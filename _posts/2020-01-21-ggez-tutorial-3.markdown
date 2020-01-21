---
title:  "Разработка игры на ggez, часть 3: Обработка ввода"
meta_description: "Третья часть руководства по разработке игры на rust с помощью движка ggez"
date:   2020-01-21 11:40:16
---

В последней части нашего руководства мы нарисовали пару ракеток для игры в «**Пинг**-**Понг**». Думаю, игра закончиться быстро, если они будут стоять на месте! Давайте, заставим наши ракетки двигаться!

Когда мы ранее говорили о [EventHandler](https://docs.rs/ggez/0.5.1/ggez/event/trait.EventHandler.html), мы затрагивали только обязательные методы, но эта черта также обладает другими, определенными по-умолчанию, в частности для обработки ввода: нажатие клавиш, движение мыши и т. д.

Так как для управления ракетками мы будем использовать клавиатуру, нам интересны методы [key_down_event](https://docs.rs/ggez/0.5.1/ggez/event/trait.EventHandler.html#method.key_down_event) и [key_up_event](https://docs.rs/ggez/0.5.1/ggez/event/trait.EventHandler.html#method.key_up_event), обрабатывающие нажатие и отпуск клавиш, соответственно. Отмечу, что `key_down_event` имеет реализацию по-умолчанию, которая завершает работу программы по нажатию `esc`, не стоит об этом забывать при переопределении метода.

В качестве аргументов эти методы принимают: [KeyCode](https://docs.rs/ggez/0.5.1/ggez/input/keyboard/enum.KeyCode.html) - перечисление всех клавиш; [KeyMods](https://docs.rs/ggez/0.5.1/ggez/input/keyboard/struct.KeyMods.html) - различные модификаторы, такие как зажатый `shift` или `ctrl` и флаг, определяющий повторное нажатие. Все это позволяет перехватить все возможные комбинации клавиш.

## Захват ввода
В начале, создадим структуру, которая будет хранить состояние ввода. Мы могли бы хранить данные прямо в нашем глобальном состоянии, но лучше добавить новую абстракцию, чтобы сохранить читаемость кода.

```rust
struct InputState {
    right_yaxis: f32,
    left_yaxis: f32,
}

struct GameState {
    left: Rect,
    right: Rect,
    input: InputState,
}
```

Реализуем наши методы:

```rust
impl EventHandler for GameState {
    fn key_down_event(
        &mut self,
        ctx: &mut Context,
        keycode: KeyCode,
        _keymod: KeyMods,
        _repeat: bool,
    ) {
        match keycode {
            KeyCode::W => {
                self.input.left_yaxis = -1.0;
            }
            KeyCode::S => {
                self.input.left_yaxis = 1.0;
            }
            KeyCode::O => {
                self.input.right_yaxis = -1.0;
            }
            KeyCode::L => {
                self.input.right_yaxis = 1.0;
            }
            KeyCode::Escape => event::quit(ctx),
            _ => (),
        }
    }

    fn key_up_event(&mut self, _ctx: &mut Context, keycode: KeyCode, _keymod: KeyMods) {
        match keycode {
            KeyCode::W | KeyCode::S => {
                self.input.left_yaxis = 0.0;
            }
            KeyCode::O | KeyCode::L => {
                self.input.right_yaxis = 0.0;
            }
            _ => (),
        }
    }
```

Хотя кода много, он довольно прост. Мы указываем отклонение по оси Y, для левой ракетки при нажатии `W`или  `S`, а для правой `O` или `L`. А при отпуске тех же клавиш, сбрасываем отклонение к нулю.

Почему мы используем отклонение? Все просто, это делает наш код более переносимым, к примеру, если мы захотим управлять не только по средствам клавиатуры, но и с джойстика, где ввод представлен отклонением стиков.

## Движение объектов
Прежде чем двигать наши ракетки, хочется упомянуть `FPS` (_Frames per Second_). В играх мы стремимся отрисовать как можно больше кадров, для более плавной картинки. Но иногда, в таких, простых играх как наша, стоит ограничить вычисления. Эти слишком обширная тема и мы не будет затрагивать ее в данной статье. Для ограничения вычислений, мы воспользуемся [check_update_time](https://docs.rs/ggez/0.5.1/ggez/timer/fn.check_update_time.html), которая возьмет всю работу на себя.

```rust
const DESIRED_FPS: u32 = 60;
const MOVE_SPEED: f32 = 1.9;

impl EventHandler for GameState {
    fn update(&mut self, ctx: &mut Context) -> GameResult<()> {
        while timer::check_update_time(ctx, DESIRED_FPS) {
            self.left.y = (self.left.y + self.input.left_yaxis * MOVE_SPEED)
                .min(ARENA_HEIGHT - PADDLE_HEIGHT)
                .max(0.0);

            self.right.y = (self.right.y + self.input.right_yaxis * MOVE_SPEED)
                .min(ARENA_HEIGHT - PADDLE_HEIGHT)
                .max(0.0);
        }
        Ok(())
    }
}
```

Здесь, мы задаем новое значение координат по оси Y, для каждой из кареток. Чтобы было удобнее управлять скоростью мы ввели новую переменную `MOVE_SPEED`. А чтобы наши каретки "не уезжали" за границы игрового поля, мы задали границы с помощью минимального и максимального значения.

Готово! Теперь мы можем скомпилировать и запустить наш код с помощью `cargo run` и проверить, что первый игрок будет управлять ракеткой с одной стороны клавиатуры, а второй - с другой.

Полный пример:

```rust
use ggez::event::{self, EventHandler, KeyCode, KeyMods};
use ggez::graphics::{self, Rect};
use ggez::timer;
use ggez::{conf, Context, ContextBuilder, GameResult};

const ARENA_WIDTH: f32 = 500.0;
const ARENA_HEIGHT: f32 = 500.0;

const PADDLE_WIDTH: f32 = 10.0;
const PADDLE_HEIGHT: f32 = 80.0;

const DESIRED_FPS: u32 = 60;
const MOVE_SPEED: f32 = 1.9;

struct InputState {
    right_yaxis: f32,
    left_yaxis: f32,
}

struct GameState {
    left: Rect,
    right: Rect,
    input: InputState,
}

impl EventHandler for GameState {
    fn update(&mut self, ctx: &mut Context) -> GameResult<()> {
        while timer::check_update_time(ctx, DESIRED_FPS) {
            self.left.y = (self.left.y + self.input.left_yaxis * MOVE_SPEED)
                .min(ARENA_HEIGHT - PADDLE_HEIGHT)
                .max(0.0);

            self.right.y = (self.right.y + self.input.right_yaxis * MOVE_SPEED)
                .min(ARENA_HEIGHT - PADDLE_HEIGHT)
                .max(0.0);
        }
        Ok(())
    }

    fn draw(&mut self, ctx: &mut Context) -> GameResult<()> {
        graphics::clear(ctx, graphics::BLACK);

        let left = graphics::Mesh::new_rectangle(
            ctx,
            graphics::DrawMode::fill(),
            self.left,
            graphics::WHITE,
        )?;

        let right = graphics::Mesh::new_rectangle(
            ctx,
            graphics::DrawMode::fill(),
            self.right,
            graphics::WHITE,
        )?;

        graphics::draw(ctx, &left, graphics::DrawParam::default())?;
        graphics::draw(ctx, &right, graphics::DrawParam::default())?;

        graphics::present(ctx)?;
        Ok(())
    }

    fn key_down_event(
        &mut self,
        ctx: &mut Context,
        keycode: KeyCode,
        _keymod: KeyMods,
        _repeat: bool,
    ) {
        match keycode {
            KeyCode::W => {
                self.input.left_yaxis = -1.0;
            }
            KeyCode::S => {
                self.input.left_yaxis = 1.0;
            }
            KeyCode::O => {
                self.input.right_yaxis = -1.0;
            }
            KeyCode::L => {
                self.input.right_yaxis = 1.0;
            }
            KeyCode::Escape => event::quit(ctx),
            _ => (),
        }
    }

    fn key_up_event(&mut self, _ctx: &mut Context, keycode: KeyCode, _keymod: KeyMods) {
        match keycode {
            KeyCode::W | KeyCode::S => {
                self.input.left_yaxis = 0.0;
            }
            KeyCode::O | KeyCode::L => {
                self.input.right_yaxis = 0.0;
            }
            _ => (),
        }
    }
}

impl GameState {
    fn new() -> Self {
        let left = Rect::new(
            0.0,                                      // x
            ARENA_HEIGHT * 0.5 - PADDLE_HEIGHT * 0.5, // y
            PADDLE_WIDTH,
            PADDLE_HEIGHT,
        );
        let right = Rect::new(
            ARENA_WIDTH - PADDLE_WIDTH,               // x
            ARENA_HEIGHT * 0.5 - PADDLE_HEIGHT * 0.5, // y
            PADDLE_WIDTH,
            PADDLE_HEIGHT,
        );

        let input = InputState {
            right_yaxis: 0.0,
            left_yaxis: 0.0,
        };

        Self { left, right, input }
    }
}

fn main() {
    let mut cb = ContextBuilder::new("game01", "author")
        .window_setup(conf::WindowSetup::default().title("My game!"))
        .window_mode(conf::WindowMode::default().dimensions(ARENA_WIDTH, ARENA_HEIGHT));

    let (ctx, event_loop) = &mut cb.build().unwrap();

    let mut state = GameState::new();

    match event::run(ctx, event_loop, &mut state) {
        Ok(_) => println!("Exited cleanly."),
        Err(e) => println!("Error occured: {}", e),
    }
}
```