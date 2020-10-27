---
title:  "Безопасные bash-скрипты или set -euxo pipefail"
meta_description: "Пример работы с bash-скриптами и обеспечание безопасной работы больших bash-сценариев"
date:   2020-10-27 19:03:56
---

В 2020 году когда есть такие языки как python, ruby и go, использовать bash приходится редко, но иногда без него не обойтись.

Bash не похож на высокоуровневые языки программирования, он не предоставляет привычных гарантий. К примеру, если в python обратиться к неинициализированной переменной, то скрипт тут же завершиться, не выполнив ни одной инструкции. В bash это не так, любая переменная, к которой вы обратились, но не инициализировали, будет заменена на пустую строку. Только представьте сколько можно наворотить дел, если подобная переменная была в параметрах у команды `rm -rf`.

К счастью, можно изменить поведение оболочки используя встроенные функции, в частности [set](https://www.gnu.org/software/bash/manual/html_node/The-Set-Builtin.html). С ее помощью, можно значительно повысить безопасность.

## set -e
Указав параметр `-e` скрипт немедленно завершит работу, если любая команда выйдет с ошибкой. По-умолчанию, игнорируются любые неудачи и сценарий продолжет выполнятся. Если предполагается, что команда может завершиться с ошибкой, но это не критично, можно использовать пайплайн `|| true`.

Без `-e`:

```bash
#!/bin/bash

./non-existing-command
echo "RUNNING"

# output
# ------
# line 3: non-existing-command: command not found
# RUNNING
```

С использованием `-e`:

```bash
#!/bin/bash
set -e

./non-existing-command || true
./non-existing-command
echo "RUNNING"

# output
# ------
# line 4: non-existing-command: command not found
# line 5: non-existing-command: command not found
```

## set -o pipefail
Но `-e` не идеален. Bash возвращает только код ошибки последней команды в пайпе (конвейере). И параметр `-e` проверяет только его. Если нужно убедиться, что все команды в пайпах завершились успешно, нужно использовать `-o pipefail`.  

Без `-o pipefail`:

```bash
#!/bin/bash
set -e

./non-existing-command | echo "PIPE"
echo "RUNNING"

# output
# ------
# PIPE
# line 4: non-existing-command: command not found
# RUNNING
```

С использованием `-o pipefail`:

```bash
#!/bin/bash
set -eo pipefail

./non-existing-command | echo "PIPE"
echo "RUNNING"

# output
# ------
# PIPE
# line 4: non-existing-command: command not found
```

## set -u
Наверно самый полезный параметр - `-u`. Благодаря ему оболочка проверяет инициализацию переменных в скрипте. Если переменной не будет, скрипт немедленно завершиться. Данный параметр достаточно умен, чтобы нормально работать с переменной по-умолчанию `${MY_VAR:-$DEFAULT}` и условными операторами  (`if`, `while`,  и др).

Без `-u`:

```bash
#!/bin/bash

echo "${MY_VAR}"
echo "RUNNING"

# output
# ------
# 
# RUNNING
```

С использованием `-u`:

```bash
#!/bin/bash
set -u

echo "${MY_VAR}"
echo "RUNNING"

# output
# ------
# line 4: MY_VAR: unbound variable
```

## set -x
Параметр `-x` очень полезен при отладке. С помощью него bash печатает в стандартный вывод все команды перед их исполнением. Стоит учитывать, что все переменные будут уже доставлены, и с этим нужно быть аккуратнее, к примеру если используете пароли.

Без `-x`:

```bash
#!/bin/bash

MY_VAR="a"
echo "${MY_VAR}"
echo "RUNNING"

# output
# ------
# a
# RUNNING
```

С использованием `-x`:

```bash
#!/bin/bash
set -x

echo "${MY_VAR}"
echo "RUNNING"

# output
# ------
# + MY_VAR=a
# + echo a
# a
# + echo RUNNING
# RUNNING
```

## Вывод
Не стоит забывать, что все эти параметры можно объединять и комбинировать между собой! Думаю, при работе с bash будет хорошим тоном начинать каждый сценарий с  `set -euxo pipefail`. 

