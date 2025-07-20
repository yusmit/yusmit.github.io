# **1. Основные вопросы по Docker**  

**Q1: Что такое Docker и для чего он используется?**  
**A1:** Docker — это платформа для контейнеризации приложений. Она позволяет упаковывать приложение и его зависимости в изолированный контейнер, который может работать на любой системе с Docker.  
Основные преимущества:  
- Изоляция приложений  
- Переносимость между средами (dev, test, prod)  
- Быстрое развертывание и масштабирование  

---  

**Q2: В чем разница между образом (image) и контейнером (container)?**  
**A2:**  
- **Образ (Image)** — это шаблон (read-only), на основе которого создаются контейнеры. Он содержит файловую систему, зависимости и настройки приложения.  
- **Контейнер (Container)** — это запущенный экземпляр образа (read-write). Контейнеры изолированы и работают независимо.  

---  

**Q3: Как создать Docker-образ?**  
**A3:** Образ создается с помощью `Dockerfile` и команды:  
```bash
docker build -t my-image:1.0 .
```  
Пример `Dockerfile`:  
```dockerfile
FROM alpine:latest
RUN apk add --no-cache python3
COPY app.py /app/
CMD ["python3", "/app/app.py"]
```  

---  

**Q4: Какие основные команды Docker вы знаете?**  
**A4:**  
- `docker run -d --name my-container nginx` – запуск контейнера в фоне  
- `docker ps` – список запущенных контейнеров  
- `docker ps -a` – все контейнеры (включая остановленные)  
- `docker stop my-container` – остановка контейнера  
- `docker rm my-container` – удаление контейнера  
- `docker images` – список образов  
- `docker rmi nginx` – удаление образа  
- `docker logs my-container` – просмотр логов  
- `docker exec -it my-container bash` – вход в контейнер  

---  

**Q5: Как пробросить порты в Docker?**  
**A5:** Через флаг `-p`:  
```bash
docker run -p 8080:80 nginx  # Хост:Контейнер
```  

---  

**Q6: Как монтировать volumes в Docker?**  
**A6:** Через флаг `-v`:  
```bash
docker run -v /host/path:/container/path nginx
```  
Или именованные volumes:  
```bash
docker volume create my-volume
docker run -v my-volume:/data nginx
```  

---  

# **2. Вопросы по Docker Compose**  

**Q7: Что такое Docker Compose и зачем он нужен?**  
**A7:** Docker Compose — это инструмент для управления многоконтейнерными приложениями через YAML-файл (`docker-compose.yml`). Позволяет запускать несколько сервисов одной командой.  

---  

**Q8: Как запустить и остановить сервисы через Docker Compose?**  
**A8:**  
```bash
docker-compose up -d  # Запуск в фоне
docker-compose down   # Остановка и удаление
```  

---  

**Q9: Как выглядит простой `docker-compose.yml`?**  
**A9:** Пример с Nginx и MySQL:  
```yaml
version: '3.8'
services:
  web:
    image: nginx:latest
    ports:
      - "80:80"
  db:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: secret
```  

---  

**Q10: Как передавать переменные окружения в Docker Compose?**  
**A10:** Через `environment` или `.env`-файл:  
```yaml
services:
  app:
    image: my-app
    environment:
      - DB_HOST=db
      - DB_USER=admin
```  
Или через `.env`:  
```env
DB_PASSWORD=12345
```  
И в `docker-compose.yml`:  
```yaml
environment:
  - DB_PASSWORD=${DB_PASSWORD}
```  

---  

**Q11: Как сделать depends_on в Docker Compose?**  
**A11:** Указать зависимости между сервисами:  
```yaml
services:
  app:
    depends_on:
      - db
  db:
    image: postgres
```  

---  

**Q12: Как пересобрать образ через Docker Compose?**  
**A12:**  
```bash
docker-compose up -d --build
```  

---  

# **3. Продвинутые вопросы**  

**Q13: Как работает Docker Network?**  
**A13:** Docker создает виртуальные сети для связи между контейнерами. По умолчанию:  
- `bridge` – сеть по умолчанию для контейнеров  
- `host` – контейнер использует сеть хоста  
- `none` – без сети  

В Docker Compose сервисы в одной сети могут общаться по имени сервиса.  

---  

**Q14: Что такое Docker Swarm и Kubernetes?**  
**A14:**  
- **Docker Swarm** – встроенный оркестратор Docker для кластеризации.  
- **Kubernetes (K8s)** – более мощный оркестратор для управления контейнерами в продакшене.  

---  

**Q15: Как чистить Docker от ненужных образов и контейнеров?**  
**A15:**  
```bash
docker system prune -a  # Удалить всё неиспользуемое
docker volume prune     # Очистить volumes
```  

---

# **3. Docker: Основы**  

**Q1: Как Docker отличается от виртуальной машины (VM)?**  
**A1:**  
- **Docker** использует ядро хоста и работает в изолированных пространствах процессов (namespaces, cgroups). Контейнеры легче, быстрее и потребляют меньше ресурсов.  
- **VM** требует полноценной ОС и гипервизора, что делает её более тяжелой.  

**Q2: Что такое Docker Hub?**  
**A2:** Это публичный реестр образов Docker (как GitHub для кода). Можно скачивать (`docker pull`) и загружать (`docker push`) образы.  

**Q3: Как посмотреть историю изменений образа?**  
**A3:**  
```bash
docker history <image_name>
```  

**Q4: Как копировать файлы в/из контейнера?**  
**A4:**  
```bash
docker cp file.txt container_id:/path/  # В контейнер  
docker cp container_id:/path/file.txt .  # Из контейнера  
```  

**Q5: Что делает команда `docker inspect`?**  
**A5:** Выводит детальную информацию о контейнере/образе (настройки сети, volumes, env-переменные и т. д.).  

---

### **4. Docker: Продвинутые вопросы**  

**Q6: Что такое multi-stage build в Docker? Зачем он нужен?**  
**A6:** Позволяет использовать несколько `FROM` в одном `Dockerfile`, чтобы уменьшить итоговый образ.  
Пример:  
```dockerfile
FROM golang:1.18 as builder  
WORKDIR /app  
COPY . .  
RUN go build -o myapp  

FROM alpine:latest  
COPY --from=builder /app/myapp /  
CMD ["/myapp"]  
```  

**Q7: Как ограничить ресурсы контейнера (CPU, RAM)?**  
**A7:**  
```bash
docker run --cpus=1 --memory=512m nginx  
```  

**Q8: Как работает `docker save` и `docker load`?**  
**A8:**  
- `docker save -o myimage.tar myimage:latest` – сохраняет образ в `.tar`.  
- `docker load -i myimage.tar` – загружает образ из архива.  

**Q9: Что такое `.dockerignore`?**  
**A9:** Аналог `.gitignore` – исключает файлы из контекста сборки (ускоряет `docker build`).  

**Q10: Как debug-ить зависший контейнер?**  
**A10:**  
1. `docker logs <container_id>` – проверить логи.  
2. `docker exec -it <container_id> sh` – зайти внутрь.  
3. `docker stats` – посмотреть потребление ресурсов.  

---

# **5. Docker Compose**  

**Q11: Как переопределить `CMD` или `ENTRYPOINT` в `docker-compose.yml`?**  
**A11:**  
```yaml
services:  
  app:  
    image: myapp  
    command: ["python", "app.py"]  # Переопределяет CMD  
    entrypoint: /entrypoint.sh     # Переопределяет ENTRYPOINT  
```  

**Q12: Как сделать healthcheck в Docker Compose?**  
**A12:**  
```yaml
services:  
  db:  
    image: postgres  
    healthcheck:  
      test: ["CMD-SHELL", "pg_isready -U postgres"]  
      interval: 5s  
      timeout: 3s  
      retries: 3  
```  

**Q13: Как использовать переменные в `docker-compose.yml`?**  
**A13:**  
1. Через `${VAR}` в `docker-compose.yml`:  
   ```yaml
   environment:  
     - DB_PASSWORD=${DB_PASS}  
   ```  
2. Или через `.env`-файл рядом с `docker-compose.yml`.  

**Q14: Как масштабировать сервисы в Docker Compose?**  
**A14:**  
```bash
docker-compose up -d --scale web=3  # Запустит 3 инстанса сервиса `web`  
```  

**Q15: Как подключить контейнер к существующей сети?**  
**A15:**  
```yaml
services:  
  app:  
    networks:  
      - my-network  

networks:  
  my-network:  
    external: true  
```  

---

# **6. Docker: Безопасность**  

**Q16: Как запускать контейнер в режиме read-only?**  
**A16:**  
```bash
docker run --read-only alpine  
```  
Или в `docker-compose.yml`:  
```yaml
services:  
  app:  
    read_only: true  
```  

**Q17: Как ограничить права контейнера (без root)?**  
**A17:**  
```bash
docker run --user 1000:1000 nginx  
```  
Или в `Dockerfile`:  
```dockerfile
USER 1000  
```  

**Q18: Что такое `docker scan`?**  
**A18:** Команда для проверки образов на уязвимости (интеграция с Snyk).  

---

# **7. Практические задачи**  

**Q19: Как собрать образ и сразу запустить контейнер?**  
**A19:**  
```bash
docker build -t myapp . && docker run -d myapp  
```  

**Q20: Как удалить все остановленные контейнеры?**  
**A20:**  
```bash
docker container prune  
```  

**Q21: Как посмотреть, какие volumes не используются?**  
**A21:**  
```bash
docker volume ls -f dangling=true  
```  

**Q22: Как перезапустить контейнер с новыми env-переменными?**  
**A22:**  
```bash
docker stop my-container && docker run -e NEW_VAR=value my-image  
```  

---

# **8. Docker: Углубленные вопросы**  

**Q1: Как Docker реализует изоляцию процессов?**  
**A1:**  
- Использует **namespaces** (PID, NET, IPC, MNT, UTS, USER) для изоляции процессов, сети, файловой системы.  
- **cgroups** (control groups) для ограничения ресурсов (CPU, RAM, I/O).  
- **Capabilities** для ограничения прав контейнера (например, запрет `CAP_NET_ADMIN`).  
- **Seccomp** и **AppArmor/SELinux** для фильтрации системных вызовов.  

**Доп. вопрос:** *Как проверить, какие namespaces использует контейнер?*  
→ `ls -la /proc/<PID>/ns/`  

---

**Q2: Как устроена Docker-сеть (network drivers)?**  
**A2:**  
- **Bridge** (по умолчанию) – виртуальная сеть между контейнерами.  
- **Host** – контейнер использует сеть хоста (нет изоляции).  
- **Overlay** – для связи между контейнерами на разных хостах (Swarm/Kubernetes).  
- **Macvlan** – контейнер получает реальный MAC-адрес.  
- **None** – нет сети.  

**Доп. вопрос:** *Как сделать кастомную bridge-сеть с фиксированными IP?*  
→  
```bash
docker network create --subnet=172.20.0.0/16 mynet  
docker run --net mynet --ip 172.20.0.2 nginx  
```  

---

**Q3: Multi-stage builds – когда использовать и как оптимизировать?**  
**A3:**  
- Используется для уменьшения размера финального образа (например, сборка в `golang`, а запуск из `alpine`).  
- Можно копировать артефакты между стадиями (`COPY --from=builder`).  
- Оптимизация:  
  - Минимизация слоев (объединять `RUN` через `&&`).  
  - Использование `.dockerignore`.  
  - Выбор минимального базового образа (`scratch`, `alpine`).  

**Доп. вопрос:** *Как уменьшить размер образа с Python-приложением?*  
→ Использовать `python:slim`, удалять кэш pip (`--no-cache-dir`), multi-stage build.  

---

**Q4: Как debug-ить проблемы с производительностью контейнера?**  
**A4:**  
1. **Логи:** `docker logs --tail=100 -f <container>`  
2. **Ресурсы:** `docker stats`, `docker top <container>`  
3. **Интерактивный debug:** `docker exec -it <container> sh`  
4. **Инструменты:**  
   - `htop`, `strace` внутри контейнера.  
   - `perf` (если разрешены capabilities).  
5. **Профилирование сети:** `tcpdump`, `wireshark`.  

**Доп. вопрос:** *Контейнер падает с `OOMKilled` – как найти утечку памяти?*  
→  
- `docker inspect` → проверка `OOMKilled: true`.  
- Инструменты: `valgrind`, `pprof` (для Go), `jmap` (для Java).  

---

# **9. Docker Compose: Продвинутые сценарии**  

**Q5: Как реализовать blue-green deployment с Docker Compose?**  
**A5:**  
1. Два сервиса (`app_v1`, `app_v2`) с разными версиями.  
2. Reverse-proxy (Nginx/Traefik) с переключением между ними.  
3. Использование healthcheck для проверки работоспособности.  
4. Постепенное обновление через `docker-compose up --scale`.  

**Пример:**  
```yaml
services:  
  app_v1:  
    image: myapp:v1  
    networks:  
      - app_network  
  app_v2:  
    image: myapp:v2  
    networks:  
      - app_network  
  proxy:  
    image: nginx  
    ports:  
      - "80:80"  
    depends_on:  
      - app_v1  
      - app_v2  
```  

**Доп. вопрос:** *Как сделать rollback?*  
→ Изменить `depends_on` в proxy и перезапустить.  

---

**Q6: Как управлять секретами (secrets) в Docker Compose?**  
**A6:**  
1. **Docker Swarm Secrets**:  
   ```yaml
   secrets:  
     db_password:  
       file: ./db_password.txt  
   services:  
     db:  
       secrets:  
         - db_password  
   ```  
2. **.env-файлы** (небезопасно для prod).  
3. **Hashicorp Vault + сторонние решения**.  

**Доп. вопрос:** *Как передать секрет в runtime без файла?*  
→  
```bash
echo "secret" | docker secret create my_secret -  
```  

---

**Q7: Как настроить динамическое масштабирование сервисов?**  
**A7:**  
1. **Docker Swarm**:  
   ```bash
   docker service scale my_service=5  
   ```  
2. **Kubernetes** (если используется).  
3. **Автоматическое масштабирование**:  
   - Мониторинг (Prometheus + Grafana).  
   - Скрипты на основе метрик (CPU/RAM).  

**Доп. вопрос:** *Как сделать autoscaling для веб-сервиса?*  
→  
- Мониторинг запросов (RPS).  
- `docker-compose up --scale web=3`.  

---

# **10. Безопасность и Production-развертывание**  

**Q8: Как защитить Docker-демон?**  
**A8:**  
1. **Отключить удаленный API** или использовать TLS-аутентификацию.  
2. **Запуск от non-root пользователя** (`dockerd --userns-remap`).  
3. **Ограничить capabilities** (`--cap-drop ALL --cap-add NET_BIND_SERVICE`).  
4. **Read-only файловая система** (`--read-only`).  
5. **AppArmor/SELinux** для ограничения доступа.  

**Доп. вопрос:** *Как проверить уязвимости в образе?*  
→  
```bash
docker scan my_image  
trivy image my_image  
```  

---

**Q9: Как настроить логирование в Production?**  
**A9:**  
1. **Драйверы логирования**:  
   - `json-file` (по умолчанию).  
   - `syslog`, `journald`, `fluentd`, `awslogs`.  
2. **Сбор логов**:  
   - ELK Stack (Elasticsearch + Logstash + Kibana).  
   - Loki + Grafana.  
3. **Ротация логов**:  
   ```json
   { "log-driver": "json-file", "log-opts": { "max-size": "10m", "max-file": "3" } }  
   ```  

**Доп. вопрос:** *Как парсить логи из `stdout`?*  
→ Использовать `grep`, `jq` (для JSON), или отправлять в Fluentd.  

---

**Q10: Как развернуть Docker в Kubernetes?**  
**A1:**  
1. **Сборка образа** → Push в Registry (Docker Hub, ECR, GCR).  
2. **Deployment-манифест**:  
   ```yaml
   apiVersion: apps/v1  
   kind: Deployment  
   metadata:  
     name: myapp  
   spec:  
     replicas: 3  
     template:  
       spec:  
         containers:  
         - name: myapp  
           image: myrepo/myapp:v1  
   ```  
3. **Service/Ingress** для доступа.  

**Доп. вопрос:** *Как сделать rolling update?*  
→  
```bash
kubectl set image deployment/myapp myapp=myrepo/myapp:v2  
```  

---

# **11. Docker: Углублённые вопросы по архитектуре**  

**Q1: Как Docker реализует OverlayFS и какие есть альтернативные драйверы хранения?**  
**A1:**  
- **OverlayFS** – основной драйвер (объединяет слои `lowerdir`, `upperdir`, `workdir`).  
- **Alternatives:**  
  - `aufs` (устаревший, но поддерживается).  
  - `btrfs`, `zfs` – для продвинутого управления томами.  
  - `devicemapper` (для старых систем, требует LVM).  
  - **Windows:** `windowsfilter` (для Windows-контейнеров).  

**Доп. вопрос:** *Как проверить текущий storage driver?*  
→ `docker info | grep "Storage Driver"`  

---

**Q2: Как работает Docker-сеть в режиме `user-defined bridge` и чем она лучше default bridge?**  
**A2:**  
- **Default bridge (`docker0`)** – автоматическое DNS-разрешение отключено, контейнеры видят друг друга только по IP.  
- **User-defined bridge:**  
  - Автоматический DNS между контейнерами (можно обращаться по имени сервиса).  
  - Лучшая изоляция (можно создать несколько сетей).  
  - Возможность настройки MTU, подсетей и gateway.  

**Доп. вопрос:** *Как прицепить работающий контейнер к новой сети?*  
→  
```bash
docker network connect my_network my_container  
```  

---

**Q3: Как ускорить сборку Docker-образов в CI/CD?**  
**A3:**  
1. **Кэширование слоёв**:  
   - Оптимизировать порядок команд в `Dockerfile` (меняющиеся шаги – в конец).  
   - Использовать `--cache-from` в CI (`docker build --cache-from=my-image:latest`).  
2. **Multi-stage builds** – уменьшение финального образа.  
3. **BuildKit**:  
   ```bash
   DOCKER_BUILDKIT=1 docker build --ssh default .  
   ```  
   - Параллельная сборка.  
   - Кэширование секретов (`--mount=type=secret`).  
4. **Distributed caching** (Bazel, Earthly).  

**Доп. вопрос:** *Как использовать SSH-агент при сборке?*  
→  
```dockerfile
RUN --mount=type=ssh git clone git@github.com:my/repo.git  
```  

---

## **12. Docker: Безопасность и Production**  

**Q4: Как защитить Docker-демон в продакшне?**  
**A4:**  
1. **Отключить небезопасный API**:  
   - `/var/run/docker.sock` → доступ только для trusted users.  
   - Если нужен удалённый API – только с TLS (`--tlsverify`).  
2. **User namespaces** (`--userns-remap=default`).  
3. **Read-only rootfs**:  
   ```bash
   docker run --read-only --tmpfs /tmp alpine  
   ```  
4. **Seccomp/AppArmor профили**:  
   ```bash
   docker run --security-opt seccomp=/path/to/profile.json  
   ```  
5. **Запрет root в контейнере**:  
   ```dockerfile
   USER 1000  
   ```  

**Доп. вопрос:** *Как проверить, какие capabilities есть у контейнера?*  
→  
```bash
docker inspect --format='{{.HostConfig.Capabilities}}' my_container  
```  

---

**Q5: Как настроить логирование в Docker для продакшна?**  
**A5:**  
1. **Драйверы**:  
   - `json-file` (с ротацией).  
   - `fluentd` → Elasticsearch/Kibana.  
   - `awslogs` (для AWS).  
2. **Ротация логов**:  
   ```json
   { "log-driver": "json-file", "log-opts": { "max-size": "10m", "max-file": "3" } }  
   ```  
3. **Сбор логов**:  
   - Loki + Grafana (легковесная альтернатива ELK).  
   - Filebeat → Logstash.  

**Доп. вопрос:** *Как отправить логи в Syslog?*  
→  
```bash
docker run --log-driver=syslog --log-opt syslog-address=udp://1.2.3.4:514 nginx  
```  

---

**Q6: Как debug-ить "висящий" контейнер?**  
**A6:**  
1. **Инструменты**:  
   - `docker exec -it my_container sh` → `top`, `htop`, `strace`.  
   - `nsenter` (если `exec` не работает).  
2. **Логи ядра**:  
   ```bash
   dmesg | grep -i oom  
   ```  
3. **Профилирование**:  
   - `perf` (если разрешены capabilities).  
   - `docker stats` → CPU/RAM.  
4. **Network**:  
   - `tcpdump -i any -n port 80`.  

**Доп. вопрос:** *Как debug-ить зависший `docker build`?*  
→  
```bash
DOCKER_BUILDKIT=0 docker build --no-cache .  # Отключить BuildKit  
```  

---

## **13. Docker Compose: Продвинутые сценарии**  

**Q7: Как сделать zero-downtime deployment с Docker Compose?**  
**A7:**  
1. **Blue-Green**:  
   - Два стека (`app_v1`, `app_v2`).  
   - Переключение трафика (Nginx/Traefik).  
2. **Healthchecks**:  
   ```yaml
   healthcheck:  
     test: ["CMD", "curl", "-f", "http://localhost"]  
     interval: 5s  
     timeout: 3s  
     retries: 3  
   ```  
3. **Rolling update**:  
   ```bash
   docker-compose up -d --no-deps --scale web=3  
   ```  

**Доп. вопрос:** *Как автоматизировать rollback?*  
→ Скрипт + мониторинг (Prometheus + Alertmanager).  

---

**Q8: Как использовать Docker Compose с Kubernetes?**  
**A8:**  
1. **`kompose`** – конвертация `docker-compose.yml` в Kubernetes-манифесты:  
   ```bash
   kompose convert  
   ```  
2. **Для продакшна**:  
   - Ручная оптимизация (Resource Limits, Liveness probes).  
   - Helm-чарты вместо Compose.  

**Доп. вопрос:** *Какие ограничения у `docker-compose` для Kubernetes?*  
→ Нет поддержки:  
- StatefulSets, DaemonSets.  
- HPA (автомасштабирование).  
- Ingress (только простые Service).  

---

## **14. Интеграция с другими системами**  

**Q9: Как интегрировать Docker с мониторингом (Prometheus)?**  
**A9:**  
1. **Экспорт метрик**:  
   - `cAdvisor` (мониторинг контейнеров).  
   - `node-exporter` (метрики хоста).  
2. **Конфиг Prometheus**:  
   ```yaml
   scrape_configs:  
     - job_name: 'docker'  
       static_configs:  
         - targets: ['cadvisor:8080']  
   ```  
3. **Grafana** – дашборды.  

**Доп. вопрос:** *Как мониторить Docker Swarm?*  
→ `docker service create --mode=global prom/node-exporter`.  

---

**Q10: Как развернуть stateful-сервис (БД) в Docker?**  
**A1:**  
1. **Volumes**:  
   ```yaml
   volumes:  
     db_data:  
       driver: local  
   services:  
     db:  
       volumes:  
         - db_data:/var/lib/postgresql  
   ```  
2. **Репликация**:  
   - Для PostgreSQL: `patroni`, `pgpool`.  
   - Для MySQL: `group replication`.  
3. **Бэкапы**:  
   - `pg_dump` + cron в отдельном контейнере.  

**Доп. вопрос:** *Как мигрировать данные между хостами?*  
→  
```bash
docker run --volumes-from db -v $(pwd):/backup alpine tar cvf /backup/data.tar /var/lib/postgresql  
```  

---
