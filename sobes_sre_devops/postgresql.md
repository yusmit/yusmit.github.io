Вот список вопросов и ответов для собеседования администратора СУБД PostgreSQL, включая вопросы про **Patroni**, **PgBouncer** и **HAProxy** в контексте их совместного использования с PostgreSQL.  

---

### **1. Общие вопросы по PostgreSQL**  

#### **Вопрос:** Какие основные методы репликации поддерживает PostgreSQL?  
**Ответ:**  
- **Физическая репликация (Streaming Replication)** – бинарная репликация WAL-файлов.  
- **Логическая репликация** – репликация на уровне записей (публикации/подписки).  
- **Синхронная/асинхронная репликация** – в синхронной репликации мастер ждёт подтверждения записи от реплики.  

#### **Вопрос:** Как настроить Streaming Replication в PostgreSQL?  
**Ответ:**  
1. На мастере:  
   ```sql
   ALTER SYSTEM SET wal_level = 'replica';  
   ALTER SYSTEM SET max_wal_senders = 5;  
   ALTER SYSTEM SET synchronous_standby_names = 'standby1';  -- если синхронная  
   ```  
2. На реплике:  
   ```bash
   pg_basebackup -h master -D /var/lib/postgresql/standby -U replicator -P -R  
   ```  
3. В `postgresql.auto.conf` реплики:  
   ```conf
   primary_conninfo = 'host=master port=5432 user=replicator password=secret'  
   ```  

---

### **2. Вопросы по Patroni**  

#### **Вопрос:** Что такое Patroni и как он работает с PostgreSQL?  
**Ответ:**  
**Patroni** – это система управления кластеризацией PostgreSQL, обеспечивающая автоматический failover. Использует **DCS (Distributed Configuration Store)** (etcd, ZooKeeper, Consul) для координации.  

#### **Вопрос:** Как Patroni определяет, что мастер "умер"?  
**Ответ:**  
- Проверяет `ttl` ключа в DCS (если мастер не обновляет ключ, значит он "мёртв").  
- Проверяет доступность PostgreSQL (`pg_isready`).  
- Если мастер не отвечает, Patroni инициирует выборы нового мастера среди реплик.  

#### **Вопрос:** Как настроить Patroni с etcd?  
**Ответ:**  
Пример конфига `patroni.yml`:  
```yaml
scope: my_cluster  
namespace: /service/  
name: node1  

restapi:  
  listen: 0.0.0.0:8008  
  connect_address: 192.168.1.1:8008  

etcd:  
  hosts: "etcd1:2379,etcd2:2379,etcd3:2379"  

bootstrap:  
  dcs:  
    ttl: 30  
    loop_wait: 10  
    retry_timeout: 10  
    maximum_lag_on_failover: 1048576  
    postgresql:  
      use_pg_rewind: true  
      parameters:  
        wal_level: replica  
        hot_standby: "on"  
```  

---

### **3. Вопросы по PgBouncer**  

#### **Вопрос:** Зачем нужен PgBouncer в связке с PostgreSQL?  
**Ответ:**  
- **Снижает нагрузку на БД**, переиспользуя соединения (пуллинг).  
- **Уменьшает время подключения** (не создаёт новый процесс/поток в PostgreSQL).  
- Поддерживает **транзакционный**, **сессионный** и **оперативный** режимы.  

#### **Вопрос:** Как настроить PgBouncer для работы с Patroni?  
**Ответ:**  
В `pgbouncer.ini`:  
```ini
[databases]  
mydb = host=postgres-master dbname=mydb port=5432  

[pgbouncer]  
listen_port = 6432  
auth_type = md5  
auth_file = /etc/pgbouncer/userlist.txt  
pool_mode = transaction  
server_reset_query = DISCARD ALL  
```  

#### **Вопрос:** Как автоматически обновлять `pgbouncer.ini` при смене мастера?  
**Ответ:**  
Использовать **`hook` в Patroni**:  
```yaml
postgresql:  
  callbacks:  
    on_role_change: /etc/patroni/update_pgbouncer.sh  
```  
Скрипт `update_pgbouncer.sh` должен обновлять `host=` в конфиге PgBouncer и перезагружать его.  

---

### **4. Вопросы по HAProxy**  

#### **Вопрос:** Как HAProxy помогает в балансировке PostgreSQL?  
**Ответ:**  
- **Определяет мастер/реплику** через health-check (`pg_isready` или `SELECT pg_is_in_recovery()`).  
- **Балансирует read-only запросы** на реплики.  
- **Обеспечивает отказоустойчивость** (переключает трафик при падении мастера).  

#### **Вопрос:** Как настроить HAProxy для работы с Patroni?  
**Ответ:**  
Конфиг `haproxy.cfg`:  
```conf
frontend postgres  
  bind *:5000  
  default_backend postgres_master  

backend postgres_master  
  option httpchk GET /master  
  server node1 192.168.1.1:5432 check port=8008  
  server node2 192.168.1.2:5432 check port=8008  
  server node3 192.168.1.3:5432 check port=8008  

backend postgres_replicas  
  balance roundrobin  
  option httpchk GET /replica  
  server node1 192.168.1.1:5432 check port=8008  
  server node2 192.168.1.2:5432 check port=8008  
  server node3 192.168.1.3:5432 check port=8008  
```  

#### **Вопрос:** Как HAProxy определяет, кто мастер?  
**Ответ:**  
Через **HTTP-чек Patroni API** (`http://node:8008/master` возвращает 200, если нода — мастер).  

---

### **5. Комплексные вопросы**  

#### **Вопрос:** Опишите полную схему отказоустойчивого кластера PostgreSQL с Patroni, PgBouncer и HAProxy.  
**Ответ:**  
1. **PostgreSQL + Patroni**:  
   - 3 ноды (1 мастер, 2 реплики).  
   - DCS (etcd) для координации.  
2. **PgBouncer**:  
   - На каждом сервере, перенаправляет запросы на текущего мастера.  
   - Обновляется через `on_role_change` в Patroni.  
3. **HAProxy**:  
   - Балансирует read/write (мастер) и read-only (реплики).  
   - Health-check через Patroni API.  

#### **Вопрос:** Что делать, если Patroni не может выбрать нового мастера?  
**Ответ:**  
- Проверить **доступность DCS** (etcd/Consul).  
- Проверить **настройки `ttl` и `loop_wait`**.  
- Вручную назначить мастера (`patronictl failover --force`).  

---

## **🔹 Основы PostgreSQL**  

### **1. Чем отличается `VACUUM` от `VACUUM FULL`?**  
**Ответ:**  
- `VACUUM` – помечает "мертвые" строки как свободное пространство (не возвращает его ОС, но позволяет повторно использовать).  
- `VACUUM FULL` – перезаписывает таблицу, полностью освобождая место (блокирует таблицу, возвращает память ОС).  

### **2. Как работает MVCC (Multiversion Concurrency Control) в PostgreSQL?**  
**Ответ:**  
- Каждая транзакция видит **согласованный снимок данных** (snapshot).  
- При изменении строки создается **новая версия**, а старая помечается `xmin`/`xmax`.  
- `VACUUM` удаляет старые версии, которые больше не нужны.  

### **3. Что такое WAL (Write-Ahead Log) и зачем он нужен?**  
**Ответ:**  
- WAL – журнал предзаписи, в который сначала пишутся изменения, а затем в саму БД.  
- **Задачи:**  
  - Обеспечение **восстановления после сбоя** (crash recovery).  
  - Репликация (Streaming Replication).  
  - Point-in-Time Recovery (PITR).  

### **4. Как работает индексирование в PostgreSQL? Какие типы индексов знаешь?**  
**Ответ:**  
- **B-tree** – стандартный индекс (подходит для `=`, `>`, `<`, `ORDER BY`).  
- **Hash** – только для точного совпадения (`=`, быстрее B-tree, но не поддерживает сортировку).  
- **GIN** – для составных данных (JSON, массивы, полнотекстовый поиск).  
- **GiST/SP-GiST** – геоданные, специализированные поиски.  
- **BRIN** – для больших таблиц с линейной сортировкой (меньше места, но менее точен).  

---

## **🔹 Репликация и High Availability**  

### **5. В чем разница между синхронной и асинхронной репликацией?**  
**Ответ:**  
- **Синхронная (`synchronous_commit = on`)** – мастер ждёт подтверждения записи от реплики (гарантия сохранности, но медленнее).  
- **Асинхронная (`synchronous_commit = off`)** – мастер не ждёт реплику (быстрее, но возможна потеря данных при сбое).  

### **6. Как сделать Point-in-Time Recovery (PITR)?**  
**Ответ:**  
1. Включить `wal_level = replica` и архивацию:  
   ```sql
   ALTER SYSTEM SET archive_mode = on;  
   ALTER SYSTEM SET archive_command = 'cp %p /var/lib/postgresql/wal_archive/%f';  
   ```  
2. Сделать бэкап:  
   ```bash
   pg_basebackup -D /backup -Ft -X fetch  
   ```  
3. Восстановить до нужного времени:  
   ```bash
   pg_restore --recovery-target-time="2024-01-01 12:00:00"  
   ```  

### **7. Как проверить лаг реплики?**  
**Ответ:**  
```sql
SELECT pg_current_wal_lsn() - pg_last_wal_replay_lsn() AS replication_lag_bytes;  
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag_seconds;  
```  

---

## **🔹 Оптимизация и производительность**  

### **8. Как найти самые медленные запросы в PostgreSQL?**  
**Ответ:**  
1. Включить `pg_stat_statements`:  
   ```sql
   CREATE EXTENSION pg_stat_statements;  
   ```  
2. Запрос:  
   ```sql
   SELECT query, total_time, calls, mean_time  
   FROM pg_stat_statements  
   ORDER BY total_time DESC  
   LIMIT 10;  
   ```  

### **9. Как работает `EXPLAIN ANALYZE`?**  
**Ответ:**  
- `EXPLAIN` – показывает **план выполнения** запроса.  
- `ANALYZE` – **выполняет запрос** и добавляет реальные метрики (время, строки).  

Пример:  
```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE age > 30;  
```  

### **10. Как ускорить `JOIN` больших таблиц?**  
**Ответ:**  
- Добавить **индексы** на поля соединения.  
- Увеличить `work_mem` (если используется `Hash Join`).  
- Использовать `PARTITIONING` для больших таблиц.  

---

## **🔹 Безопасность и администрирование**  

### **11. Как ограничить доступ к определённым таблицам?**  
**Ответ:**  
```sql
REVOKE ALL ON TABLE users FROM PUBLIC;  
GRANT SELECT, INSERT ON TABLE users TO app_user;  
```  

### **12. Как настроить SSL в PostgreSQL?**  
**Ответ:**  
1. Сгенерировать сертификаты:  
   ```bash
   openssl req -new -x509 -nodes -out server.crt -keyout server.key  
   ```  
2. В `postgresql.conf`:  
   ```ini
   ssl = on  
   ssl_cert_file = 'server.crt'  
   ssl_key_file = 'server.key'  
   ```  

### **13. Как сделать дамп (backup) без блокировки?**  
**Ответ:**  
```bash
pg_dump --format=directory --jobs=4 --no-lock dbname -f /backup  
```  

---

## **🔹 Продвинутые вопросы**  

### **14. Что такое TOAST и как он работает?**  
**Ответ:**  
- **TOAST (The Oversized-Attribute Storage Technique)** – механизм хранения больших данных (текст, JSONB).  
- Если строка > 2 КБ, PostgreSQL сжимает её или выносит в отдельную таблицу.  

### **15. Как работает параллельное выполнение запросов?**  
**Ответ:**  
- Запрос разбивается на **worker-процессы** (настройки `max_parallel_workers`).  
- Требует `ORDER BY`, `GROUP BY`, `JOIN` больших таблиц.  

### **16. Как настроить логирование медленных запросов?**  
**Ответ:**  
В `postgresql.conf`:  
```ini
log_min_duration_statement = 1000  # логировать запросы > 1 сек  
log_statement = 'all'  
```  

---

## **🔹 Практические задачи**  

### **17. Как переименовать базу данных?**  
**Ответ:**  
```sql
ALTER DATABASE old_name RENAME TO new_name;  
```  
(Но нужно убить все соединения перед этим.)  

### **18. Как посмотреть активные подключения?**  
**Ответ:**  
```sql
SELECT * FROM pg_stat_activity;  
```  

### **19. Как убить зависший запрос?**  
**Ответ:**  
```sql
SELECT pg_cancel_backend(pid);  -- мягкое завершение  
SELECT pg_terminate_backend(pid);  -- жёсткое завершение  
```  

---

## **🔹 Итоговый сложный вопрос**  

### **20. Опиши, как настроить отказоустойчивый кластер PostgreSQL с автоматическим failover.**  
**Ответ:**  
1. **Patroni + etcd** – для автоматического переключения.  
2. **PgBouncer** – пуллинг соединений.  
3. **HAProxy** – балансировка запросов.  
4. **Monitoring (Prometheus + Grafana)** – сбор метрик.  

Примерная схема:  
```
[Client] → HAProxy → PgBouncer → [PostgreSQL (Patroni)]  
                          ↑  
                       etcd/ZooKeeper  
```  

---
# **🔹 Глубокий разбор Patroni**  

## **1. Архитектура Patroni**  
**Вопрос:** Как устроен Patroni? Какие компоненты критичны для его работы?  
**Ответ:**  
- **DCS (Distributed Configuration Store)** – etcd/Consul/ZooKeeper, хранит состояние кластера.  
- **Patroni Agent** – демон на каждой ноде, управляет PostgreSQL.  
- **REST API** – для управления и мониторинга (порт 8008).  
- **Watchdog (опционально)** – предотвращает split-brain через аппаратный сброс.  

## **2. Failover в Patroni**  
**Вопрос:** Как Patroni принимает решение о failover?  
**Ответ:**  
1. **Мастер перестаёт обновлять ключ в DCS** (TTL истекает).  
2. Реплики обнаруживают это и **начинают выборы нового мастера**.  
3. **Критерии выбора:**  
   - Наименьший лаг репликации (`maximum_lag_on_failover`).  
   - Приоритет (`priority` в конфиге).  
   - Наличие `pg_rewind` для автоматического подхвата.  

## **3. Настройка pg_rewind**  
**Вопрос:** Зачем нужен `pg_rewind` и как его настроить?  
**Ответ:**  
- **Для чего:** Позволяет бывшему мастеру **подключиться к новому мастеру без полного копирования**.  
- **Как включить:**  
  ```sql
  ALTER SYSTEM SET wal_log_hints = on;  -- обязательно!  
  ```  
  В `patroni.yml`:  
  ```yaml
  bootstrap:  
    dcs:  
      postgresql:  
        use_pg_rewind: true  
  ```  

## **4. Split-Brain и как его избежать**  
**Вопрос:** Что такое split-brain и как Patroni его предотвращает?  
**Ответ:**  
- **Split-Brain** – ситуация, когда **оба узла считают себя мастерами**.  
- **Защита:**  
  - **Watchdog** (через `/dev/watchdog` или программный).  
  - **fencing** – принудительное отключение старого мастера.  
  - **Конфигурация Patroni:**  
    ```yaml
    postgresql:  
      use_unix_socket: true  
      remove_data_directory_on_rewind_failure: true  
    ```  

## **5. Как работает автоматическое восстановление старого мастера?**  
**Ответ:**  
1. После failover Patroni проверяет, можно ли применить `pg_rewind`.  
2. Если да – переводит ноду в режим реплики.  
3. Если нет – **пересоздаёт data directory** с `pg_basebackup`.  

---

# **🔹 Глубокий разбор PgBouncer**  

## **1. Режимы пулинга PgBouncer**  
**Вопрос:** В чём разница между `session`, `transaction` и `statement` режимами?  
**Ответ:**  
| Режим       | Когда соединение возвращается в пул? | Подходит для |  
|-------------|-----------------------------------|--------------|  
| **session**  | После закрытия клиента | Долгие сессии (например, Django). |  
| **transaction** | После `COMMIT`/`ROLLBACK` | OLTP-нагрузка (лучший баланс). |  
| **statement**  | Сразу после запроса | Только для `SELECT` (не для транзакций!). |  

## **2. Как интегрировать PgBouncer с Patroni?**  
**Вопрос:** Как PgBouncer узнаёт, кто текущий мастер?  
**Ответ:**  
1. **Способ 1: Динамический конфиг**  
   - Использовать скрипт, который обновляет `pgbouncer.ini` при смене мастера:  
     ```bash
     patronictl list | grep -oP 'Leader.*\K(\d+\.\d+\.\d+\.\d+)' > /etc/pgbouncer/pgbouncer.ini  
     ```  
   - Перезагрузить PgBouncer:  
     ```bash
     pgbouncer -R -d /etc/pgbouncer/pgbouncer.ini  
     ```  

2. **Способ 2: DNS + SRV-записи**  
   - Настроить Consul/etcd для обновления DNS при failover.  

## **3. Как избежать проблем с prepared statements в PgBouncer?**  
**Вопрос:** Почему ломаются prepared statements в режиме `transaction`?  
**Ответ:**  
- **Причина:** PgBouncer не сохраняет состояние prepared statements между транзакциями.  
- **Решение:**  
  - Использовать `server_reset_query = DISCARD ALL` (сбрасывает состояние).  
  - Или перейти на `session` режим.  

## **4. Мониторинг PgBouncer**  
**Вопрос:** Какие метрики важно мониторить?  
**Ответ:**  
- **`SHOW STATS`:**  
  ```sql
  SELECT * FROM pgbouncer.stats;  
  ```  
  - **`total_requests`** – общее число запросов.  
  - **`avg_req`** – среднее время запроса.  
  - **`pool_waiting`** – клиенты в очереди.  

- **`SHOW POOLS`:**  
  ```sql
  SELECT * FROM pgbouncer.pools;  
  ```  
  - **`cl_active`** – активные соединения.  
  - **`cl_waiting`** – ждущие соединения.  

---

# **🔹 Совместная работа Patroni + PgBouncer + HAProxy**  

## **1. Полная схема взаимодействия**  
```
[Client]  
   ↓  
[HAProxy] → Определяет мастер через Patroni API (http://:8008/master)  
   ↓  
[PgBouncer] → Пуллинг соединений (transaction mode)  
   ↓  
[PostgreSQL (Patroni)] → Автоматический failover через etcd  
```  

## **2. Как настроить HAProxy для автоматического определения мастера?**  
**Конфиг HAProxy:**  
```conf
backend postgres_master  
  option httpchk GET /master  
  http-check expect status 200  
  server node1 192.168.1.1:5432 check port=8008  
  server node2 192.168.1.2:5432 check port=8008  
```  

## **3. Что делать, если Patroni сменил мастер, но PgBouncer не переключился?**  
**Решение:**  
- **Вариант 1:** Настроить `on_role_change` hook в Patroni:  
  ```yaml
  postgresql:  
    callbacks:  
      on_role_change: /etc/patroni/update_pgbouncer.sh  
  ```  
- **Вариант 2:** Использовать **консул-темплейт** для динамического обновления PgBouncer.  

---

# **🔹 Частые проблемы и решения**  

## **1. Patroni не может выбрать мастер**  
**Причины:**  
- **Нет кворума в DCS** (проверить `etcdctl cluster-health`).  
- **Нет доступа к API** (проверить `curl http://127.0.0.1:8008/health`).  

## **2. PgBouncer возвращает ошибку "pooler error"**  
**Причины:**  
- **Слишком много соединений** (увеличить `max_client_conn`).  
- **Нет свободных серверных соединений** (увеличить `pool_size`).  

## **3. Реплика отстаёт после failover**  
**Решение:**  
```sql
SELECT pg_wal_replay_resume();  -- если реплика "застряла"  
```  
Или увеличить `wal_keep_segments`.  

---

# **🔹 Итог**  
- **Patroni** – лучший инструмент для автоматического failover.  
- **PgBouncer** – обязателен для пулинга соединений.  
- **HAProxy** – балансировщик, который работает с Patroni API.  


---

#### **🔹 Основные команды PostgreSQL**  

| Команда | Описание | Пример |
|---------|----------|--------|
| `psql -U user -d dbname` | Вход в консоль PostgreSQL | `psql -U postgres -d test_db` |
| `\l` | Список всех БД | `\l` |
| `\c dbname` | Переключиться на БД | `\c test_db` |
| `\dt` | Показать таблицы | `\dt` |
| `\d+ table_name` | Описание таблицы | `\d+ users` |
| `\du` | Список пользователей | `\du` |
| `\x` | Включить/выключить расширенный вывод | `\x` |
| `CREATE DATABASE dbname;` | Создать БД | `CREATE DATABASE test_db;` |
| `DROP DATABASE dbname;` | Удалить БД | `DROP DATABASE test_db;` |
| `VACUUM (VERBOSE) table_name;` | Очистка мёртвых строк | `VACUUM VERBOSE users;` |
| `VACUUM FULL table_name;` | Полная очистка (блокирует таблицу) | `VACUUM FULL users;` |
| `REINDEX TABLE table_name;` | Перестроить индекс | `REINDEX TABLE users;` |
| `EXPLAIN ANALYZE SELECT ...` | Анализ выполнения запроса | `EXPLAIN ANALYZE SELECT * FROM users;` |
| `SELECT pg_size_pretty(pg_database_size('dbname'));` | Размер БД | `SELECT pg_size_pretty(pg_database_size('test_db'));` |
| `SELECT pg_reload_conf();` | Перезагрузить конфиг без перезапуска | `SELECT pg_reload_conf();` |

---

#### **🔹 Управление репликацией**  

| Команда | Описание | Пример |
|---------|----------|--------|
| `SELECT pg_is_in_recovery();` | Проверить, реплика ли | `SELECT pg_is_in_recovery();` |
| `SELECT pg_last_wal_receive_lsn();` | Позиция WAL на реплике | `SELECT pg_last_wal_receive_lsn();` |
| `SELECT pg_last_wal_replay_lsn();` | Последний применённый WAL | `SELECT pg_last_wal_replay_lsn();` |
| `SELECT pg_wal_replay_pause();` | Приостановить репликацию | `SELECT pg_wal_replay_pause();` |
| `SELECT pg_wal_replay_resume();` | Возобновить репликацию | `SELECT pg_wal_replay_resume();` |
| `SELECT * FROM pg_stat_replication;` | Статус репликации | `SELECT * FROM pg_stat_replication;` |

---

#### **🔹 Patroni**  

| Команда | Описание | Пример |
|---------|----------|--------|
| `patronictl list` | Статус кластера | `patronictl list` |
| `patronictl show-config` | Показать конфиг | `patronictl show-config` |
| `patronictl restart cluster` | Перезапуск кластера | `patronictl restart my_cluster` |
| `patronictl failover` | Ручной failover | `patronictl failover --candidate=node2` |
| `patronictl reinit` | Переинициализация ноды | `patronictl reinit my_cluster node3` |
| `patronictl edit-config` | Изменить конфиг | `patronictl edit-config my_cluster` |
| `patronictl query --command "SELECT ..."` | Выполнить SQL на мастере | `patronictl query -c "SELECT 1;"` |

---

#### **🔹 PgBouncer**  

| Команда | Описание | Пример |
|---------|----------|--------|
| `psql -p 6432 -U pgbouncer pgbouncer` | Вход в консоль PgBouncer | `psql -p 6432 -U pgbouncer pgbouncer` |
| `SHOW VERSION;` | Версия PgBouncer | `SHOW VERSION;` |
| `SHOW STATS;` | Статистика запросов | `SHOW STATS;` |
| `SHOW POOLS;` | Статус пулов | `SHOW POOLS;` |
| `SHOW CLIENTS;` | Активные клиенты | `SHOW CLIENTS;` |
| `SHOW SERVERS;` | Серверные соединения | `SHOW SERVERS;` |
| `RELOAD;` | Перезагрузить конфиг | `RELOAD;` |
| `PAUSE;` | Приостановить новые соединения | `PAUSE;` |
| `RESUME;` | Возобновить работу | `RESUME;` |
| `KILL client_id;` | Завершить соединение | `KILL 123;` |

---

#### **🔹 HAProxy**  

| Команда | Описание | Пример |
|---------|----------|--------|
| `systemctl restart haproxy` | Перезапуск HAProxy | `systemctl restart haproxy` |
| `haproxy -c -f /etc/haproxy/haproxy.cfg` | Проверка конфига | `haproxy -c -f /etc/haproxy/haproxy.cfg` |
| `echo "show stat" | socat /run/haproxy/admin.sock stdio` | Статистика HAProxy | `echo "show stat" | socat /run/haproxy/admin.sock stdio` |
| `echo "show info" | socat /run/haproxy/admin.sock stdio` | Информация о процессе | `echo "show info" | socat /run/haproxy/admin.sock stdio` |

---

### **🔹 Полезные команды Linux для админа PostgreSQL**  

| Команда | Описание | Пример |
|---------|----------|--------|
| `pg_isready -h host -p port` | Проверить доступность PostgreSQL | `pg_isready -h 127.0.0.1 -p 5432` |
| `pg_dump -U user -d dbname > backup.sql` | Дамп БД | `pg_dump -U postgres -d test_db > backup.sql` |
| `pg_restore -U user -d dbname backup.dump` | Восстановление БД | `pg_restore -U postgres -d test_db backup.dump` |
| `du -sh /var/lib/postgresql/` | Размер каталога PostgreSQL | `du -sh /var/lib/postgresql/` |
| `tail -f /var/log/postgresql/postgresql-14-main.log` | Просмотр логов | `tail -f /var/log/postgresql/postgresql-14-main.log` |
| `netstat -tulnp | grep postgres` | Проверить открытые порты PostgreSQL | `netstat -tulnp | grep postgres` |

---

### **Шпаргалка по основным параметрам PostgreSQL, Patroni, PgBouncer и HAProxy**

#### **🔹 PostgreSQL: ключевые параметры конфигурации (postgresql.conf)**

| Параметр | Описание | Рекомендуемое значение |
|----------|----------|------------------------|
| `shared_buffers` | Объем памяти для кэширования данных | 25% от RAM |
| `work_mem` | Память для операций сортировки и хэшей | 4-32 MB |
| `maintenance_work_mem` | Память для операций обслуживания (VACUUM и др.) | 64-512 MB |
| `wal_level` | Уровень журналирования (WAL) | `replica` (для репликации) |
| `max_wal_senders` | Максимальное число процессов отправки WAL | 5-10 |
| `synchronous_commit` | Синхронность записи WAL | `on` (гарантия сохранности) |
| `checkpoint_timeout` | Интервал между контрольными точками | `5min` |
| `max_connections` | Максимальное число подключений | 100-500 (зависит от RAM) |
| `random_page_cost` | Стоимость случайного доступа к странице | 1.1 (SSD), 4.0 (HDD) |
| `effective_cache_size` | Оценка размера кэша ОС | 50-75% от RAM |

---

#### **🔹 Patroni: ключевые параметры (patroni.yml)**

| Параметр | Описание | Пример значения |
|----------|----------|----------------|
| `scope` | Имя кластера | `my_cluster` |
| `name` | Уникальное имя ноды | `node1` |
| `restapi.listen` | Адрес и порт REST API | `0.0.0.0:8008` |
| `etcd.hosts` | Адреса DCS (etcd/Consul) | `etcd1:2379, etcd2:2379` |
| `bootstrap.dcs.ttl` | TTL ключа мастера (сек) | `30` |
| `bootstrap.dcs.loop_wait` | Интервал проверки состояния (сек) | `10` |
| `postgresql.use_pg_rewind` | Использовать pg_rewind для восстановления | `true` |
| `postgresql.parameters.wal_level` | Наследуется из postgresql.conf | `replica` |
| `postgresql.callbacks.on_role_change` | Скрипт при смене роли | `/path/to/script.sh` |

---

#### **🔹 PgBouncer: ключевые параметры (pgbouncer.ini)**

| Параметр | Описание | Пример значения |
|----------|----------|----------------|
| `listen_port` | Порт для подключений | `6432` |
| `pool_mode` | Режим пулинга | `transaction` |
| `max_client_conn` | Максимальное число клиентских подключений | `1000` |
| `default_pool_size` | Размер пула на одну БД | `20` |
| `reserve_pool_size` | Резервный пул для пиковой нагрузки | `5` |
| `server_reset_query` | Запрос при возврате соединения в пул | `DISCARD ALL` |
| `auth_type` | Метод аутентификации | `md5` |
| `auth_file` | Файл с пользователями | `/etc/pgbouncer/userlist.txt` |
| `log_connections` | Логировать подключения | `1` (вкл) |

---

#### **🔹 HAProxy: ключевые параметры (haproxy.cfg)**

| Параметр | Описание | Пример значения |
|----------|----------|----------------|
| `bind *:5000` | Порт для подключений | `*:5432` (для PostgreSQL) |
| `mode tcp` | Режим работы (TCP/HTTP) | `tcp` |
| `balance roundrobin` | Алгоритм балансировки | `roundrobin` |
| `option httpchk` | HTTP-чек для определения мастера | `GET /master` |
| `server` | Определение серверов бэкенда | `node1 192.168.1.1:5432 check port=8008` |
| `timeout connect` | Таймаут подключения (мс) | `5000` |
| `timeout server` | Таймаут ответа сервера (мс) | `15000` |
| `retries` | Число попыток переподключения | `3` |

---

### **🔹 Полезные команды для проверки параметров**

#### **PostgreSQL**
```sql
-- Показать текущие параметры
SELECT name, setting, unit FROM pg_settings WHERE name IN ('shared_buffers', 'work_mem', 'wal_level');

-- Изменить параметр динамически
ALTER SYSTEM SET work_mem = '16MB';
SELECT pg_reload_conf();
```

#### **Patroni**
```bash
# Проверить текущий конфиг
patronictl show-config

# Применить изменения
patronictl edit-config
```

#### **PgBouncer**
```sql
-- В консоли PgBouncer (psql -p 6432 pgbouncer)
SHOW CONFIG;
```

#### **HAProxy**
```bash
# Проверить конфиг
haproxy -c -f /etc/haproxy/haproxy.cfg

# Статистика в реальном времени
echo "show stat" | socat /var/run/haproxy/admin.sock stdio
```

---

### **Шпаргалка по формату pg_hba.conf в PostgreSQL**

Файл `pg_hba.conf` (Host-Based Authentication) управляет аутентификацией клиентов PostgreSQL. Каждая строка определяет правило доступа.

---

## **🔹 Формат записи в pg_hba.conf**
```
# TYPE  DATABASE        USER           ADDRESS                 METHOD       [OPTIONS]
```

| Поле       | Описание                                                                 | Пример значения                |
|------------|--------------------------------------------------------------------------|--------------------------------|
| **TYPE**   | Тип подключения: `local` (сокет), `host` (TCP/IP), `hostssl` (SSL), `hostnossl` (без SSL) | `host`, `local`, `hostssl`     |
| **DATABASE**| Имя БД (`all`, `sameuser`, `samerole`, `replication` или конкретное имя) | `all`, `mydb`, `replication`   |
| **USER**   | Имя пользователя (`all`, `+группа` для ролей)                            | `postgres`, `all`, `+admin`    |
| **ADDRESS**| IP-адрес/маска (`0.0.0.0/0` — все адреса)                               | `192.168.1.0/24`, `::1/128`    |
| **METHOD** | Метод аутентификации (см. таблицу ниже)                                 | `md5`, `scram-sha-256`, `trust`|
| **OPTIONS**| Доп. параметры (`clientcert=1` для проверки сертификата)                 | `clientcert=1`                 |

---

## **🔹 Методы аутентификации (METHOD)**

| Метод                | Описание                                                                 | Безопасность |
|----------------------|--------------------------------------------------------------------------|--------------|
| `trust`             | Безусловный доступ (без пароля)                                         | ❌ Опасен     |
| `reject`            | Запрет доступа                                                          | ✅            |
| `md5`               | Аутентификация по хешу MD5 (устаревший, но широко используется)         | ⚠️ Условно    |
| `scram-sha-256`     | Современный безопасный метод (рекомендуется)                            | ✅            |
| `password`          | Пароль передается в открытом виде                                       | ❌            |
| `peer`              | Проверка имени ОС-пользователя (только для `local`)                     | ✅ (локально) |
| `cert`              | Аутентификация по SSL-сертификату (`clientcert=1`)                      | ✅            |
| `ident`             | Проверка через ident-сервер (устаревший)                                | ⚠️            |
| `pam`               | Аутентификация через PAM                                                | ✅            |
| `ldap`              | Аутентификация через LDAP                                               | ✅            |

---

## **🔹 Примеры правил**

### 1. Разрешить доступ всем пользователям с локального хоста (через сокет)
```
local   all             all                                     peer
```

### 2. Разрешить доступ по паролю (SCRAM-SHA-256) для пользователя `admin` к БД `mydb` с определенного IP
```
host    mydb           admin            192.168.1.100/32       scram-sha-256
```

### 3. Запретить доступ из внешней сети
```
host    all             all             0.0.0.0/0               reject
```

### 4. Разрешить репликацию с определенного IP
```
host    replication     replicator       192.168.1.50/32        scram-sha-256
```

### 5. SSL + проверка сертификата
```
hostssl all             all             0.0.0.0/0               cert clientcert=1
```

---

## **🔹 Важные замечания**

1. **Порядок правил важен** — PostgreSQL использует первое совпавшее правило.
2. **Изменения вступают в силу после перезагрузки**:
   ```bash
   pg_ctl reload
   # или через SQL
   SELECT pg_reload_conf();
   ```
3. **Для репликации** обязательно нужно правило с `TYPE=host` и `DATABASE=replication`.
4. **Не используйте `trust` для внешних подключений** — это дыра в безопасности!
5. **Лучшие практики**:
   - Используйте `scram-sha-256` вместо `md5`.
   - Для администраторов — `peer` (локально) или `cert` (SSL).
   - Для приложений — ограничьте IP-адреса.

---

## **🔹 Как проверить текущие правила?**
```sql
SELECT * FROM pg_hba_file_rules;
```
---

### **🔹 Схема (Schema) в PostgreSQL**

Схема — это **логический контейнер** внутри базы данных, который группирует таблицы, индексы, функции и другие объекты.  
 
1. **Организация** – помогает структурировать БД (например, отдельно `hr` для кадров, `finance` для финансов).  
2. **Безопасность** – можно назначать права доступа на уровне схемы.  
3. **Изоляция** – несколько схем могут содержать таблицы с одинаковыми именами (`hr.employees` и `finance.employees`).  

#### **Примеры**  
```sql
-- Создать схему
CREATE SCHEMA hr;

-- Создать таблицу в схеме
CREATE TABLE hr.employees (id SERIAL, name TEXT);

-- Дать доступ пользователю
GRANT USAGE ON SCHEMA hr TO manager;
```

#### **Особенности**  
- По умолчанию используется схема `public`.  
- Поиск объектов идет по пути (`search_path`):  
  ```sql
  SHOW search_path;  -- Покажет текущий путь (например, "$user, public")
  ```

---

### **🔹 Индекс (Index) в PostgreSQL**

Индекс — это **дополнительная структура данных**, которая ускоряет поиск и сортировку в таблице (аналогично оглавлению в книге).  

- Ускоряет `SELECT`, `WHERE`, `JOIN`, `ORDER BY`.  
- Замедляет `INSERT`/`UPDATE`/`DELETE` (так как нужно обновлять сам индекс).  

#### **Типы индексов**  
| Тип       | Когда использовать                     | Пример |
|-----------|---------------------------------------|--------|
| **B-tree** | Стандартный индекс (для `=`, `>`, `<`, `BETWEEN`). | `CREATE INDEX idx_name ON users (name);` |
| **Hash**   | Только для точного совпадения (`=`). | `CREATE INDEX idx_id ON users USING HASH (id);` |
| **GIN**    | Для составных данных (массивы, JSON, полнотекстовый поиск). | `CREATE INDEX idx_tags ON posts USING GIN (tags);` |
| **GiST**   | Для геоданных и сложных типов (например, `geometry`). | `CREATE INDEX idx_location ON shops USING GiST (location);` |
| **BRIN**   | Для очень больших таблиц с линейной сортировкой (экономит место). | `CREATE INDEX idx_time ON logs USING BRIN (created_at);` |

#### **Пример создания**  
```sql
-- Одноколоночный индекс
CREATE INDEX idx_email ON users (email);

-- Составной индекс (для условий с несколькими полями)
CREATE INDEX idx_name_age ON users (last_name, age);

-- Уникальный индекс
CREATE UNIQUE INDEX idx_unique_username ON users (username);
```

#### **Когда индексы не работают?**  
1. Если в запросе используется **функция** (например, `WHERE LOWER(name) = 'alice'`).  
   **Решение:** Создать индекс по выражению:  
   ```sql
   CREATE INDEX idx_lower_name ON users (LOWER(name));
   ```  
2. Если выбирается **большая часть таблицы** (PostgreSQL решит, что сканировать таблицу дешевле).  

#### **Как проверить использование индекса?**  
```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';
```
Вывод покажет, был ли использован индекс (например, `Index Scan using idx_email on users`).

---

### **🔹 Сравнение схемы и индекса**
| Характеристика       | Схема (Schema)                          | Индекс (Index)                     |
|----------------------|-----------------------------------------|------------------------------------|
| **Что содержит?**     | Таблицы, индексы, функции, представления | Указатели на данные в таблицах     |
| **Основная цель**     | Логическая организация БД               | Ускорение запросов                 |
| **Влияние на скорость** | Нет                                     | Ускоряет `SELECT`, замедляет `INSERT` |
| **Пример**           | `CREATE SCHEMA hr;`                     | `CREATE INDEX idx_name ON users;`  |

---

### **🔹 Практические советы**  
1. **Для схем**:  
   - Используйте схемы, если в БД > 50 таблиц.  
   - Назначайте права на уровне схем (`GRANT SELECT ON ALL TABLES IN SCHEMA hr TO analyst;`).  
2. **Для индексов**:  
   - Индексируйте только часто используемые в `WHERE`/`JOIN` столбцы.  
   - Для текста используйте `GIN` с `pg_trgm` для поиска по подстроке:  
     ```sql
     CREATE EXTENSION pg_trgm;
     CREATE INDEX idx_name_search ON users USING GIN (name gin_trgm_ops);
     ```  
