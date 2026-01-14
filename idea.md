## Ідея коротко

**Link Loom** — це централізований сервіс оркестрації DAG-воркфлоу для керування workflow від кількох юзер-приложень (на будь-яких мовах програмування).

Ключова ціль — навчитися писати:
* **складний, конкурентний, масштабований Java-код** (Server)
* **REST API і async архітектуру** (зв'язок із user backends)
* **множинні клієнти** (Web, Desktop, CLI)

---

## 1. Core концепції

### 1.1 Workflow (DAG)

* Workflow = набір Task з залежностями
* Task має:
    * id, name
    * тип (http, shell, java)
    * залежності на інші tasks
    * retry policy, timeout
    * конфігурація (URL для http, class для java, command для shell)

```text
Приклад DAG:

fetch_data ──┐
             ├─► process_data ──┐
             │                  ├─► send_email
validate ────┘                  │
                                │
             retry_email ────────┘
```

### 1.2 Execution Model

* **Scheduler** вирішує *коли* запускати (по cron або события)
* **Workflow Engine** вирішує *який порядок* (DAG traversal)
* **Executor** вирішує *де* і *як* виконувати (HTTP callback до user backend)
* **User Backend** виконує реальну бізнес-логіку й повертає результат

---

## 2. Архітектура (High Level)

```
┌─────────────────────────────────────┐
│      LINK LOOM SERVER               │
│                                     │
│  ┌─────────────┐   ┌────────────┐  │
│  │  Scheduler  │   │  Workflow  │  │
│  │             │   │   Engine   │  │
│  └──────┬──────┘   └─────┬──────┘  │
│         │ Events         │         │
│         └────────┬───────┘         │
│                  │                 │
│         ┌────────▼────────┐        │
│         │    Executor     │        │
│         │                 │        │
│         │ ┌─────────────┐ │        │
│         │ │ RetryPolicy │ │        │
│         │ │ Timeout     │ │        │
│         │ └─────────────┘ │        │
│         └────────┬────────┘        │
│                  │                 │
│         ┌────────▼────────┐        │
│         │  HTTP Client    │        │
│         └────────┬────────┘        │
└─────────────────┼──────────────────┘
                  │ REST API
        ┌─────────┴──────────┐
        │                    │
        ▼                    ▼
    ┌──────────────┐   ┌──────────────┐
    │  User       │   │ User         │
    │  Backend 1  │   │ Backend N    │
    │             │   │              │
    │ POST /      │   │ POST /       │
    │ tasks/...   │   │ tasks/...    │
    └──────────────┘   └──────────────┘
        │ Logic              │ Logic
        ▼                    ▼
    Fetch, Process,      Other Business
    Database Ops         Operations
```

---

## 3. Модулі (Maven multi-module)

```
server/
├── pom.xml (parent)
├── common/
│   └── DTO, Enum, Exceptions, Utils
├── api/
│   └── REST Controllers
├── scheduler-core/
│   └── Scheduling Logic (cron, triggers)
├── workflow-engine/
│   └── DAG Builder, State Machine
├── executor/
│   └── Task Execution, Retry, Timeout
├── worker/
│   └── Task Runners (HTTP, Shell, Java)
├── persistence/
│   └── JPA Entities, Repositories
└── observability/
    └── Metrics, Logs, Tracing

clients/
├── web-ui/
│   └── React Dashboard
├── desktop-app/
│   └── Electron App
└── cli/
    └── Go/Rust CLI Tool

sdk/
├── java-client/
├── python-client/
├── go-client/
└── js-client/
```

---

## 4. Workflow DSL

### YAML формат

```yaml
id: daily-report
name: Daily Report
schedule: "0 3 * * *"  # Cron: 3 AM every day
retryStrategy: exponential

tasks:
  - id: fetch_data
    type: http
    method: POST
    url: http://user-backend:8080/api/tasks/fetch-data
    timeout: 5m
    maxAttempts: 3
    retryOn: [TimeoutException, ConnectionException]
    
  - id: validate
    type: http
    method: POST
    url: http://user-backend:8080/api/tasks/validate
    timeout: 2m
    dependsOn: [fetch_data]
    
  - id: process
    type: http
    method: POST
    url: http://user-backend:8080/api/tasks/process
    timeout: 10m
    dependsOn: [fetch_data, validate]
    
  - id: send_email
    type: http
    method: POST
    url: http://user-backend:8080/api/tasks/send-email
    timeout: 1m
    dependsOn: [process]
```

### Реалізація

* SnakeYAML для парсингу YAML
* WorkflowValidator для перевірки
* DagBuilder для побудови графу

---

## 5. Scheduler Core

### Функції

* **Cron Scheduling:** Запуск по cron expression
* **Event-based:** Запуск по external eventos (HTTP trigger, message, file)
* **Debounce/Delay:** Затримка перед запуском
* **Distributed Lock:** Гарантія що workflow запускається тільки один раз одночасно (навіть з кількома сервер-инстансами)

### Java фокус

* `ScheduledExecutorService` для periodic задач
* Database lock для distributed scheduling
* Thread-safe state management

---

## 6. Workflow Engine

### Стани Task-а

```text
PENDING → READY → RUNNING → SUCCESS
                     ↓
                   FAILED → RETRY → READY (повтор) → ...
                     ↓
                    DEAD (max retries exceeded)
```

### Реалізація

* **State Machine:** Контроль переходів між станами
* **Immutable State:** Кожна зміна стану — новий об'єкт
* **Event-driven:** Кожна зміна генерує подію (для observability)
* **DAG Traversal:** DFS/BFS для визначення готових задач

---

## 7. Executor & HTTP Callbacks

### Executor

* Отримує готову задачу від Workflow Engine
* Вибирає task runner (HTTP, Shell, Java)
* Передає задачу user backend (HTTP POST callback)
* Чекає результату з таймаутом
* На помилку — retry logic

### HTTP Callback (до User Backend)

```
POST /api/tasks/fetch-data HTTP/1.1
Host: user-backend:8080
Content-Type: application/json

{
  "executionId": "exec-abc-123",
  "taskId": "fetch_data",
  "input": {
    "date": "2024-01-14",
    "limit": 1000
  },
  "attempt": 1,
  "timeout": 300000
}
```

### User Backend відповідь

```json
{
  "status": "SUCCESS",
  "output": {
    "recordCount": 1000,
    "checksum": "abc123"
  },
  "logs": "Fetched 1000 records from API",
  "duration": 5234
}
```

---

## 8. Retry / Fault Tolerance

```yaml
retryPolicy:
  maxAttempts: 5
  backoffStrategy: exponential  # 1s → 2s → 4s → 8s → 16s
  retryOn:
    - TimeoutException
    - ConnectionException
    - HttpServerError (5xx)
  
  dontRetryOn:
    - ValidationException
    - ResourceNotFoundException (4xx)
```

* **Circuit Breaker:** Не сипати запити до неживого backend
* **Dead Letter Queue:** Задачи що не пройшли max retries
* **Partial Workflow Recovery:** При крахі executor — replay невиконаних задач

---

## 9. Persistence

### PostgreSQL (metadata)

```sql
-- Workflows
CREATE TABLE workflows (
  id VARCHAR PRIMARY KEY,
  name VARCHAR,
  schedule VARCHAR,  -- cron
  definition TEXT,   -- YAML
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);

-- Executions (runs)
CREATE TABLE executions (
  id VARCHAR PRIMARY KEY,
  workflow_id VARCHAR,
  status VARCHAR,  -- PENDING, RUNNING, SUCCESS, FAILED
  started_at TIMESTAMP,
  completed_at TIMESTAMP,
  created_at TIMESTAMP
);

-- Task executions
CREATE TABLE task_executions (
  id VARCHAR PRIMARY KEY,
  execution_id VARCHAR,
  task_id VARCHAR,
  status VARCHAR,
  attempt INT,
  started_at TIMESTAMP,
  completed_at TIMESTAMP,
  output TEXT,
  error TEXT
);
```

### MongoDB (logs & history)

```javascript
// Execution logs (для efficient querying)
db.execution_logs.insertOne({
  _id: ObjectId(),
  execution_id: "exec-abc-123",
  task_id: "fetch_data",
  level: "INFO",
  message: "Fetched 1000 records",
  timestamp: ISODate(),
  context: {...}
});
```

---

## 10. Observability

* **Metrics (Micrometer):**
    * `scheduler.executions.total` — кількість execution
    * `execution.duration` — час виконання
    * `task.retry.count` — кількість retry

* **Logs (Structured):**
    * JSON format для easy parsing
    * Context tracking (execution ID, task ID)
    * Level-based filtering

* **Tracing (OpenTelemetry):**
    * Request path через всі компоненти
    * Latency breakdown

* **Web UI:**
    * DAG view з interactive граф
    * Task timeline з timeline виконання
    * Retry history
    * Log viewer з search

---

## 11. Клієнти

### Web UI (React)

* **Dashboard:** Список workflow з статусами
* **Workflow Details:** DAG граф + metadata
* **Execution History:** Таблиця виконань
* **Execution Timeline:** Visual timeline задач
* **Logs:** Real-time tail з search

### Desktop App (Electron)

* Те саме що Web UI
* + System notifications
* + Offline cache
* + Native file dialogs

### CLI (Go/Rust)

```bash
loom workflow list
loom workflow describe daily-report
loom execution logs exec-123 --follow
loom workflow trigger daily-report
```

---

## 12. SDK для User Backends

### Java SDK

```java
@Autowired
private LinkLoomClient client;

// Register workflow
WorkflowDto workflow = loadYaml("workflow.yaml");
client.registerWorkflow(workflow);

// Create task handler
@PostMapping("/api/tasks/fetch-data")
public TaskResult fetchData(@RequestBody TaskRequest req) {
  // Твоя логіка
  return new TaskResult("SUCCESS", output, logs);
}
```

### Python SDK

```python
from linkloom import LinkLoomClient

client = LinkLoomClient("http://link-loom:8080")
workflow = load_yaml("workflow.yaml")
client.register_workflow(workflow)

@app.post("/api/tasks/fetch-data")
def fetch_data(request):
    # Твоя логіка
    return {"status": "SUCCESS", "output": ...}
```

---

## 13. Роадмап (10–12 тижнів)

### **Тиждень 1–2: Foundation**
* ✅ DTO, Enums, Exceptions (common module)
* ✅ DAG Builder + DagTraversal (workflow-engine)
* ✅ State Machines (task + workflow)
* Tests: Unit tests для DAG

### **Тиждень 3–4: Scheduling & Execution**
* ✅ CronScheduler (scheduler-core)
* ✅ TaskExecutor з HTTP callbacks (executor)
* ✅ Basic REST API (api module)
* Tests: Integration tests

### **Тиждень 5–6: Persistence & Retry**
* ✅ JPA Entities (persistence)
* ✅ RetryPolicy, TimeoutWatcher
* ✅ DeadLetterQueue
* Tests: Database tests

### **Тиждень 7–8: Observability & Hardening**
* ✅ Metrics, Structured Logs
* ✅ Distributed locking (scheduler)
* ✅ Circuit breaker
* Tests: Load tests

### **Тиждень 9–10: Web UI**
* ✅ React Dashboard
* ✅ DAG visualization (React Flow)
* ✅ Execution timeline
* ✅ Real-time log viewer

### **Тиждень 11–12: Desktop & CLI**
* ✅ Electron Desktop App
* ✅ Go/Rust CLI Tool
* ✅ Documentation
* ✅ Docker/K8s ready

---

## 14. Чому це цінно

**На інтерв'ю:**
- DAG algorithms (topological sort, cycle detection)
- Concurrency (ExecutorService, CompletableFuture, locks)
- Distributed systems (HTTP, eventual consistency)
- State machines
- REST API design
- Database design (PostgreSQL + MongoDB)
- System design (service + clients)

**На практиці:**
- Реальна архітектура (як Airflow, Temporal, Argo)
- Масштабованість від самого початку
- Multi-language support (Java backend, any user backend)
- Production-ready observability
- DevOps-friendly deployment

**Технічна глибина:**
- Advanced Java (reflection, annotations, concurrent collections)
- Spring Boot advanced (auto-configuration, AOP, async)
- YAML parsing & validation
- HTTP client & server best practices
- Database transactions & migrations