---
title:  "Параллельный запуск заданий в PySpark"
meta_description: "Параллельный запуск заданий в PySpark с помощью FAIR scheduler"
date:   2020-08-27 09:10:00
---

Порой требуется обработать небольшие куски данных используя разные подходы, к примеру применяя для каждого свой фильтр в зависимости от источника данных; или тренировка нескольких ML-моделей разом. Конечно, можно запустить обработку в отдельных приложениях (разные SparkContext), но это не оптимальный вариант (оверхед на создание контекста, не читаемый jobhistory, и тд), будет лучше использовать встроенные механизмы самого PySpark.

PySpark имеет [планировщик задач](http://spark.apache.org/docs/latest/job-scheduling.html#scheduling-within-an-application), по-умолчанию, работающий в режиме `FIFO`. То есть каждое задание будет получать приоритет на использование всех ресурсов кластера. Это приводит к тому, что небольшие задания могут запускаться не параллельно, а последовательно, если мало внимания уделить работе с партициями. Поэтому в подобных случаях лучше отдать предпочтение "справедливому" (`FAIR`) планировщику, который равномерно распределит ресурсы кластера.

Наглядный пример, с использованием `FAIR scheduler` и `ThreadPool`:

```python
from multiprocessing.pool import ThreadPool

from pyspark import SparkContext, StorageLevel
from pyspark.sql import SparkSession
from pyspark.sql import functions as f
from pyspark.sql import types as t


THREADS = 5
SCHEMA = t.StructType([
    t.StructField('source', t.StringType(), False),
    t.StructField('city', t.StringType(), False),
    t.StructField('population', t.FloatType(), False),
])


def processing_source(df, source):
    df_local = df.filter(f.col('source') == source)
    df_local = '...'


sc = SparkContext(appName='city by source')
ss = SparkSession(sc)

df = ss.read.format('csv').load('/path/to/file.csv', schema=SCHEMA)
df = df.select('...')  # prepare data
df = df.persist(StorageLevel.DISK_ONLY_2)  # or df.cache()

pool = ThreadPool(THREADS)
pool.map(lambda s: processing_source(df, s), SOURCE_LIST)
pool.close()
pool.join()

# run
# spark-submit --conf spark.scheduler.mode=FAIR ... task.py
```
