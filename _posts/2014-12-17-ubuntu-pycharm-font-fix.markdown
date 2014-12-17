---
layout: post
title:  "Исправляем кривое отображение шрифтов в PyCharm"
date:   2014-12-17 19:03:01
tags: [programming]
image: "/images/post-01.png"
---

При использовании Ubuntu наткнулся на проблему не корректного отображения шрифтов в PyCharm, да и наверно это касается всех продуктов JetBrains. Погуглив нашел пару решений, и вот одно из рабочих (проверялось на Ubuntu 14.04, с PyCharm 3 и 4).


#### Infinality

Для начала ставим Infinality и настраиваем его:

{% highlight bash %}
$ sudo add-apt-repository ppa:no1wantdthisname/ppa
$ sudo apt-get update && sudo apt-get update upgrade
$ sudo apt-get install fontconfig-infinality
$ sudo /etc/fonts/infinality/infctl.sh setstyle osx
$ sudo nano /etc/profile.d/infinality-settings.sh
{% endhighlight %}

Убедитесь что в файле `infinality-settings.sh`, была установлена нужная тема:

{% highlight bash %}
$ sudo nano /etc/profile.d/infinality-settings.sh
{% endhighlight %}

{% highlight bash %}
USE_STYLE="UBUNTU"
{% endhighlight %}


#### OpenJDK с fontfix

Также нужно поставить заплатку для шрифтов на OpenJDK:

{% highlight bash %}
$ sudo apt-add-repository -y ppa:no1wantdthisname/openjdk-fontfix
$ sudo apt-get update
$ sudo apt-get install openjdk-7-jdk
{% endhighlight %}


### Конфигурация и запуск

Теперь нам нужно отредактировать файл `$PYCHARM_HOME/bin/pycharm.vmoptions` или `pycharm64.vmoptions`, в зависимости от разрядности вашей системы, добавив в конец:

{% highlight bash %}
-Dawt.useSystemAAFontSettings=lcd
-Dswing.aatext=true
-Dsun.java2d.xrender=true
{% endhighlight %}


Если у вас одна версия java на машине, то следующий шаг можно не делать, иначе, отредактируйте файл `$IDEA_HOME/bin/pycharm.sh`, заменив строку:

{% highlight bash %}
eval "$JDK/bin/java" $ALL_JVM_ARGS -Djb.restart.code=88 $MAIN_CLASS_NAME "$@"
{% endhighlight %}

на

{% highlight bash %}
eval "/usr/lib/jvm/java-7-openjdk-amd64/jre/bin/java" $ALL_JVM_ARGS -Djb.restart.code=88 $MAIN_CLASS_NAME "$@"
{% endhighlight %}


После чего перезагрузите машину. Profit!
