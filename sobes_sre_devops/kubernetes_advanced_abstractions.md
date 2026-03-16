# Kubernetes: Продвинутые Абстракции — Шпаргалка

## 1) StatefulSet
**Когда использовать:** stateful-нагрузки (Kafka, БД, очереди), где важны стабильные имена и постоянные тома.

**Ключевые свойства:**
- Стабильные имена Pod: `name-0`, `name-1`, ...
- Упорядоченный запуск/остановка/обновление.
- Отдельный PVC на каждый Pod через `volumeClaimTemplates`.
- Обычно нужен headless Service (`clusterIP: None`).

**Запомнить для интервью:**
- Deployment для stateless, StatefulSet для stateful.
- Удаление Pod не удаляет его PVC автоматически.

## 2) DaemonSet
**Когда использовать:** один Pod на каждой ноде.

**Типичные кейсы:**
- сбор логов (Fluent Bit/Fluentd),
- сбор метрик (node-exporter),
- сетевые агенты (CNI),
- security-агенты.

**Ключевые свойства:**
- Новый узел добавился -> Pod DaemonSet запускается автоматически.
- Узел удалился -> соответствующий Pod пропадает.
- Можно ограничивать узлы через `nodeSelector`/`affinity`/`tolerations`.

## 3) Job и CronJob

### Job
Разовая задача до успешного завершения.

Важно:
- `restartPolicy: OnFailure|Never`
- `backoffLimit` — число повторных попыток.

### CronJob
Запуск Job по расписанию.

Важно:
- `spec.schedule` (cron-выражение)
- `successfulJobsHistoryLimit` / `failedJobsHistoryLimit`
- `spec.suspend: true` — приостановка запуска новых задач.

## 4) Планирование Pod

### NodeSelector
Самый простой фильтр по меткам ноды.

### Taints / Tolerations
- `taint` на ноде ограничивает посадку Pod.
- `toleration` в Pod разрешает посадку на tainted-ноду.

Эффекты taint:
- `NoSchedule` — новые Pod без toleration не сядут.
- `PreferNoSchedule` — желательно не сажать.
- `NoExecute` — выгоняет уже запущенные Pod без toleration.

### Affinity / Anti-Affinity
Более гибкие правила размещения:
- `nodeAffinity` — требования к ноде,
- `podAffinity` — ставить рядом с нужными Pod,
- `podAntiAffinity` — разносить Pod по узлам/зонам.

**Для отказоустойчивости:**
- anti-affinity по `kubernetes.io/hostname` для разнесения реплик.

## 5) Init-контейнеры
Запускаются **до** основного контейнера и должны завершиться успешно.

**Когда нужны:**
- подготовка файлов/конфигов,
- ожидание зависимости,
- миграции/инициализация,
- выставление прав,
- прогрев кэша.

**Важно:**
- Init-контейнеры выполняются последовательно.
- Пока они не завершены, приложение не стартует.

## 6) Что отвечать кратко (30–60 секунд)

### StatefulSet vs Deployment
Deployment — для stateless. StatefulSet — для stateful: стабильные имена, порядок, персональные PVC.

### Job vs CronJob
Job — один запуск до результата. CronJob — периодические запуски по расписанию.

### NodeSelector vs Affinity
NodeSelector — простое совпадение метки. Affinity — гибкая логика с обязательными и предпочтительными правилами.

### Taints/Tolerations
Taint «закрывает» ноду от неподходящих Pod, toleration дает конкретному Pod право сесть на такую ноду.

### Init-контейнеры
Подготавливают окружение до старта приложения: зависимости, файлы, миграции, проверки готовности.

## 7) Практический минимум команд
```bash
# StatefulSet / DaemonSet / Job / CronJob
kubectl get sts,ds,job,cronjob -A
kubectl describe sts <name> -n <ns>
kubectl describe cronjob <name> -n <ns>

# Проверка планирования
kubectl get pod <pod> -n <ns> -o wide
kubectl describe pod <pod> -n <ns>
kubectl get nodes --show-labels
kubectl describe node <node> | grep -i -A3 Taints

# Проверка PVC для StatefulSet
kubectl get pvc -n <ns>
```

## 8) Частые ошибки
- Использовать Deployment для stateful БД/очередей.
- Не делать anti-affinity для критичных реплик.
- Не ограничивать DaemonSet по узлам (агент ставится везде).
- Забывать чистить историю CronJob/Job.
- Держать логику инициализации в основном контейнере вместо init-контейнера.
