## Загальна структура проєкту

```
link-loom/
├── server/                      # Link Loom Server (Java Backend)
│   ├── pom.xml (root, multi-module)
│   ├── common/
│   ├── api/
│   ├── scheduler-core/
│   ├── workflow-engine/
│   ├── executor/
│   ├── worker/
│   ├── persistence/
│   └── observability/
│
├── clients/                     # Клієнтські програми
│   ├── web-ui/                  # Web Dashboard (React/Vue)
│   ├── desktop-app/             # Desktop App (Electron)
│   └── cli/                      # CLI Tool (Go/Rust)
│
├── sdk/                         # Client Libraries
│   ├── java-client/             # Maven package for user backends
│   ├── python-client/           # Pip package
│   ├── go-client/               # Go module
│   └── js-client/               # NPM package
│
└── docs/
    ├── api-spec.md              # OpenAPI/Swagger
    ├── architecture.md
    ├── user-guide.md
    └── deployment.md
```

---

## 1. СЕРВЕР (Link Loom Backend)

### Структура модулів

```
server/
├── pom.xml (parent)
├── build-docker.sh
├── docker-compose.yml
│
├── common/
│   ├── src/main/java/com/linkloom/common/
│   │   ├── dto/
│   │   │   ├── WorkflowDto.java
│   │   │   ├── TaskDto.java
│   │   │   ├── ExecutionDto.java
│   │   │   └── TaskExecutionDto.java
│   │   ├── enums/
│   │   │   ├── TaskType.java
│   │   │   ├── TaskStatus.java
│   │   │   └── WorkflowStatus.java
│   │   ├── exceptions/
│   │   │   └── SchedulerException.java
│   │   └── utils/
│   │       └── IdGenerator.java
│   └── pom.xml
│
├── api/
│   ├── src/main/java/com/linkloom/api/
│   │   ├── controller/
│   │   │   ├── WorkflowController.java
│   │   │   ├── ExecutionController.java
│   │   │   ├── TaskController.java
│   │   │   └── HealthController.java
│   │   ├── request/
│   │   │   ├── CreateWorkflowRequest.java
│   │   │   ├── RunWorkflowRequest.java
│   │   │   └── TaskCallbackRequest.java
│   │   ├── response/
│   │   │   ├── WorkflowResponse.java
│   │   │   ├── ExecutionResponse.java
│   │   │   └── TaskResultResponse.java
│   │   ├── mapper/
│   │   │   └── WorkflowMapper.java
│   │   └── ApiApplication.java
│   ├── resources/
│   │   └── application.yml
│   └── pom.xml
│
├── scheduler-core/
│   ├── src/main/java/com/linkloom/scheduler_core/
│   │   ├── scheduler/
│   │   │   ├── WorkflowScheduler.java
│   │   │   ├── CronScheduler.java
│   │   │   └── EventScheduler.java
│   │   ├── trigger/
│   │   │   ├── Trigger.java (interface)
│   │   │   ├── CronTrigger.java
│   │   │   └── ManualTrigger.java
│   │   ├── lock/
│   │   │   ├── SchedulerLock.java (interface)
│   │   │   └── DbSchedulerLock.java
│   │   └── service/
│   │       └── ScheduleService.java
│   └── pom.xml
│
├── workflow-engine/
│   ├── src/main/java/com/linkloom/workflow_engine/
│   │   ├── model/
│   │   │   ├── Workflow.java
│   │   │   ├── Task.java
│   │   │   └── TaskDependency.java
│   │   ├── dag/
│   │   │   ├── DagBuilder.java
│   │   │   └── DagTraversal.java
│   │   ├── state/
│   │   │   ├── WorkflowStateMachine.java
│   │   │   └── TaskStateMachine.java
│   │   └── engine/
│   │       ├── WorkflowEngine.java
│   │       └── TaskDispatcher.java
│   └── pom.xml
│
├── executor/
│   ├── src/main/java/com/linkloom/executor/
│   │   ├── executor/
│   │   │   ├── TaskExecutor.java (interface)
│   │   │   ├── HttpTaskExecutor.java
│   │   │   └── ShellTaskExecutor.java
│   │   ├── retry/
│   │   │   ├── RetryPolicy.java
│   │   │   └── RetryExecutor.java
│   │   ├── timeout/
│   │   │   └── TimeoutWatcher.java
│   │   └── load/
│   │       └── WorkerLoadBalancer.java
│   └── pom.xml
│
├── worker/
│   ├── src/main/java/com/linkloom/worker/
│   │   ├── worker/
│   │   │   ├── WorkerNode.java
│   │   │   ├── WorkerRegistry.java
│   │   │   └── WorkerHeartbeat.java
│   │   ├── task/
│   │   │   ├── TaskRunner.java (interface)
│   │   │   ├── HttpTaskRunner.java
│   │   │   ├── JavaTaskRunner.java
│   │   │   └── ShellTaskRunner.java
│   │   └── sandbox/
│   │       └── ExecutionSandbox.java
│   └── pom.xml
│
├── persistence/
│   ├── src/main/java/com/linkloom/persistence/
│   │   ├── entity/
│   │   │   ├── WorkflowEntity.java
│   │   │   ├── WorkflowRunEntity.java
│   │   │   ├── TaskRunEntity.java
│   │   │   └── ExecutionLogEntity.java
│   │   ├── repository/
│   │   │   ├── WorkflowRepository.java
│   │   │   ├── ExecutionRepository.java
│   │   │   └── TaskRunRepository.java
│   │   ├── mapper/
│   │   │   └── EntityMapper.java
│   │   └── config/
│   │       └── PersistenceConfig.java
│   └── pom.xml
│
└── observability/
    ├── src/main/java/com/linkloom/observability/
    │   ├── metrics/
    │   │   └── SchedulerMetrics.java
    │   ├── logging/
    │   │   └── StructuredLogger.java
    │   └── tracing/
    │       └── ExecutionTracer.java
    └── pom.xml
```

---

## 2. WEB UI (React/Vue)

```
clients/web-ui/
├── package.json
├── tsconfig.json
├── vite.config.ts (або webpack.config.js)
├── public/
│   └── index.html
├── src/
│   ├── api/
│   │   ├── client.ts              # HTTP клієнт для Link Loom API
│   │   ├── workflows.ts
│   │   ├── executions.ts
│   │   └── tasks.ts
│   ├── components/
│   │   ├── WorkflowList.tsx        # Таблиця workflow
│   │   ├── WorkflowDetails.tsx     # DAG граф + інформація
│   │   ├── ExecutionHistory.tsx    # Історія запусків
│   │   ├── ExecutionTimeline.tsx   # Timeline задач
│   │   ├── TaskLogs.tsx            # Логи задачи (tail)
│   │   └── Sidebar.tsx
│   ├── pages/
│   │   ├── Dashboard.tsx           # Головна сторінка
│   │   ├── WorkflowPage.tsx        # Деталі workflow
│   │   ├── ExecutionPage.tsx       # Деталі execution
│   │   └── SettingsPage.tsx
│   ├── hooks/
│   │   ├── useWorkflows.ts
│   │   ├── useExecutions.ts
│   │   └── usePolling.ts           # Real-time updates
│   ├── types/
│   │   └── index.ts                # TypeScript types
│   ├── App.tsx
│   └── index.tsx
├── tailwind.config.js              # або MUI/Bootstrap config
└── Dockerfile                      # For containerization

**Функціонал:**
- Dashboard: список workflow з статусами
- Workflow details: DAG граф (React Flow), metadata
- Execution list: таблиця з фільтрацією
- Execution timeline: 
  - Кожна задача як блок
  - Коло біля задачи (зелене/червоне)
  - Timeline показує order & duration
- Task logs: real-time output (WebSocket або polling)
- Create/Edit workflow: форма для YAML
- Trigger workflow: кнопка для ручного запуску
```

---

## 3. DESKTOP APP (Electron)

```
clients/desktop-app/
├── package.json
├── tsconfig.json
├── electron-builder.yml
├── src/
│   ├── main/
│   │   ├── index.ts              # Main Electron process
│   │   ├── preload.ts            # Security context
│   │   └── ipc/
│   │       ├── workflow-ipc.ts
│   │       └── execution-ipc.ts
│   ├── renderer/
│   │   ├── components/           # Те саме що в Web UI
│   │   ├── pages/
│   │   ├── hooks/
│   │   └── App.tsx
│   ├── storage/
│   │   ├── sqlite-store.ts       # Local DB for cache
│   │   └── config-store.ts       # User settings
│   └── notifications/
│       └── notifier.ts           # Desktop notifications
├── resources/
│   └── icon.icns / icon.png      # App icon
└── Dockerfile                    # Для дистрибуції

**Додатковий функціонал vs Web UI:**
- System tray icon для quick status
- Desktop notifications при зміні статусу
- Local SQLite cache (offline mode)
- Keyboard shortcuts (Cmd+Enter для trigger, etc.)
- Native file dialogs для upload YAML
- Settings: Link Loom server URL, refresh interval
```

---

## 4. CLI TOOL (Go або Rust)

```
clients/cli/
├── main.go (або main.rs)
├── go.mod (або Cargo.toml)
├── cmd/
│   ├── workflow.go
│   │   ├── list.go           # loom workflow list
│   │   ├── describe.go       # loom workflow describe ID
│   │   ├── create.go         # loom workflow create file.yaml
│   │   ├── delete.go         # loom workflow delete ID
│   │   └── trigger.go        # loom workflow trigger ID
│   ├── execution.go
│   │   ├── list.go           # loom execution list --workflow ID
│   │   ├── describe.go       # loom execution describe ID
│   │   ├── logs.go           # loom execution logs ID --follow
│   │   ├── cancel.go         # loom execution cancel ID
│   │   └── retry.go          # loom execution retry ID
│   ├── status.go             # loom status (quick overview)
│   ├── watch.go              # loom watch (live mode)
│   └── config.go             # loom config set/get
├── client/
│   ├── http_client.go        # Link Loom API client
│   ├── workflows.go
│   └── executions.go
├── formatter/
│   ├── table.go              # Таблиця вывода
│   ├── json.go               # JSON format
│   └── yaml.go               # YAML format
├── config/
│   └── config.go             # ~/.loom/config
└── Dockerfile                # For distribution

**Примери команд:**
```bash
$ loom workflow list
$ loom workflow describe daily-report
$ loom execution logs exec-123 --follow
$ loom workflow trigger daily-report --wait
$ loom status                    # Quick status
$ loom watch                     # Live refresh
$ loom config set server-url http://localhost:8080
```
```

---

## 5. CLIENT LIBRARIES (SDK)

### Java Client
```
sdk/java-client/
├── pom.xml
└── src/main/java/com/linkloom/client/
├── LinkLoomClient.java       # Main interface
├── WorkflowClient.java
├── ExecutionClient.java
├── http/
│   └── RestTemplateClient.java
└── config/
└── LinkLoomAutoConfiguration.java
```

Використання:
```java
@Autowired
private LinkLoomClient client;

// Register workflow
WorkflowDto wf = loadYaml("workflow.yaml");
client.registerWorkflow(wf);

// Trigger manually
client.triggerWorkflow("daily-report");

// Get execution status
ExecutionDto exec = client.getExecution("exec-123");
```

### Python Client
```
sdk/python-client/
├── setup.py
├── linkloom/
│   ├── __init__.py
│   ├── client.py              # Main class
│   ├── workflow.py
│   ├── execution.py
│   └── http.py
```

### Go Client
```
sdk/go-client/
├── go.mod
├── client.go
├── workflow.go
├── execution.go
└── http.go
```

---

## Потік даних

```
┌──────────────────────────────────────────┐
│ USER BACKEND (any language)              │
│                                          │
│ 1. LinkLoomClient.registerWorkflow()     │
│    └─> POST /api/workflows (Link Loom)   │
│                                          │
│ 2. Дожидаемся callback от Link Loom      │
│    POST /api/tasks/my-task               │
│                                          │
│ 3. Выполняем бизнес-логику               │
│                                          │
│ 4. Возвращаем результат                  │
│    └─> {status, output, logs}            │
└──────────────────────────────────────────┘

         ▲
         │ REST (JSON)
         │
┌────────▼──────────────────────────────────┐
│  LINK LOOM SERVER                        │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │ REST API Gateway                   │  │
│  │ - POST /workflows                  │  │
│  │ - GET /executions                  │  │
│  │ - POST /callback (from user)       │  │
│  └────────────────────────────────────┘  │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │ Scheduler + Workflow Engine        │  │
│  │ - DAG builder                      │  │
│  │ - Task dispatcher                  │  │
│  │ - State machine                    │  │
│  └────────────────────────────────────┘  │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │ Task Executor                      │  │
│  │ - HTTP client for callbacks        │  │
│  │ - Retry logic                      │  │
│  │ - Timeout watcher                  │  │
│  └────────────────────────────────────┘  │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │ Persistence                        │  │
│  │ - PostgreSQL (metadata)            │  │
│  │ - MongoDB (logs)                   │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
         │
         ▲ REST (JSON)
         │
┌────────┴──────────────────────────────────┐
│  CLIENTS (Web / Desktop / CLI)            │
│                                          │
│  GET /workflows                          │
│  GET /executions                         │
│  POST /workflows/:id/trigger             │
└──────────────────────────────────────────┘
```

---

## Деплоймент

### Development (docker-compose)
```bash
cd server
docker-compose up
# Link Loom at http://localhost:8080
# Web UI at http://localhost:3000
# PostgreSQL at localhost:5432
# MongoDB at localhost:27017
```

### Production (Kubernetes)
```
link-loom-server pod (replicas: 3)
├── Java backend
├── PostgreSQL StatefulSet
├── MongoDB StatefulSet
└── Web UI Deployment

User Backends розташовуються в своїх clusters
Спілка через public Link Loom API endpoint
```

---

## Summary

| Компонент | Мова | Мета | Користувач |
|-----------|------|------|-----------|
| Server | Java | Scheduler + Executor | DevOps, Infrastructure |
| Web UI | React/TS | Dashboard | DevOps, Analyst, PM |
| Desktop | Electron | Offline + Notifications | DevOps, Power Users |
| CLI | Go/Rust | Scripting, DevOps | DevOps, Automation |
| Java SDK | Java | User Backend integration | Backend Dev |
| Python SDK | Python | User Backend integration | Data Science, Backend Dev |
| Go SDK | Go | User Backend integration | Go Backend Dev |