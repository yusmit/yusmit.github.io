# Kubernetes: Продвинутые Абстракции (Конспект Урока)

## Что разберем
- StatefulSet и DaemonSet.
- Job и CronJob.
- Pod Scheduling: NodeSelector, Taints/Tolerations, Affinity/Anti-Affinity.
- Init-контейнеры.

## 1. StatefulSet

`StatefulSet` нужен для stateful-нагрузок, где важны:
- стабильные имена pod;
- стабильные сетевые идентификаторы;
- персональное постоянное хранилище для каждого pod;
- упорядоченное создание/удаление/обновление.

Типовые кейсы:
- базы данных;
- брокеры сообщений (Kafka и т.п.);
- key-value хранилища.

### Ключевые свойства
- Имена pod по шаблону: `<statefulset-name>-<ordinal>` (например: `kafka-0`, `kafka-1`, `kafka-2`).
- Каждый pod получает свой PVC через `volumeClaimTemplates`.
- Нужен headless Service (`clusterIP: None`) для стабильной DNS-идентификации.

### Минимальный пример (Kafka)
{% raw %}
```yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka
  labels:
    app: kafka
spec:
  clusterIP: None
  selector:
    app: kafka
  ports:
  - name: kafka
    port: 9092
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
spec:
  serviceName: kafka
  replicas: 3
  selector:
    matchLabels:
      app: kafka
  template:
    metadata:
      labels:
        app: kafka
    spec:
      containers:
      - name: kafka
        image: confluentinc/cp-kafka:latest
        ports:
        - containerPort: 9092
          name: kafka
        volumeMounts:
        - name: kafka-storage
          mountPath: /var/lib/kafka/data
  volumeClaimTemplates:
  - metadata:
      name: kafka-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```
{% endraw %}

## 2. DaemonSet

`DaemonSet` гарантирует запуск pod на каждом подходящем узле.

Типовые кейсы:
- node-level лог-агенты (Fluentd/Vector);
- node-exporter и другие системные метрики;
- CNI/сетевые агенты.

### Особенности
- На каждый узел - один pod DaemonSet (если узел подходит по селекторам).
- Новый узел в кластере -> pod создается автоматически.
- Удалили узел -> pod исчезает вместе с ним (не мигрирует).

### Пример DaemonSet (лог-агент)
{% raw %}
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      containers:
      - name: fluentd
        image: fluent/fluentd:latest
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```
{% endraw %}

## 3. Job и CronJob

## Job
`Job` выполняет задачу до успешного завершения.

Когда использовать:
- миграции БД;
- разовые бэкапы;
- batch-обработку.

Ключевые поля:
- `restartPolicy: OnFailure | Never`;
- `backoffLimit` (сколько повторов при ошибке).

## CronJob
`CronJob` запускает Job по расписанию (как cron).

Ключевые поля:
- `spec.schedule` — cron-выражение;
- `successfulJobsHistoryLimit`;
- `failedJobsHistoryLimit`;
- `spec.suspend: true|false` — пауза расписания.

### Когда что использовать
- `Job`: единичная задача «прямо сейчас».
- `CronJob`: регулярная задача по времени.

## 4. Pod Scheduling

## NodeSelector
Самый простой механизм: pod запускается только на узлах с нужной меткой.

```
nodeSelector:
  disktype: ssd
```

## Taints и Tolerations
- `Taint` ставится на узел и ограничивает размещение pod.
- `Toleration` в pod разрешает запуск на узле с таким taint.

Эффекты taint:
- `NoSchedule`;
- `PreferNoSchedule`;
- `NoExecute`.

Пример toleration:
```
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
```

## Affinity и Anti-Affinity
- `nodeAffinity` — правила выбора узлов;
- `podAffinity` — размещать рядом с pod по меткам;
- `podAntiAffinity` — разносить pod по узлам/зонам.

Пример anti-affinity (не класть одинаковые pod на один узел):
```
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values: ["nginx"]
      topologyKey: kubernetes.io/hostname
```

## 5. Init-контейнеры

`initContainers` выполняются до старта основных контейнеров.

Свойства:
- запускаются строго по порядку;
- каждый должен успешно завершиться;
- только после этого стартуют app-контейнеры.

Когда полезны:
- подготовить конфиг/файлы;
- дождаться внешнего сервиса;
- прогреть кэш;
- выполнить миграции/инициализацию среды.

### Пример
{% raw %}
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
  - name: install
    image: busybox:1.28
    command: ["sh", "-c", "wget -O /work-dir/index.html http://info.cern.ch"]
    volumeMounts:
    - name: workdir
      mountPath: /work-dir
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: workdir
      mountPath: /usr/share/nginx/html
  volumes:
  - name: workdir
    emptyDir: {}
```
{% endraw %}

## Что важно запомнить
- `StatefulSet` для stateful-сервисов со стабильной идентичностью и томами.
- `DaemonSet` для node-level агентов на каждом узле.
- `Job` — разовая задача, `CronJob` — задача по расписанию.
- Scheduling управляется `NodeSelector`, `Taints/Tolerations`, `Affinity/Anti-Affinity`.
- `Init`-контейнеры подготавливают окружение до старта приложения.
