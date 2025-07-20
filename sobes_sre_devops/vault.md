### **Подготовка к собеседованию на Senior SRE: углублённый разбор Hashicorp Vault**  

Hashicorp Vault — критически важный инструмент в инфраструктуре, особенно для SRE, отвечающих за безопасность, надёжность и автоматизацию. Давай разберём ключевые аспекты, которые нужно знать, включая архитектуру, безопасность, интеграции и практическое применение.  

---  

## **1. Основные концепции и архитектура Vault**  

### **Что такое Vault?**  
Vault — это система для управления секретами (пароли, API-ключи, сертификаты) и защиты данных через:  
- **Безопасное хранение** (шифрование на rest и in transit).  
- **Динамические секреты** (временные учётные данные для БД, облачных провайдеров).  
- **Шифрование как сервис** (без хранения данных).  
- **Аутентификацию и авторизацию** (AppRole, JWT/OIDC, Kubernetes, LDAP).  

### **Ключевые компоненты**  
- **Storage Backend** (Consul, DynamoDB, etcd, файловое хранилище) — где хранятся данные.  
- **Secrets Engines** (KV, PKI, AWS, Database) — генерация и управление секретами.  
- **Auth Methods** (Token, AppRole, Kubernetes Auth) — как клиенты аутентифицируются.  
- **Policies** (HCL или JSON) — кто и к каким секретам имеет доступ.  
- **Audit Logging** — запись всех операций для compliance.  

---  

## **2. Безопасность и эксплуатация**  

### **Seal/Unseal и инициализация**  
- При старте Vault **запечатан** (sealed) — данные недоступны.  
- **Unseal** требует порогового числа ключей Shamir (например, 3 из 5).  
- **Auto-unseal** (через AWS KMS, GCP KMS, Azure Key Vault) упрощает процесс.  

### **Как Vault защищает данные?**  
- **Шифрование** (AES-GCM, ChaCha20-Poly1305).  
- **Lease и TTL** — автоматический отзыв секретов.  
- **Revocation** — мгновенная инвалидация всех выданных секретов.  
- **Root Token Rotation** — смена root-токена после инициализации.  

### **Что делать при компрометации?**  
- **Revoke всех сессий** (`vault token revoke -self`).  
- **Rotate master key** (`vault operator rekey`).  
- **Обновить корневой токен** (`vault operator rotate-root`).  

---  

## **3. Интеграции и практическое применение**  

### **Динамические секреты для БД**  
Пример с PostgreSQL:  
1. Включить Database Secrets Engine:  
   ```sh
   vault secrets enable database
   ```  
2. Настроить подключение:  
   ```sh
   vault write database/config/postgresql \
     plugin_name=postgresql-database-plugin \
     connection_url="postgresql://{{username}}:{{password}}@db.example.com:5432/postgres" \
     allowed_roles="readonly" \
     username="vault-admin" \
     password="securepassword"
   ```  
3. Создать роль с TTL:  
   ```sh
   vault write database/roles/readonly \
     db_name=postgresql \
     creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';" \
     default_ttl="1h"
   ```  
Теперь приложения могут запрашивать временные учётные данные.  

### **PKI как сервис**  
Vault может быть **корневым** или **промежуточным CA**:  
```sh
vault secrets enable pki
vault write pki/root/generate/internal common_name=example.com ttl=8760h
vault write pki/roles/web-server allow_any_name=true max_ttl=72h
```  
Сертификаты автоматически отзываются по истечении TTL.  

### **Интеграция с Kubernetes**  
1. Включить Kubernetes Auth:  
   ```sh
   vault auth enable kubernetes
   ```  
2. Настроить доступ для Pod’ов:  
   ```sh
   vault write auth/kubernetes/config \
     kubernetes_host=https://kubernetes.default.svc \
     token_reviewer_jwt=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
   ```  
3. Создать Policy и Role для доступа:  
   ```sh
   vault policy write app-policy - <<EOF
   path "secret/data/app/*" {
     capabilities = ["read"]
   }
   EOF

   vault write auth/kubernetes/role/app \
     bound_service_account_names=app-sa \
     bound_service_account_namespaces=default \
     policies=app-policy \
     ttl=1h
   ```  
Теперь Pod с ServiceAccount `app-sa` может читать секреты.  

---  

## **4. Мониторинг и бекапы**  

### **Audit Logging**  
Обязательно включить:  
```sh
vault audit enable file file_path=/var/log/vault_audit.log
```  
Логируются все запросы (но не сами секреты).  

### **Резервное копирование**  
- **Snapshot API**:  
  ```sh
  vault operator raft snapshot save backup.snap
  ```  
- **Хранить ключи unseal в безопасном месте** (1Password, KMS, физический сейф).  

---  

## **5. Вопросы для самопроверки**  
1. Как Vault обрабатывает компрометацию root-токена?  
2. В чём разница между `kv-v1` и `kv-v2`?  
3. Как настроить автоматическую ротацию SSH-ключей через Vault?  
4. Как ограничить доступ к секретам на основе меток в Kubernetes?  
5. Какие альтернативы Vault ты рассматривал и почему выбрал именно его?  

---  
### **Отказоустойчивые схемы Hashicorp Vault**  

Чтобы Vault оставался доступным даже при сбоях, его можно развернуть в **высокодоступной (HA)** конфигурации. Рассмотрим ключевые подходы к обеспечению отказоустойчивости.  

---

## **1. Режим High Availability (HA) в Vault**  

Vault поддерживает HA через **встроенный механизм репликации** (leader-follower).  

### **Как это работает?**  
- **Один активный (leader) инстанс** — обрабатывает запросы на запись.  
- **Несколько standby-реплик (followers)** — готовы взять на себя роль лидера при его падении.  
- **Хранилище (storage backend)** должно быть распределённым (Consul, Raft, DynamoDB, etcd).  

### **Типы storage backends для HA**  

| **Тип хранилища** | **Описание** |  
|------------------|------------|  
| **Consul** | Кластер Consul обеспечивает кворум и автоматический failover. |  
| **Integrated Storage (Raft)** | Встроенное распределённое хранилище (рекомендуется HashiCorp). |  
| **DynamoDB / etcd** | Облачные managed-решения для хранения данных Vault. |  

**Пример конфигурации с Raft (наиболее современный вариант):**  
```hcl
storage "raft" {
  path    = "/vault/data"
  node_id = "vault-node-1"
}

cluster_addr = "https://vault-node-1:8201"
api_addr     = "https://vault-node-1:8200"
```

---

## **2. Развёртывание кластера Vault**  

### **Минимальная HA-конфигурация (3 ноды)**  
1. **Запуск первой ноды (инициализация):**  
   ```sh
   vault server -config=/etc/vault/config.hcl
   vault operator init -key-shares=5 -key-threshold=3
   ```  
   (Сохраняем **unseal keys** и **root token**.)  

2. **Добавление второй и третьей ноды:**  
   ```sh
   vault operator raft join https://vault-node-1:8200
   ```  

3. **Проверка статуса кластера:**  
   ```sh
   vault operator raft list-peers
   ```  

### **Как происходит failover?**  
- Если **leader падает**, оставшиеся ноды проводят выборы.  
- Новый leader избирается по алгоритму **Raft Consensus**.  
- Клиенты автоматически перенаправляются на новый leader.  

---

## **3. Multi-Datacenter Replication (DR / Performance)**  

Для глобальной отказоустойчивости можно настроить **репликацию между дата-центрами**:  

| **Тип репликации** | **Описание** |  
|--------------------|------------|  
| **Performance Replication** | Асинхронная репликация (читаемые копии в других регионах). |  
| **DR Replication** | Синхронная репликация для аварийного восстановления. |  

**Настройка Performance Replication:**  
1. Включить на primary-кластере:  
   ```sh
   vault write -f sys/replication/performance/primary/enable
   ```  
2. Получить **secondary token**:  
   ```sh
   vault write sys/replication/performance/primary/secondary-token id=secondary-1
   ```  
3. Активировать на secondary:  
   ```sh
   vault write sys/replication/performance/secondary/enable token=<TOKEN>
   ```  

---

## **4. Backup & Disaster Recovery**  

### **Резервное копирование состояния Vault**  
1. **Снимок (snapshot) Raft-хранилища:**  
   ```sh
   vault operator raft snapshot save backup.snap
   ```  
2. **Восстановление из снапшота:**  
   ```sh
   vault operator raft snapshot restore backup.snap
   ```  

### **Что делать при потере кворума?**  
Если большинство нод недоступно:  
1. **Восстановить из снапшота.**  
2. **Принудительно перезапустить кластер:**  
   ```sh
   vault operator raft force-leave
   ```  

---

## **5. Рекомендации по отказоустойчивости**  

✅ **Минимум 3 ноды** (для кворума).  
✅ **Разные зоны доступности** (AZs / регионы).  
✅ **Auto-unseal** (через KMS, Cloud HSM).  
✅ **Мониторинг** (Health checks, Prometheus + Grafana).  
✅ **Регулярные снапшоты** (S3, GCS).  

---

### **Отказоустойчивый кластер Hashicorp Vault с бекендом на PostgreSQL**  

Для построения **HA-кластера Vault** с хранением данных в **PostgreSQL** потребуется:  
- **3+ ноды Vault** (режим High Availability).  
- **Отказоустойчивый PostgreSQL** (Patroni + Streaming Replication).  
- **Механизм автоматического unseal** (например, через AWS KMS или HashiCorp Auto-unseal).  

---

## **1. Архитектура решения**  

```
                       +---------------------+
                       |   Load Balancer     |
                       | (HAProxy / Nginx)  |
                       +----------+----------+
                                  |
           +----------------------+----------------------+
           |                      |                      |
+----------v----------+ +----------v----------+ +----------v----------+
|    Vault Node 1     | |    Vault Node 2     | |    Vault Node 3     |
| (Leader / Follower) | | (Standby Follower)  | | (Standby Follower)  |
+---------------------+ +---------------------+ +---------------------+
           |                      |                      |
           +----------------------+----------------------+
                                  |
                     +------------v------------+
                     | PostgreSQL HA Cluster   |
                     | (Primary + 2 Replicas) |
                     | (Patroni + PgBouncer)  |
                     +------------------------+
```

---

## **2. Настройка PostgreSQL для Vault**  

### **Требования к PostgreSQL**  
- **Режим `READ COMMITTED`** (Vault требует этого уровня изоляции).  
- **Логическая репликация** (`wal_level = logical`).  
- **Отказоустойчивость** (Patroni, Streaming Replication).  

**Пример `postgresql.conf`:**  
```ini
wal_level = logical
max_connections = 200
synchronous_commit = remote_apply  # Для синхронной репликации
```

### **Настройка Patroni (HA PostgreSQL)**  
```yaml
# patroni.yml (для Primary)
scope: vault-cluster
name: pg-node-1

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.1.1:8008

etcd:  # или Consul для координации
  hosts: ["10.0.1.1:2379", "10.0.1.2:2379", "10.0.1.3:2379"]

bootstrap:
  dcs:
    ttl: 30
    retry_timeout: 10
    synchronous_mode: true
    synchronous_mode_strict: true
    postgresql:
      use_pg_rewind: true
      parameters:
        wal_level: logical

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.0.1.1:5432
  data_dir: /var/lib/postgresql/14/main
  pg_hba:
    - host all all 10.0.0.0/16 md5
```

---

## **3. Настройка Vault с PostgreSQL Storage Backend**  

### **Конфигурация Vault (`config.hcl`)**  
```hcl
storage "postgresql" {
  connection_url = "postgresql://vault_user:securepassword@pgbouncer.example.com:5432/vault?sslmode=require"
  ha_enabled     = true
  ha_table       = "vault_ha_locks"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_cert_file = "/etc/vault/tls/cert.pem"
  tls_key_file  = "/etc/vault/tls/key.pem"
  cluster_address = "10.0.1.1:8201"
}

api_addr = "https://vault-node-1:8200"
cluster_addr = "https://vault-node-1:8201"
disable_mlock = true
```

### **Инициализация кластера**  
1. **Запуск первой ноды:**  
   ```sh
   vault server -config=/etc/vault/config.hcl
   ```  
2. **Инициализация и сохранение ключей:**  
   ```sh
   vault operator init -key-shares=5 -key-threshold=3
   ```  
3. **Добавление остальных нод:**  
   ```sh
   vault operator raft join https://vault-node-1:8200
   ```  

---

## **4. Обеспечение автоматического Unseal**  

### **Вариант 1: AWS KMS**  
```hcl
seal "awskms" {
  region     = "us-east-1"
  kms_key_id = "alias/vault-unseal-key"
}
```

### **Вариант 2: HashiCorp Auto-unseal**  
```hcl
seal "pkcs11" {
  lib            = "/usr/lib/libcklog2.so"
  slot           = "0"
  key_label      = "vault-hsm-key"
  hmac_key_label = "vault-hmac-key"
}
```

---

## **5. Проверка отказоустойчивости**  

### **Тест 1: Падение leader-ноды**  
1. Определить текущего leader:  
   ```sh
   vault status
   ```  
2. Остановить leader:  
   ```sh
   systemctl stop vault
   ```  
3. Проверить, что новый leader избран:  
   ```sh
   vault operator raft list-peers
   ```  

### **Тест 2: Падение PostgreSQL Primary**  
1. Запустить **failover в Patroni**:  
   ```sh
   patronictl failover --force
   ```  
2. Убедиться, что Vault продолжает работать:  
   ```sh
   vault kv get secret/test
   ```  

---

## **6. Резервное копирование**  

### **Снапшот Vault**  
```sh
vault operator raft snapshot save vault-backup.snap
```

### **PgBak для PostgreSQL**  
```sh
pg_dump -U vault_user -h pgbouncer.example.com -Fc vault > vault-db.dump
```

---

## **Вывод**  
✅ **3+ ноды Vault** с PostgreSQL-бекендом.  
✅ **Patroni + Streaming Replication** для HA PostgreSQL.  
✅ **Автоматический unseal** (KMS / HSM).  
✅ **Регулярные снапшоты** (Vault + PostgreSQL).  

Такой кластер переживёт:  
- Падение **любой ноды Vault**.  
- Падение **PostgreSQL Primary**.  
- Сетевые сбои (благодаря **PgBouncer** и балансировщику).  

# **Основные CLI-команды Hashicorp Vault**  
Vault CLI — основной инструмент для управления секретами, аутентификации, настройки политик и администрирования кластера.  

---

## **1. Общие команды**  
| Команда | Описание |  
|---------|----------|  
| `vault status` | Проверить статус Vault (seal/unseal, leader) |  
| `vault login [METHOD]` | Аутентификация (например, `vault login -method=userpass username=admin`) |  
| `vault logout` | Выйти из текущей сессии |  
| `vault token lookup` | Информация о текущем токене |  
| `vault token renew` | Продлить срок действия токена |  
| `vault read <path>` | Чтение секрета (например, `vault read secret/data/myapp`) |  
| `vault write <path> key=value` | Запись секрета |  
| `vault delete <path>` | Удаление секрета |  
| `vault list <path>` | Просмотр списка ключей |  

---

## **2. Управление Secrets Engines**  
| Команда | Описание |  
|---------|----------|  
| `vault secrets list` | Список включенных Secrets Engines |  
| `vault secrets enable -path=kv kv-v2` | Включить KV Secrets Engine (версия 2) |  
| `vault secrets disable kv/` | Отключить Secrets Engine |  
| `vault kv put secret/myapp password=123` | Записать секрет в KV |  
| `vault kv get secret/myapp` | Прочитать секрет |  

---

## **3. Управление аутентификацией (Auth Methods)**  
| Команда | Описание |  
|---------|----------|  
| `vault auth list` | Список методов аутентификации |  
| `vault auth enable userpass` | Включить аутентификацию по логину/паролю |  
| `vault write auth/userpass/users/admin password=123` | Создать пользователя |  
| `vault auth enable kubernetes` | Включить аутентификацию через Kubernetes |  

---

## **4. Политики (Policies)**  
| Команда | Описание |  
|---------|----------|  
| `vault policy list` | Список политик |  
| `vault policy read default` | Просмотр политики |  
| `vault policy write mypolicy policy.hcl` | Создать/обновить политику |  

**Пример `policy.hcl`:**  
```hcl
path "secret/data/myapp/*" {
  capabilities = ["read", "list"]
}
```

---

## **5. Управление PKI (сертификаты)**  
| Команда | Описание |  
|---------|----------|  
| `vault secrets enable pki` | Включить PKI Secrets Engine |  
| `vault write pki/root/generate/internal common_name=example.com` | Создать корневой CA |  
| `vault write pki/roles/web allow_any_name=true` | Создать роль для выпуска сертификатов |  
| `vault write pki/issue/web common_name=test.example.com` | Выпустить сертификат |  

---

## **6. Администрирование кластера**  
| Команда | Описание |  
|---------|----------|  
| `vault operator init` | Инициализировать Vault (первый запуск) |  
| `vault operator unseal` | Распечатать Vault |  
| `vault operator raft list-peers` | Список нод в HA-кластере |  
| `vault operator raft snapshot save backup.snap` | Создать бэкап |  
| `vault operator rotate-root` | Сменить root-токен |  

---

## **Доступ к Vault через API (REST)**  
Vault предоставляет **HTTP API** для интеграции с приложениями.  

### **1. Базовые запросы**  
- **Аутентификация** (например, через токен):  
  ```bash
  curl \
    --header "X-Vault-Token: s.1234567890" \
    http://127.0.0.1:8200/v1/secret/data/myapp
  ```  

- **Чтение секрета** (KV v2):  
  ```bash
  curl \
    --header "X-Vault-Token: s.1234567890" \
    http://127.0.0.1:8200/v1/secret/data/myapp | jq
  ```  

- **Запись секрета**:  
  ```bash
  curl \
    --header "X-Vault-Token: s.1234567890" \
    --request POST \
    --data '{"data": {"password": "123"}}' \
    http://127.0.0.1:8200/v1/secret/data/myapp
  ```  

### **2. Аутентификация через API**  
- **Userpass**:  
  ```bash
  curl \
    --request POST \
    --data '{"password": "123"}' \
    http://127.0.0.1:8200/v1/auth/userpass/login/admin
  ```  

- **Kubernetes (из Pod)**:  
  ```bash
  curl \
    --request POST \
    --data '{"jwt": "'$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)'", "role": "myapp"}' \
    http://vault:8200/v1/auth/kubernetes/login
  ```  

### **3. Динамические секреты (БД)**  
```bash
curl \
  --header "X-Vault-Token: s.1234567890" \
  http://127.0.0.1:8200/v1/database/creds/myapp-role
```  

---
### **Основные проблемы в HashiCorp Vault и методы их устранения**  

Vault — сложная система, и при эксплуатации могут возникать различные проблемы. Рассмотрим **типичные сценарии**, их **диагностику** и **способы решения**.  

---

## **1. Vault недоступен (Sealed)**  
**Симптомы:**  
- Запросы возвращают `500 Internal Server Error` или `503 Service Unavailable`.  
- Команда `vault status` показывает `Sealed: true`.  

**Причины:**  
- Сервер перезагрузился, и Vault не распечатан.  
- Недостаточно ключей unseal.  
- Проблемы с хранилищем (PostgreSQL, Consul и т. д.).  

**Решение:**  
1. **Проверить статус:**  
   ```sh
   vault status
   ```  
2. **Распечатать Vault:**  
   ```sh
   vault operator unseal
   ```  
   (Требуется **пороговое число ключей Shamir**.)  
3. **Автоматизировать unseal:**  
   - Использовать **Auto-unseal** (AWS KMS, GCP KMS, Azure Key Vault).  
   - Настроить **systemd-юнит** для автоматического unseal после перезагрузки.  

---

## **2. Проблемы с хранилищем (Storage Backend)**  
**Симптомы:**  
- Ошибки типа `no route to host`, `connection refused`.  
- Vault падает с ошибками `failed to lock storage`.  

**Причины:**  
- PostgreSQL/Consul/etc. недоступен.  
- Проблемы с сетью или правами доступа.  

**Решение:**  
1. **Проверить подключение к хранилищу:**  
   ```sh
   psql -h <postgres-host> -U vault_user -d vault
   ```  
2. **Логи Vault:**  
   ```sh
   journalctl -u vault -f
   ```  
3. **Восстановить кластер хранилища:**  
   - Для **PostgreSQL**: проверить Patroni (`patronictl list`).  
   - Для **Consul**: `consul members`.  
4. **Резервное копирование и восстановление:**  
   ```sh
   vault operator raft snapshot save backup.snap
   vault operator raft snapshot restore backup.snap
   ```  

---

## **3. Ошибки аутентификации (Permission Denied)**  
**Симптомы:**  
- `permission denied`, `missing client token`.  
- Политики не применяются.  

**Причины:**  
- Неправильные **policies**.  
- Истёкший **токен**.  
- Ошибки в **auth methods** (Kubernetes, OIDC и т. д.).  

**Решение:**  
1. **Проверить токен:**  
   ```sh
   vault token lookup
   ```  
2. **Проверить политики:**  
   ```sh
   vault policy read <policy-name>
   ```  
3. **Проверить auth method:**  
   - Для **Kubernetes**:  
     ```sh
     vault read auth/kubernetes/config
     ```  
   - Для **OIDC**:  
     ```sh
     vault read auth/oidc/config
     ```  

---

## **4. Проблемы с динамическими секретами (БД, AWS и т. д.)**  
**Симптомы:**  
- Динамические секреты не генерируются.  
- Ошибки типа `failed to create credentials`.  

**Причины:**  
- Неправильные настройки **Secrets Engine**.  
- Проблемы с подключением к внешней системе (БД, AWS API).  

**Решение:**  
1. **Проверить Secrets Engine:**  
   ```sh
   vault read database/config/<db-name>
   ```  
2. **Проверить подключение к БД:**  
   ```sh
   vault write database/rotate-root/<db-name>
   ```  
3. **Логи Vault:**  
   ```sh
   vault read sys/logs
   ```  

---

## **5. Проблемы с производительностью**  
**Симптомы:**  
- Медленные ответы на запросы.  
- Высокая нагрузка на CPU/память.  

**Причины:**  
- Большое количество **арендованных секретов (leases)**.  
- Проблемы с хранилищем (медленный PostgreSQL/Consul).  

**Решение:**  
1. **Оптимизировать TTL:**  
   ```sh
   vault read sys/leases/config
   ```  
2. **Очистить старые leases:**  
   ```sh
   vault lease revoke -prefix secret/
   ```  
3. **Масштабировать хранилище:**  
   - Для **PostgreSQL**: добавить реплики, настроить PgBouncer.  
   - Для **Consul**: увеличить кластер.  

---

## **6. Проблемы с репликацией (HA/DR)**  
**Симптомы:**  
- Репликация не работает.  
- Вторичный кластер не синхронизируется.  

**Причины:**  
- Сетевые проблемы между дата-центрами.  
- Ошибки в настройке **Performance/DR Replication**.  

**Решение:**  
1. **Проверить статус репликации:**  
   ```sh
   vault read sys/replication/status
   ```  
2. **Перезапустить репликацию:**  
   ```sh
   vault write sys/replication/performance/secondary/enable token=<TOKEN>
   ```  

---

## **7. Аварийное восстановление (Disaster Recovery)**  
**Симптомы:**  
- Кластер полностью недоступен.  
- Потеря данных.  

**Решение:**  
1. **Восстановить из снапшота:**  
   ```sh
   vault operator raft snapshot restore backup.snap
   ```  
2. **Пересоздать кластер:**  
   - Заново инициализировать Vault (`vault operator init`).  
   - Восстановить политики и конфиги из бэкапа.  

---

## **Итог: чеклист для диагностики**  
1. **Vault sealed?** → `vault status` + unseal.  
2. **Хранилище доступно?** → Проверить PostgreSQL/Consul.  
3. **Есть ли доступ?** → `vault token lookup` + проверка политик.  
4. **Динамические секреты работают?** → Проверить Secrets Engine.  
5. **Медленные запросы?** → Оптимизировать leases + мониторинг.  
6. **Репликация сломана?** → `vault read sys/replication/status`.  

Для сложных случаев используйте **логи (`journalctl -u vault`)** и **мониторинг (Prometheus + Grafana)**.  
