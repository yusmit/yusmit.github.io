# Cloud Patterns для SRE/DevOps: теория, вопросы с ответами, кейсы, красные флаги

## Теория
### Что важно для интервью
- Сетевой дизайн: VPC/VNet, private/public subnets, routing, NAT, security groups.
- IAM и доступы: least privilege, role-based access, short-lived credentials.
- Managed vs self-hosted: стоимость, операционная нагрузка, vendor lock-in.
- Надежность: multi-AZ, multi-region, backup/restore, RTO/RPO.
- Наблюдаемость и безопасность как часть архитектуры, а не «потом».

### Базовые паттерны
1. Stateless app + managed DB + cache + queue.
2. Blue/Green или Canary deployment для безопасных релизов.
3. DR-план: backup policy + restore drills + runbooks.
4. Zero-trust access между сервисами (mTLS/service identity).

## Вопросы с ответами
### Вопрос 1: Когда выбирать managed service, а когда self-hosted?
Ответ:
- Managed: быстрее запуск, меньше ops-рутины, SLA от провайдера.
- Self-hosted: больше контроля, ниже риск lock-in, но выше стоимость сопровождения.
- На интервью важно назвать trade-offs: cost, control, compliance, team capacity.

### Вопрос 2: Как снизить vendor lock-in?
Ответ:
- Использовать открытые стандарты и portable runtime (Kubernetes, Terraform-модули с абстракцией).
- Избегать жесткой привязки к проприетарным API без fallback.
- Планировать data portability (экспорт, миграционный сценарий).

### Вопрос 3: Как проектировать сеть в cloud для production?
Ответ:
- Разделить публичный и приватный периметр.
- Минимизировать входящие порты, egress-контроль, отдельные security policies.
- Разнести критичные компоненты по зонам доступности.

### Вопрос 4: Что проверять в DR-ready системе?
Ответ:
- Бэкапы не только делаются, но и регулярно восстанавливаются в тестовом контуре.
- Понятны RTO/RPO по каждому сервису.
- Есть runbook и ответственные за аварийное восстановление.

## Практические кейсы
### Кейс: «Регион облака частично недоступен»
1. Оценить impact и affected services.
2. Включить failover на standby-контур.
3. Ограничить деградацию через feature flags/rate limiting.
4. Синхронизировать команды и коммуникацию для бизнеса.
5. После стабилизации: postmortem и корректировки DR-плана.

### Кейс: «Счет за облако вырос на 40%»
1. Разбивка расходов: compute/storage/network/managed services.
2. Проверка overprovisioning и idle-ресурсов.
3. Оптимизация autoscaling, storage tiering, retention policy.
4. Ввести FinOps-практику и budget alerts.

## Красные флаги в ответах
- «Облако автоматически решает отказоустойчивость» без конкретики по архитектуре.
- Отсутствие RTO/RPO и DR-процедур.
- Полное игнорирование IAM и принципа least privilege.
- Нет оценки стоимости и эксплуатационной сложности решений.
