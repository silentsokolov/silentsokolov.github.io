---
title:  "Spark UDF или всегда используйте explain"
meta_description: "Разбераемся почему Spark выполняет UDF несколько раз"
date:   2021-02-27 12:06:30
---

Всякий, кто хоть раз разбирался с User Defined Functions (UDF) в Spark натыкался на следующее утверждение:

> Use the higher-level standard Column-based functions with Data set operators whenever possible before reverting to using your own  custom UDF functions since UDFs are a blackbox for Spark and so it does not even try to optimize them.  

И еще вереницу статей на тему, что использовать UDF крайне нежелательно и лучше работать только со встроенными функциями. И все это действительно так, но иногда без UDF обойтись нельзя. А материалов на тему "как правильно писать UDF" днем с огнем не найдешь. 

Поэтому главное правило: **если есть UDF всегда используйте** `explain`.

Часто Catalyst (движок оптимизаций) работает с UDF так, как если бы это было обычные встроенные функции, что иногда приводит к нежелательному поведению. Предположим, что нам нужно найти подсеть в наборе IP адресов. Мы определяем UDF и пишем набор инструкций для трансформации данных:

```python
from pyspark.sql import functions as f


def to_subnet(v):
    # body
    return v


udf_to_subnet = f.udf(to_subnet)

df.select(
    f.col('ip'),
    f.col('unix_timestamp'),
).filter(
    f.col('ip').isNotNull()
).groupBy(
    f.col('ip'),
).agg(
    f.min(f.col('ts')).alias('ts')
).withColumn(
    'subnet', to_subnet(f.col('ip'))
).filter(
    f.col('subnet').isNotNull()
)
```

При разборе плана запроса мы увидим, что наша функция была вызвана дважды (внимание на _BatchEvalPython_).

```
== Physical Plan ==
*(4) Project [ip#243, unix_timestamp#252L, pythonUDF0#261 AS subnet#256]
+- BatchEvalPython [to_subnet(ip#243)], [pythonUDF0#261]
   +- *(3) HashAggregate(keys=[ip#243], functions=[min(unix_timestamp#244L)], output=[ip#243, unix_timestamp#252L, ip#243])
      +- Exchange hashpartitioning(ip#243, 200), true, [id=#246]
         +- *(2) HashAggregate(keys=[ip#243], functions=[partial_min(unix_timestamp#244L)], output=[ip#243, min#263L])
            +- *(2) Project [_c2#136 AS ip#243, unix_timestamp(gettimestamp(_c3#137, dd/MMM/yyyy:HH:mm:ss Z, Some(Europe/Moscow)), yyyy-MM-dd HH:mm:ss, Some(Europe/Moscow)) AS unix_timestamp#244L]
               +- *(2) Project [_c2#136, _c3#137]
                  +- *(2) Filter isnotnull(pythonUDF0#260)
                     +- BatchEvalPython [to_subnet(_c2#136)], [pythonUDF0#260]
                        +- *(1) Project [_c2#136, _c3#137]
                           +- *(1) Filter isnotnull(_c2#136)
                              +- FileScan csv [_c2#136,_c3#137] Batched: false, DataFilters: [isnotnull(_c2#136)], Format: CSV, Location: InMemoryFileIndex[hdfs://..., PartitionFilters: [], PushedFilters: [IsNotNull(_c2)], ReadSchema: struct<_c2:string,_c3:string>
```

В случае если функция выполняет несложные операции, то это не так критично, но иначе мы серьезно потеряем в производительности.

Ответ, почему так происходит прост: все UDF по умолчанию _детерминированные_, поэтому Catalyst не стесняется использовать их многократно. Чтобы избежать подобного поведения, функцию нужно пометить как _недетерминированную_:

```python
udf_to_subnet = f.udf(to_subnet).asNondeterministic()
```

Теперь план выглядит не только иначе, но и порой работает в несколько раз быстрее:

```
== Physical Plan ==
*(3) Project [ip#264, unix_timestamp#273L, pythonUDF0#281 AS subnet#277]
+- *(3) Filter isnotnull(pythonUDF0#281)
   +- BatchEvalPython [to_subnet(ip#264)], [pythonUDF0#281]
      +- *(2) HashAggregate(keys=[ip#264], functions=[min(unix_timestamp#265L)], output=[ip#264, unix_timestamp#273L, ip#264])
         +- Exchange hashpartitioning(ip#264, 200), true, [id=#290]
            +- *(1) HashAggregate(keys=[ip#264], functions=[partial_min(unix_timestamp#265L)], output=[ip#264, min#283L])
               +- *(1) Project [_c2#136 AS ip#264, unix_timestamp(gettimestamp(_c3#137, dd/MMM/yyyy:HH:mm:ss Z, Some(Europe/Moscow)), yyyy-MM-dd HH:mm:ss, Some(Europe/Moscow)) AS unix_timestamp#265L]
                  +- *(1) Filter isnotnull(_c2#136)
                     +- FileScan csv [_c2#136,_c3#137] Batched: false, DataFilters: [isnotnull(_c2#136)], Format: CSV, Location: InMemoryFileIndex[hdfs://..., PartitionFilters: [], PushedFilters: [IsNotNull(_c2)], ReadSchema: struct<_c2:string,_c3:string>
```

Как видите, правило обозначенное вначале помогло сэкономить время и ресурсы (а часто и деньги). Поэтому еще раз: **если есть UDF всегда используйте** `explain`.