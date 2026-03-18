
# Структура и создание Helm-чартов

Helm-чарт — это набор файлов и директорий, организованных определённым образом для упаковки и развёртывания Kubernetes-приложений.

## Минимальная структура чарта

Для работы простейшего чарта необходимы три компонента:

- **`Chart.yaml`** — метаданные чарта
- **`values.yaml`** — значения по умолчанию
- **`templates/`** — директория с шаблонами манифестов

Остальные файлы (`README.md`, `LICENSE`, `charts/`, `values.schema.json`) опциональны.

## Полная структура чарта

```
mychart/
├── Chart.yaml                # метаданные чарта (обязательный)
├── values.yaml               # значения по умолчанию (обязательный)
├── templates/                # шаблоны манифестов (обязательный)
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── _helpers.tpl          # вспомогательные шаблоны
│   └── NOTES.txt             # инструкции после установки
├── charts/                   # зависимости (сабчарты)
├── values.schema.json        # схема валидации значений
├── LICENSE                   # лицензия
└── README.md                 # документация
```

## Основные компоненты

### Chart.yaml — метаданные чарта

Обязательный файл с информацией о чарте:

```yaml
apiVersion: v2           # версия API чарта (v2 для Helm 3)
name: mychart            # имя чарта
version: 0.1.0           # версия чарта (SemVer2)
description: A Helm chart for Kubernetes  # описание (необязательно)
appVersion: "1.16.0"     # версия приложения (необязательно)
```

### values.yaml — значения по умолчанию

Содержит переменные, которые пользователь может переопределить при установке:

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.16.0"

service:
  port: 80
```

### templates/ — шаблоны манифестов

Директория с Go-шаблонами, которые рендерятся в Kubernetes-манифесты. Пример `templates/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
```

## Дополнительные файлы

### values.schema.json — валидация значений

JSON-схема для проверки входных данных:

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "properties": {
    "port": {
      "description": "Port",
      "minimum": 0,
      "type": "integer"
    },
    "protocol": {
      "type": "string"
    }
  },
  "required": ["protocol", "port"],
  "type": "object"
}
```

### NOTES.txt — инструкции после установки

Файл в `templates/`, содержимое которого выводится после успешной установки:

```
Thank you for installing {{ .Chart.Name }}.

Your application is available at:
  http://{{ .Values.service.name }}:{{ .Values.service.port }}

Enjoy!
```

### charts/ — зависимости

Директория для хранения сабчартов (зависимостей). Сюда помещаются другие чарты, от которых зависит текущий.

## Создание чарта

### Генерация скелета

```bash
helm create mychart
```

Команда создаст структуру:

```
mychart/
├── charts/
├── Chart.yaml
├── templates/
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt
│   ├── serviceaccount.yaml
│   ├── service.yaml
│   └── tests/
│       └── test-connection.yaml
└── values.yaml
```

## Проверка чарта

### 1. Линтинг — проверка синтаксиса

```bash
helm lint mychart/
```

Проверяет YAML-разметку и структуру чарта.

### 2. Проверка шаблонов

```bash
helm template mychart/
```

Показывает результат рендеринга шаблонов без установки.

### 3. Пробный запуск (dry-run)

```bash
helm install my-release mychart/ --dry-run
```

Имитирует установку в кластер, проверяет, примет ли Kubernetes манифесты.

### Упаковка чарта

```bash
helm package mychart/
# Результат: mychart-0.1.0.tgz
```

## Параметризация шаблонов

### Встроенные объекты Helm

| Объект | Описание |
|--------|----------|
| `.Values` | Значения из `values.yaml` и `--set` |
| `.Release` | Информация о релизе (имя, версия и т.д.) |
| `.Chart` | Метаданные из `Chart.yaml` |
| `.Files` | Доступ к файлам чарта |
| `.Capabilities` | Информация о кластере |

### Переменные и подстановки

```yaml
name: {{ .Release.Name }}-deployment
replicas: {{ .Values.replicaCount }}
image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
```

### Функции и конвейеры (pipeline)

```yaml
replicas: {{ .Values.replicaCount | default 1 }}
image: {{ .Values.image.tag | default "latest" | quote }}
```

Популярные функции:
- `default` — значение по умолчанию
- `quote` — заключает в кавычки
- `upper` / `lower` — регистр
- `indent` / `nindent` — добавляет отступы

### Условные конструкции

```yaml
metadata:
  name: {{ .Release.Name }}-deployment
  {{- if .Values.nameLabel }}
  labels:
    name: {{ .Values.nameLabel }}
  {{- end }}
```

`{{-` удаляет пробелы слева, `-}}` — справа.

### Циклы (range)

```yaml
containers:
  {{- range .Values.containers }}
  - name: {{ .name }}
    image: {{ .image }}
  {{- end }}
```

Внутри цикла `.` ссылается на текущий элемент.

## Лучшие практики

### Именование
- Имя чарта: **строчные буквы и цифры**, слова через дефис (`nginx`, `argo-cd`)
- Переменные: **camelCase** с первой строчной (`replicaCount`, `revisionHistoryLimit`)
- Файлы шаблонов: **дефисная нотация** (`deployment.yaml`, `service.yaml`)

### Версионирование
- Версия чарта: **SemVer2** (`MAJOR.MINOR.PATCH`)
- При изменении MAJOR сбрасываются MINOR и PATCH
- При изменении MINOR сбрасывается PATCH

### values.yaml
- **Предпочитать плоскую структуру** вложенной (проще для пользователей)
- **Документировать все переменные**:
  ```yaml
  # serverPort is the HTTP listener port
  serverPort: 9191
  ```
- **Явно указывать типы**: строки в кавычках, числа без кавычек
- Для больших чисел использовать строки: `number: "123456789"`

### templates
- Один ресурс — один файл
- Имена файлов отражают тип ресурса (`deployment.yaml`, `svc.yaml`)
- Отступы — **2 пробела** (не табуляция)

### Документирование
- `README.md` — описание, установка, настройка
- `NOTES.txt` — инструкции после установки
- `LICENSE` — лицензионное соглашение
