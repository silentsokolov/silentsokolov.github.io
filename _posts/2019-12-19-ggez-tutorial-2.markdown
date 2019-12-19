---
title:  "Разработка игры на ggez, часть 1: Работа с графикой"
meta_description: "Вторая часть руководства по разработке игры на rust с помощью движка ggez"
date:   2019-12-19 14:36:16
---

Теперь, когда мы создали окно и разобрались с основными понятиями в `ggez`, можно приступить к созданию самой игры. Предлагаю, начать с простого - «**Пинг**-**Понг**».

Определимся с элементами, который игрок видит на экране: своя и ракетка соперника, мяч и счет игры. Начнем с ракеток.

Для работы с графикой в `ggez` есть модуль `graphics`, который предоставляет инструменты для рисования геометрических фигур, работы с изображениями и текстом, а также управляет координатной сеткой и выводом на экран.

Наверно, ключевым элементом можно назвать трейт [Drawable](https://docs.rs/ggez/0.5.1/ggez/graphics/trait.Drawable.html), любой объект, который вы хотите вывести на экран должен реализовывать его.

## Создание ракеток
Мы могли бы реализовать свою структуру, которая представляла бы ракетки. Но зачем усложнять? `graphics` предоставляет структуру [Rect](https://docs.rs/ggez/0.5.1/ggez/graphics/struct.Rect.html) для создания простых 2D прямоугольников. Нам достаточно задать константы ширины и высоты, для упрощения работы с кодом.

```rust
const PADDLE_WIDTH: f32 = 10.0;
const PADDLE_HEIGHT: f32 = 80.0;
```

Игроков в пинг-понге для и ракеток должно быть две - Left и Right. Чтобы в будущем управлять их положением, поместим их в `GameState`:

```rust
use ggez::graphics::{self, Rect};

struct GameState {
    left: Rect,
    right: Rect,
}
```

Модифицируем `GameState`добавив метод `new`. Нам известны размеры нашего игрового поля (500x500), поэтому рассчитать начальное положение ракеток не составит труда. Система координат по умолчанию имеет начало в верхнем левом углу экрана, при этом Y увеличивается вниз.

```rust
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

        Self { left, right }
    }
}
```

## Рисование
Как уже говорилось в предыдущей части нашего руководства, для рисования объектов на экране есть метод `draw()`, давайте воспользуемся им и нарисуем наши ракетки.

```rust
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
```

Разберем этот код построчно.

```rust
graphics::clear(ctx, graphics::BLACK);
```

Метод `draw()` вызывается каждый раз когда отрисовывается кадр, поэтому перед началом рисования, стоит очистить экран, что и делает [clear](https://docs.rs/ggez/0.5.1/ggez/graphics/fn.clear.html), здесь же можно задать цвет фона.

```rust
let left = graphics::Mesh::new_rectangle(
    ctx,
    graphics::DrawMode::fill(),
    self.left,
    graphics::WHITE,
)?;
```

[graphics::Mesh](https://docs.rs/ggez/0.5.1/ggez/graphics/struct.Mesh.html) предоставлять методы для создания полигональной сетки. В коде выше, мы создаем прямоугольник. Через [DrawMode](https://docs.rs/ggez/0.5.1/ggez/graphics/enum.DrawMode.html#method.fill) указываем, что он должен быть "залит", а не отображался в виде контура. Передаем наш прямо угольник (координаты), и указываем что цвет должен быть белый.

```rust
graphics::draw(ctx, &left, graphics::DrawParam::default())?;
```

Рисуем нашу ракетку. В [DrawParam](https://docs.rs/ggez/0.5.1/ggez/graphics/struct.DrawParam.html) мы можем указать различные трансформации над объектом,  к примеру измениться его положение в пространстве (наклонить) или скалировать его величину.

```rust
graphics::present(ctx)
```

[present](https://docs.rs/ggez/0.5.1/ggez/graphics/fn.present.html) говорит графической системе вывести все на экран.

Готово! Теперь мы можем скомпилировать и запустить наш код с помощью `cargo run` и получить уже не пустое окно, а окно с двумя нашими ракетками.

Полный пример:

```rust
use ggez::event::{self, EventHandler};
use ggez::graphics::{self, Rect};
use ggez::{conf, Context, ContextBuilder, GameResult};

const ARENA_WIDTH: f32 = 500.0;
const ARENA_HEIGHT: f32 = 500.0;

const PADDLE_WIDTH: f32 = 10.0;
const PADDLE_HEIGHT: f32 = 80.0;

struct GameState {
    left: Rect,
    right: Rect,
}

impl EventHandler for GameState {
    fn update(&mut self, _ctx: &mut Context) -> GameResult<()> {
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

        Self { left, right }
    }
}

fn main() {
    let mut cb = ContextBuilder::new("game02", "author")
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