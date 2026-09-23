# Thesis Proposal
 
| Поле | Значення |
|---|---|
| Студент | Куруляк Едуард Олександрович |
| Науковий керівник | Прохоров Георгій Валерійович |
| Спеціальність | F2 Інженерія програмного забезпечення |
| Факультет / кафедра | Факультет математики та інформатики / кафедра програмного забезпечення комп'ютерних систем |
 
## Українська версія
 
### Попередня робоча назва проєкту
 
**UA:** Відмовостійка мультитенантна платформа гарантованої доставки вебхуків для подієво-орієнтованої інтеграції програмних систем
 
**EN:** Fault-Tolerant Multi-Tenant Webhook Delivery Platform for Event-Driven Software Integration
 
### Актуальність та проблематика
 
Вебхуки — основний механізм асинхронної інтеграції SaaS-систем: платіжних сервісів, CRM, CI/CD-платформ, маркетплейсів. Відправлення HTTP-запиту тривіальне, проте гарантована доставка події потребує розв'язання низки задач розподілених систем:
 
- недоступність отримувача, тайм-аути та помилкові відповіді призводять до втрати подій;
- повторні спроби спричиняють дублікати, паралельна доставка — порушення порядку;
- можливі підробка запитів від імені відправника та SSRF-атаки на внутрішню мережу через URL endpoint-а;
- виникає ефект «шумного сусіда»: високе навантаження або тривалі збої одного тенанта погіршують доставку для решти.
Компанії, як правило, реалізують цю логіку самостійно і неповно. Винесення гарантованої доставки в окрему платформу усуває дублювання зусиль і забезпечує єдиний рівень надійності та безпеки.
 
### Мета роботи
 
Спроєктувати, реалізувати та розгорнути в хмарному Kubernetes-середовищі мультитенантну платформу доставки вебхуків із гарантією at-least-once. Експериментально підтвердити відсутність втрат подій та ізоляцію тенантів в умовах відмов і навантаження.
 
### Сформована ідея продукту
 
Модульний моноліт із модулями, виділеними за bounded contexts: Ingestion, Routing, Delivery, Tenancy, Observability. Межі модулів контролюються автоматизованими архітектурними перевірками (fitness functions) у CI.
 
API та delivery-воркери збираються з однієї кодової бази, але розгортаються й масштабуються незалежно. Асинхронна обробка побудована на Apache Kafka. Моделі запису та читання розділено (CQRS): журнал доставок і метрики формуються проєкціями з потоку подій.
 
Вибір модульного моноліту замість мікросервісної архітектури обґрунтовано в ADR: немає розподілених транзакцій, операційна складність нижча, а незалежне масштабування критичних компонентів збережено.
 
Адміністративна панель дозволяє керувати endpoint-ами, підписками та секретами, переглядати історію доставок, DLQ і метрики.
 
### Детальний опис функціоналу (Core Features)
 
1. **Надійний прийом подій.** API приймає події з idempotency key, унікальним у межах тенанта. Подія та запис outbox фіксуються в одній транзакції PostgreSQL (Transactional Outbox), після чого ретранслятор публікує їх у Kafka. Гарантія доставки — at-least-once. Кожен запит містить webhook-id для дедуплікації на боці отримувача.
2. **Повторні спроби без блокування черги.** Exponential backoff із jitter. Відкладені спроби плануються поза Kafka-партицією, що виключає head-of-line blocking. Після вичерпання спроб подія переміщується в Dead Letter Queue. Масовий replay реалізовано як керований довготривалий процес (Process Manager) з відстеженням прогресу, паузою та скасуванням.
3. **Захист від збоїв отримувача та життєвий цикл endpoint-а.** Circuit Breaker і Bulkhead (ліміт паралельних запитів) на рівні endpoint-а. Endpoint активується лише після верифікації володіння (challenge-response). Endpoint, недоступний довше за заданий поріг, вимикається автоматично, а тенант отримує сповіщення.
4. **Безпека доставки.** Запити підписуються HMAC-SHA256 за специфікацією Standard Webhooks. Ротація секретів має перехідний період дії двох секретів. Replay-атаки блокуються через timestamp, секрети зберігаються з envelope encryption. Захист від SSRF: перевірка резолвленої IP-адреси на належність до приватних діапазонів, фіксація IP на час запиту (захист від DNS rebinding), заборона редиректів, ізольований egress-контур.
5. **Ізоляція тенантів.** Rate limiting за алгоритмом Token Bucket виконується атомарно в Redis. Пропускна здатність розподіляється між тенантами справедливо (Deficit Round Robin). Дані ізольовано на рівні PostgreSQL Row-Level Security.
6. **Маршрутизація та контракти подій.** Endpoint-и підписуються на потрібні типи подій. Payload валідується за JSON Schema, схеми версіонуються. Опційний FIFO-режим забезпечує упорядковану доставку за ключем.
7. **Observability.** Журнал усіх спроб доставки. Метрики: success rate, p95 latency, consumer lag. Наскрізне трасування API → outbox → Kafka → воркер → HTTP з передаванням trace context у заголовках Kafka. SLO зі сповіщеннями за швидкістю вичерпання error budget.
### Нефункціональні вимоги
 
Значення цільові й уточнюються після базового заміру.
 
| Характеристика | Цільове значення | Метод перевірки |
|---|---|---|
| Втрата прийнятих подій | 0 при відмові будь-якого pod-а API, воркера або брокера | Експеримент із відмовами під навантаженням |
| Латентність прийому | p95 ≤ 100 мс при 500 запитах/с | k6 |
| Затримка доставки | p95 ≤ 2 с за доступного отримувача | k6, трасування |
| Ізоляція тенантів | деградація p95 для інших тенантів ≤ 10%, коли один тенант генерує 80% трафіку до недоступного endpoint-а | k6, Toxiproxy |
| Покриття доменних модулів | ≥ 80% | SonarQube quality gate |
| Вразливості образів | 0 рівня Critical/High | Trivy |
 
### Інженерна інфраструктура
 
**CI/CD (GitHub Actions).** Послідовність етапів:
 
1. лінтинг і перевірка типів;
2. unit- та інтеграційні тести (Testcontainers);
3. SonarQube quality gate;
4. SAST (Semgrep), SCA, secret scanning (gitleaks);
5. збирання образу, Trivy (образ та IaC), SBOM (Syft), підпис образу (Cosign);
6. розгортання в staging, DAST (OWASP ZAP), smoke-тест k6;
7. промоція в production.
**Хмарне розгортання.** Kubernetes. Інфраструктура описана як код (Terraform), застосунок — Helm-чартами, доставка — через GitOps (Argo CD). Kafka розгортається оператором Strimzi. Воркери автоматично масштабуються за consumer lag (KEDA). Graceful shutdown доводить in-flight доставки до завершення.
 
**Контроль технічного боргу.** SonarQube quality gate, fitness functions (dependency-cruiser), ADR у репозиторії.
 
**Безпека застосунку.** Модель загроз STRIDE, цільовий рівень верифікації OWASP ASVS Level 2.
 
### Методика верифікації
 
Навантажувальне тестування (k6) поєднується з ін'єкцією мережевих збоїв (Toxiproxy) та примусовим завершенням pod-ів. Під час експериментів перевіряється інваріант: кожна прийнята подія доставлена або перебуває в DLQ. Результати зіставляються з нефункціональними вимогами.
 
### Попередній інструментарій та технологічний стек
 
| Категорія | Технології |
|---|---|
| Мова програмування | TypeScript |
| Архітектурні патерни | Modular Monolith, Bounded Contexts, Event-Driven Architecture, CQRS, Transactional Outbox, Idempotent Consumer, Process Manager, Circuit Breaker, Bulkhead, Dead Letter Queue, ADR |
| Фреймворки | NestJS, Next.js, Prisma ORM, Turborepo |
| Бази даних та сховища | PostgreSQL, Redis |
| Брокер повідомлень | Apache Kafka (Strimzi) |
| Тестування | Jest, Testcontainers, fast-check, k6, Toxiproxy |
| DevSecOps | GitHub Actions, SonarQube, Semgrep, gitleaks, Trivy, Syft, Cosign, OWASP ZAP, dependency-cruiser |
| Інфраструктура | Docker, Kubernetes, Terraform, Helm, Argo CD, KEDA |
| Observability | OpenTelemetry, Prometheus, Alertmanager, Grafana, Jaeger |
 
---
 
## English Version
 
### Preliminary Working Title
 
Fault-Tolerant Multi-Tenant Webhook Delivery Platform for Event-Driven Software Integration
 
### Relevance and Problem Statement
 
Webhooks are the primary mechanism for asynchronous integration between SaaS systems: payment providers, CRMs, CI/CD platforms, marketplaces. Sending an HTTP request is trivial, but guaranteeing event delivery requires solving several distributed-systems problems:
 
- receiver unavailability, timeouts, and error responses cause event loss;
- retries produce duplicates, and concurrent delivery breaks event ordering;
- attackers can forge requests on behalf of the sender or use endpoint URLs for SSRF attacks against internal networks;
- the "noisy neighbor" effect lets one tenant's high load or persistent failures degrade delivery for everyone else.
Companies typically implement this logic in-house, and often incompletely. Extracting guaranteed delivery into a dedicated platform removes duplicated effort and provides a uniform level of reliability and security.
 
### Objective
 
Design, implement, and deploy to a cloud Kubernetes environment a multi-tenant webhook delivery platform with an at-least-once guarantee. Experimentally demonstrate zero event loss and tenant isolation under failures and load.
 
### Product Concept and Architecture
 
A modular monolith whose modules follow bounded contexts: Ingestion, Routing, Delivery, Tenancy, Observability. Automated architecture checks (fitness functions) in CI enforce module boundaries.
 
The API and delivery workers are built from a single codebase but deployed and scaled independently. Asynchronous processing runs on Apache Kafka. Write and read models are separated (CQRS): delivery logs and metrics are built as projections from the event stream.
 
An ADR justifies choosing a modular monolith over microservices: there are no distributed transactions and operational complexity is lower, while critical components can still scale independently.
 
The admin panel lets operators manage endpoints, subscriptions, and secrets, and inspect delivery history, the DLQ, and metrics.
 
### Core Features
 
1. **Reliable ingestion.** The API accepts events with an idempotency key that is unique per tenant. The event and its outbox record are committed in a single PostgreSQL transaction (Transactional Outbox), and a relay then publishes them to Kafka. Delivery is at-least-once. Every request carries a webhook-id for receiver-side deduplication.
2. **Non-blocking retries.** Exponential backoff with jitter. Delayed attempts are scheduled outside the Kafka partition, which eliminates head-of-line blocking. Events that exhaust their retries move to a Dead Letter Queue. Bulk replay runs as a managed long-running process (Process Manager) with progress tracking, pause, and cancellation.
3. **Receiver failure protection and endpoint lifecycle.** Per-endpoint Circuit Breaker and Bulkhead (concurrency limit). An endpoint is activated only after ownership verification (challenge-response). An endpoint that stays unavailable beyond a threshold is disabled automatically, and the tenant is notified.
4. **Delivery security.** Requests are signed with HMAC-SHA256 per the Standard Webhooks specification. Secret rotation includes a grace period during which two secrets are valid. Timestamps block replay attacks, and secrets are stored with envelope encryption. SSRF protection covers resolved-IP validation against private ranges, IP pinning for the duration of the request (DNS rebinding protection), a ban on redirects, and an isolated egress path.
5. **Tenant isolation.** Token Bucket rate limiting runs atomically in Redis. Throughput is shared fairly across tenants (Deficit Round Robin). Data is isolated with PostgreSQL Row-Level Security.
6. **Routing and event contracts.** Endpoints subscribe to the event types they need. Payloads are validated against JSON Schema, and schemas are versioned. An optional FIFO mode delivers events in order per ordering key.
7. **Observability.** A log of every delivery attempt. Metrics: success rate, p95 latency, consumer lag. End-to-end tracing API → outbox → Kafka → worker → HTTP, with trace context propagated in Kafka headers. SLOs with error-budget burn-rate notifications.
### Non-Functional Requirements
 
Values are targets, to be refined after a baseline measurement.
 
| Attribute | Target | Verification |
|---|---|---|
| Loss of accepted events | 0 on failure of any API, worker, or broker pod | Failure experiment under load |
| Ingestion latency | p95 ≤ 100 ms at 500 req/s | k6 |
| Delivery latency | p95 ≤ 2 s with a healthy receiver | k6, tracing |
| Tenant isolation | ≤ 10% p95 degradation for other tenants while one tenant sends 80% of traffic to an unavailable endpoint | k6, Toxiproxy |
| Domain module coverage | ≥ 80% | SonarQube quality gate |
| Image vulnerabilities | 0 Critical/High | Trivy |
 
### Engineering Infrastructure
 
**CI/CD (GitHub Actions).** Stages in order:
 
1. linting and type checking;
2. unit and integration tests (Testcontainers);
3. SonarQube quality gate;
4. SAST (Semgrep), SCA, secret scanning (gitleaks);
5. image build, Trivy (image and IaC), SBOM (Syft), image signing (Cosign);
6. staging deployment, DAST (OWASP ZAP), k6 smoke test;
7. promotion to production.
**Cloud deployment.** Kubernetes. Infrastructure is defined as code (Terraform), the application as Helm charts, and delivery runs through GitOps (Argo CD). Kafka is deployed with the Strimzi operator. Workers autoscale on consumer lag (KEDA). Graceful shutdown completes in-flight deliveries.
 
**Technical debt control.** SonarQube quality gate, fitness functions (dependency-cruiser), and ADRs kept in the repository.
 
**Application security.** STRIDE threat model, target verification level OWASP ASVS Level 2.
 
### Verification Methodology
 
Load testing (k6) is combined with network fault injection (Toxiproxy) and forced pod termination. The experiments check one invariant: every accepted event is either delivered or present in the DLQ. Results are compared against the non-functional requirements.
 
### Preliminary Tooling and Technology Stack
 
| Category | Technologies |
|---|---|
| Programming language | TypeScript |
| Architectural patterns | Modular Monolith, Bounded Contexts, Event-Driven Architecture, CQRS, Transactional Outbox, Idempotent Consumer, Process Manager, Circuit Breaker, Bulkhead, Dead Letter Queue, ADR |
| Frameworks | NestJS, Next.js, Prisma ORM, Turborepo |
| Databases and storage | PostgreSQL, Redis |
| Message broker | Apache Kafka (Strimzi) |
| Testing | Jest, Testcontainers, fast-check, k6, Toxiproxy |
| DevSecOps | GitHub Actions, SonarQube, Semgrep, gitleaks, Trivy, Syft, Cosign, OWASP ZAP, dependency-cruiser |
| Infrastructure | Docker, Kubernetes, Terraform, Helm, Argo CD, KEDA |
| Observability | OpenTelemetry, Prometheus, Alertmanager, Grafana, Jaeger |