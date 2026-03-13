# Risks и Missing Requirements

## Текущие риски подготовки
1. Хорошая база по Linux, Kubernetes, Ansible, PostgreSQL и observability уже есть, но платформенные темы нового поколения покрыты неравномерно.
2. Есть риск уверенно говорить о Kubernetes в целом, но проседать на узких вопросах про `Talos Linux`, immutable-подход и операционную модель без SSH.
3. Тема балансировки частично раскрыта, но для SRE-вакансий с упором на `HAProxy` под высокой нагрузкой нужен более системный устный ответ.
4. По observability есть база, но нужен отдельный уверенный контур по `VictoriaMetrics`, `PromQL`, качеству метрик и правилам алертинга, связанным с `SLO`.
5. IaC покрыт в основном через `Ansible` и `Terraform`; вопросы по `Puppet` и `Pulumi` требуют хотя бы уверенного сравнительного ответа.
6. Недостаточно связаны между собой темы `service mesh`, `mTLS`, canary и управления east-west трафиком.
7. `Cilium` и `eBPF` - это рискованная зона для SRE-вакансий: без отдельной проработки легко уйти в общие слова.
8. Security-модель Kubernetes частично покрыта, но не хватает целостного ответа про `RBAC`, admission policy, `Pod Security`, network policy и особенности Talos.

## Missing Requirements
1. `Talos Linux`: философия immutable-ноды, machine config, модель эксплуатации и типовые trade-offs.
2. `HAProxy` для high-load: `maxconn`, таймауты, health checks, connection reuse, наблюдаемость и сценарии деградации.
3. `VictoriaMetrics` и `PromQL`: хранение метрик, cardinality, запись/чтение, базовые SLO- и burn-rate-запросы.
4. `Puppet` и `Pulumi`: где уместны, чем отличаются от `Ansible` и `Terraform`, какие плюсы и риски.
5. `Service mesh` (`Istio`, `Linkerd`): зачем нужен, когда оправдан, как работает `mTLS`, canary и traffic shifting.
6. `Cilium` и `eBPF`: network policy, observability потоков, kube-proxy replacement, ограничения и риски.
7. Security-модель Kubernetes и Talos: hardening control plane/node plane, доступ к API, policy-as-code, supply chain.

## Что уже закрыто этим обновлением
1. Добавлены новые блоки в `kubernetes_core.md` по `Talos`, `service mesh`, `Cilium/eBPF` и security-модели.
2. Добавлены блоки в `loadbalance.md` по `HAProxy` под высокую нагрузку.
3. Добавлены блоки в `observability_incident_response.md` и `sre.md` по `VictoriaMetrics`, `PromQL`, `SLO/error budget`.
4. Добавлены сравнительные материалы в `linux.md` и `devsecops_security_compliance.md`.
5. Обновлен `interview_readiness_tracker.md`, чтобы новые темы не терялись из поля зрения.
