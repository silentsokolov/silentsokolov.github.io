---
title:  "Работа с C/C++ расширениями в PySpark"
date:   2019-05-22 09:46:00
---

Последние годы Apache Spark зарекомендовал себя как отличный инструмент для работы с большими данными. Не только из-за высокой производительности, но и из-за простоты использования.

PySpark -- отличный инструмент для работы со Spark на Python. Он отлично подходит для быстрого прототипирования обработчиков данных, и все бы ничего, но порой приходится использовать сторонние пакеты.  Обычно достаточно упаковать код в `.zip` или `.egg` использовать `--py-files foo.zip,bar.egg`, как сказано в [документации](https://spark.apache.org/docs/latest/submitting-applications.html). Но это работает не всегда, и огромную долю проблем вызывают python-пакеты с C/C++ кодом. К примеру, можно встретить такое:

```
Caused by: org.apache.spark.api.python.PythonException: Traceback (most recent call last):
  File "/opt/../lib/spark/python/lib/pyspark.zip/pyspark/worker.py", line 359, in main
    func, profiler, deserializer, serializer = read_command(pickleSer, infile)
  File "/opt/../lib/spark/python/lib/pyspark.zip/pyspark/worker.py", line 64, in read_command
    command = serializer._read_with_length(file)
  File "/opt/../lib/spark/python/lib/pyspark.zip/pyspark/serializers.py", line 172, in _read_with_length
    return self.loads(obj)
  File "/opt/../lib/spark/python/lib/pyspark.zip/pyspark/serializers.py", line 577, in loads
    return pickle.loads(obj, encoding=encoding)
  File "./hll-py3.6-linux-x86_64.egg/hll.py", line 7, in <module>
    __bootstrap__()
  File "./hll-py3.6-linux-x86_64.egg/hll.py", line 3, in __bootstrap__
    import sys, pkg_resources, imp
ModuleNotFoundError: No module named 'pkg_resources'
```

Конечно, можно поискать альтернативную pure-python реализацию, но это может привести к потере производительности. Можно использовать `conda`, но не факт что там будет нужный вам пакет. Еще можно установить расширение на все ноды в кластере, но тогда мы будет зависеть от инфраструктуры, что тоже не хорошо.

В нашем проекте мы решили использовать другой подход -- динамически загружать модуль в ходе работы.

1. Для начала нужно скомпилировать C код в совместимую библиотеку, в ручную или в крайнем случае достать из яйца.
2. Пробросить библиотеку в Spark, через `--files /path/lib/foo.so,/project/conf.ini`
3. С помощью `imp` импортировать модуль. У себя мы написали небольшую обертку для удобства:

```python
import imp
import sys

from pyspark import SparkFiles


def load_lib(name, so_path):
    if name not in sys.modules:
        return imp.load_dynamic(name, SparkFiles.get(so_path))
    return sys.modules[name]
```

Вызов будет следующим:

```python
foo = load_lib('foo', 'foo.so')
data = foo.call_function('hello')
```

В итоге мы получаем все преимущества С расширений, независимость от инфраструктуры и решаем проблемы связанные зависимостями в замен на не совсем "удобный" вызов функций.
