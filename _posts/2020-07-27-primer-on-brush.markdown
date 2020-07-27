---
title:  "Передача константных значений в PySpark UDF"
meta_description: "Передача константных значений в PySpark UDF"
date:   2020-07-27 14:56:00
---

При работе с PySpark RDD передача константных значений в функцию довольно тривиальна, то с Dataframe все капельку сложнее.

## Простые константы
Это числа, строки, туплы. Такие константы проще всего передавать через [pyspark.sql.functions.lit](https://spark.apache.org/docs/latest/api/python/pyspark.sql.html#pyspark.sql.functions.lit). Это функция создает колонку из значения.

```python
from pyspark.sql import functions as f
from pyspark.sql import types as t


@f.udf(t.IntegerType())
def foo_bar_udf(col, value):
    if col == 'foo':
        return value
    return 0

def main():
    df = df.withColumn('new_col', foo_bar_udf(f.col('col1'), f.lit(1)))
```

## Комплексные константы
К сожалению, при передаче в функцию lit значений list, dict и тд, возникнет ошибка, что данные тип не поддерживается. Возможно, в версиях Spark выше 2.3 это уже возможно, но на сегодняшний день, чтобы обойти это, приходится городить лямбда-костыли.

```python
def foo_bar(col, value):
    return value.get(col, 0)


def main():
    d = {'foo': 1, 'bar': 2}

    foo_bar_udf = f.udf(
        lambda col: foo_bar(col, d),
        t.IntegerType()
    )

    df = df.withColumn('new_col', foo_bar_udf(f.col('col1')))
```