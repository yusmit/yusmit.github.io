# Observability и Incident Response: теория, вопросы с ответами, кейсы, красные флаги

## Теория
### Базовая модель observability
- Metrics: тренды и SLI/SLO.
- Logs: событийный контекст и расследование.
- Traces: путь запроса и latency bottlenecks.

### Golden signals
- Latency
- Traffic
- Errors
- Saturation

### Incident lifecycle
1. Detection
2. Triage
3. Mitigation
4. Recovery
5. Postmortem

## Вопросы с ответами
### Вопрос 1: Чем observability отличается от monitoring?
Ответ:
- Monitoring отвечает на «что уже сломалось».
- Observability отвечает на «почему сломалось» в сложных системах за счет корреляции метрик, логов и трасс.

### Вопрос 2: Какие алерты считаются качественными?
Ответ:
- Actionable: по алерту понятно, что делать первым шагом.
- Ownership: есть команда/роль-владелец.
- Noise control: low false positive, dedup/grouping.
- Привязка к SLO или риску для пользователя.

### Вопрос 3: Что делаете в первые 15 минут P1-инцидента?
Ответ:
1. Подтвердить impact и scope.
2. Назначить incident commander и канал коммуникации.
3. Включить mitigation (rollback, traffic shaping, feature flag).
4. Параллельно фиксировать timeline.

### Вопрос 4: Что обязательно включить в postmortem?
Ответ:
- Хронологию, root cause, contributing factors.
- Почему не сработали ранние сигналы.
- Конкретные action items с owner и дедлайном.

## Практические кейсы
### Кейс 1: Latency выросла, error rate стабильный
1. Сегментация по endpoint/zone/revision.
2. Проверка saturation (CPU, memory, I/O, pool).
3. Сверка с внешними зависимостями и сетевыми задержками.
4. Коррекция алертов и runbook после инцидента.

### Кейс 2: Алерты «шумят» после релиза
1. Проверить пороги и burn-rate логику.
2. Проверить readiness/probes и warm-up.
3. Проверить rollout window.
4. Обновить alerting policy и документацию.

## Красные флаги
- Перечислять только инструменты без процесса.
- Игнорировать коммуникацию и ownership в инциденте.
- Не различать симптом и первопричину.
- Не фиксировать follow-up после инцидента.
