---
title:  "Spark UDF на Rust"
meta_description: "Разбор подхода написания udf для apache spark на языке rust"
date:   2021-06-25 08:03:21
---

Apache Spark превосходный инструмент для обработки больших данных, мы используем PySpark (интерфейс для Spark на Python), у которого есть ряд ограничений по сравнению с версией на Scala, но это компенсируется гибкостью, скорость прототипирования и богатой экосистемой языка Python. Одна из главных головных болей в PySpark это UDF, они проигрывают в сравнении со Scala в скорости работы и иногда потребляют больше памяти. Если с первым можно смириться, то со втором приходится повозиться, так как ресурсы кластера небезграничны.

Чтобы бороться с этими проблемами применяют разные подходы такие как `pandas_udf` и вызов Scala из Python. `pandas_udf`  хорош, когда UDF простой, так как сильно ограничен в типах; второй, конечно, гибкий, быстрый и вообще, но без знаний Scala не обойтись.

Последний каплей в моем терпении стало, странное поведение метода [разбора URL в Java](https://stackoverflow.com/questions/28568188/java-net-uri-get-host-with-underscores) (соответственно и в Scala), из-за чего стало невозможно использовать его в работе, и пришлось прибегнуть к UDF. Реализация на Python почти в два раза замедлила работу джобы. В тот момент и родилась идея попробовать применить Rust. _Да, идея безумна и граничит с бредом, но когда это кого-то останавливало?_. Дальнейшее погружение в эту тему показало, что вызвать из Java методы Rust довольно просто и накладные расходы [не такие значительные](https://github.com/dyu/ffi-overhead).

План в следующем: написать класс-обертку на Java, который будет вызывать реализацию на Rust, так как Java по сути родной язык Spark, мы не будем тратить лишние ресурсы на сериализацию объектов (если использовать Python->Rust) и позволит использовать UDF как в PySpark, так и версиях для других языков.

Далее мы погрузимся в реализацию, для этого мы разберем по шагам код в [репозитории](https://github.com/silentsokolov/spark-udf-rust) с примером готовой UDF.

## Немного Java
Прежде всего, нам нужно написать обертку - UDF на Java, но место того, чтобы реализовывать логику, эта функция будет иметь специальный метод, вызов которого будет приводить к обращению к Rust. Чтобы иметь возможность работать с Java и в целом обеспечить связь двух технологий, мы будем использовать крейт [jni-rs](https://docs.rs/jni/0.19.0/jni/).

За основу UDF будет взят обобщенный (generic) класс `UDF1` из пакета `org.apache.spark`. Раз он обобщенный, мы должны указать тип входных и выходные данных, в нашем случае это `String` и `String[]` , а также заимплементить метод `call`.

```java
package com.github.silentsokolov;
import java.io.IOException;
import org.apache.spark.sql.api.java.UDF1;

public class App implements UDF1<String, String[]> {
    // метод "вызывающий" rust
    public static native String[] nativeUDF(String url);
	
    // чтобы иметь возможность обращаться к rust
    // мы должны подключить библиотеку, реализующую метод
    // иначе получим ошибку
    static {
        NativeUtils.loadLibraryFromJar("/libs/darwin-amd64.dylib");
    }

    // данный метод будет вызван при использовании UDF
    @Override
    public String[] call(String url) throws Exception {
        return App.nativeUDF(url);
    }
}
```

## Связующее звено
Следующем шагом мы должны создать файл с заголовками, где будут указаны имя и типы, которых мы должны придерживаться при реализации в нашем коде Rust. К счастью, в Java сгенерировать заголовки довольно просто: `javac -h . App`. Для удобства, в коде проекта есть `Makefile`, позволяющий упростить эту операцию: `make java_compile`, после ее вызова будет создан файл `src/main/native/include/com_github_silentsokolov_App.h` со следующим содержимом:

```c
/*
 * Class:     com_github_silentsokolov_App
 * Method:    nativeUDF
 * Signature: (Ljava/lang/String;)[Ljava/lang/String;
 */
JNIEXPORT jobjectArray JNICALL Java_com_github_silentsokolov_App_nativeUDF
  (JNIEnv *, jclass, jstring);
```

Это даем нам название функции, которое мы должны использовать, принимаемые ей аргументы и возвращаемый тип.

## Непосредственно Rust
Из заголовков мы получили нужную информацию, теперь достаточно перенести ее на Rust:

```rust
pub extern "system" fn Java_com_github_silentsokolov_App_nativeUDF(
    env: JNIEnv,
    _class: JClass,
    url: JString,
) -> jobjectArray {
}
```

Как вы, наверно, уже поняли мы не можем напрямую использовать стандартные типы Rust в определении функции. Для этого jni предоставляет обертки:  jstring, jobjectArray и другие. Благо они легко конвертируются к стандартным типами, поэтому при реализации непосредственно самой логики можно использовать их.

В коде также можете найти ряд полезных инструментов, как отлов ошибок и конвертация их к Java исключениям, запуск тестов и другие.

В заключении, мы должны скомпилировать библиотеку: `cargo build --release`. 

Опять же, можно воспользоваться `make rust_test` - для запуска тестов и `make rust_build` - для сборки.

## Готовим Jar
Чтобы сделать наш код переносимым, нужно создать jar-файл. Тут не все так просто, по-умолчанию в jar файл нельзя поместить подключаемые библиотеки. Но к счастью, это можно обойти, для этого воспользуемся `NativeUtils.loadLibraryFromJar` (thanks _Adam Heinrich_). У этого подхода есть свои минусы, о которых поговорим позже, но благодаря ему, отдельно переносить файл библиотеки между машинами не придется.

После компиляции, нужно перенести файл из `target/release/**.so` в `src/main/resources/libs/` и скомпилировать jar:  `mvn package`.

Уже догадались? Да, для удобства есть `make java_package`.

## Все в одном месте
Уже не раз упомянутый `Makefile` можно назвать краеугольным камнем в нашем коде, благодаря ему репозитории становиться не просто примером, а настоящим шаблоном для создания UDF на Rust. Все что требуется поправить типы в Java, реализовать логику в Rust и запустить `make build` - на выходе будет создан jar готовый к использованию.

## Подключаем к Spark
Осталось дело за малым - подключить нашу библиотеку к Spark и проверить ее в работе:

```python
# spark-submit --jars /path/to/udf.jar ...

spark.udf.registerJavaFunction('parseUrl', 'com.github.silentsokolov.App', t.ArrayType(t.StringType(), True))

df = spark.createDataFrame(
    [
        ('https://www.github.com/rust-lang/rust/issues?labels=E-easy&state=open#hash'),
        ('https://google.com/'),
    ],
    ['url']
)
df = df.select(f.expr('parseUrl(url)'))
df.show()
```

## Benchmark
Первое и главное, выше описанный подход имеет смысл только при сложных UDF, в случае конкатенации двух строк - UDF на Rust проиграет в скорости не только нативной реализации, но даже UDF на Python! Причина не в связке Rust-Java, а в том, что при работе нашего jar файла требуется "распаковать" файл библиотеки в систему! То есть файл, который был помещен в `src/main/resources/libs/` , копируется во временную директорию, иначе Java просто неспособна к нему обратиться.

Теперь, можно и к тестам, у нас получилось [следующее](https://github.com/silentsokolov/spark-udf-rust/blob/main/.docs/benchmark.ipynb):
```
native: 7 s ± 197 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
rust: 7.66 s ± 459 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
python: 14 s ± 310 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

NOTE: под `native` понимается вызов встроенных функций Spark
NOTE: 16 CPU, 64Gb RAM, Ubuntu 16.04, Spark 3.1.1 (default settings), Dataset 1kk urls.

Как видим версия на Rust +/- равна по скорости к нативной реализации, что по сути мы и добивались: без знаний Java реализовывать быстрые UDF с минимальным потреблением памяти.

## Заключение
Во-первых, если вы знаете Java/Scala, то стоит использовать их! Не стоит городить костыли на ровном месте. Во-вторых, если вы, как и я, топите за Rust в больших данных, стоит присмотреться к таким инструментам как [datafusion](https://github.com/apache/arrow-datafusion), [ballista](https://github.com/apache/arrow-datafusion/blob/master/ballista/README.md), [polars](https://github.com/pola-rs/polars). Иначе, если готовы к экспериментам, то надеюсь изыскания выше помогут в достижении ваших целей.