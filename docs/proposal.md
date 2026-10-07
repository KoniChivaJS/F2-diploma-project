# Thesis Proposal

| Поле | Значення |
|---|---|
| Студент | Куруляк Едуард Олександрович |
| Науковий керівник | Прохоров Георгій Валерійович |
| Спеціальність | F2 Інженерія програмного забезпечення |
| Факультет / кафедра | Факультет математики та інформатики / кафедра програмного забезпечення комп'ютерних систем |
| Статус | Тему затверджено керівником з зауваженнями (ревізія 2) |

## Українська версія

### Попередня робоча назва проєкту

**UA:** Дослідження стійкості до навантаження платформи гарантованої доставки вебхуків за різних конфігурацій обчислювальних ресурсів та сховищ даних

**EN:** Load Resilience Study of a Guaranteed Webhook Delivery Platform under Varying Compute Resource and Data Storage Configurations

### Актуальність та проблематика

Вебхуки — основний механізм асинхронної інтеграції SaaS-систем: платіжних сервісів, CRM, CI/CD-платформ, маркетплейсів. Відправлення HTTP-запиту тривіальне, проте гарантована доставка події потребує розв'язання низки задач розподілених систем:

- недоступність отримувача, тайм-аути та помилкові відповіді призводять до втрати подій;
- повторні спроби спричиняють дублікати, паралельна доставка — порушення порядку;
- можливі підробка запитів від імені відправника та SSRF-атаки на внутрішню мережу через URL endpoint-а;
- виникає ефект «шумного сусіда»: високе навантаження або тривалі збої одного тенанта погіршують доставку для решти.

Надійність такої системи визначається не лише алгоритмами доставки, а й середовищем виконання: кількістю обчислювальних ядер, обсягом оперативної пам'яті, типом СУБД та брокера повідомлень. Відкриті кількісні дані про вплив цих факторів на пропускну здатність, латентність і збереження подій практично відсутні, тому вибір конфігурації в реальних системах здійснюється емпірично, що призводить або до надлишкових витрат на інфраструктуру, або до втрати подій під піковим навантаженням.

### Мета роботи

Спроєктувати та реалізувати мультитенантну платформу доставки вебхуків із гарантією at-least-once і експериментально дослідити вплив кількості ядер CPU, обсягу оперативної пам'яті, типу СУБД та брокера повідомлень на пропускну здатність, латентність і відсутність втрат подій за різних профілів навантаження та в умовах відмов. За результатами визначити точки насичення системи та мінімальні конфігурації, що забезпечують задані нефункціональні вимоги.

### Сформована ідея продукту

Модульний моноліт із модулями, виділеними за bounded contexts: Ingestion, Routing, Delivery, Tenancy, Observability. Межі модулів контролюються автоматизованими архітектурними перевірками (fitness functions) у CI.

Сховище даних і брокер повідомлень підключаються через порти та адаптери (Ports & Adapters). Це дозволяє змінювати СУБД і брокер без зміни доменної логіки, що є необхідною умовою порівняльного експериментального дослідження.

API та delivery-воркери збираються з однієї кодової бази, але розгортаються й масштабуються незалежно. Моделі запису та читання розділено (CQRS): журнал доставок і метрики формуються проєкціями з потоку подій.

Вибір модульного моноліту замість мікросервісної архітектури обґрунтовано в ADR: немає розподілених транзакцій, операційна складність нижча, а незалежне масштабування критичних компонентів збережено.

Адміністративна панель дозволяє керувати endpoint-ами, підписками та секретами, переглядати історію доставок, DLQ і метрики.

### Детальний опис функціоналу (Core Features)

1. **Надійний прийом і маршрутизація подій.** API приймає події з idempotency key, унікальним у межах тенанта. Подія та запис outbox фіксуються в одній транзакції (Transactional Outbox), після чого ретранслятор публікує їх у брокер повідомлень. Endpoint-и підписуються на потрібні типи подій, payload валідується за JSON Schema. Гарантія доставки — at-least-once; кожен запит містить webhook-id для дедуплікації на боці отримувача.
2. **Повторні спроби без блокування черги.** Exponential backoff із jitter. Відкладені спроби плануються поза чергою брокера, що виключає head-of-line blocking. Після вичерпання спроб подія переміщується в Dead Letter Queue з можливістю повторного відправлення (replay).
3. **Захист від збоїв отримувача.** Circuit Breaker і Bulkhead (ліміт паралельних запитів) на рівні endpoint-а. Endpoint, недоступний довше за заданий поріг, вимикається автоматично, а тенант отримує сповіщення.
4. **Безпека доставки.** Запити підписуються HMAC-SHA256 за специфікацією Standard Webhooks. Ротація секретів має перехідний період дії двох секретів. Replay-атаки блокуються через timestamp, секрети зберігаються з envelope encryption. Захист від SSRF: перевірка резолвленої IP-адреси на належність до приватних діапазонів, фіксація IP на час запиту (захист від DNS rebinding), заборона редиректів.
5. **Ізоляція тенантів.** Rate limiting за алгоритмом Token Bucket виконується атомарно в Redis. Пропускна здатність розподіляється між тенантами справедливо (Deficit Round Robin).
6. **Експериментальний стенд.** Автоматизований запуск сценаріїв навантаження з параметризацією конфігурації (кількість ядер CPU, обсяг RAM, СУБД, брокер, кількість реплік воркерів), ін'єкцією відмов, збором метрик і формуванням звіту за кожним прогоном.
7. **Observability.** Журнал усіх спроб доставки. Метрики: success rate, p50/p95/p99 latency, consumer lag, використання CPU та RAM. Наскрізне трасування API → outbox → брокер → воркер → HTTP з передаванням trace context. Дашборди для аналізу результатів експериментів.

### Методика експериментального дослідження

**Фактори та рівні:**

| Фактор | Рівні |
|---|---|
| Кількість ядер CPU | 1, 2, 4 |
| Обсяг оперативної пам'яті | 512 МБ, 1 ГБ, 2 ГБ |
| СУБД | PostgreSQL, MySQL, MongoDB |
| Брокер повідомлень | Apache Kafka, RabbitMQ |
| Кількість реплік delivery-воркерів | 1, 3, 5 |

**Профілі навантаження:**

| Профіль | Опис |
|---|---|
| Базовий | Постійне навантаження від 500 запитів/хв з поступовим збільшенням |
| Східчастий (ramp-up) | Зростання інтенсивності до точки насичення системи |
| Піковий (spike) | Раптове багаторазове зростання навантаження на короткий проміжок часу |
| Тривалий (soak) | Постійне навантаження протягом 1–2 год для виявлення витоків пам'яті та деградації |
| З відмовами | Навантаження з ін'єкцією мережевих збоїв (Toxiproxy) та примусовим завершенням pod-ів |

**Метрики:** пропускна здатність (подій/с), латентність прийому та наскрізна затримка доставки (p50, p95, p99), частка помилок, кількість втрачених подій, використання CPU та RAM, consumer lag, час відновлення після відмови.

**Порядок проведення:** від базової конфігурації (2 CPU, 1 ГБ RAM, PostgreSQL, Kafka) змінюється по одному фактору, після чого перевіряються окремі комбінації факторів. Ресурсні обмеження задаються через Kubernetes requests/limits. Кожен прогін повторюється щонайменше тричі. Результати подаються у вигляді таблиць і графіків та зіставляються з нефункціональними вимогами. Під час кожного прогону перевіряється інваріант: кожна прийнята подія доставлена або перебуває в DLQ.

### Нефункціональні вимоги

Цільові значення наведено для базової конфігурації й уточнюються після базового заміру.

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
5. збирання образу, Trivy (образ та IaC), SBOM (Syft);
6. розгортання в staging, smoke-тест k6;
7. промоція в production.

**Хмарне розгортання.** Kubernetes. Інфраструктура описана як код (Terraform), застосунок — Helm-чартами, доставка — через GitOps (Argo CD). Воркери автоматично масштабуються за consumer lag (KEDA). Graceful shutdown доводить in-flight доставки до завершення.

**Контроль технічного боргу.** SonarQube quality gate, fitness functions (dependency-cruiser), ADR у репозиторії.

**Безпека застосунку.** Модель загроз STRIDE, цільовий рівень верифікації OWASP ASVS Level 2.

### Попередній інструментарій та технологічний стек

| Категорія | Технології |
|---|---|
| Мова програмування | TypeScript |
| Архітектурні патерни | Modular Monolith, Bounded Contexts, Ports & Adapters, Event-Driven Architecture, CQRS, Transactional Outbox, Idempotent Consumer, Circuit Breaker, Bulkhead, Dead Letter Queue, ADR |
| Фреймворки | NestJS, Next.js, Prisma ORM, Turborepo |
| Бази даних та сховища | PostgreSQL, MySQL, MongoDB, Redis |
| Брокери повідомлень | Apache Kafka, RabbitMQ |
| Тестування | Jest, Testcontainers, fast-check, k6, Toxiproxy |
| DevSecOps | GitHub Actions, SonarQube, Semgrep, gitleaks, Trivy, Syft, dependency-cruiser |
| Інфраструктура | Docker, Kubernetes, Terraform, Helm, Argo CD, KEDA |
| Observability | OpenTelemetry, Prometheus, cAdvisor, Grafana, Jaeger |

---

## English Version

### Preliminary Working Title

Load Resilience Study of a Guaranteed Webhook Delivery Platform under Varying Compute Resource and Data Storage Configurations

### Relevance and Problem Statement

Webhooks are the primary mechanism for asynchronous integration between SaaS systems: payment providers, CRMs, CI/CD platforms, marketplaces. Sending an HTTP request is trivial, but guaranteeing event delivery requires solving several distributed-systems problems:

- receiver unavailability, timeouts, and error responses cause event loss;
- retries produce duplicates, and concurrent delivery breaks event ordering;
- attackers can forge requests on behalf of the sender or use endpoint URLs for SSRF attacks against internal networks;
- the "noisy neighbor" effect lets one tenant's high load or persistent failures degrade delivery for everyone else.

The reliability of such a system depends not only on its delivery algorithms but also on its runtime environment: the number of CPU cores, the amount of RAM, and the choice of database and message broker. Public quantitative data on how these factors affect throughput, latency, and event retention is scarce, so production configurations are chosen empirically, which leads either to infrastructure overspending or to event loss under peak load.

### Objective

Design and implement a multi-tenant webhook delivery platform with an at-least-once guarantee, and experimentally investigate how the number of CPU cores, the amount of RAM, the database, and the message broker affect throughput, latency, and event loss under different load profiles and failure conditions. Based on the results, identify the system's saturation points and the minimal configurations that meet the specified non-functional requirements.

### Product Concept and Architecture

A modular monolith whose modules follow bounded contexts: Ingestion, Routing, Delivery, Tenancy, Observability. Automated architecture checks (fitness functions) in CI enforce module boundaries.

The database and the message broker are connected through ports and adapters (Ports & Adapters). This allows replacing them without changing domain logic, which is a prerequisite for the comparative experimental study.

The API and delivery workers are built from a single codebase but deployed and scaled independently. Write and read models are separated (CQRS): delivery logs and metrics are built as projections from the event stream.

An ADR justifies choosing a modular monolith over microservices: there are no distributed transactions and operational complexity is lower, while critical components can still scale independently.

The admin panel lets operators manage endpoints, subscriptions, and secrets, and inspect delivery history, the DLQ, and metrics.

### Core Features

1. **Reliable ingestion and routing.** The API accepts events with an idempotency key that is unique per tenant. The event and its outbox record are committed in a single transaction (Transactional Outbox), and a relay then publishes them to the message broker. Endpoints subscribe to the event types they need, and payloads are validated against JSON Schema. Delivery is at-least-once; every request carries a webhook-id for receiver-side deduplication.
2. **Non-blocking retries.** Exponential backoff with jitter. Delayed attempts are scheduled outside the broker queue, which eliminates head-of-line blocking. Events that exhaust their retries move to a Dead Letter Queue and can be replayed.
3. **Receiver failure protection.** Per-endpoint Circuit Breaker and Bulkhead (concurrency limit). An endpoint that stays unavailable beyond a threshold is disabled automatically, and the tenant is notified.
4. **Delivery security.** Requests are signed with HMAC-SHA256 per the Standard Webhooks specification. Secret rotation includes a grace period during which two secrets are valid. Timestamps block replay attacks, and secrets are stored with envelope encryption. SSRF protection covers resolved-IP validation against private ranges, IP pinning for the duration of the request (DNS rebinding protection), and a ban on redirects.
5. **Tenant isolation.** Token Bucket rate limiting runs atomically in Redis. Throughput is shared fairly across tenants (Deficit Round Robin).
6. **Experimental test bench.** Automated execution of load scenarios parameterized by configuration (CPU cores, RAM, database, broker, number of worker replicas), with failure injection, metric collection, and a report for every run.
7. **Observability.** A log of every delivery attempt. Metrics: success rate, p50/p95/p99 latency, consumer lag, CPU and RAM usage. End-to-end tracing API → outbox → broker → worker → HTTP with trace context propagation. Dashboards for analyzing experiment results.

### Experimental Methodology

**Factors and levels:**

| Factor | Levels |
|---|---|
| CPU cores | 1, 2, 4 |
| RAM | 512 MB, 1 GB, 2 GB |
| Database | PostgreSQL, MySQL, MongoDB |
| Message broker | Apache Kafka, RabbitMQ |
| Delivery worker replicas | 1, 3, 5 |

**Load profiles:**

| Profile | Description |
|---|---|
| Baseline | Constant load starting at 500 requests/min with gradual increase |
| Ramp-up | Increasing intensity up to the system's saturation point |
| Spike | A sudden multi-fold load increase for a short period |
| Soak | Constant load for 1–2 hours to detect memory leaks and degradation |
| With failures | Load combined with network fault injection (Toxiproxy) and forced pod termination |

**Metrics:** throughput (events/s), ingestion latency and end-to-end delivery latency (p50, p95, p99), error rate, number of lost events, CPU and RAM usage, consumer lag, recovery time after failure.

**Procedure:** starting from a baseline configuration (2 CPU, 1 GB RAM, PostgreSQL, Kafka), one factor is changed at a time, followed by selected factor combinations. Resource limits are set with Kubernetes requests/limits. Every run is repeated at least three times. Results are presented as tables and charts and compared against the non-functional requirements. Every run checks the invariant: each accepted event is either delivered or present in the DLQ.

### Non-Functional Requirements

Target values apply to the baseline configuration and will be refined after a baseline measurement.

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
5. image build, Trivy (image and IaC), SBOM (Syft);
6. staging deployment, k6 smoke test;
7. promotion to production.

**Cloud deployment.** Kubernetes. Infrastructure is defined as code (Terraform), the application as Helm charts, and delivery runs through GitOps (Argo CD). Workers autoscale on consumer lag (KEDA). Graceful shutdown completes in-flight deliveries.

**Technical debt control.** SonarQube quality gate, fitness functions (dependency-cruiser), and ADRs kept in the repository.

**Application security.** STRIDE threat model, target verification level OWASP ASVS Level 2.

### Preliminary Tooling and Technology Stack

| Category | Technologies |
|---|---|
| Programming language | TypeScript |
| Architectural patterns | Modular Monolith, Bounded Contexts, Ports & Adapters, Event-Driven Architecture, CQRS, Transactional Outbox, Idempotent Consumer, Circuit Breaker, Bulkhead, Dead Letter Queue, ADR |
| Frameworks | NestJS, Next.js, Prisma ORM, Turborepo |
| Databases and storage | PostgreSQL, MySQL, MongoDB, Redis |
| Message brokers | Apache Kafka, RabbitMQ |
| Testing | Jest, Testcontainers, fast-check, k6, Toxiproxy |
| DevSecOps | GitHub Actions, SonarQube, Semgrep, gitleaks, Trivy, Syft, dependency-cruiser |
| Infrastructure | Docker, Kubernetes, Terraform, Helm, Argo CD, KEDA |
| Observability | OpenTelemetry, Prometheus, cAdvisor, Grafana, Jaeger |
