---
title:  "Умное управление настройками git"
meta_description: "Использование Conditional includes в конфигурации git"
date:   2021-01-29 10:46:30
---

Порой приходиться работать с git-репозиториями, которые требуют различные индивидуальные настройки. К примеру, вы работаете с несколькими компаниями и для каждого репозитория необходимо указывать корпоративный email. Когда компаний и репозиториев мало, то можно вручную вносить изменения для каждого: `git config user.email username@example.com`. Но сейчас, во времена микросервисов, такой подход может быть утомительным.

Для решения этой проблемы, при работе с настройками git можно использовать `include` и `includeIf`. Они позволяют подтягивать часть конфигураций из другого источника. `includeIf` также содержит условие в зависимости от которого произойдет включение (вставка) конфигурации или нет. С всеми возможными вариантами условий можно знакомиться в документации [Conditional includes](https://git-scm.com/docs/git-config#_conditional_includes). 

В нашем случае, мы можем настроить email и подпись в зависимости от каталога, где находиться репозиторий. В `.gitconfig`:

```
[includeIf "gitdir:projects/company_a/"]
  path = .gitconfig-company-a
[includeIf "gitdir:projects/company_b/"]
  path = .gitconfig-company-b
```

К примеру, `.gitconfig-company-a` может быть следующим:

```
[user]
  email = username@company-a.com
  signingkey = ******
```

Теперь любой git-репозиторий, путь до которого включает `projects/company_a/` будет использовать нужный адрес электронной почты и собственный ключ для подписания коммитов. Стоит также отметить, что `gitdir` проверяет весь путь, поэтому не советую задавать условие, которое содержит один единственный каталог.

Готово! Теперь у вас всегда будет нужный email, и ни кто из коллег не осудит вас за использование личного электронного адреса (и служба безопасности тоже).

