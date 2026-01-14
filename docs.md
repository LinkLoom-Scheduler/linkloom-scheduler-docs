# Link Loom

Оркестрація робочих процесів — це координація виконання кількох автоматизованих завдань у визначеному порядку, що забезпечує зв'язок між системами для досягнення кінцевого результату. Link Loom — це **централізований сервіс оркестрації** на базі Spring Boot, призначений для керування DAG-воркфлоу з одного місця для кількох юзер-приложень.

Система працює як **інфраструктурний сервіс** (аналогічно GitHub Actions, Jenkins, Airflow), де:
- **Link Loom Server** (central) — запускає workflow по розкладу, керує залежностями, моніторить статус
- **User Backends** (distributed) — отримують HTTP callback від Link Loom й виконують свою бізнес-логіку
- **Клієнти** (Web UI, Desktop, CLI) — дозволяють управляти workflow і переглядати статус

Основна мета Link Loom — забезпечити масштабовану, розвязану, надійну систему оркестрації, де різні сервіси можуть описувати свої workflow, а Link Loom керує їх виконанням централізовано.

---

## Архітектура: Service + Clients

```
                    ┌─────────────────────┐
                    │   Link Loom Server  │
                    │  (Scheduler + API)  │
                    └──────────┬──────────┘
                               │
            ┌──────────────────┼──────────────────┐
            │                  │                  │
            ▼                  ▼                  ▼
    ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
    │  Web UI      │   │  Desktop App │   │  CLI Tool    │
    │ (React/Vue)  │   │  (Electron)  │   │ (Go/Rust)    │
    └──────┬───────┘   └──────┬───────┘   └──────┬───────┘
           │                  │                  │
           └──────────────────┼──────────────────┘
                              │ REST API
                              │
            ┌─────────────────┴─────────────────┐
            │                                   │
            ▼                                   ▼
    ┌─────────────────┐            ┌────────────────────┐
    │ User Backend 1  │            │  User Backend N    │
    │ (Spring Boot)   │            │  (Any Language)    │
    │                 │            │                    │
    │ ┌─────────────┐ │            │ ┌──────────────┐   │
    │ │ link-loom   │ │            │ │ link-loom    │   │
    │ │ client (lib)│ │            │ │ client (lib) │   │
    │ └─────────────┘ │            │ └──────────────┘   │
    │                 │            │                    │
    │ POST /tasks/... │            │ POST /tasks/...    │
    │ (callback)      │            │ (callback)         │
    └─────────────────┘            └────────────────────┘
```

---

## Сценарії використання

- **CI/CD-пайплайни:** Автоматизація етапів складання, тестування та розгортання програмного забезпечення. Link Loom координує виконання послідовних завдань у конвеєрі, забезпечуючи швидкість і узгодженість процесу.

- **Обробка великих даних (ETL):** Координація потоків обробки даних (завантаження, трансформація, завантаження) в аналітичних системах. Оркестрація таких процесів підвищує надійність, видимість і продуктивність обробки даних.

- **Машинне навчання і Data Science:** Управління життєвим циклом ML-моделей — від підготовки даних і навчання до оцінювання та розгортання. Система дозволяє автоматично відстежувати стан моделі і виконувати повторне навчання за потреби.

- **Бізнес-процеси та інтеграція систем:** Автоматизація багатокрокових операцій із взаємодією між різними сервісами — наприклад, реєстрація користувача, обробка замовлення, затвердження документів, узгодження. Link Loom забезпечує послідовне виконання залежних задач і повідомлення про стан процесу.

- **IT-операції та інциденти:** Підтримка автоматичного реагування на події — резервне копіювання, оновлення інфраструктури, планування завдань у кластері. Використання Link Loom робить виконання операційних процесів надійним та узгодженим.

---

## Основні особливості

- **Модель DAG:** Опис кожного робочого процесу як орієнтований ациклічний граф (DAG) завдань дозволяє чітко вказати залежності. DAG-структура спрощує підтримку складних процесів і забезпечує максимальний паралелізм виконання.

- **Service-Driven архітектура:** Link Loom — це central service, який не залежить від мови програмування юзер-backend. Спілка відбувається через REST API й HTTP callbacks. Юзер-backend можна писати на Java, Python, Go, Node.js — довільно.

- **Розвязана система (Decoupling):** User backends комунікують з Link Loom через HTTP, що дозволяє:
    - Незалежно розгортати і оновлювати компоненти
    - Масштабувати один юзер-backend без впливу на інших
    - Переживати крахи юзер-backend без втрати стану workflow

- **Надійне виконання:** Вбудовані механізми повторних спроб (retry), таймаутів, deadletter queue для неудачних задач. Система гарантує, що task буде виконана або явно помічена як dead.

- **Централізований моніторинг:** Один UI для перегляду всіх workflow від усіх юзер-backend. Історія виконань, метрики, логи в одному місці.

- **Гнучка інтеграція:** Link Loom розуміє різні типи задач (HTTP, Shell, Java) й може інтегруватися з будь-якою системою через REST API.

---

## Типовий флоу використання

### 1. Користувач описує workflow

У своєму проєкті (User Backend):

```yaml
# workflows/daily-report.yaml
id: daily-report
schedule: "0 3 * * *"
tasks:
  - id: fetch_data
    type: http
    method: POST
    url: http://user-backend:8080/api/tasks/fetch-data
    
  - id: process_data
    type: http
    method: POST
    url: http://user-backend:8080/api/tasks/process
    dependsOn: [fetch_data]
    
  - id: send_email
    type: http
    method: POST
    url: http://user-backend:8080/api/tasks/send-email
    dependsOn: [process_data]
```

### 2. Користувач реєструє workflow в Link Loom

```java
// User Backend
@Service
public class WorkflowService {
    @Autowired
    private LinkLoomClient linkLoomClient;
    
    @PostConstruct
    public void registerWorkflows() {
        WorkflowDto workflow = readYaml("workflows/daily-report.yaml");
        linkLoomClient.registerWorkflow(workflow);
        // POST http://link-loom:8080/api/workflows
    }
}
```

### 3. Link Loom запускає по розкладу

```
О 03:00 AM:

1. Scheduler: "час запускати daily-report"
2. Workflow Engine: будує DAG (fetch → process → email)
3. Executor: "fetch_data готова" 
   → POST http://user-backend:8080/api/tasks/fetch-data
4. User Backend обробляє, повертає результат
5. Executor: "отримав результат, process готова"
   → POST http://user-backend:8080/api/tasks/process
6. User Backend обробляє, повертає
7. Executor: "email готова"
   → POST http://user-backend:8080/api/tasks/send-email
8. Execution завершена, Link Loom записує статус
```

### 4. Користувач переглядає статус

**Web UI:** http://link-loom:3000
- Список усіх workflow
- DAG граф
- Історія запусків
- Логи кожної задачи
- Можливість ручного запуску

**Desktop App:** Electron додаток
- Той же функціонал що й Web UI
- Localized notifications
- Offline режим (кеш)

**CLI:** Команди з терміналу
```bash
$ loom workflow list
$ loom workflow describe daily-report
$ loom execution list --workflow daily-report
$ loom execution logs exec-123 --follow
$ loom workflow trigger daily-report
```

---

## Архітектура системи

Link Loom Server складається з таких компонентів:

```
┌───────────────────────────────────────────────┐
│         REST API Gateway                      │
│  (WorkflowController, ExecutionController)    │
└─────────────────┬─────────────────────────────┘
                  │
    ┌─────────────┴──────────────┐
    │                            │
    ▼                            ▼
┌──────────────────┐      ┌──────────────────┐
│ Scheduler Core   │      │ Workflow Engine  │
│                  │      │                  │
│ - CronScheduler  │      │ - DAG Builder    │
│ - EventScheduler │      │ - DAG Traversal  │
│ - Lock Manager   │      │ - State Machine  │
└──────────────────┘      └──────────────────┘
         │                        │
         └─────────────┬──────────┘
                       │
                       ▼
            ┌──────────────────────┐
            │  Task Executor       │
            │                      │
            │ - RetryPolicy        │
            │ - TimeoutWatcher     │
            │ - LoadBalancer       │
            └──────────────────────┘
                       │
        ┌──────────────┴──────────────┐
        │                             │
        ▼                             ▼
    ┌─────────────┐          ┌──────────────┐
    │ HTTP Client │          │ Task Runners │
    │             │          │              │
    │ POST /tasks │          │ - Http Task  │
    │ /callback   │          │ - Shell Task │
    └─────────────┘          └──────────────┘
        │
        ▼
    User Backends
    
        │
        ▼
┌───────────────────────────────────────────────┐
│  Persistence Layer                            │
│                                               │
│ - PostgreSQL (metadata, state)                │
│ - MongoDB (logs, history)                     │
└───────────────────────────────────────────────┘
```

### Ключові компоненти

- **REST API:** Endpoints для реєстрації workflow, запуску execution, отримання статусу.

- **Scheduler Core:** Відповідає за запуск workflow по cron або подіям. Перевіряє, чи пора запускати, блокує дублі запусків.

- **Workflow Engine:** Керує DAG логікою. Аналізує залежності, визначає готові до виконання задачи, відстежує прогрес.

- **Task Executor:** Виконавець задач. Отримує готову задачу, вибирає як її виконувати (HTTP callback, Shell, Java), контролює таймаути й retry.

- **Persistence:** PostgreSQL для метаданих (workflow, execution states), MongoDB для логів і історії.

- **Observability:** Metrics (Micrometer), Structured Logs, OpenTelemetry tracing.

---

## Клієнтські програми

### 1. Web UI (React/Vue)

**Функціонал:**
- Dashboard з статусом workflow
- CRUD workflow (create, read, update, delete)
- Перегляд DAG граф (інтерактивна візуалізація)
- Історія execution з таймлайном
- Логи задач (real-time tail)
- Ручний запуск workflow
- Фільтрація й пошук

**Технологія:** React + TypeScript + REST API client

**Де розміщується:** `ui/web-app/`

---

### 2. Desktop App (Electron)

**Функціонал:**
- Той же функціонал що й Web UI
- Desktop notifications при зміні статусу
- Офлайн режим (локальний кеш)
- Системний трей для моніторингу
- Keyboard shortcuts
- Local storage налаштувань

**Технологія:** Electron + React + SQLite (локальний кеш)

**Де розміщується:** `ui/desktop-app/`

---

### 3. CLI Tool

**Функціонал:**
```bash
# Workflow management
$ loom workflow list                    # Список workflow
$ loom workflow create file.yaml        # Загрузити workflow
$ loom workflow describe daily-report   # Деталі workflow
$ loom workflow delete daily-report     # Видалити workflow

# Execution management
$ loom execution list --workflow daily-report    # Список runs
$ loom execution describe exec-123               # Деталі execution
$ loom execution logs exec-123 --follow          # Логи (tail -f)
$ loom execution cancel exec-123                 # Скасувати

# Workflow triggering
$ loom workflow trigger daily-report             # Ручний запуск
$ loom workflow trigger daily-report --wait      # Чекати завершення

# Monitoring
$ loom status                           # Quick status all workflows
$ loom watch                            # Watch mode (live updates)
```

**Технологія:** Go або Rust (standalone binary)

**Де розміщується:** `cli/`

---

## Технологічний стек

### Server (Link Loom)
- **Backend:** Java 17+ + Spring Boot 3.x
- **Database:** PostgreSQL (metadata), MongoDB (logs)
- **Message Queue:** Optional (RabbitMQ/Kafka для large-scale)
- **Observability:** Micrometer, OpenTelemetry, Spring Boot Actuator
- **Container:** Docker, Kubernetes-ready

### Web UI
- **Framework:** React 18+ або Vue 3+
- **State Management:** Redux / Pinia
- **HTTP Client:** Axios / Fetch API
- **Visualization:** React Flow (для DAG grapu)
- **Styling:** Tailwind CSS / Material UI

### Desktop App
- **Framework:** Electron + React
- **Local Storage:** SQLite / LevelDB
- **Networking:** Electron IPC + HTTP
- **Packaging:** electron-builder

### CLI
- **Language:** Go або Rust
- **HTTP Client:** Built-in libraries
- **TUI:** cobra (Go) або clap (Rust)
- **Distribution:** Single binary executable

---

## Переваги Service + Client підходу

| Аспект | Переваги |
|--------|----------|
| **Масштабованість** | 1 Link Loom керує сотнями workflow від N user backends |
| **Мова-agnostic** | User backend на Java, Python, Go, Node.js — довільно |
| **Незалежність** | Кожен user backend розгортається окремо |
| **Моніторинг** | Один центральний UI для всіх workflow |
| **Fault Isolation** | Крах одного user backend не впливає на інших |
| **Development** | Frontend, backend, CLI розробляються паралельно |
| **Deployment** | Server + Clients розгортаються незалежно |

---

## Deployment архітектура

```
┌─────────────────────────────────────────┐
│  Kubernetes Cluster (або Docker Compose)│
│                                         │
│  ┌──────────────────────────────────┐   │
│  │  Link Loom Pods (1-N replicas)  │   │
│  │  - Server                        │   │
│  │  - Scheduler                     │   │
│  │  - Executor                      │   │
│  └──────────────────────────────────┘   │
│                                         │
│  ┌──────────────────────────────────┐   │
│  │  PostgreSQL StatefulSet          │   │
│  │  (metadata persistence)          │   │
│  └──────────────────────────────────┘   │
│                                         │
│  ┌──────────────────────────────────┐   │
│  │  MongoDB StatefulSet             │   │
│  │  (logs, execution history)       │   │
│  └──────────────────────────────────┘   │
│                                         │
│  ┌──────────────────────────────────┐   │
│  │  Web UI Pods (Nginx + React)    │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘

User Backends розташовуються окремо в своїх K8s clusters
і спілкуються з Link Loom через external API
```

---

## Джерела та посилання

Ця документація описує Link Loom як централізований сервіс оркестрації DAG-воркфлоу з множинними клієнтами (Web, Desktop, CLI). Архітектура дозволяє масштабувати систему для управління workflow від різних user backends (на будь-яких мовах програмування).

**Концепції** базуються на досвіді впровадження workflow orchestrators (Airflow, Argo, Temporal) у production.

### Рекомендовані ресурси

- [The Complete Guide to Workflow Orchestration](https://kairntech.com/blog/articles/the-complete-guide-to-workflow-orchestration/)
- [DAG - Argo Workflows - The workflow engine for Kubernetes](https://argo-workflows.readthedocs.io/en/latest/walk-through/dag/)
- [Temporal: The Microservices Orchestration Platform](https://temporal.io/)
- [Apache Airflow Architecture](https://airflow.apache.org/docs/apache-airflow/stable/concepts/overview.html)
- [Spring Boot and Kubernetes](https://spring.io/guides/gs/spring-boot-kubernetes/)