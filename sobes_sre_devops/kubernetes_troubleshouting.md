# Решение проблем с приложениями в Kubernetes с практическими примерами.

## Оглавление
1.  [Отладка Pods](#отладка-pods)
    - [Pod в состоянии Pending](#pod-в-состоянии-pending)
    - [Pod в состоянии Waiting (ImagePullBackOff)](#pod-в-состоянии-waiting-imagepullbackoff)
    - [Pod в состоянии Terminating](#pod-в-состоянии-terminating)
    - [Pod падает с ошибкой (CrashLoopBackOff)](#pod-падает-с-ошибкой-crashloopbackoff)
    - [Pod ведет себя нестандартно](#pod-ведет-себя-нестандартно)
2.  [Отладка Сервисов (Services)](#отладка-сервисов-services)
    - [Сервис недоступен (No endpoints)](#сервис-недоступен-no-endpoints)
    - [Сервис не перенаправляет трафик](#сервис-не-перенаправляет-трафик)
    - [Проблемы с LoadBalancer / NodePort](#проблемы-с-loadbalancer--nodeport)
    - [Неправильное распределение нагрузки](#неправильное-распределение-нагрузки)
    - [DNS-проблемы](#dns-проблемы)
3.  [Отладка Кластера (Узлы)](#отладка-кластера-узлы)
    - [Узел недоступен или NotReady](#узел-недоступен-или-notready)
    - [Узел не принимает Pods (Taints/Cordon)](#узел-не-принимает-pods-taintscordon)
    - [Высокая нагрузка на узел](#высокая-нагрузка-на-узел)
4.  [Практика: Отладка кривого приложения](#практика-отладка-кривого-приложения)

---

## Отладка Pods

Первый шаг в отладке Pods — просмотр их состояния и событий. Это наша точка входа в расследование.
```bash
kubectl describe pods <pod_name>
```

### Pod в состоянии Pending

Если Pod завис в статусе `Pending`, планировщик (kube-scheduler) не может назначить его на узел.

**Пример:** У нас нет узлов, которые могут выделить 4 CPU для пода.
```bash
# Проверяем события
kubectl describe pod resource-hungry-pod
...
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  10s   default-scheduler  0/3 nodes are available: 3 Insufficient cpu.
```
**Что делать:**
1.  **Нехватка ресурсов:** Удалите ненужные поды или увеличьте ресурсы кластера (добавьте узлы). Проверьте запросы ресурсов в своей спецификации Pod.
2.  **`hostPort`:** Если вы используете `hostPort`, это сильно ограничивает планирование. Вместо этого используйте `Service` типа `NodePort` или `LoadBalancer`.
3.  **PersistentVolume (PVC):** Под не запланируется, пока требуемый PVC не будет связан (bound) с PV.
    ```bash
    kubectl get pvc
    # Ищите PVC в статусе Pending
    ```

### Pod в состоянии Waiting (ImagePullBackOff)

Pod назначен на узел, но не может запуститься. Самая частая причина — проблемы с образом.

**Пример:** Опечатка в имени образа.
```bash
kubectl get pods
NAME                        READY   STATUS             RESTARTS   AGE
my-app-6b4b9c7b68-abc123    0/1     ImagePullBackOff   0          2m

kubectl describe pod my-app-6b4b9c7b68-abc123
...
Events:
  Type     Reason                 Age   From                Message
  ----     ------                 ----  ----                -------
  Normal   Pulling                15s   kubelet, node-1     Pulling image "my-app:lates" # Опечатка!
  Warning  Failed                 15s   kubelet, node-1     Failed to pull image "my-app:lates": rpc error: ...
  Warning  Failed                 15s   kubelet, node-1     Error: ErrImagePull
  Normal   BackOff                5s    kubelet, node-1     Back-off pulling image "my-app:lates"
```
**Что делать:**
*   Проверьте имя образа и тег.
*   Убедитесь, что образ загружен в registry (Docker Hub, Yandex Container Registry и т.д.).
*   Если registry приватный, проверьте наличие и корректность `imagePullSecrets`.
*   Попробуйте выполнить `docker pull <image>` вручную на узле, чтобы проверить доступность.

### Pod в состоянии Terminating

Если Pod завис в статусе `Terminating`, это часто означает, что процесс удаления заблокирован.

**Пример:** Вебхук (например, проверяющий безопасность) блокирует удаление.
```bash
kubectl get pods
NAME                      READY   STATUS        RESTARTS   AGE
my-app-6b4b9c7b68-xyz789  1/1     Terminating   0          10m

# Проверяем, нет ли финализаторов, которые не могут отработать
kubectl get pod my-app-6b4b9c7b68-xyz789 -o yaml | grep finalizers
#   finalizers:
#   - some.security.webhook/finalizer
```
**Что делать:**
1.  Проверьте наличие и логи `ValidatingWebhookConfiguration` или `MutatingWebhookConfiguration`, которые могут относится к этому поду.
2.  Если нужно срочно удалить под (например, для дебага), можно убрать финализаторы:
    ```bash
    # ВНИМАНИЕ: Это принудительное удаление!
    kubectl patch pod <pod-name> -p '{"metadata":{"finalizers":[]}}' --type=merge
    ```

### Pod падает с ошибкой (CrashLoopBackOff)

Pod запускается, но сразу падает, и Kubernetes пытается перезапустить его снова и снова.

**Пример:** Приложение внутри пода падает с ошибкой "port already in use".
```bash
kubectl get pods
NAME                        READY   STATUS             RESTARTS      AGE
crashing-app-6b4b9c7b68     0/1     CrashLoopBackOff   5 (10s ago)   2m

# Смотрим логи — это первый и главный источник информации
kubectl logs crashing-app-6b4b9c7b68
# Error: listen tcp :8080: bind: address already in use

# Если логов нет, проверяем события
kubectl describe pod crashing-app-6b4b9c7b68
...
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Started    40s (x4 over 2m)   kubelet            Started container my-container
  Normal   Created    39s (x5 over 2m)   kubelet            Created container my-container
  Warning  BackOff    20s (x7 over 2m)   kubelet            Back-off restarting failed container
```

**Что делать:**
*   **Смотрите логи!** (`kubectl logs <pod-name> --previous` если контейнер уже перезапустился).
*   Проверьте настройки `livenessProbe` и `readinessProbe`. Возможно, проба настроена неправильно (слишком строгий таймаут, неверный путь) и убивает контейнер. Временное отключение пробы поможет это проверить.

### Pod ведет себя нестандартно

Приложение работает, но отвечает не теми данными или выдает ошибки в половине случаев.

**Пример:** В манифесте Deployment была опечатка в переменной окружения.
{% raw %}
```yaml
# В манифесте ошибка: DATABSE_URL вместо DATABASE_URL
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
      - name: app
        env:
        - name: DATABSE_URL  # <-- Опечатка
          value: "postgres://db:5432"
```
{% endraw %}

**Что делать:**
1.  **Интерактивная отладка:** Подключитесь к запущенному контейнеру и проверьте переменные окружения или файлы конфигурации.
    ```bash
    kubectl exec -it my-app-pod -- /bin/sh
    # (внутри пода) env | grep DATABASE
    # (внутри пода) cat /etc/config/app.conf
    ```
2.  **Валидация манифеста:** Если вы подозреваете ошибку в YAML, удалите под (он пересоздастся) или используйте `--validate` (хотя `kubectl apply` делает это по умолчанию). Для более тщательной проверки используйте специализированные инструменты, например `kubeval`.
3.  **Сравнение конфигов:** Сравните то, что вы отправляли, с тем, что есть в кластере.
    ```bash
    kubectl get deployment my-app -o yaml > deployment_on_server.yaml
    # Сравните этот файл с вашим локальным манифестом
    ```

## Отладка Сервисов (Services)

Вы создали Deployment и Service, но приложение недоступно. Действуем по плану.

### Сервис недоступен (No endpoints)

Самая частая причина — селектор сервиса не соответствует меткам подов.

**Пример:** Под имеет метку `app: my-app`, а сервис ищет `app: my-app-frontend`.
```bash
# Проверяем поды и их метки
kubectl get pods --show-labels
NAME                      READY   STATUS    LABELS
my-app-6b4b9c7b68-abc123  1/1     Running   app=my-app,pod-template-hash=...

# Проверяем сервис
kubectl describe service my-app-service
Name:                     my-app-service
Selector:                 app=my-app-frontend  # <-- Не совпадает!
...
Endpoints:                <none>               # <-- Ключевой признак!

# Проверяем endpoints напрямую
kubectl get endpoints my-app-service
NAME             ENDPOINTS   AGE
my-app-service   <none>      5m
```
**Что делать:** Исправьте селектор в сервисе, чтобы он совпадал с метками пода.

### Сервис не перенаправляет трафик

Endpoints есть, но при обращении к сервису получаете timeout или connection refused.

**Пример:** Поды слушают на порту `5000`, а сервис настроен на порт `8080`.
```bash
kubectl get endpoints my-app-service
NAME             ENDPOINTS                                   AGE
my-app-service   10.244.1.2:5000,10.244.2.3:5000            2m

# Проверяем targetPort в сервисе
kubectl get service my-app-service -o yaml | grep targetPort
    targetPort: 8080  # <-- Несоответствие!
```

**Что делать:**
*   Убедитесь, что `targetPort` в сервисе соответствует `containerPort`, который слушает ваше приложение.
*   Проверьте, что само приложение в поде запустилось и слушает порт. Сделайте `curl localhost:5000` внутри пода через `kubectl exec`.

### Проблемы с LoadBalancer / NodePort

Сервис типа `LoadBalancer` не выдает внешний IP.

```bash
kubectl get svc my-app-service
NAME             TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
my-app-service   LoadBalancer   10.96.123.45   <pending>     80:31234/TCP   5m
```
**Что делать:**
*   Если вы в облаке (Yandex Cloud, AWS, GCP), убедитесь, что контроллер облачного балансировщика работает.
*   Проверьте квоты и политики безопасности в облаке.
*   Для `NodePort` проверьте, что правила файрвола (Security Groups) разрешают трафик на порт NodePort (в примере выше это `31234`) с вашего IP-адреса.

### Неправильное распределение нагрузки

Трафик идет только на один под из пяти.

**Что делать:**
*   **Readiness Probe:** Проверьте, все ли поды считаются готовыми принимать трафик.
    ```bash
    kubectl get pods
    NAME                      READY   STATUS    RESTARTS   AGE
    my-app-6b4b9c7b68-abc123  1/1     Running   0          1m
    my-app-6b4b9c7b68-def456  0/1     Running   0          1m  # <-- Readiness probe failed!
    ```
    Опишите проблемный под, чтобы узнать, почему проба готовности не проходит.

### DNS-проблемы

Не удается обратиться к сервису по имени из другого пода.

**Что делать:**
*   **Проверка:** Зайдите в любой под и попробуйте выполнить nslookup.
    ```bash
    kubectl run -it --rm debug --image=nicolaka/netshoot -- /bin/bash
    bash-5.1# nslookup my-app-service.default.svc.cluster.local
    ```
*   **Разные Namespace:** Если сервис в namespace `default`, а под в `test`, используйте полное доменное имя: `my-app-service.default.svc.cluster.local` или короткую форму `my-app-service.default`.

## Отладка Кластера (Узлы)

Если с приложением все ок, проверяем состояние узлов.
```bash
kubectl get nodes
```

### Узел недоступен или NotReady

Узел в статусе `NotReady`.

**Что делать:**
1.  **Опишите узел:** часто там бывают подсказки.
    ```bash
    kubectl describe node <unhealthy-node>
    ...
    Conditions:
      Type             Status  LastHeartbeatTime  Reason
      ----             ------  -----------------  ------
      MemoryPressure   False   ...                KubeletHasSufficientMemory
      DiskPressure     True    ...                KubeletHasDiskPressure   # <-- Проблема!
      ...
    ```
2.  **Зайдите на сам узел** (если есть доступ) и проверьте:
    *   Статус kubelet: `systemctl status kubelet` или `journalctl -u kubelet -f`.
    *   Свободное место на диске (`df -h`).
    *   Сетевые подключения.

### Узел не принимает Pods (Taints/Cordon)

Узел есть, но поды на него не ставятся.

**Причина 1: Узел закордонен администратором.**
```bash
kubectl describe node worker-1
...
Taints:             node.kubernetes.io/unschedulable:NoSchedule
...
```
**Решение:** `kubectl uncordon worker-1`

**Причина 2: На узле есть "грязные пятна" (Taints), а у пода нет терпимости (Tolerations).**
```bash
kubectl describe node gpu-node
...
Taints:             nvidia.com/gpu=present:NoSchedule  # Такие поды просто так не зайдут
```
**Решение:** Добавьте соответствующую `tolerations` в спецификацию вашего пода.

### Высокая нагрузка на узел

Узел работает медленно.

**Что делать:**
*   Используйте `kubectl top` для быстрой диагностики.
    ```bash
    # Смотрим загрузку узлов
    kubectl top node
    NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
    worker-1   3500m        87%    12Gi            75%

    # Смотрим, какие поды жрут ресурсы на этом узле
    kubectl top pod --all-namespaces --field-selector spec.nodeName=worker-1
    ```
*   Пересмотрите `requests` и `limits` для подов, чтобы узел не был перегружен.

## Практика: Отладка кривого приложения

Давайте представим, что мы развернули приложение, и оно не работает. Пройдем путь от начала до конца.

**Задача:** Развернуть простое веб-приложение, которое должно отвечать "Hello, World!" на порту 8080.

**Проблема:** После развертывания, сервис не отвечает.

### Шаг 1: Деплоим "кривое" приложение
```bash
# Создаем deployment с багом
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-buggy-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: buggy-app
  template:
    metadata:
      labels:
        app: buggy-app-frontend  # <--- БАГ 1: Метка не совпадет с сервисом позже
    spec:
      containers:
      - name: app
        image: nginx:latest  # <--- БАГ 2: Используем nginx вместо нашего приложения
        ports:
        - containerPort: 8080 # <--- БАГ 3: nginx слушает 80, а мы указываем 8080
        env:
        - name: MESSAGE
          value: "Hello, World!"
        # Баг 4: Нет команды для запуска кастомного приложения, nginx просто запустится сам
---
apiVersion: v1
kind: Service
metadata:
  name: buggy-service
spec:
  selector:
    app: buggy-app  # Селектор ищет поды с меткой app=buggy-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: NodePort
EOF
```

### Шаг 2: Наблюдаем за симптомами

```bash
kubectl get pods,svc
# Видим, что под есть, но воркнется ли он?
# Сервис есть, но работает ли?

# Получаем порт NodePort для проверки
NODE_PORT=$(kubectl get svc buggy-service -o jsonpath="{.spec.ports[0].nodePort}")
# Пытаемся достучаться (worker-ip нужно заменить на реальный IP узла)
curl http://<worker-ip>:$NODE_PORT
# curl: (7) Failed to connect... - Не работает
```

### Шаг 3: Диагностика

**Шаг 3.1: Проверяем под**
```bash
kubectl get pods --show-labels
NAME                             READY   STATUS    RESTARTS   AGE   LABELS
my-buggy-app-6b4b9c7b68-xyz789   1/1     Running   0          2m    app=buggy-app-frontend,pod-template-hash=...
```
*   **Находка:** Метка пода — `app=buggy-app-frontend`, а не `app=buggy-app`. Это БАГ 1.

**Шаг 3.2: Проверяем сервис и endpoints**
```bash
kubectl describe service buggy-service
...
Selector:                 app=buggy-app
...
Endpoints:                <none>   # <-- Ключевая улика! Нет подов за сервисом.
```
*   **Вывод:** Сервис не видит под из-за несовпадения меток.

**Шаг 3.3: Исправляем первичную проблему**
Давайте временно исправим метку пода, чтобы проверить дальше. Так как под управляется Deployment, проще всего исправить сам Deployment.
```bash
# Правим deployment, меняем метку пода с 'app: buggy-app-frontend' на 'app: buggy-app'
kubectl edit deployment my-buggy-app
# Меняем строку app: buggy-app-frontend на app: buggy-app в секции template.metadata.labels
# Сохраняем и выходим. Deployment пересоздаст под с правильной меткой.
```

**Шаг 3.4: Проверяем снова**
```bash
kubectl get pods --show-labels
# Теперь у пода метка app=buggy-app
kubectl get endpoints buggy-service
NAME            ENDPOINTS                               AGE
buggy-service   10.244.1.10:8080                        1m   # <-- Появился endpoint!
```
Ура, эндпоинты появились. Пробуем снова обратиться к сервису:
```bash
curl http://<worker-ip>:$NODE_PORT
# curl: (7) Failed to connect... - Все еще не работает
```

**Шаг 3.5: Смотрим внутрь пода**
Под есть, эндпоинты есть, но трафик не доходит. Подключаемся к поду.
```bash
# Проверяем, какие процессы внутри
kubectl exec -it my-buggy-app-<new-hash> -- ps aux
# Видим процесс nginx: master process

# Проверяем, на каком порту слушает приложение
kubectl exec -it my-buggy-app-<new-hash> -- netstat -tulpn | grep LISTEN
# tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      1/nginx: master pro
```
*   **Находка:** Внутри пода приложение (nginx) слушает на порту `80`. А наш targetPort в сервисе и containerPort в поде указаны как `8080`. Это БАГ 2 и 3.
*   Проверяем переменную окружения:
    ```bash
    kubectl exec -it my-buggy-app-<new-hash> -- env | grep MESSAGE
    # MESSAGE=Hello, World!
    ```
    Переменная есть, но nginx ее не читает, потому что мы просто запустили стандартный nginx, а не свое приложение. Это БАГ 4.

### Шаг 4: Окончательное исправление

Мы поняли, что манифест полностью не соответствует приложению. Правильный манифест для нашего "Hello, World!" приложения (например, написанного на Python или Node.js) должен был бы выглядеть иначе.

Мы исправим текущую ситуацию, настроив nginx правильно (для демонстрации) или, что правильнее, заменив образ на правильный.

**Вариант быстрого фикса:** Изменим `targetPort` в сервисе с 8080 на 80.

```bash
kubectl edit service buggy-service
# Меняем targetPort: 8080 на targetPort: 80
```

### Шаг 5: Финальная проверка

```bash
# Ждем немного и пробуем снова
curl http://<worker-ip>:$NODE_PORT
# <!DOCTYPE html>
# <html>
# <head>
# <title>Welcome to nginx!</title>  # <-- Ура, ответ есть! (Это nginx, но сервис работает)
```
