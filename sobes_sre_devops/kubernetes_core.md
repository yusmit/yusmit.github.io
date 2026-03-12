# Kubernetes Core: теория, вопросы с ответами, кейсы, красные флаги

## Теория
### Архитектура control plane
- `kube-apiserver`: точка входа в кластер и API.
- `etcd`: источник истины о состоянии кластера.
- `kube-scheduler`: назначает Pod на ноды.
- `kube-controller-manager`: reconciliation loop (desired vs current state).

### Node plane
- `kubelet`: агент ноды, исполняет PodSpec.
- Container runtime: запуск контейнеров (`containerd` и др.).
- `kube-proxy`: сервисная маршрутизация.

### Надежность и rollout
- `readinessProbe` регулирует включение Pod в трафик.
- `livenessProbe` перезапускает зависший контейнер.
- `startupProbe` защищает медленный старт.
- `RollingUpdate` снижает риск downtime.

## Вопросы с ответами
### Вопрос 1: Чем отличаются Deployment, StatefulSet и DaemonSet?
Ответ:
- Deployment: stateless workload, rolling update, replica management.
- StatefulSet: стабильные имена/тома, порядок запуска/остановки.
- DaemonSet: по одному Pod на ноду (агенты/логгеры/мониторинг).

### Вопрос 2: Почему Pod может быть в Pending?
Ответ:
- Недостаточно ресурсов (`requests`), нет подходящей ноды по taints/affinity.
- Ограничения quota/limitrange.
- Ошибки в PVC/PV binding.

### Вопрос 3: Как диагностировать CrashLoopBackOff?
Ответ:
1. `kubectl logs --previous`.
2. `kubectl describe pod` (events, probe failures).
3. Проверить env/secrets/config mounts.
4. Проверить OOMKilled и лимиты ресурсов.

### Вопрос 4: Что происходит при потере quorum в etcd?
Ответ:
- Кластер не может безопасно принимать запись состояния.
- Control plane деградирует: операции управления/обновления блокируются.
- Требуется восстановление quorum или корректный DR-сценарий.

## Практические кейсы
### Кейс 1: Rollout завис
1. Проверить `kubectl rollout status` и events.
2. Проверить readiness/liveness/startup probes.
3. Проверить метрики приложения/инфры на момент релиза.
4. Принять решение: rollback или fix-forward.

### Кейс 2: Рост 5xx после деплоя
1. Сегментировать по версии/подам.
2. Проверить конфиги и секреты релиза.
3. Проверить saturation (CPU/memory/IO/conn pool).
4. Сверить с trace path и зависимостями.

## Красные флаги
- Путать `readiness` и `liveness`.
- Игнорировать `events` и `describe` при диагностике.
- Говорить «K8s сам чинит всё» без ограничений и trade-offs.
- Не учитывать влияние `requests/limits` на scheduling и стабильность.
