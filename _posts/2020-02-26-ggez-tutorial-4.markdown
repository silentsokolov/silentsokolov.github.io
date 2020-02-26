---
title:  "Разработка игры на ggez, часть 4: Коллизия"
meta_description: "Четвертая часть руководства по разработке игры на rust с помощью движка ggez"
date:   2020-02-26 10:20:33
---

В этой части руководства мы создадим мяч для нашей игры, и заставим его двигаться, а главное отскакивать от стен и наших ракеток.

Для работы с такими понятиями как коллизия, нужна математика. `ggez` не реализует собственные математические функции, вместо этого он использует популярную библиотеку `nalgebra`. Она предоставляет огромное количество мат. функций и удобные обертки к ним. Но не переживайте, сложного в нашей игре ничего не будет.

## Создание мяча
Так как наш мяч будет обладать некоторыми свойствами - создадим под него отдельную структуру, в которую поместим следующее: текущее место положение (в виде координат), радиус, а так же значение скорости. Скорость будет отдельно для X и для Y, объяснение этому будет позже.

Для хранения текущего положения мяча, мы будем использовать `nalgebra` и специальный тип - [nalgebra::geometry::Point2](https://docs.rs/nalgebra/0.19.0/nalgebra/geometry/type.Point2.html). Данный тип обладает множеством полезных методов и используется повсеместно в `ggez`, он идеально подходит для хранения координат.

```rust
const LINE_WIDTH: f32 = 2.0;
const BALL_RADIUS: f32 = 10.0;
const BALL_VELOCITY_X: f32 = 2.0;
const BALL_VELOCITY_Y: f32 = 2.0;

struct Ball {
    point: na::Point2<f32>,
    radius: f32,
    velocity: [f32; 2],
}

struct GameState {
    left: Rect,
    right: Rect,
    ball: Ball,
    input: InputState,
}

impl GameState {
    fn new() -> Self {
          // часть функции опущено
        let ball = Ball {
            point: na::Point2::new(ARENA_WIDTH * 0.5, ARENA_HEIGHT * 0.5),
            radius: BALL_RADIUS,
            velocity: [BALL_VELOCITY_X, BALL_VELOCITY_Y],
        };
    }
}
```

Теперь с помощью уже известной нам функции `draw()` можно нарисовать мяч на экране, для этого воспользуемся функцией [new_circle](https://docs.rs/ggez/0.5.1/ggez/graphics/struct.Mesh.html#method.new_circle):

```rust
fn draw(&mut self, ctx: &mut Context) -> GameResult<()> {
    // часть функции опущено
    let ball = graphics::Mesh::new_circle(
        ctx,
        graphics::DrawMode::fill(),
        self.ball.point,
        self.ball.radius,
        0.1,
        graphics::WHITE,
    )?;

    graphics::draw(ctx, &ball, graphics::DrawParam::default())?;
}
```

У любопытных мог возникнуть вопрос, что такое [tolerance](https://docs.rs/lyon_geom/0.9.0/lyon_geom/#flattening)? По ссылке вы найдете подробное объяснение, а для нетерпеливых скажу лишь, что чем меньше это число, тем более округлей выглядит наш мяч.

## Движение мяча и коллизия
В начале заставим мяч двигаться, что этого дополним нам метод `update()`:

```rust
// Move the ball
self.ball.point.x += self.ball.velocity[0];
self.ball.point.y += self.ball.velocity[1];
```

Отмечу что мы используем постоянную скорость, то есть она будет привязана к кол-ву кадров в секунду. Для нашей простой игры это подходящий вариант, но для более масштабных проектов вы должны использовать [delta timing](https://en.wikipedia.org/wiki/Delta_timing).

Теперь займемся коллизией, для начала заставим наш мяч отскакивать от нижней и верхней границы игрового поля. Тут нам пригодится наша скорость по X и Y, которую мы задали ранее. Все что требуется для симуляции отскакивания, то есть изменения вектора движения, так это заменить знак скорости на противоположный. Для X - при приближении к боковым границам, а для Y - к нижней и верхней. Чтобы мяч не "втыкался" в стену мы так же учитываем его радиус. Дополним наш `update()` метод:

```rust
if (self.ball.point.y - self.ball.radius < 0.0)
                || (self.ball.point.y >= ARENA_HEIGHT - self.ball.radius)
{
    self.ball.velocity[1] = -self.ball.velocity[1];
}
```

Чтобы мяч отталкивался от ракеток, нужно действовать по другому, так как ракетки не только движутся, но и имеют объем. Для этого мы воспользуемся формулой для проверки нахождения точки в прямоугольнике. Точной будет центр мяча, а прямоугольником - ракетка. Выделим этот код в отдельную функцию, не забываем про радиус мяча:

```rust
fn ball_in_rect(ball: &Ball, rect: &Rect) -> bool {
    let x1 = rect.x - ball.radius;
    let y1 = rect.y - ball.radius;
    let x2 = rect.x + rect.w + ball.radius;
    let y2 = rect.y + rect.h + ball.radius;

    ball.point.x >= x1 && ball.point.x <= x2 && ball.point.y >= y1 && ball.point.y <= y2
}
```

В `update()` проверяем коллизию, и если она произошла, то как и в случае с границами, просто меняем скорость на противоположную, но теперь по X:

```rust
if ball_in_rect(&self.ball, &self.right) || ball_in_rect(&self.ball, &self.left) {
    self.ball.velocity[0] = -self.ball.velocity[0];
}
```

Осталось последнее. Сейчас если игрок не попадет по мячу, он продолжит двигаться за краями игрового поля, что бы этого не произошло, нужно возвращать его в центр, когда он пересекает боковую границу:

```rust
if (self.ball.point.x + self.ball.radius * 2.0 < 0.0)
                || (self.ball.point.x >= ARENA_WIDTH + self.ball.radius * 2.0)
{
    self.ball.point.x = ARENA_WIDTH * 0.5;
    self.ball.point.y = ARENA_HEIGHT * 0.5;
}
```

Готово! Теперь мы можем скомпилировать и запустить наш код с помощью `cargo run` и проверить, мяч движется и отскакивает от стен и ракеток. В финальном коде мы добавили линию, разделяющую поле пополам.

Полный пример:

```rust
use ggez::event::{self, EventHandler, KeyCode, KeyMods};
use ggez::graphics::{self, Rect};
use ggez::timer;
use ggez::{conf, Context, ContextBuilder, GameResult};

use ggez::nalgebra as na;

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
            if (self.ball.point.x + self.ball.radius * 2.0 < 0.0)
                || (self.ball.point.x >= ARENA_WIDTH + self.ball.radius * 2.0)
            {
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

        Self {
            left,
            right,
            ball,
            line,
            input,
        }
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