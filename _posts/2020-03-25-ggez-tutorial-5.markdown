---
title:  "Разработка игры на ggez, часть 5: Игровой счет"
meta_description: "Пятая часть руководства по разработке игры на rust с помощью движка ggez"
date:   2020-03-26 10:04:23
---

В этой, заключительной, части руководства мы выведем на экран игровой счет, и чтобы это сделать мы познакомимся с тем как в `ggez` происходит работа с ассетами, а именно со шрифтом.

## Подготовка окружения
Перед тем как работать с ассетами мы должны указать в какой директории их следует искать. Кстати, для ассетов в `ggez` используется термин "ресурсы". У [ContextBuilder](https://docs.rs/ggez/0.5.1/ggez/struct.ContextBuilder.html) существует метод [add_resource_path](https://docs.rs/ggez/0.5.1/ggez/struct.ContextBuilder.html#method.add_resource_path), который принимает путь до директории с файлами. Мы можем указать абсолютный путь, но, чтобы сохранить гибкость кода лучше использовать относительный, но это не так просто. Из-за того, что при запуске через `cargo run`, используется код из директории `target`, путь до файлов будет относительно этой директории. Чтобы этого избежать воспользуемся пакетом `env`  и его методом возвращающим директорию текущего проекта.

```rust
let mut current_dir = env::current_dir().unwrap();
current_dir.push("resources");

let mut cb = ContextBuilder::new("game01", "author")
    .add_resource_path(current_dir)
    .window_setup(conf::WindowSetup::default().title("My game!"))
    .window_mode(conf::WindowMode::default().dimensions(ARENA_WIDTH, ARENA_HEIGHT));
```

## Шрифт и счетчики
Чтобы многократно не читать файл со шрифтом загрузим его один раз и сохраним в `GameState`, там же сохраним текущий счет для обоих игроков.

Здесь тоже придется обновить уже существующий код. Так как при загрузке любого файла, будь то шрифт или спрайт или текстура нам потребуется контекст игры.  Передадим его в конструктор `GameState`. `ggez` не предоставляет универсальных дескрипторов для загрузки файлов, для каждого формата есть свой - у шрифтов это [ggez::graphics::Font](https://docs.rs/ggez/0.5.1/ggez/graphics/struct.Font.html). Обратите внимание, что путь к файлу указывается как абсолютный путь относительно директории указаной в методе `add_resource_path`.

```rust
struct GameState {
    left: Rect,
    right: Rect,
    ball: Ball,
    line: [na::Point2<f32>; 2],
    input: InputState,
    font: graphics::Font,
    left_score: i32,
    right_score: i32,
}

impl GameState {
    fn new(ctx: &mut Context) -> Self {
        // часть функции опущено
        let font = graphics::Font::new(ctx, "/fonts/monaco.ttf").unwrap();

        Self {
            left,
            right,
            ball,
            line,
            input,
            font,
            left_score: 0,
            right_score: 0,
        }
    }
}
```

У нас уже есть код отвечающий за сброс положение мяча, при промахе. И поэтому для достаточно обновлять счетчик, когда игрок пропускает мяч. Правый счетчик - когда левый игрок пропускает мяч и на оборот.

```rust
fn update(&mut self, ctx: &mut Context) -> GameResult<()> {
      while timer::check_update_time(ctx, DESIRED_FPS) {
          // часть функции опущено
        let reset = if self.ball.point.x + self.ball.radius * 2.0 < 0.0 {
            self.left_score += 1;
            true
        } else if self.ball.point.x >= ARENA_WIDTH + self.ball.radius * 2.0 {
            self.right_score += 1;
            true
        } else {
            false
        };

        if reset {
            self.ball.point.x = ARENA_WIDTH * 0.5;
            self.ball.point.y = ARENA_HEIGHT * 0.5;
        }
    }
    Ok(())
}
```

## Вывод счета на экран
Как говорилось во второй части этого руководства, чтобы что-то вывести на экран, оно должно реализовывать трайт [Drawable](https://docs.rs/ggez/0.5.1/ggez/graphics/trait.Drawable.html), для текста такой объект уже есть - [ggez::graphics::Text](https://docs.rs/ggez/0.5.1/ggez/graphics/struct.Text.html). Но если мы просто передадим ему текст в конструктор, он будет использовать шрифт по-умолчанию (системный), мы же хотим использовать нужный нам. Тут стоит упомянуть, что текст, довольно сложная вещь для вывода на экран. Поэтому часто он состоит из отдельных элементов, которые вначале собираются в кэше, а лишь за чем переводятся в структуры для вывода на экран. В `ggez` этими элементами являются [ggez::graphics::TextFragment](https://docs.rs/ggez/0.5.1/ggez/graphics/struct.TextFragment.html) и в них уже можно указать и шрифт и цвет и даже размерность.

```rust
fn draw(&mut self, ctx: &mut Context) -> GameResult<()> {
    // часть функции опущено
    let right_count = graphics::Text::new(
        graphics::TextFragment::new(format!("P1: {}", self.right_score)).font(self.font),
    );

    let left_count = graphics::Text::new(
        graphics::TextFragment::new(format!("P2: {}", self.left_score)).font(self.font),
    );

    graphics::draw(
        ctx,
        &right_count,
        (na::Point2::new(ARENA_WIDTH * 0.5 - 100.0, 0.0),),
    )?;
    graphics::draw(
        ctx,
        &left_count,
        (na::Point2::new(ARENA_WIDTH * 0.5 + 60.0, 0.0),),
    )?;

    graphics::present(ctx)?;
    Ok(())
}
```

Готово! `cargo run` соберет и запустит наше уже готовую игру. 

На этом данное руководство заканчивается, его можно назвать лишь "вводным", так как `ggez` предоставляет еще массу возможностей для создания полноценных 2D игр, даже в созданным "пинг-понг" есть масса путей для улучшений, к примеру, создать приветственное меню или экран с результатами. Все в ваших руках!

Полный пример:

```rust
use std::env;

use ggez::event::{self, EventHandler, KeyCode, KeyMods};
use ggez::graphics::{self, Rect};
use ggez::nalgebra as na;
use ggez::timer;
use ggez::{conf, Context, ContextBuilder, GameResult};

const ARENA_WIDTH: f32 = 500.0;
const ARENA_HEIGHT: f32 = 500.0;

const PADDLE_WIDTH: f32 = 10.0;
const PADDLE_HEIGHT: f32 = 80.0;

const DESIRED_FPS: u32 = 60;

const MOVE_SPEED: f32 = 1.9;
const LINE_WIDTH: f32 = 2.0;
const BALL_RADIUS: f32 = 10.0;
const BALL_VELOCITY_X: f32 = 2.0;
const BALL_VELOCITY_Y: f32 = 2.0;

struct InputState {
    right_yaxis: f32,
    left_yaxis: f32,
}

struct Ball {
    point: na::Point2<f32>,
    radius: f32,
    velocity: [f32; 2],
}

struct GameState {
    left: Rect,
    right: Rect,
    ball: Ball,
    line: [na::Point2<f32>; 2],
    input: InputState,
    font: graphics::Font,
    left_score: i32,
    right_score: i32,
}

fn ball_in_rect(ball: &Ball, rect: &Rect) -> bool {
    let x1 = rect.x - ball.radius;
    let y1 = rect.y - ball.radius;
    let x2 = rect.x + rect.w + ball.radius;
    let y2 = rect.y + rect.h + ball.radius;

    ball.point.x >= x1 && ball.point.x <= x2 && ball.point.y >= y1 && ball.point.y <= y2
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

        // Move the ball
        self.ball.point.x += self.ball.velocity[0];
        self.ball.point.y += self.ball.velocity[1];

        // Bounce at the top or the bottom of the arena
        if (self.ball.point.y - self.ball.radius < 0.0)
            || (self.ball.point.y >= ARENA_HEIGHT - self.ball.radius)
        {
            self.ball.velocity[1] = -self.ball.velocity[1];
        }

        // Bounce at the paddles
        if ball_in_rect(&self.ball, &self.right) || ball_in_rect(&self.ball, &self.left) {
            self.ball.velocity[0] = -self.ball.velocity[0];
        }

        // Reset
        let reset = if self.ball.point.x + self.ball.radius * 2.0 < 0.0 {
            self.left_score += 1;
            true
        } else if self.ball.point.x >= ARENA_WIDTH + self.ball.radius * 2.0 {
            self.right_score += 1;
            true
        } else {
            false
        };

        if reset {
            self.ball.point.x = ARENA_WIDTH * 0.5;
            self.ball.point.y = ARENA_HEIGHT * 0.5;
        }
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

    let ball = graphics::Mesh::new_circle(
        ctx,
        graphics::DrawMode::fill(),
        self.ball.point,
        self.ball.radius,
        0.1,
        graphics::WHITE,
    )?;

    let line = graphics::Mesh::new_line(ctx, &self.line, LINE_WIDTH, graphics::WHITE)?;

    graphics::draw(ctx, &left, graphics::DrawParam::default())?;
    graphics::draw(ctx, &right, graphics::DrawParam::default())?;
    graphics::draw(ctx, &ball, graphics::DrawParam::default())?;
    graphics::draw(ctx, &line, graphics::DrawParam::default())?;

    let right_count = graphics::Text::new(
        graphics::TextFragment::new(format!("P1: {}", self.right_score)).font(self.font),
    );

    let left_count = graphics::Text::new(
        graphics::TextFragment::new(format!("P2: {}", self.left_score)).font(self.font),
    );

    graphics::draw(
        ctx,
        &right_count,
        (na::Point2::new(ARENA_WIDTH * 0.5 - 100.0, 0.0),),
    )?;
    graphics::draw(
        ctx,
        &left_count,
        (na::Point2::new(ARENA_WIDTH * 0.5 + 60.0, 0.0),),
    )?;

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
    fn new(ctx: &mut Context) -> Self {
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

        let ball = Ball {
            point: na::Point2::new(ARENA_WIDTH * 0.5, ARENA_HEIGHT * 0.5),
            radius: BALL_RADIUS,
            velocity: [BALL_VELOCITY_X, BALL_VELOCITY_Y],
        };

        let line = [
            na::Point2::new(ARENA_WIDTH * 0.5, 0.0),
            na::Point2::new(ARENA_WIDTH * 0.5, ARENA_HEIGHT),
        ];

        let input = InputState {
            right_yaxis: 0.0,
            left_yaxis: 0.0,
        };

        let font = graphics::Font::new(ctx, "/fonts/monaco.ttf").unwrap();

        Self {
            left,
            right,
            ball,
            line,
            input,
            font,
            left_score: 0,
            right_score: 0,
        }
    }
}

fn main() {
let mut current_dir = env::current_dir().unwrap();
current_dir.push("resources");

let mut cb = ContextBuilder::new("game01", "author")
    .add_resource_path(current_dir)
    .window_setup(conf::WindowSetup::default().title("My game!"))
    .window_mode(conf::WindowMode::default().dimensions(ARENA_WIDTH, ARENA_HEIGHT));

    let (ctx, event_loop) = &mut cb.build().unwrap();

    let mut state = GameState::new(ctx);

    match event::run(ctx, event_loop, &mut state) {
        Ok(_) => println!("Exited cleanly."),
        Err(e) => println!("Error occured: {}", e),
    }
}
```