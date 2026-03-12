# System Design для SRE/DevOps: теория, вопросы с ответами, кейсы, красные флаги

## Теория
### Что оценивают на интервью
- Декомпозицию системы и критичные потоки.
- Надежность: SPOF, failover, деградация.
- Trade-offs: latency/consistency, reliability/cost, complexity/operability.
- Наличие observability, security и DR как части дизайна.

### Каркас ответа
1. Уточнить требования (RPS, latency, SLO, RTO/RPO).
2. Нарисовать high-level архитектуру.
3. Разобрать data/control paths.
4. Показать отказоустойчивость и recovery.
5. Показать мониторинг, безопасность и rollout.

## Вопросы с ответами
### Вопрос 1: Active-active или active-passive?
Ответ:
- Active-active: выше доступность и распределение нагрузки, но сложнее consistency.
- Active-passive: проще операционно, но хуже утилизация ресурсов и выше RTO.
- Выбор зависит от бизнес-требований к доступности и допустимой сложности.

### Вопрос 2: Что заложить в v1 reliability-платформы?
Ответ:
- SLO/SLI, минимальный observability stack, alerting policy.
- Backup/restore + регулярные drills.
- Canary/rollback strategy и runbooks.
- Capacity baseline + autoscaling policy.

### Вопрос 3: Как проектировать для 10k RPS API?
Ответ:
- Stateless app tier + горизонтальное масштабирование.
- Кэш и очереди для сглаживания пиков.
- DB-схема с read/write стратегией и защитой от hot spots.
- Наблюдаемость по golden signals и saturation.

### Вопрос 4: Что делать при недоступности региона?
Ответ:
- Переключение на standby/второй регион по заранее проверенной процедуре.
- Ограниченная деградация фич и защита критичных путей.
- После инцидента — пересмотр RTO/RPO и DR runbook.

## Практические кейсы
### Кейс 1: Пиковая нагрузка убивает БД
1. Определить bottleneck (CPU/IO/locks/slow queries).
2. Включить mitigation: cache, queue, throttling.
3. Оптимизировать критичные запросы и лимиты.
4. Обновить capacity model и performance тесты.

### Кейс 2: Частые инциденты после релизов
1. Проверить качество rollout-стратегии.
2. Усилить pre-deploy проверки и policy gates.
3. Добавить canary analysis и auto-rollback.
4. Закрыть системные причины, не только симптомы.

## Красные флаги
- Нет явных требований перед выбором архитектуры.
- Игнорирование стоимости и операционной сложности.
- Нет DR-плана и проверок восстановления.
- Ответы только «по технологии», без процессов и ownership.
