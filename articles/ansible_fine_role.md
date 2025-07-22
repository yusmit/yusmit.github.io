# Как запилить годную ролюху в Ansible
Гайдов и практик по написанию ролей - куча. Все их можно легко найти - приводить их не буду. В данной статье я попытаюсь структурировать все мои шишки, полученные в рамках написания и эксплуатации ролей Ansible и рассказать о том как ~~легко~~ написать роль ~~без регистрации и СМС~~.

## Содержание

* [Используй теги](#%D1%82%D0%B5%D0%B3%D0%B8).
* [Безопасная работа с секретами в Ansible: no\_log: true.](#no_log)
* [Проверка пререквизитов и корректности переменных.](#%D0%BF%D1%80%D0%B5%D1%80%D0%B5%D0%BA%D0%B2%D0%B8%D0%B7%D0%B8%D1%82%D1%8B)
* [Обработка ошибок в Ansible (block/rescue).](#%D0%BE%D0%B1%D1%80%D0%B0%D0%B1%D0%BE%D1%82%D0%BA%D0%B0_%D0%BE%D1%88%D0%B8%D0%B1%D0%BE%D0%BA)
* [Неидемпотентная роль - выстрел в ногу.](#%D0%B8%D0%B4%D0%B5%D0%BC%D0%BF%D0%BE%D1%82%D0%B5%D0%BD%D1%82%D0%BD%D0%BE%D1%81%D1%82%D1%8C)
* [Не ленись - пиши модули.](#%D0%BC%D0%BE%D0%B4%D1%83%D0%BB%D0%B8)
* [Помни о хендлерах.](#handlers)
* [Хочешь мира - пиши документацию.](#readme)
* [Тестируй все изменения](#test).

## Алгоритм написания роли в Ansible

Хорошая роль Ansible должна быть **модульной, переиспользуемой и хорошо документированной**. Вот пошаговый алгоритм создания роли:

### Определение функционала роли

Проанализируй свою потребность и ответь на вопрос "Что должна делать роль"?

* Устанавливать и настраивать один сервис (Nginx, PostgreSQL, Docker и т. д.).
* Настраивать системные параметры (например, `sysctl`, `limits.conf`).
* Разворачивать приложение (например, WordPress, Prometheus).
* Конфигурировать отдельный сервис.
* Выдавать права и пр. и др.

При определении функционала не стоит смешивать несколько несвязанных задач (например, установка Nginx + настройка БД).

### Нюансы написание ролюхи.

* Разбивать сложные задачи на подзадачи (`include_tasks`) и [использовать **теги**](#%D1%82%D0%B5%D0%B3%D0%B8) **(**`tags`**)** для выборочного запуска.
* [Безопасная работа с секретами в Ansible: no\_log: true.](#no_log)
* [Проверка пререквизитов и корректности переменных.](#%D0%BF%D1%80%D0%B5%D1%80%D0%B5%D0%BA%D0%B2%D0%B8%D0%B7%D0%B8%D1%82%D1%8B)
* Чтобы избежать конфликтов и повысить читаемость кода, **все переменные** (включая временные) должны начинаться с **префикса имени роли**.
* [Обработка ошибок в Ansible (block/rescue).](#%D0%BE%D0%B1%D1%80%D0%B0%D0%B1%D0%BE%D1%82%D0%BA%D0%B0_%D0%BE%D1%88%D0%B8%D0%B1%D0%BE%D0%BA)
* [Неидемпотентная роль - выстрел в ногу.](#%D0%B8%D0%B4%D0%B5%D0%BC%D0%BF%D0%BE%D1%82%D0%B5%D0%BD%D1%82%D0%BD%D0%BE%D1%81%D1%82%D1%8C)
* [Не ленись - пиши модули.](#%D0%BC%D0%BE%D0%B4%D1%83%D0%BB%D0%B8)
* [Помни о хендлерах.](#handlers)
* [Хочешь мира - пиши документацию.](#readme)
* [Тестируй все изменения](#test).

```yaml
---
- name: Check variables
  include_tasks:
    file: pre_task.yml
    apply:
      tags: always
  tags: always

- name: Install Nginx
  apt:
    name: nginx
    state: present
  tags: install

- name: Copy Nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: restart nginx
  tags: config

- name: Flush handlers at end
  meta: flush_handlers
  tags: always
```

### Разделение функционала по тегам

Теги - замечательный инструмент.

* **Теги** дают контроль над этапами: `install|update|config|certs|remove`.
* **Обеспечивают четкое разделение задач**, что упрощает поддержку.
* **Обеспечивают безопасность**: Тег `remove` не конфликтует с `install`.
* **Гибкость**: Можно комбинировать (`deploy = install + config + certs`).

Примерная структура тегов:

| Тег | Действие |
| --- | --- |
| `install` | Установка ПО |
| `update` | Обновление (если версия изменилась) |
| `config` | Настройка конфигов |
| `certs` | Обновление TLS/SSL |
| `remove, never` | Полное удаление ПО |

Особое внимание стоит уделить применению (apply) тегов, тут важно учесть, что роль должна проходить в *dryrun (check\_mode)* перед установкой полностью, а после установки, с использованием любого из тегов. Если лениво подписывать каждую таску тегом можно применить вот такую конструкцию:

```yaml
- name: Check variables
  include_tasks:
    file: pre_task.yml
    apply:
      tags: check # применяет тег ко всем таскам в файле
  tags: check
```

**Иногда получается вот такая структура:**

```bash
my_app/
├── tasks/
│   ├── install.yml
│   ├── update.yml
│   ├── config.yml
│   ├── certs.yml
│   ├── remove.yml
│   └── main.yml  # импорт всех задач с тегами
```

### Безопасная работа с секретами в Ansible: no\_log: true

Для защиты чувствительных данных (пароли, ключи, токены) в логах Ansible нужно использовать `no_log: true`. Это предотвращает запись секретов в:

* Консольный вывод
* Файлы логов
* Системы мониторинга

#### Правильная реализация

```yaml
# Для отдельных задач с секретами
- name: Set database password
  ansible.builtin.lineinfile:
    path: /etc/app.conf
    line: "DB_PASSWORD={{ db_password }}"
  no_log: true  # ← Важно!

# Для целых блоков
- name: Secrets handling block
  block:
    - name: Create API key
      ansible.builtin.command: generate-key.sh
      register: api_key_result

    - name: Deploy key to vault
      ansible.builtin.uri:
        url: "https://vault.example.com"
        body: "{{ api_key_result.stdout }}"
  no_log: true  # Скрывает ВЕСЬ вывод блок

# Когда переменная содержит секрет:
- name: Configure secret token
  ansible.builtin.template:
    src: token.j2
    dest: /etc/secrets/token
  vars:
    secret_token: "{{ vaulted_token }}"  # Переменная из vault
  no_log: true

# Для результатов выполненных задач (register):
- name: Get sensitive data
  ansible.builtin.command: decrypt.sh
  register: decrypted_data
  no_log: true

- name: Use secured data
  debug:
    msg: "Data processed successfully"
  when: decrypted_data.rc == 0

# Для модулей с секретами (например uri):
- name: Auth request
  ansible.builtin.uri:
    url: "https://api.example.com/login"
    body:
      username: admin
      password: "{{ vaulted_pass }}"
  no_log: true

# Комбинируйте с Ansible Vault:
vars:
  db_password: !vault |
    $ANSIBLE_VAULT;1.1;AES256
    643865...

# Комбинируйте с Hashicorp Vault
vars:
  msql_password: "{{ lookup('community.hashi_vault.hashi_vault', 'secret=secret/data/hello token=my_vault_token url=http://myvault_url:8200') }}"
```

### Как проверить защиту?

1. Запустите плейбук с `-vvv`
2. Убедитесь что в выводе нет:

   ```
   - "Changed": true, "DB_PASSWORD": "s3cr3t"
   + "output": "********"
   ```

### Проверка пререквизитов в pre\_tasks

Чтобы гарантировать корректную работу роли, все требования должны проверяться **до** её выполнения. Для этого используем `pre_tasks` в плейбуке tasks/main.yml.

```yaml
---
- name: "1. Check OS compatibility (Ubuntu/Debian)"
  ansible.builtin.fail:
    msg: "Unsupported OS. Required: Ubuntu/Debian"
  when: ansible_facts['distribution'] not in ['Ubuntu', 'Debian']

- name: "2. Verify free disk space > 1GB"
  ansible.builtin.command: df -BG /
  register: disk_space
  changed_when: false
  failed_when:
    - disk_space.rc != 0
    or (disk_space.stdout | regex_search('\\d+G')) | int < 1

# Проверка доступности портов
- name: "3. Check ports 80/443 are available"
  ansible.builtin.wait_for:
    port: "{{ item }}"
    state: stopped
    timeout: 1
  loop: [80, 443]
  ignore_errors: true
  register: ports_check
  failed_when: ports_check.results | selectattr('failed') | list | length > 0

# Зависимости (установка)
- name: "Ensure curl is installed"
  apt:
    name: curl
    state: present

# Блокирующие проверки
- name: "Fail if Docker not found"
  ansible.builtin.command: docker --version
  register: docker_check
  failed_when: docker_check.rc != 0

# Неблокирующие предупреждения
- name: "Warn about low RAM"
  ansible.builtin.debug:
    msg: "Recommended: 4GB RAM (found {{ ansible_memtotal_mb }}MB)"
  changed_when: false
  when: ansible_memtotal_mb < 4096

# использование модуля assert

- name: After version 2.7 both O(msg) and O(fail_msg) can customize failing assertion message
  ansible.builtin.assert:
    that:
      - my_param <= 100
      - my_param >= 0
    fail_msg: "'my_param' must be between 0 and 100"
    success_msg: "'my_param' is between 0 and 100"
```

#### Нюансы реализации:

1. Используйте параметры модулей `fail` и `assert` для вывода понятных сообщений об ошибках. Старайтесь придерживаться одного из этих модулей в роли - будет красивее смотреться и быстрее дебажить.
2. Все проверки должны иметь `changed_when: false`.
3. Для "тяжелых" проверок добавляйте `run_once: true`.
4. Укажите все проверки в [`README.md`](http://README.md)

**Итог:**

* Роль выполняется только при соблюдении всех условий
* Четкие сообщения об ошибках
* Нет "тихих" сбоев на этапе выполнения

### Обработка ошибок в Ansible (block/rescue)

Чтобы роль **не падала** при некритических ошибках (например, если сервис временно недоступен), используем связку `block`**+**`rescue`.
Это аналог `try/catch` в других языках.

Допустим, мы **копируем конфиг** и **перезапускаем сервис**, но хотим:

1. **Продолжить выполнение**, если конфиг скопировался, но сервис не перезапустился.
2. **Записать ошибку** в лог, но не прерывать всю роль.

```yaml
---
- name: Critical operations block
  block:
    - name: Task 1 - Copy config
      ansible.builtin.template:
        src: config.j2
        dest: /etc/app/config.conf
      register: taskresult
      notify: restart app

    - name: Task 2 - Validate config
      ansible.builtin.command: app --validate
      register: taskresult
      changed_when: false

  rescue:
    - name: Add error to list
      set_fact:
        role_errors: "{{ role_errors | default([]) + [{
          'task': ansible_failed_task.name,
          'error': ansible_failed_result.msg
        }] }}"

    - name: Continue execution
      meta: continue

- name: Print all errors (if any)
  ansible.builtin.debug:
    var: role_errors
  failed_when: role_errors | length > 0
```

#### Как это работает:

1. **Основной блок (block)**

   * Каждая задача регистрирует результат в `taskresult`
   * При ошибке - переход в `rescue`
2. **Обработка ошибок (rescue)**

   * Добавляем форматированную ошибку в список.
   * Продолжаем выполнение (`meta: continue`)
3. **Финальный отчет**

   * После всех задач выводим список ошибок (если они есть) и если они есть выдаем ошибку.

#### Пример вывода при ошибках:

```json
"role_errors": [
  {
    "task": "Task 2 - Validate config",
    "error": "Command 'app --validate' returned 1: ERROR: Invalid config"
  }
]
```

#### Преимущества такого подхода:

1. **Полная трассировка ошибок** - видно какие именно задачи упали
2. **Аккуратный вывод** - все ошибки собираются в одном месте
3. **Гибкость** - можно добавить дополнительные поля (время, хост и т.д.)
4. **Скорость исправления и отладки** - можно разом собрать все ошибки и попытаться их исправить.

### Неидемпотентная роль - выстрел в ногу

Идемпотентность — ключевое требование к Ansible-ролям. Это означает, что:
✔ Повторный запуск роли **не должен делать лишних изменений**
✔ Система после каждого запуска **должна приходить в одинаковое состояние**

Как добиться идемпотентности?

Большинство модулей Ansible **уже идемпотентны** (apt, yum, template и др.). Но иногда (**Для командных модулей** (`command`, `shell`, `raw`) всегда) нужно ручное управление через:

* changed\_when: false # Всегда показывает "ok" (даже если что-то делал)
* changed\_when: условие # Кастомное условие для "changed"

Примеры использования.

```yaml
# Команды, которые всегда меняют состояние
- name: Check service stataus
  command: systemctl is-active nginx
  register: nginx_status
  changed_when: false  # ← Не влияет на систему, поэтому "ok"

- name: Force reload (если нужно)
  command: systemctl reload nginx
  when: nginx_status.stdout != "active"

# Кастомная проверка изменений
- name: Apply config if changed
  template:
    src: app.conf.j2
    dest: /etc/app.conf
  register: config_result
  changed_when: config_result.changed  # Стандартное поведение (можно опустить)

# Условный "changed" для скриптов
- name: Run database migration
  command: /opt/app/migrate.py
  register: migration_result
  changed_when:
    - "'Success' in migration_result.stdout"  # ← "changed" только при успехе
    - migration_result.rc == 0

# Для задач с always_run
- name: Validate config (выполняется всегда)
  command: validate_config.sh
  changed_when: false
  check_mode: no
  always_run: yes

# Для обработчиков (handlers) handlers/main.yml:
- name: migrate app
  command: /opt/app/migrate.py
  changed_when: false  # ← Чтобы не показывал "changed" при каждом вызове

# Для сложных проверок: Используйте failed_when вместе с changed_when:
- name: Check license
  command: check_license.sh
  register: license_check
  changed_when: false
  failed_when:
    - license_check.rc != 0
    - "'Expired' in license_check.stdout"
```

#### Как проверить идемпотентность

* Запустите роль дважды
* ansible-playbook playbook.yml && ansible-playbook playbook.yml
* Ищите задачи с `changed!=0` при повторном запуске — это точки неидемпотентности.

### Вынос сложной логики в кастомные модули Ansible

Когда в роли появляются **сложные проверки, вычисления или работа с API**, их лучше выносить в **отдельные модули**. Это:

* Упрощает поддержку кода
* Повышает производительность (модули выполняются на Python)
* Позволяет переиспользовать логику

Когда нужно выносить логику в модуль?

1. **Появление большого количества вложенных циклов** - циклы в ansible реализованы, скажем прямо, не очень удобно и мой болезненный опыт показал, что если невозможно избежать большого количества вложенных циклов, то целесообразно их упаковать в модуль - так будет удобнее поддерживать и быстрее выполняться роль
2. **Сложная валидация -** Парсинг JSON/XML, Проверка сертификатов/подписей
3. **Работа с API -** Запросы к Kubernetes, AWS, Database
4. **Громоздкие вычисления -** Обработка больших данных, Математические операции
5. **Специфичная логика -** Генерация конфигов со сложными условиями

Создаем кастомный модуль

```bash
roles/
└── my_role/
    ├── library/          # Сюда кладем модули
    │   └── cert_validator.py
    ├── tasks/
    │   └── main.yml
    └── defaults/
        └── main.yml
```

Пример модуля (`library/cert_validator.py`):

```python
#!/usr/bin/python3

# Используйте AnsibleModule
from ansible.module_utils.basic import AnsibleModule
import OpenSSL.crypto
from datetime import datetime

# Пишите документацию к модулю
DOCUMENTATION = r'''
module: cert_validator
description: Check SSL certificate expiry
options:
  cert_path:
    description: Path to PEM certificate
    required: true
    type: str
'''

def check_cert(cert_path):
    # Обрабатывайте ошибки
    try:
        with open(cert_path, 'rb') as f:
            cert = OpenSSL.crypto.load_certificate(
                OpenSSL.crypto.FILETYPE_PEM, f.read()
            )
        expiry_date = datetime.strptime(
            cert.get_notAfter().decode('utf-8'), '%Y%m%d%H%M%SZ'
        )
        return {
            'valid': datetime.now() < expiry_date,
            'expiry_date': expiry_date.isoformat()
        }
    except Exception as e:
        return {'error': str(e)}

def main():
    module = AnsibleModule(
        argument_spec=dict(
            cert_path=dict(type='str', required=True)
        )
    )
    result = check_cert(module.params['cert_path'])
    if 'error' in result:
        module.fail_json(msg=result['error'])
    module.exit_json(**result)

if __name__ == '__main__':
    main()
```

### Используем модуль в роли

```yaml
---
- name: Validate SSL certificate
  cert_validator:
    cert_path: "{{ nginx__ssl_cert }}"
  register: cert_check

- name: Fail if cert invalid
  ansible.builtin.fail:
    msg: "Certificate expires on {{ cert_check.expiry_date }}"
  when: not cert_check.valid
```

### Преимущества подхода

* **Производительность -** Модуль выполняет кучу команд со сложной логикой (в отличие от набора последовательных set\_fact и циклов ansible).
* **Безопасность -** Нет риска инъекций (в отличие от сырых команд).
* **Идемпотентность -** Встроенная поддержка `changed`/`failed` состояний.
* **Тестируемость -** Модуль можно проверить отдельно от роли.

### Советы по разработке модулей

* Добавляйте документацию
* Обрабатывайте ошибки
* Тестируйте локально

  ```bash
  python library/cert_validator.py '{"cert_path":"/tmp/cert.pem"}'
  ```

#### Альтернативы для простых случаев

Если модуль — это overkill, используйте:

1. `ansible.builtin.script`

   ```yaml
   - name: Run validation script
     ansible.builtin.script:
       cmd: scripts/validate_cert.sh {{ cert_path }}
   ```
2. **Фильтры Jinja2**

   ```yaml
   - set_fact:
       is_valid: "{{ cert_data | regex_search('VALID') }}"
   ```

### Обработчики (handlers/main.yml)

**Хендлеры – это "отложенные задачи"**, которые:

* **Срабатывают только при изменениях** (если был `notify`).
* **Выполняются один раз**, даже если их вызвали несколько раз.
* **Помогают избежать лишних действий** (например, множественных перезапусков сервиса).

#### Когда использовать хендлеры?

* **Перезапуск сервисов** после изменения конфигов.
* **Перечитывание конфигурации** после изменения конфигов.
* **Перезагрузка демонов** после настройки параметров.
* **Отправка уведомлений** (например, оповещение в почту при изменениях).

И всегда добавь в конец роли принудительный вызов хендлеров. И конечно появляется логичный вопрос - а почему, зачем? Ответ: потому, что все хендлеры по умолчанию срабатывают не по завершению роли, а по завершению плея. И если в одном плее указано несколько ролей есть секция tasks и post\_task, и они завязаны на результат выполнения конкретной роли, то тогда обязательно надо принудительно хендлеры в конце этой роли вызывать. ( Огромное спасибо за информацию [FactorT](https://habr.com/ru/users/FactorT/) )

```yaml
- name: Flush handlers at end
  meta: flush_handlers
  tags: always
```

### Документация Ansible-роли (README.md)

Хорошая документация помогает другим братьям-администраторам быстро понять, как использовать роль (и не приставать к тебе с глупыми вопросами, отвлекая от размышления о вечном). Вот структура `README.md`:  

* Название роли
* Ключевые переменные
* Теги
* Пререквизиты
* Сценарии использования
* Примеры переменных в defaults/main.yml
* Советы по использованию

### Тестирование роли

Тестирование ролей Ansible должно проводиться как в кластерной среде, так и в отдельной (standalone) конфигурации. Это необходимо для обеспечения корректной работы ролей в различных условиях и на всех поддерживаемых операционных системах и конфигурациях.
Если в функционале ролей происходят изменения, перед слиянием (мержем) необходимо обновить тесты с использованием фреймворка Molecule и **явно протестировать новый функционал**. Это позволит гарантировать, что новые изменения не нарушают существующую функциональность и что роли продолжают работать корректно. Чувствую твой правомерный гнев - "зачем тестировать, ведь ~~у меня локально~~ на моей продуктивной инфраструктуре работает", но если роль ты пишешь не только для себя, то надо протестировать различные варианты.

Надеюсь, что вышеизложенное поможет тебе в трудовыебуднях.