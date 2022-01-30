---
title:  "Spark и Rust: Дополнение"
meta_description: "Запуск Rust на apache spark"
date:   2022-01-30 08:03:21
---

Некоторые время назад мы уже катались темы [использования языка Rust в Apache Spark](https://silentsokolov.github.io/apache-spark-udfs-in-rust) приложениях при помощи UDF. Но если вам не нужна вся мощь типов - то есть более простой подход запустить Rust код. Этим методом является - [pipe()](https://spark.apache.org/docs/latest/api/python/reference/api/pyspark.RDD.pipe.html). Ок, я немного приврал в заголовке, так как этот подход можно применять с любой программой.

По сути `pipe()`  передает содержимое RDD в `stdin` переданной программе и читает ее `stdout` . В [документации](https://spark.apache.org/docs/latest/api/python/reference/api/pyspark.RDD.pipe.html) присутствует исчерпывающий пример:

```python
sc.parallelize(['1', '2', '', '3']).pipe('cat').collect()
# ['1', '2', '', '3']
```

Но ничего не запрещает вместо `cat` использовать собственную программу. Ниже небольшой код на Rust, который берет строку и преобразует ее в строку JSON.

```rust
use serde::{Deserialize, Serialize};
use std::io::{self, BufRead};

#[derive(Serialize, Deserialize)]
struct Block {
    bar: String,
}

fn main() {
    let stdin = io::stdin();
    for line in stdin.lock().lines() {
        let block = Block { bar: line.unwrap() };

        println!("{}", serde_json::to_string(&block).unwrap());
    }
}
```

Не стоит забывать, что при работе в кластерном режиме, каждая из нод кластера должна иметь копию исполняемой программы. Тот же `cat` есть почти в каждый unix-системе, но нашу программу нужно еще доставить. Это можно сделать в ручную, или "запеч" в образ при создании ноды, или использовать `--files` / `addFile()`.

```python
# spark-submit --files /path/to/my-rust-bin ...

df = sc.parallelize(['1', '2', '', '3']).pipe('my-rust-bin').toDf()

df.show()
```

Безусловно этот подход проще, чем написание UDF, но он также имеет ряд недостатков: во-первых, нужно помнить о компиляции кода под разные платформы, во-вторых если выработает с DataFrame, то использование RDD - снизит производительность. Но так или иначе это еще один вариант оптимизации Spark приложений.