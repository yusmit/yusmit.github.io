# Покрываем плэйбуками ansible IaC в части предоставления доступа
**Инфраструктура как код** (*Infrastructure-as-Code; Iac*) — это подход для управления и описания инфраструктуры ЦОД через конфигурационные файлы, а не через ручное редактирование конфигураций на серверах или интерактивное взаимодействие.

Итак, мы уже [научились](https://habr.com/ru/articles/746864/) быстро поднимать инфраструктуру с помощью IaC, завели кучу репозиторриев и готовы восстановить исковерканную или сломанную инфраструктуру в части готовности сервисов. В качестве следующего этапа можно рассмотреть предоставление доступов к ресурсам и документирование предоставления таких доступов. Такой подход позволит не только быстро находить на каком основании Вася дропнул табличку на проде, но и ускорить предоставление доступов и сократить количество ошибок.

Я покажу своё видение структуры такой ansible-роли (далее - роль).

* [Структура tasks/main.yml](#tasks/main.yml)
* [Особенности сбора фактов о текущем состоянии](#%D0%9E%D1%81%D0%BE%D0%B1%D0%B5%D0%BD%D0%BD%D0%BE%D1%81%D1%82%D0%B8%20%D1%81%D0%B1%D0%BE%D1%80%D0%B0%20%D1%84%D0%B0%D0%BA%D1%82%D0%BE%D0%B2%20%D0%BE%20%D1%82%D0%B5%D0%BA%D1%83%D1%89%D0%B5%D0%BC%20%D1%81%D0%BE%D1%81%D1%82%D0%BE%D1%8F%D0%BD%D0%B8%D0%B8)
* [Порядок работы с сущностями](#%D0%9F%D0%BE%D1%80%D1%8F%D0%B4%D0%BE%D0%BA%20%D1%80%D0%B0%D0%B1%D0%BE%D1%82%D1%8B%20%D1%81%20%D1%81%D1%83%D1%89%D0%BD%D0%BE%D1%81%D1%82%D1%8F%D0%BC%D0%B8)
* [Формирование отладочной информации](#%D0%A4%D0%BE%D1%80%D0%BC%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5%20%D0%BE%D1%82%D0%BB%D0%B0%D0%B4%D0%BE%D1%87%D0%BD%D0%BE%D0%B9%20%D0%B8%D0%BD%D1%84%D0%BE%D1%80%D0%BC%D0%B0%D1%86%D0%B8%D0%B8)

Начнем с tasks/main.yml:

```yaml
---
# Проверка корректности конфигурационных
# файлов описанных в IaC
# Проверки корректности и имен и взаимосвязей
- name: Check
  include_tasks:
    file: check.yml
  tags: always
  run_once: true

# Установка на целевой хост пакетов,
# необходимых для работы
# ansible-модулей

- name: installing the required packages
  apt:
    name: "{{ item }}"
  loop:
    - required_package_1
    - required_package_2

# Получение имени последнего коммита.
# Его можно использовать в качестве темы письма, а также
# проводить различные проверки на его содержание.

- name: get latest commit message
  shell: git log --pretty="%s" -n1
  register: latest_commit_message
  tags: always
  run_once: true
  delegate_to: localhost
  changed_when: false

# Сбор фактов о существующих сущностях
# на которые выдаются права
# и правах предоставленных пользователям

- name: Collect info about entities, roles
  become: true
  become_user: postgres
  community.postgresql.postgresql_info:
    login_unix_socket: '{{ pg_login_unix_socket }}'
    db: postgres
    filter:
      - "databases"
      - "roles"
  register: "pg_exist"
# Тегирование -  важный элемент формирования роли.
# Позволит нам изорировать работу с различным видом сущностей.
  tags:
    - pg_db_config
    - pg_roles_config
  run_once: true

# Последовательная работа с ролями

- name: Work with roles
  include_tasks:
    file: pg_roles.yml
    apply:
      tags: pg_roles_config
  when: pg_roles_config
  tags: pg_roles_config
  run_once: true

# работа с сущностями

- name: Work with db
  include_tasks:
    file: pg_db.yml
    apply:
      tags: pg_db_config
  when: pg_db_config
  tags: pg_db_config
  run_once: true

# работа с дополнительными механизмами,
# обеспечивающими доступ

- name: Work with pg_hba
  include_tasks:
    file: pg_hba.yml
    apply:
      tags: pg_hba_config
  when: pg_hba_config
  tags: pg_hba_config
  # run_once: true

# Вывод ошибок полученных в рамках работ,
# естественно красным цветом

- name: Fail message
  debug:
    msg: "Не все конфиги были применены \n{{ rescue_msgs|to_nice_yaml }}"
  failed_when: true
  when: rescue_msgs != []
  tags: always
  run_once: true
```

Здесь приведен пример сбора фактов с использованием готового модуля (строка 35), однако если такового нет придется поработать шелом и [жинжа фильтрами](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_filters.html). После выполнения команды с модулем command/shell можно сохранить вывод в переменной с помощью параметра register. Это позволит использовать вывод в дальнейших задачах плейбука. Ansible позволяет использовать фильтры Jinja2 для обработки сохраненного вывода. Фильтры позволяют разбирать строки, извлекать подстроки, разделить текст на отдельные элементы и многое другое.

```yaml
- name: Run a shell command and parse the output using regex
  hosts: your_target_hosts
  tasks:
    - name: Execute the shell command
      command: your_shell_command
      register: shell_output

    - name: Process the output using regex
      debug:
        msg: "{{ shell_output.stdout | regex_search('pattern') }}"
```

При сборе фактов формируем словари-списки идентичные тем, которые описаны в IaC. То есть для последующего сравнения мы должны получить пары:

```
pg_db_exist # существующие сущности
pg_db_iac # сущности, описанные в IaC
```

На примере работы с базами данных рассмотрим порядок работы с сущностями.

```yaml
---
# Формирование идентичного словаря
- name: Parse db from output
  set_fact:
    pg_db_exist: |
      {{pg_db_exist|combine({item.key: {'access_priv': item.value.access_priv|regex_replace("/.*\n","/")|split("/")|map('regex_search', '.*=.*')|select('string')|list, 'owner': item.value.owner}})}}        
  loop: "{{ pg_exist.databases|dict2items }}"
  when: pg_exist.databases != {}

# Формирование словаря на сущностей для создания
- set_fact:
    db_dict_to_create: '{{ db_dict_to_create | combine({ item.key : item.value })}}'
  loop: "{{ pg_db_iac|dict2items }}"
  when: item.key not in pg_db_exist

# Формирование словаря на сущностей для изменения
- set_fact:
    db_dict_to_alter: '{{ db_dict_to_alter | combine({ item.key : item.value })}}'
  loop: "{{ pg_db_iac|dict2items }}"
  when: item.key in pg_db_exist and item.value != pg_db_exist[item.key]

# Формирование словаря на сущностей для удаления
- set_fact:
    db_dict_to_delete: '{{ db_dict_to_delete | combine({ item.key : item.value })}}'
  loop: "{{ pg_db_exist|dict2items }}"
  when: item.key not in pg_db_iac

# Защита от автоматического удаления
#  на проде
- name: Verify required db
  assert:
    that: db_dict_to_delete == {}
    fail_msg: "You need to delete databases. Set db_auto_delete=true"
  when: not db_auto_delete

```

```yaml
# Вывод информации о том, что было,
# что будет, чем сердце успокоится
# он поможет нам не только отлаживаться
# в рамках рутинной работы,
# но и сформировать первичный словарь/список,
# который будет внесен в IaC
- debug:
    msg: "pg_db_exist\n {{ pg_db_exist|to_yaml }}\n\n pg_db_iac\n {{ pg_db_iac|to_yaml }}\n\n/
  {%- if db_dict_to_alter != {} -%}DB will be altered\n{{ db_dict_to_alter|to_yaml }}\n\n{% endif %}
  {%- if db_dict_to_create != {} -%}DB will be created\n{{ db_dict_to_create|to_yaml }}\n\n{% endif %}
  {%- if db_dict_to_delete != {} -%}DB will be deleted\n{{ db_dict_to_delete|to_yaml }}\n\n{% endif %}"

# Сами действия с сущностями
- name: Delete databases
  include: pg_db_action.yml
  vars:
    pg_db_action: 'absent'
  loop: "{{ db_dict_to_delete|dict2items }}"
  when: db_auto_delete and db_dict_to_delete != {}

- name: Create databases
  include: pg_db_action.yml
  vars:
    pg_db_action: 'present'
    pg_db_create: true
    loop_access_priv: "{{ item.value.access_priv }}"
  loop: "{{ db_dict_to_create|dict2items }}"
  when: db_dict_to_create != {}

- name: Update databases
  include: pg_db_action.yml
  vars:
    pg_db_action: 'present'
    pg_db_update: "{{ pg_db_exist }}"
    access_priv_out: "{{ pg_db_exist[item.key].access_priv | difference(item.value.access_priv) }}"
    access_priv_in: "{{ item.value.access_priv | difference(pg_db_exist[item.key].access_priv) }}"
  loop: "{{ db_dict_to_alter|dict2items }}"
  when: db_dict_to_alter != {}
```

Формирование отладочной информации - важный этап, который поможет понять почему не отработали скрипты и где закралась ошибка. Понятное, дело чем больше кода мы покроем такой обработкой - тем больше отладочной информации получим. Рассмотрим пример.

```yaml
# Работаем с блоками ansible
- name: Block
  block:
    - name: Action {{ pg_db_action }} a database with name {{ item.key }}
      become: true
      become_user: postgres
      community.postgresql.postgresql_db:
        login_unix_socket: '{{ pg_login_unix_socket }}'
        name: "{{ item.key }}"
        state: "{{ pg_db_action }}"
        owner: "{{ item.value.owner }}"
      register: actionresult
# В случае неудачи мы не падаем
# и выходим из роли,
# а продолжаем выполнение,
# записав ошибку в список ошибок,
# который мы выводим в конце роли
  rescue:
    - set_fact:
        rescue_msgs: "{{ rescue_msgs|default([]) +
        [ 'Action '+pg_db_action+' a database with name '
        + item.key+' unsucsessful because '+actionresult.msg] }}"
```

Надеюсь, что данная статья поможет структурировать мысли и оформить вашу инфраструкту.