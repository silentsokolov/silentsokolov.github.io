---
title:  "Использование gzip для сжатия запросов boto3"
meta_description: "Разбор подхода написания udf для apache spark на языке rust"
date:   2021-07-31 14:40:21
---

По умолчанию в библиотеке `boto3` не включено сжатие для запросов, что может стать серьезной проблемой, так как AWS выдвигает серьезные ограничения к объему передаваемых данных. Все бы ничего, если бы сжатие можно было включить флагом или параметром `boto3`, но этого нет.

Так как `boto3` является оберткой HTTP клиента, самым очевидным вариантом реализовать сжатие - перехватить запрос до того, как он будет отправлен. 

Для подобных ситуаций в библиотеке предусмотрена [система евентов](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/events.html), которая позволяет зарегистрировать функцию-обработчик. Как только событие произойдет обработчик, будет вызван с определенными аргументами. Используя данную систему, несложно сжать данные и указать нужный HTTP заголовок.

```python
import boto3
import gzip

s3 = boto3.client('s3')

event_system = s3.meta.events

def gzip_request_body(request, **kwargs):
    gzipped_body = gzip.compress(request.body)
    request.headers.add_header('Content-Encoding', 'gzip')
    request.data = gzipped_body

event_system.register('provide-client-params.s3.PutObject', gzip_request_body)

# Your code here
```

Система событий очень гибкая: можно зарегистрировать несколько обработчиков или подключиться в пачке эвентов. Что позволяет писать более практичный код и не заниматься monkey-патчингом.