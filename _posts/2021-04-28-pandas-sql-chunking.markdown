---
title:  "Использование памяти при загрузке данных в pandas из базы"
meta_description: "Сократить использование памяти в pandas с помощью серверых курсоров или потоковой передачи данных"
date:   2021-04-28 19:57:30
---

При работе с pandas'ом часто данные подтягиваются из реляционной базы данных. Для этих целей в pandas есть удобный метод [read_sql](https://pandas.pydata.org/docs/reference/api/pandas.read_sql.html). Но не все так просто, при работе с массивными датасетами этот метод быстро исчерпает оперативную память и может упасть с ошибкой `out of memory`. Знающие люди могут возразить, что данная функция имеет возможность пакетной обработки, но к сожалению даже используя `chunksize` использование памяти будет значительным.

_Отмечу, что это касается pandas 1.1.*, возможно в будущих версиях эту проблему уже исправили._ Но пока [проблема не решена](https://github.com/pandas-dev/pandas/issues/35689).

При использовании функции в лоб:

```python
import pandas as pd
from sqlalchemy import create_engine

engine = create_engine()

dataframe = pd.read_sql("SELECT * FROM table1", engine)
``` 

В память будет помещено несколько блоков одних и тех же данных: первый это сырые данные полученные из бд полученные из `dbapi_cursor.fetchall()`, далее SQLAlchemy проведет манипуляции со строками и в конце сам блок кортежей, который использует pandas. Не совсем эффективно.

При использовании функции с `chunksize`:

```python
import pandas as pd
from sqlalchemy import create_engine

engine = create_engine()

dataframe = pd.read_sql("SELECT * FROM table1", engine, chunksize=10000)
``` 

Потребление памяти безусловно будет ниже, но SQLAlchemy извлечет все данные и лишь, затем, по кускам, будет передавать в функцию создающую dataframe. Это из-за того, что SQLAlchemy использует разбиение данных локально, а не на стороне сервера.

Чтобы получить настоящую пакетную обработку, нужно указать SQLAlchemy использовать [потоковую передачу данных](https://docs.sqlalchemy.org/en/14/core/connections.html#engine-stream-results). Тогда вместо того, чтобы загружать все строки в память, он будет загружать строки из базы данных итеративно. Код конечно измениться кардинально:

```python
import pandas as pd
from sqlalchemy import create_engine

engine = create_engine()
conn = engine.connect().execution_options(stream_results=True)

dataframe = pd.read_sql("SELECT * FROM table1", chunksize=10000)
for chunk_df in pd.read_sql("SELECT * FROM table1", conn, chunksize=10000):
    print(chunk_df)
``` 

При выполнении будет использоваться сколько памяти, сколько занимает один чанк плюс дополнительные расходы на работу со строками.

Как видно у использования потоковой передачи, есть значительный минус - не получиться получать один dataframe. Но как показывает практика при больших объемах это нужно крайне редко и подсчет можно произвести по частям.
 
