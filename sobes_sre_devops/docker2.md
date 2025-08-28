### **Разница между `CMD` и `ENTRYPOINT`**

Обе инструкции определяют, какую команду выполнить при запуске контейнера. Их взаимодействие — ключ к пониманию.
**Краткий ответ:**
*   `ENTRYPOINT` — это **что запускать** (непереопределяемая команда).
*   `CMD` — это **аргументы по умолчанию** для этой команды, которые можно легко переопределить.

**Когда что использовать:**
*   Используйте `ENTRYPOINT`, когда ваш контейнер — это **исполняемая утилита** (например, `curl`, `bash`, ваш собственный бинарник).
*   Используйте `CMD`, когда вам нужно предоставить **легко переопределяемые аргументы по умолчанию** (например, запуск скрипта, который можно заменить на `bash` для дебага).
*   Часто их используют **вместе**, чтобы получить гибкость и удобство.

#### **1. `CMD` (Command)**

*   **Предназначение:** Предоставить **аргументы по умолчанию** для выполняемой команды.
*   **Поведение:** Команда, указанная в `CMD`, **может быть переопределена** пользователем при запуске контейнера через `docker run [image] [новая команда]`.
*   **Синтаксис:**
    *   *Shell form:* `CMD echo "Hello World"`
    *   *Exec form (рекомендуется):* `CMD ["executable", "param1", "param2"]`
*   **Аналогия:** `CMD` — это аргументы по умолчанию для вашей программы.
    > Представьте, что ваша программа — это `curl`. Вы можете задать для нее аргументы по умолчанию: `CMD ["-I", "https://google.com"]`. Но пользователь может их легко поменять.

#### **2. `ENTRYPOINT`**

*   **Предназначение:** Определить **исполняемую команду**, которая будет запущена при старте контейнера. Контейнер будет работать как эта команда.
*   **Поведение:** Команда и параметры, указанные в `ENTRYPOINT`, **НЕ переопределяются** при запуске контейнера. Вместо этого, аргументы, переданные в `docker run`, **добавляются** в конец команды `ENTRYPOINT`.
*   **Синтаксис:** *Предпочтительна только Exec form:* `ENTRYPOINT ["executable", "param1", "param2"]`
*   **Аналогия:** `ENTRYPOINT` — это сама программа, которую нельзя поменять.
    > Продолжая аналогию с `curl`: `ENTRYPOINT ["curl"]`. Теперь контейнер — это всегда `curl`. Пользователь может только добавлять флаги: `docker run my-curl -I` превратится в `curl -I`.

---

### **Как они работают вместе?**

Это самая важная часть. Две инструкции взаимодействуют по следующему правилу:

| ENTRYPOINT | CMD | Что выполнится при `docker run` | Что выполнится при `docker run [image] [arg]` |
| :--- | :--- | :--- | :--- |
| Не задан | `CMD ["cmd", "param"]` | **`cmd param`** | **`<arg>`** (CMD полностью игнорируется) |
| `ENTRYPOINT ["entry"]` | Не задан | **`entry`** | **`entry <arg>`** |
| `ENTRYPOINT ["entry"]` | `CMD ["cmd", "param"]` | **`entry cmd param`** | **`entry <arg>`** (CMD игнорируется) |

**Проще говоря:**
*   Аргументы из `docker run` заменяют `CMD` и добавляются к `ENTRYPOINT`.
*   Если нужно переопределить `ENTRYPOINT`, это можно сделать флагом `--entrypoint`: `docker run --entrypoint /bin/bash my-image`

---

### **Примеры из жизни**

**Пример 1: Утилита для запросов (лучший случай для `ENTRYPOINT`)**
```dockerfile
FROM alpine
RUN apk add --no-cache curl
ENTRYPOINT ["curl"]
CMD ["-I", "https://google.com"]
```
*   `docker run my-curl` → выполнится `curl -I https://google.com`
*   `docker run my-curl -v https://ya.ru` → выполнится `curl -v https://ya.ru`

**Пример 2: Контейнер с приложением (лучший случай для `CMD`)**
```dockerfile
FROM python:3.9
COPY app.py .
CMD ["python", "app.py"]
```
*   `docker run my-app` → запустится `python app.py`
*   `docker run -it my-app /bin/bash` → вы попадете в bash, а приложение не запустится. Это нужно для дебага.

**Пример 3: Жестко заданная команда (только `ENTRYPOINT`)**
```dockerfile
FROM my-base-image
COPY my-script.sh .
RUN chmod +x my-script.sh
ENTRYPOINT ["./my-script.sh"]
```
*   Этот скрипт запустится всегда, независимо от аргументов `docker run`. Аргументы будут переданы скрипту как `$1, $2...`.

---

# **Best Practices для Dockerfile**

### **1. Используйте официальные и минимальные базовые образы**
*   **Плохо:** `FROM ubuntu:latest` (большой образ, может содержать уязвимости).
*   **Хорошо:** `FROM python:3.9-slim` или `FROM alpine:3.16` (меньше поверхность атаки, меньше размер).
*   **Для GO:** `FROM golang:1.19-alpine AS builder` -> `FROM scratch` (минимальный образ).

### **2. Указывайте точную версию тега (не latest)**
*   **Плохо:** `FROM node:latest`
*   **Хорошо:** `FROM node:18.12.1-slim`
*   **Зачем:** Это обеспечивает детерминированность сборки. Образ `latest` сегодня и завтра — это два разных образа.

### **3. Оптимизируйте кэширование слоев: меняющиеся инструкции — в конец**
Сборка происходит послойно. Docker кэширует слои. Изменился слой — все последующие слои пересобираются.
```dockerfile
# ПЛОХО: Кэш собьется при любом изменении в коде
COPY . .
RUN apt update && apt install -y python3-pip
RUN pip install -r requirements.txt

# ХОРОШО: Зависимости установятся один раз и закэшируются
RUN apt update && apt install -y python3-pip
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

### **4. Объединяйте RUN-инструкции и очищайте кэш**
Каждая инструкция `RUN` создает новый слой. Минимизируйте их количество.
```dockerfile
# ПЛОХО: 3 слоя
RUN apt update
RUN apt install -y curl
RUN rm -rf /var/lib/apt/lists/*

# ХОРОШО: 1 слой, кэш очищен
RUN apt update && apt install -y curl \
    && rm -rf /var/lib/apt/lists/*
```

### **5. Используйте .dockerignore**
Создайте файл `.dockerignore` в той же директории, что и `Dockerfile`. Он работает как `.gitignore` и исключает ненужные файлы из контекста сборки, ускоряя ее и делая образ безопаснее.
```
**/node_modules
**/.git
**/.env
**/README.md
**/*.log
Dockerfile
.dockerignore
```

### **6. Используйте multi-stage builds для production**
Это золотой стандарт. Собирайте приложение в одном образе (со всеми компиляторами), а копируйте только готовый артефакт в чистый финальный образ.
```dockerfile
# STAGE 1: Build
FROM golang:1.19 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp .

# STAGE 2: Run (минимальный образ)
FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/myapp .
CMD ["./myapp"]
```

### **7. Используйте не-root пользователя**
Запуск от root внутри контейнера — это security anti-pattern.
```dockerfile
FROM node:16-slim
# Создаем пользователя и группу
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
# Меняем владельца файлов
RUN chown -R appuser:appgroup /app
USER appuser # Переключаемся на non-root пользователя
CMD ["node", "index.js"]
```

### **8. Отдавайте предпочтение форме Exec для CMD и ENTRYPOINT**
*   **Shell form:** `CMD python app.py` (запускается через `/bin/sh -c`, плохо обрабатывает сигналы остановки).
*   **Exec form (рекомендуется):** `CMD ["python", "app.py"]` (процесс становится PID 1, правильно получает сигналы).

### **9. Указывайте метаданные (LABEL)**
Это помогает управлять образами.
```dockerfile
LABEL maintainer="team@example.com"
LABEL version="1.0"
LABEL description="This is a cool app"
```

### **10. Сканируйте образы на уязвимости**
Используйте `docker scan` (или Trivy, Grype) в CI/CD пайплайне.
```bash
docker build -t myapp:1.0 .
docker scan myapp:1.0
```

---

# **Best Practices для Docker Compose**

### **1. Используйте правильную версию schema**
Всегда указывайте версию. Для современных возможностей используйте `3.8` или выше.
```yaml
version: '3.8'
services:
  web:
    ...
```

### **2. Не используйте `latest` тег для образов**
То же правило, что и в Dockerfile.
```yaml
services:
  web:
    # ПЛОХО
    image: nginx:latest
    # ХОРОШО
    image: nginx:1.23.3
```

### **3. Храните конфиденциальные данные в секретах**
Никогда не пишите пароли в plain text в `docker-compose.yml`.
**Способ 1: Переменные окружения из `.env` файла**
```yaml
# .env
DB_PASSWORD=supersecret
```
```yaml
# docker-compose.yml
services:
  db:
    image: postgres
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
```

**Способ 2: Docker Secrets (в Swarm mode)**
```yaml
services:
  db:
    image: postgres
    secrets:
      - db_password
secrets:
  db_password:
    file: ./db_password.txt
```

### **4. Явно определяйте сети (Networks)**
Не полагайтесь на сеть по умолчанию. Создавайте изолированные сети для сервисов.
```yaml
services:
  web:
    networks:
      - frontend
  db:
    networks:
      - backend

networks:
  frontend:
  backend:
```
Это улучшает безопасность и изоляцию (веб-сервер не сможет напрямую подключиться к БД, если они в разных сетях).

### **5. Явно определяйте тома (Volumes)**
Именованные тома предпочтительнее bind mounts для production, так как ими управляет Docker.
```yaml
services:
  db:
    image: postgres
    volumes:
      # Именованный том (рекомендуется для данных)
      - db_data:/var/lib/postgresql/data
      # Bind mount (для конфигов или кода)
      - ./postgres.conf:/etc/postgresql.conf:ro # 'ro' - только для чтения

volumes:
  db_data: # Том будет создан автоматически
```

### **6. Настраивайте healthchecks**
Это критически важно для определения работоспособности сервисов и правильной работы `depends_on`.
```yaml
services:
  web:
    image: nginx
    depends_on:
      db:
        condition: service_healthy # Ждать, пока БД не станет healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
  db:
    image: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
```

### **7. Ограничивайте ресурсы (CPU, Memory)**
Не позволяйте контейнерам съедать все ресурсы хоста.
```yaml
services:
  web:
    image: nginx
    deploy: # Работает и в обычном Compose
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
```

### **8. Используйте профили (profiles)**
Запускайте сервисы только когда нужно (например, для разработки, но не для production).
```yaml
services:
  frontend:
    # ... 
  dev-tools:
    profiles: ["dev"] # Сервис будет запущен только с флагом --profile dev
    image: debug-tools

  backend:
    profiles: ["prod", "dev"] # Сервис будет запущен для prod и dev
    image: backend
```
Запуск: `docker compose --profile dev up`

### **9. Декомпозируйте большие конфиги**
С помощью `extends` (в старых версиях) или включения других файлов.
**Способ 1 (современный): `include`**
```yaml
# docker-compose.yml
include:
  - backend-compose.yml
  - frontend-compose.yml
```

### **10. Не используйте Docker Compose для Production напрямую**
Docker Compose — отличный инструмент для разработки, тестирования и staging. Для production используйте оркестраторы:
*   **Docker Swarm:** `docker stack deploy -c docker-compose.yml myapp` (поддерживает не все опции Compose).
*   **Kubernetes:** С помощью `kompose convert` можно сгенерить манифесты, но лучше писать их руками под требования K8s.

---

## **Чеклист для идеального образа и композа**

**Для Dockerfile:**
1.  [ ] `FROM` с точным тегом и slim-образ.
2.  [ ] Оптимизированный порядок инструкций.
3.  [ ] Объединенные `RUN` с очисткой кэша.
4.  [ ] Файл `.dockerignore`.
5.  [ ] Multi-stage build.
6.  [ ] Non-root пользователь.
7.  [ ] `CMD` в exec-форме.
8.  [ ] Просканирован на уязвимости.

**Для Docker Compose:**
1.  [ ] Версия `3.8`+.
2.  [ ] Явные теги образов.
3.  [ ] Секреты в `.env`, а не в коде.
4.  [ ] Явные сети и тома.
5.  [ ] Настроенные healthchecks.
6.  [ ] Лимиты на ресурсы.
7.  [ ] Правильный `depends_on` с `condition: service_healthy`.
