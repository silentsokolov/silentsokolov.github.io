---
title:  "Как забрать данные из Google Analytics с помощью Python"
date:   2015-02-23 12:54:00
---

На одном из проектов потребовалось собирать данные из Google Analytics для анализа. Все было бы круто, но заставить собирать данные по схеме server-to-server довольно проблематично, если не знаешь как. В официальной документации скудно описан этот процесс -  исправим эту ошибку!

Предполагаю, что у вас уже сайт подключенный к Google Analytics и объяснять как это делать не нужно.


### Google Console

Первое, что нужно сделать - это получить возможность работать с сервисами гугла, для этого создадим проект в [Google Console](https://console.developers.google.com/project).

![](/images/ga/S1.png)

И активируем Google Analytics API для проекта во вкладке "APIs".

![](/images/ga/S2.png)

Создадим новый Client ID с типом **Service account**.

![](/images/ga/S4.png)

![](/images/ga/S5.png)

По завершению будут выдан приватный ключ, которые нужно будет сохранить его на сервера. Если вы его потеряете придется заново создавать Client ID.

![](/images/ga/S6.png)

Также потребуется email.

![](/images/ga/S7.png)


### Google Analytics

Теперь, шаг, который почти ни где не описан ... надо выдать права полученному Client ID.

![](/images/ga/S8.png)

![](/images/ga/S9.png)

### Python

Все! Используем SignedJwtAssertionCredentials чтобы авторизовываться по ключу.

{% highlight python %}
import httplib2

from django.conf import settings

from apiclient.discovery import build
from oauth2client.client import SignedJwtAssertionCredentials


def prepare_credentials():
    with file(settings.SERVICE_ACCOUNT_PK12_FILE_PATH, 'rb') as key_file:
        credentials = SignedJwtAssertionCredentials(
            settings.SERVICE_ACCOUNT_EMAIL,
            key_file.read(),
            scope='https://www.googleapis.com/auth/analytics.readonly',
        )

        return credentials


def initialize_service():
    http = httplib2.Http()
    credentials = prepare_credentials()
    http = credentials.authorize(http)

    return build('analytics', 'v3', http=http)
{% endhighlight %}

Profit!
