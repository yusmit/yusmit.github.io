# Отладка плэйбуков ansible
Рябятаньки, в этом руководстве я постараюсь рассказать как дебажить playbookи, устраняя потенциальные проблемы, не дожидаясь обезвреживания продакшена. Если вы читаете это, то я уверен что вы, также как и я, прочитали десяток статей о том как установить ansible и запускать (применять - режет слух) плэйбуки для конфигурирования всего до чего дотягивается руки.

Что будет в данном гайде:

* [Использование вспомогательного отладочного вывода](#debug)
* [Проверка синтаксиса](#syntax)
* [Линтинг](#lint)
* [Холостой запуск](#check)
* [Получение подробного вывода во время запуска playbooka](#verbose)
* [Включение встроенного дебагера](#debuger)
* [Запуск с демонстрацией отличий](#diff)

## Использование вспомогательного отладочного вывода

Самое очевидное что приходит на ум это использование вспомогательного отладочного вывода. Тут нам на помощь приходит встроенный модуль `debug`. Все просто как 3 копейки - дебаг позволяет вывести сообщение или значение переменной.

{% raw %}
```yaml
---
- name: Print the gateway for each host when defined
  ansible.builtin.debug:
    msg: System {{ inventory_hostname }} has gateway {{ ansible_default_ipv4.gateway }}
  when: ansible_default_ipv4.gateway is defined

```
{% endraw %}

Даже результат выполнения shell можно вывести определяя промежуточную переменную - словарь и печатая ее по ходу выполнения:

{% raw %}
```yaml
---
- name: Get uptime information
  ansible.builtin.shell: /usr/bin/uptime
  register: result

- name: Print return information from the previous task
  ansible.builtin.debug:
    var: result.stdout
    verbosity: 2
```
{% endraw %}

## Проверка синтаксиса syntax-check

По умолчанию Ansible имеет встроенную программу проверки синтаксиса под названием --syntax-check. Эта программа проверки синтаксиса - отличный способ выявить ошибки в ваших yml-файлах перед запуском.       

Средство проверки синтаксиса анализирует файл YAML и ищет любые потенциальные ошибки, не выполняя ни одной из ваших задач playbook.

`ansible-playbook --syntax-check your_playbook.yml`

Если в вашем playbooke есть ошибки, вы увидите сообщение об ошибке и иногда даже номер строки, в которой ошибка произошла явно. Стоит сделать оговорку, что при некорректных отступах ошибка будет указана на уровне задания (таски, task).

## Линтер ansible-lint

Ansible-lint https://github.com/ansible/ansible-lint проверяет плэйбуки и роли на наличие методов и поведения, которые потенциально могут быть улучшены и конечно проверит синтаксис. Линтер обычно присутствует при тестировании molecule и не даст вам смержить в мастер-ветку сырой, некрасивый, неблагополучный код.

Запускаем линтер очень просто:

* Проверка плейбука - `ansible-lint your_playbook.yml`
* Проверка роли - линтер пройдет по всем файлам роли `ansible-lint your_rolename.yml`
* Найдет yml файлы или роли в текущей директории и проверит их `ansible-lint`

Линтер можно гибко настроить в соответствии с [соглашениями](https://ansible.readthedocs.io/projects/lint/configuring/) в команде изменяя, добавляя или убирая проверки.

## Холостой или пробный запуск - check

Пробный запуск - отличный способ проверить ваш playbook на наличие ошибок перед запуском в реальных системах. Пробный запуск позволяет имитировать запуск playbooka без изменения каких-либо данных или выполнения каких-либо действий на хостах.

`ansible-playbook --check create_user.yml`

Обратите внимание, что не все модули поддерживают флаг `--сheck` или `-С` Например, модули, которые изменяют данные или систему, не поддерживают флаг `check`. Задачи, использующие модули, поддерживающие флаг `check`, показывают результат выполнения задачи на целевых хостах и любые изменения, внесенные, если модуль был выполнен.

Естественно, если вы используете bashsible, т.е. пренебрегаете использованием модулей command или shell вместо использования модулей ansible, то состояние работы модуля shell-command будет практически всегда *changed*. Как обрабатывать такое - тема отдельной статьи. Сразу хочу предупредить, что чаще всего роль в `check`-mode будет "падать", из-за зависимостей тасок-задач друг от друга. Например, первая таска создаёт группу linux, а вторая пользователя в этой группе. Естественно вторая будет падать при отсутствии группы. Обрабатывается это игнорированием ошибки: `ignore_errors: "{{ ansible_check_mode }}"` - то есть игнорировать ошибку при использовании `check`.

## Получение подробного вывода во время запуска playbooka

Периодически возникают проблемы, и необходимо посмотреть, что делает ansible во время запуска плэйбука. Запуск ansible с флагом `v`, также как и в большинстве утилит командной строки, повышает детализацию вывода - увеличение выводимой информации поможет узнать с какими кредами (учетными данными) вы запустили задачу, какая именно таска выполняется (полезно если забыли именовать task или запутались в логике переходов), что именно и с какими переменными выполняется в системе (очень полезно при отладке параметризованных shell)

`ansible-playbook -vvv your_playbook.yml`

## Включение запуска отладчика playbook при запуске

Вы видели, что устранение неполадок в плэйбуках с помощью опций в ansible-playbook команде работает просто замечательно. Но помимо этих опций, Ansible также имеет встроенный отладчик. Он довольно хорошо описан в [документации](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_debugger.html) (но посмотрим правде в глаза - кто ж ее читает). Отладчик ansible - это мощный инструмент, который позволяет пошагово просматривать ваш плэйбук и видеть результаты каждой задачи по мере их выполнения. Чтобы включить отладчик, необходимо в файл ansible.cfg строку `enable_task_debugger = True`, в раздел "default". Как только отладчик playbook включен, любая сбойная или недоступная задача запустит встроенное приглашение взаимодействия с отладчиком.

Когда отладчик playbook настроен на запуск при каждом запуске playbook, можно поместить ключевое слово `debugger` в любое место playbook. Это позволяет выполнять отладку различных сущностей, таких как плэйбуки, роли, блоки или даже отдельные задачи.

Ключевое слово debugger поддерживает пять значений, включая наиболее часто используемое (по-умолчанию) значение (`on_failed`) при отладке воспроизведения:

1. `always` Отладчик всегда будет запускаться, когда указано, независимо от результата.
2. `never` Отладчик никогда не запустится, независимо от результата.
3. `on_failed` Отладчик запустится только в случае сбоя задачи (для сбоев задач отладки).
4. `on_skipped` Отладчик запустится только в том случае, если задача пропустила себя при выполнении
5. `on_unreachable` Отладчик запустится только в том случае, если задача недоступна. Используйте это значение, когда ваш управляемый хост недоступен или когда у задачи истекло время ожидания.

Пример использования:

{% raw %}
```yaml
---
- name: Debugger demo
  hosts: all
  debugger: on_failed
  vars:
    - pkg_name: does_not_exist
  tasks:
   - name: Install a package
     yum:
      name: "{{ pkg_name }}"
      state: present
```
{% endraw %}

## Запуск с демонстрацией отличий

Вообще считаю это незаслуженно забываемой киллер фичей - ведь практически всегда мы используем не просто установку чего либо но и конфигурирование с использованием jinja2-шаблонов. Использование флага `--diff` или `-D` позволяет увидеть содержимое конфигов, изменяемые права и владение файлов по ходу выполнения больших ролей без хождения на хост и дополнительных отладочных выводов. Согласитесь это гораздо удобнее, нежели запускаться с супер вербозом `-vvvv` и потом фильтровать вывод стараясь рассмотреть желаемые изменения.

`ansible-playbook -D your_palybook.yml`

## Выводы

Выше были описаны способы отладки ansible. Каждый из них имеет свою область применения, обусловленную достоинствами и недостатками:

|  |  |
| --- | --- |
| Способ | Применение |
| Использование вспомогательного отладочного вывода | Необходимо выводить отладочную информацию при каждом запуске и хранить ее в логах |
| Проверка синтаксиса | Предварительная проверка корректности кода |
| Линтинг | Предварительная проверка корректности кода и возможности его улучшения встроенными методами |
| Холостой запуск | Запуск на целевой системе без внесения изменений |
| Получение подробного вывода во время запуска | Необходимо вывести креды, trace выполнения команд |
| Включение встроенного дебагера | Гибкое использование отладочной информации на различных этапах выполнения |
| Запуск с демонстрацией отличий | Вывод изменений шаблонизируемых файлов, прав файлов. |