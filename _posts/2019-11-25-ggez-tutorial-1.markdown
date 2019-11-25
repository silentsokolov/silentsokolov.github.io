---
title:  "Разработка игры на ggez, часть 1: Настройка и запуск"
meta_description: "Первая часть руководства по разработке игры на rust с помощью движка ggez"
date:   2019-11-25 14:16:16
---

`ggez`- это легковесный фреймворк для создания 2D игр. Он направлен на реализацию API игрового движка [LÖVE](https://love2d.org/) и был им вдохновлен. `ggez` поддерживает кроссплатформенную 2D графику, звук и обработку событий.

В этом небольшом руководстве мы рассмотрим создание просто игры с помощью этого фреймворка.

### Создание проекта
Надеюсь, к этому моменту у вас уже установлен компилятор Rust (на текущий момент актуальная версия `1.40.0`) и `cargo`. 

Для пользователей Linux потребуется установить дополнительные библиотеки, к примеру, для Debian это будут `libasound2-dev libudev-dev pkg-config`, подробнее о системных зависимостях можно узнать [тут](https://github.com/ggez/ggez/blob/master/docs/BuildingForEveryPlatform.md).

Создадим новый проект:

```bash
cargo new --bin game01
```

Перейдем в созданный каталог и обновим `Cargo.toml`, а именно добавим в зависимости `ggez` (актуальная версия `0.5.1`):

```toml
[package]
name = "game01"
version = "0.1.0"
authors = []
edition = "2018"

[dependencies]
ggez = "0.5.1"
```

Готово! Мы можем начать работать над своей игрой.

### Создание глобального состояния (game state)
Создадим структуру, которая станет базовым элементом нашей игры. В большинстве случаем она будет отвечать за глобальное состояние нашей игры. К примеру в нем можно будет хранить счет игры или положение игрока на экране, пока обойдемся пустой структурой:

```rust
struct GameState;
```

Данная структура должна реализовывать черту[EventHandler](https://docs.rs/ggez/0.5.1/ggez/event/trait.EventHandler.html). Это основной интерфейс взаимодействия с циклом событий `ggez`. Черта определяет несколько методов, но обязательными являются два: [update()](https://docs.rs/ggez/0.5.1/ggez/event/trait.EventHandler.html#tymethod.update) и [draw()](https://docs.rs/ggez/0.5.1/ggez/event/trait.EventHandler.html#tymethod.draw). Первый вызывается при каждом обновлении игры и обычно сдержит игровую логику. Второй отвечает за отрисовку кадра. Каждый из этих методов принимает один аргумент - [Context](https://docs.rs/ggez/0.5.1/ggez/struct.Context.html). Context - это структура, которая отвечает за взаимодействие с различными ресурсами: экран, аудиосистема, таймеры и так далее. Context может быть только один, попытка создать второй приведет к панике. 

Давайте создадим эти методы:

```rust
impl EventHandler for GameState {
    fn update(&mut self, _ctx: &mut Context) -> GameResult<()> {
        Ok(())
    }

    fn draw(&mut self, _ctx: &mut Context) -> GameResult<()> {
        Ok(())
    }
}
```

### Создание цикла событий
Реализуем функцию `main`, которая будет отвечать за запуск нашей игры.  Для начала нам потребуется сам цикл событий, который и будет вызывать наши методы. Также нам потребуется Context. Для решения обеих этих задач служит [ContextBuilder](https://docs.rs/ggez/0.5.1/ggez/struct.ContextBuilder.html). Он принимает различные параметры, с помощью которых описывается окно игры. Давайте дадим имя нашей игре, и установим минимальный размер окна (500px на 500px):

```rust
let mut cb = ContextBuilder::new("game01", "author")
    .window_setup(
        conf::WindowSetup::default()
            .title("My game!")
            .samples(conf::NumSamples::Four),
    )
    .window_mode(
        conf::WindowMode::default()
            .dimensions(500.0, 500.0)
            .min_dimensions(500.0, 500.0),
    );
```

Теперь можно создать Context и цикл событий:

```rust
let (ctx, event_loop) = &mut cb.build().unwrap();
```

Конечно, создаем наш `GameState`: 

```rust
let mut state = GameState {};
```

Осталось дело за малым, запустить цикл, для этого нужно вызвать  [event::run()](https://docs.rs/ggez/0.5.1/ggez/event/fn.run.html) и передать созданные элементы параметры:

```rust
match event::run(ctx, event_loop, &mut state) {
    Ok(_) => println!("Exited cleanly."),
    Err(e) => println!("Error occured: {}", e),
}
```

Готово! Теперь мы можем скомпилировать и запустить наш код с помощью `cargo run` и получить окно нашей будущей игры.