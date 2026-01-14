## Як це виглядає на практиці

**Link Loom — це окремий сервіс** (з бекендом + API + Web UI), але він **не керує виконанням задач напрямик**.

Замість цього:
- **Link Loom** запускає workflow по розкладу й керує порядком виконання (DAG)
- **User Backends** отримують HTTP callback від Link Loom й виконують свою бізнес-логіку
- **Клієнти** (Web, Desktop, CLI) дозволяють керувати workflow й переглядати статус

Це **інфраструктурний сервіс**, як:
* GitHub Actions (central workflow runner)
* Jenkins (scheduler + executor)
* Airflow (DAG orchestrator)

---

## Архітектура проєкту: 3 компоненти

### 1. Link Loom Server (central)

* Java + Spring Boot backend
* REST API (`POST /workflows`, `GET /executions`, etc.)
* PostgreSQL + MongoDB для зберігання
* Scheduler працює 24/7
* Workflow Engine керує DAG
* Executor запускає HTTP callbacks

### 2. User Backends (distributed)

* Будь-які проєкти користувачів (Spring Boot, Flask, Go, Node.js, тощо)
* Мають `link-loom-client` (мінімальна бібліотека)
* Реєструють свої workflow в Link Loom
* Отримують HTTP POST від Link Loom з наказом виконувати задачу
* Обробляють задачу й повертають результат

### 3. Клієнти (для керування)

* **Web UI** (React) — dashboard для мониторингу й управління
* **Desktop App** (Electron) — офлайн режим + notifications
* **CLI Tool** (Go/Rust) — для DevOps, скрипти, автоматизація

---

## Типовий сценарій використання

### Крок 1. Користувач описує workflow в своєму проєкті

Файл `src/resources/workflows/daily-report.yaml` в user backend:

```yaml
id: daily-report
schedule: "0 3 * * *"  # кожного дня о 3:00 AM
tasks:
  - id: fetch_data
    type: http
    method: POST
    url: http://user-backend:8080/api/tasks/fetch-data
    
  - id: process
    type: http
    method: POST
    url: http://user-backend:8080/api/tasks/process
    dependsOn: [fetch_data]
    
  - id: send_email
    type: http
    method: POST
    url: http://user-backend:8080/api/tasks/send-email
    dependsOn: [process]
```

---

### Крок 2. Користувач реєструє workflow в Link Loom

User backend поднімається і робить:

```java
@Service
public class WorkflowSetup {
    @Autowired
    private LinkLoomClient linkLoomClient;
    
    @PostConstruct
    public void init() {
        WorkflowDto workflow = loadYaml("workflows/daily-report.yaml");
        // POST http://link-loom:8080/api/workflows
        linkLoomClient.registerWorkflow(workflow);
    }
}
```

Link Loom отримує workflow й зберігає в БД:
- Workflow ID
- Tasks + dependencies
- Schedule (cron)

---

### Крок 3. Link Loom запускає по розкладу

О **03:00:00**:

```
1. Scheduler Core: "Час запускати daily-report!"
   └─> Checks cron "0 3 * * *" ✓

2. Execution created:
   └─> execution_id: exec-abc-123
   └─> status: PENDING
   └─> workflow: daily-report

3. Workflow Engine: Будує DAG
   └─> fetch_data (no deps) → READY
   └─> process (wait for fetch_data)
   └─> send_email (wait for process)

4. Task Executor: "fetch_data готова!"
   └─> POST http://user-backend:8080/api/tasks/fetch-data
   └─> Body: {executionId: "exec-abc-123", ...}
```

---

### Крок 4. User Backend отримує callback й виконує

```java
@RestController
@RequestMapping("/api/tasks")
public class TaskController {
    
    @PostMapping("/fetch-data")
    public TaskResult fetchData(@RequestBody TaskExecutionRequest req) {
        // Твоя бізнес-логіка
        List<Data> data = fetchDataFromDatabase();
        
        return new TaskResult(
            status: "SUCCESS",
            output: data,
            logs: "Fetched 1000 records"
        );
    }
}
```

User backend повертає результат до Link Loom:

```json
{
  "status": "SUCCESS",
  "output": {...},
  "logs": "Fetched 1000 records"
}
```

---

### Крок 5. Link Loom записує результат і запускає наступну задачу

```
1. Executor отримав результат від fetch_data ✓
2. Execution updated: fetch_data → SUCCESS
3. DAG Engine: "process можна запускати"
   └─> POST http://user-backend:8080/api/tasks/process
   └─> Body: {input: {...fetched data...}}

4. User backend обробляє process → SUCCESS

5. Executor: "send_email можна запускати"
   └─> POST http://user-backend:8080/api/tasks/send-email

6. User backend відправляє email → SUCCESS

7. Execution завершена: execution_status = SUCCESS
   └─> Duration: 5 minutes 23 seconds
   └─> All tasks completed
```

---

### Крок 6. Користувач дивиться результати

#### **Web UI:** http://link-loom:3000

- Dashboard: список усіх workflow
- Клік на workflow: DAG граф з кольорами (зелений=success, червоний=failed)
- Клік на execution: timeline задач із логами
- Логи task: real-time output

#### **Desktop App:** Electron

- Те саме що Web UI
- Notifications при зміні статусу
- Offline режим (кеш execution історії)
- System tray для quick status

#### **CLI:**

```bash
$ loom workflow list
ID              NAME            SCHEDULE    CREATED
daily-report    Daily Report    0 3 * * *   2024-01-10

$ loom workflow describe daily-report
ID: daily-report
Name: Daily Report
Schedule: 0 3 * * *
Tasks: 3
  - fetch_data (type: http)
  - process (depends: fetch_data)
  - send_email (depends: process)

$ loom execution list --workflow daily-report
ID              STATUS    STARTED             DURATION
exec-abc-123    SUCCESS   2024-01-14 03:00    5m 23s
exec-def-456    SUCCESS   2024-01-13 03:00    4m 51s

$ loom execution logs exec-abc-123 --follow
[fetch_data] Fetched 1000 records
[process] Processed data, 950 valid
[send_email] Email sent to admin@example.com
EXECUTION COMPLETED SUCCESSFULLY
```

---

## Розподіл компонентів

```
┌─────────────────────────────────────────────────────────┐
│ LINK LOOM INFRASTRUCTURE (One Central Instance)         │
│                                                         │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ Java Backend Server                                 │ │
│ │ - Scheduler Core                                    │ │
│ │ - Workflow Engine (DAG)                             │ │
│ │ - Executor                                          │ │
│ │ - REST API                                          │ │
│ └─────────────────────────────────────────────────────┘ │
│                                                         │
│ ┌────────────────┐  ┌──────────────┐  ┌─────────────┐ │
│ │ PostgreSQL     │  │ MongoDB      │  │ Prometheus  │ │
│ │ (metadata)     │  │ (logs)       │  │ (metrics)   │ │
│ └────────────────┘  └──────────────┘  └─────────────┘ │
└──────────────────────┬──────────────────────────────────┘
                       │ REST API
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
    ┌────────┐   ┌────────┐    ┌──────────────┐
    │ Web UI │   │Desktop │    │ CLI Tool     │
    │(React) │   │(Elec.) │    │ (Go/Rust)    │
    └────────┘   └────────┘    └──────────────┘
        │              │              │
        └──────────────┼──────────────┘
                       │ Управління workflow
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
    ┌────────────┐ ┌────────────┐ ┌────────────┐
    │User Backend│ │User Backend│ │User Backend│
    │   (Java)   │ │ (Python)   │ │  (Node.js) │
    └────────────┘ └────────────┘ └────────────┘
        │              │              │
        │HTTP callback │HTTP callback │
        │(task execute)│(task execute)│
        ▼              ▼              ▼
    Бізнес логіка \ Fetch Data \ Send Notifications
```

---

## Для кого він

| Роль        | Як використовує                           |
|-------------|-------------------------------------------|
| Dev (Java)  | Описує workflow, реалізує task handlers    |
| Dev (Python)| Додає link-loom-client, реалізує handlers |
| DevOps      | Розгортає Link Loom, моніторить           |
| QA/Analyst  | Дивиться в Web UI, запускає вручну        |
| Бізнес      | Отримує стан workflow від вебсайту        |

---

## Мінімальний MVP

**Для Link Loom Server:**

1. REST endpoint `POST /workflows` (реєстрація)
2. REST endpoint `GET /workflows` (список)
3. YAML парсер → DAG Builder
4. @Scheduled запуск по cron
5. HTTP client для callbacks
6. `ExecutorService` для паралелізму
7. In-memory або PostgreSQL для стану
8. Логи в консоль + файл

**Для User Backend:**

1. Звичайне Spring Boot приложення
2. 2-3 REST endpoints `POST /api/tasks/*`
3. Лікже інтеграція з Link Loom (чтення YAML, HTTP POST)

**Клієнти:**

1. Проста Web UI (React) — таблиця workflow + статуси
2. CLI — curl-based або Go тулка з basic командами

Цього вже достатньо, щоб бачити **реальну цінність системи**.

---

## Чому це не "overengineering"

Бо:

* Такі системи реально існують (Airflow, Temporal, Argo)
* Java тут використовується на 100% (concurrency, scheduling, DAG алгоритми)
* Service + Client архітектура це що реально запитують на senior інтерв'ю
* Scalability від самого початку (N user backends ↔ 1 Link Loom)
* Реальний usecase для production (ETL, CI/CD, бізнес-процеси)

Це не гарячок чи toy project — це **повноцінна інфраструктурна система**.