---
layout: post
title:  "Бесплатный хостинг для Ghost на Heroku"
date:   2014-06-26 08:22:00
---

Использование **Heroku** как хостинга для вашего **Ghost** блога имеет ряд недостатков, с которыми придется мириться:

- вы не сможете загружать изображения, нужно будет использовать сервис [imgur](http://imgur.com/) или ему подобные, DropBox, тоже сойдет;
- лучше не использовать Heroku для популярных блогов, так как бесплатная его версия (и его компонентов) имеет значительные ограничения, будь то коннекты к базе или сам размер БД.

### Регистрация на Heroku

Первое, что нужно сделать это зарегистрироваться на [heroku](https://heroku.com/) и ознакомиться с [туториалом](https://devcenter.heroku.com/articles/quickstart).


### Скачать Ghost

Мы не сможем ничего написать в нашем блоге, без самого блога! Нужно скачать свежую версию Ghost с [официального сайта](https://ghost.org/download/), последняя на момент написания этой статьи 0.4.2.

После этого нужно распоковать архив, например в папку `blog`:

{% highlight bash %}
$ unzip ghost-0.4.2.zip -d blog && blog
{% endhighlight %}

### Создание репозитария

Если вы прочитали руководство по Heroku, вам уже известно, что он тесно связан с Git-репозитариями, поэтому надеюсь, что git у вас уже установлен.

Создадим репозитарий:

{% highlight bash %}
$ git init
{% endhighlight %}

Добавим в исключения папку `node_modules`, это важно, иначе ваш блог не запуститься:

{% highlight bash %}
$ echo "node_modules/" > .gitignore
{% endhighlight %}

Зафиксируем изменения:

{% highlight bash %}
$ git add -A
$ git commit -m 'Init commit'
{% endhighlight %}


### Создание Heroku-приложения

Войдем и создадим приложение:

{% highlight bash %}
$ heroku login
$ heroku create my-blog-name
{% endhighlight %}

#### База даных

По-умолчанию Ghost в качестве базы данных использует SQLite, нам она не подходит, так как она хранится локально, а Heroku, при каждом пуше, удаляет все что не храниться в репозитарии.

Самый подходящий вариант это PostgreSQL или MySQL. Со вторым сложнее так что будем использовать PostgreSQL:

{% highlight bash %}
$ heroku addons:add heroku-postgresql:dev
Adding heroku-postgresql:dev on my-blog... done, v3 (free)  
Attached as HEROKU_POSTGRESQL_RED_URL  
Database has been created and is available
{% endhighlight %}

В ответе heroku вернет идентификатор базы `HEROKU_POSTGRESQL_RED_URL` (у вас он возможно будет другим), чтобы не использовать его на прямую добавим его в глобальные переменные:

{% highlight bash %}
$ heroku pg:promote HEROKU_POSTGRESQL_RED_URL  
Promoting HEROKU_POSTGRESQL_RED_URL to DATABASE_URL... done
{% endhighlight %}

#### Почта

Для обсужевания почты установим нам понадобится еще один компонент - Mandrill:

{% highlight bash %}
$ heroku addons:add mandrill
{% endhighlight %}


### Конфигурация и настройка

#### Зависимости

Использование PostgreSQL в качестве базы для Ghost требует установки [node-postgres](https://github.com/brianc/node-postgres):

{% highlight bash %}
$ npm install pg --save
{% endhighlight %}

npm все поставит и изменит ваш `package.json` добавив новую зависимость, нужно зафиксировать это:

{% highlight bash %}
$ git add package.json
$ git commit -m 'Add node-postgres'
{% endhighlight %}

#### Настройки

Создадим файл настройки из приложенного примера:

{% highlight bash %}
$ cp config.example.js config.js
{% endhighlight %}

Внесем изменения в `Production` раздел:

{% highlight javascript %}
// ### Production
production: {
    url: 'http://my-blog-name.herokuapp.com/',
        mail: {
          transport: 'SMTP',
          host: 'smtp.mandrillapp.com',
          options: {
            service: 'Mandrill',
            auth: {
              user: process.env.MANDRILL_USERNAME,
              pass: process.env.MANDRILL_APIKEY
            }
          }
        },
        database: {
          client: 'postgres',
          connection: process.env.DATABASE_URL,
          debug: false
        },
        server: {
          host: '0.0.0.0',
          port: process.env.PORT
        }
    },
{% endhighlight %}

Зафиксируем изменения:

{% highlight bash %}
$ git add config.js
$ git commit -m 'Configured'
{% endhighlight %}


### Deploy!

#### Установка блога

Для начала укажем heroku, что он должен использовать `production` конфигурацию:

{% highlight bash %}
$ heroku config:set NODE_ENV=production
{% endhighlight %}

Все готово! Для размещения и запуска приложения выполните:

{% highlight bash %}
$ git push heroku master
{% endhighlight %}

Через некоторые время ваш блог станет доступен по адресу: [http://my-blog-name.herokuapp.com/](my-blog-name.herokuapp.com)


#### Настройка блога

Чтобы настроить свой блог просто перейдите [http://my-blog-name.herokuapp.com/ghost/](my-blog-name.herokuapp.com/ghost/) и укажите требуемые данные.


## Еще раз про картинки

Важно отметить, что если вы хотите установить логотип для своего блога - то сделать это через админку не получиться! Так как после очередного `$ git push heroku master` все локальные изменения удалятся, включая ваш логотип. Так что самым безопасным вариантом будет добавить его в репозитарий и «зашить» в код шаблона.
