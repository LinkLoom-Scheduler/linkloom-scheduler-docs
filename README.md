# Link Loom Scheduler

[![Java](https://img.shields.io/badge/Java-17+-blue)](https://www.java.com)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.2+-green)](https://spring.io/projects/spring-boot)
[![Maven](https://img.shields.io/badge/Maven-3.8+-orange)](https://maven.apache.org/)
[![License](https://img.shields.io/badge/License-MIT-yellow)](LICENSE)
[![Status](https://img.shields.io/badge/Status-Active%20Development-blue)]()

A **centralized workflow orchestration service** for managing DAG-based processes across multiple distributed backends. Similar to Airflow, Jenkins, and Temporal, but designed as a scalable, language-agnostic microservice with multiple client interfaces.

> **English** | [Українська](README.ua.md)

---

## 🎯 Overview

Link Loom is a **centralized orchestrator** that:

1. **Schedules** workflows based on cron expressions or external events
2. **Manages** task dependencies as Directed Acyclic Graphs (DAG)
3. **Executes** tasks reliably with retry logic and timeout handling
4. **Monitors** all workflow executions from a single place
5. **Integrates** with any backend service via HTTP callbacks

### Architecture

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
    │ (React)      │   │  (Electron)  │   │ (Go/Rust)    │
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
    │ POST /tasks/... │            │ POST /tasks/...    │
    │ (callback)      │            │ (callback)         │
    └─────────────────┘            └────────────────────┘
```

---

## ✨ Key Features

### 🔄 **DAG-Based Workflows**
- Define complex workflows with task dependencies
- Automatic topological sorting and execution order
- Cycle detection and validation

### 📅 **Flexible Scheduling**
- Cron-based scheduling (Unix cron expressions)
- Manual triggering via API
- Event-based triggers (optional)

### 🔁 **Fault Tolerance**
- Automatic retry with exponential backoff
- Timeout handling and dead-letter queues
- Partial workflow recovery
- Circuit breaker pattern

### 🌐 **Multi-Language Support**
- User backends in **any language** (Java, Python, Go, Node.js, etc.)
- Communication via simple HTTP callbacks
- Language-agnostic client libraries (Java, Python, Go, JS)

### 📊 **Centralized Monitoring**
- Single dashboard for all workflows
- Real-time execution timeline
- Detailed logs and metrics
- Historical data and statistics

### 📱 **Multiple Interfaces**
- **Web UI** (React) — Dashboard for monitoring and management
- **Desktop App** (Electron) — Offline mode + system notifications
- **CLI Tool** (Go/Rust) — DevOps and automation friendly

### 🎯 **Production-Ready**
- PostgreSQL for metadata persistence
- MongoDB for logs and execution history
- Kubernetes-ready deployment
- Built-in observability (metrics, structured logs, tracing)

---

## 🚀 Quick Start

### Prerequisites
- Java 17+
- Maven 3.8+
- Docker & Docker Compose (optional)
- PostgreSQL 12+ (for production)
- MongoDB 4.4+ (for production)

### 1. Clone the Repository

```bash
git clone https://github.com/yourusername/link-loom-scheduler.git
cd link-loom-scheduler
```

### 2. Build the Server

```bash
cd server
mvn clean package
```

### 3. Run with Docker Compose

```bash
docker-compose up -d
```

This starts:
- **Link Loom Server** at http://localhost:8080
- **Web UI** at http://localhost:3000
- **PostgreSQL** at localhost:5432
- **MongoDB** at localhost:27017

### 4. Create Your First Workflow

**Step 1:** Create a workflow YAML file:

```yaml
# workflow.yaml
id: daily-report
name: Daily Report
schedule: "0 3 * * *"  # 3 AM every day
tasks:
  - id: fetch_data
    type: http
    method: POST
    url: http://your-backend:8080/api/tasks/fetch-data
    
  - id: process
    type: http
    method: POST
    url: http://your-backend:8080/api/tasks/process
    dependsOn: [fetch_data]
    
  - id: send_email
    type: http
    method: POST
    url: http://your-backend:8080/api/tasks/send-email
    dependsOn: [process]
```

**Step 2:** Register the workflow (via REST API):

```bash
curl -X POST http://localhost:8080/api/workflows \
  -H "Content-Type: application/json" \
  -d @workflow.yaml
```

**Step 3:** View your workflow in Web UI:

Open http://localhost:3000 and see your workflow in action!

---

## 📚 Usage Examples

### Register Workflow Programmatically (Java)

```java
@Service
public class WorkflowSetup {
    @Autowired
    private LinkLoomClient linkLoomClient;
    
    @PostConstruct
    public void init() {
        WorkflowDto workflow = new WorkflowDto("daily-report", "Daily Report");
        workflow.setSchedule("0 3 * * *");
        
        TaskDto fetchTask = new TaskDto("fetch", "Fetch Data", TaskType.HTTP);
        fetchTask.setConfig(Map.of(
            "method", "POST",
            "url", "http://localhost:8080/api/tasks/fetch-data"
        ));
        
        TaskDto processTask = new TaskDto("process", "Process", TaskType.HTTP);
        processTask.setDependsOn(List.of("fetch"));
        processTask.setConfig(Map.of(
            "method", "POST",
            "url", "http://localhost:8080/api/tasks/process"
        ));
        
        workflow.setTasks(List.of(fetchTask, processTask));
        linkLoomClient.registerWorkflow(workflow);
    }
}
```

### Implement Task Handlers (Your Backend)

```java
@RestController
@RequestMapping("/api/tasks")
public class TaskController {
    
    @PostMapping("/fetch-data")
    public TaskResult fetchData(@RequestBody TaskExecutionRequest req) {
        // Your business logic
        List<Data> data = fetchDataFromDatabase();
        
        return new TaskResult(
            "SUCCESS",
            Map.of("records", data),
            "Fetched " + data.size() + " records"
        );
    }
    
    @PostMapping("/process")
    public TaskResult process(@RequestBody TaskExecutionRequest req) {
        Map<String, Object> input = req.getInput();
        // Process the data
        return new TaskResult("SUCCESS", processedData, "Done");
    }
}
```

### Use CLI Tool

```bash
# List all workflows
loom workflow list

# Describe a workflow
loom workflow describe daily-report

# List executions
loom execution list --workflow daily-report

# View real-time logs
loom execution logs exec-123 --follow

# Trigger a workflow manually
loom workflow trigger daily-report

# Watch mode (live updates)
loom watch
```

### Use Desktop App

```bash
# Download and run the Desktop App
# - Same features as Web UI
# - Works offline (with local cache)
# - System notifications for status changes
# - Native desktop experience
```

---

## 🏗️ Project Structure

```
link-loom-scheduler/
├── server/                          # Java Backend (Spring Boot)
│   ├── common/                      # Shared DTOs, enums, utils
│   ├── api/                         # REST API controllers
│   ├── scheduler-core/              # Scheduling logic
│   ├── workflow-engine/             # DAG engine & state machine
│   ├── executor/                    # Task execution & retry logic
│   ├── persistence/                 # Database entities & repositories
│   └── observability/               # Metrics, logs, tracing
│
├── clients/
│   ├── web-ui/                      # React dashboard
│   ├── desktop-app/                 # Electron app
│   └── cli/                         # Go/Rust CLI tool
│
├── sdk/
│   ├── java-client/                 # Maven library
│   ├── python-client/               # Pip library
│   ├── go-client/                   # Go module
│   └── js-client/                   # NPM library
│
├── docs/
│   ├── architecture.md
│   ├── api-spec.md
│   └── deployment.md
│
└── docker-compose.yml               # Local development setup
```

---

## 📖 Documentation

- **[Architecture](docs/architecture.md)** — Detailed system architecture
- **[API Specification](docs/api-spec.md)** — REST API endpoints
- **[User Guide](docs/user-guide.md)** — How to use Link Loom
- **[Deployment](docs/deployment.md)** — Production deployment guide
- **[Contributing](CONTRIBUTING.md)** — How to contribute

---

## 🛠️ Tech Stack

### Server
- **Java 17+** with Spring Boot 3.x
- **PostgreSQL** for metadata
- **MongoDB** for logs and history
- **JGraphT** for DAG algorithms
- **Micrometer** for metrics
- **Docker & Kubernetes** ready

### Web UI
- **React 18+** with TypeScript
- **React Flow** for DAG visualization
- **Tailwind CSS** for styling
- **Axios** for API communication

### Desktop App
- **Electron** for cross-platform desktop
- **React** for UI
- **SQLite** for offline caching

### CLI
- **Go** or **Rust** for performance
- **Cobra** / **Clap** for command-line interface
- Single binary executable

---

## 🔄 Typical Workflow

```
1. User Backend describes workflow in YAML
   └─> Sends to Link Loom: POST /api/workflows

2. Link Loom registers and schedules the workflow
   └─> Waits for cron time or manual trigger

3. At execution time:
   ├─> Scheduler: "It's time to run!"
   ├─> Workflow Engine: Builds DAG
   ├─> Executor: Sends HTTP callback to User Backend
   └─> User Backend: Executes task, returns result

4. Link Loom tracks progress:
   ├─> Updates execution status
   ├─> Determines next ready task
   ├─> Repeats until all tasks complete

5. User monitors via Web UI / Desktop / CLI
   └─> Sees real-time progress, logs, metrics
```

---

## 📊 Use Cases

- **ETL Pipelines** — Extract, transform, load data with complex dependencies
- **CI/CD Automation** — Orchestrate build, test, and deployment stages
- **Data Science** — ML pipeline orchestration and model training
- **Business Processes** — Order processing, approvals, integrations
- **IT Operations** — Backups, infrastructure updates, health checks
- **Scheduled Reports** — Daily, weekly, monthly reports and notifications

---

## 🔐 Security

- **API Authentication** — Token-based API authentication
- **Authorization** — Role-based access control (RBAC)
- **Encryption** — TLS/HTTPS for all communications
- **Audit Logging** — Complete execution history
- **Sandboxing** — Isolated task execution

See [Security Guide](docs/security.md) for details.

---

## 🚢 Deployment

### Local Development
```bash
docker-compose up -d
```

### Production (Kubernetes)
```bash
kubectl apply -f k8s/
# See docs/deployment.md for details
```

### Scaling
- **Horizontal Scaling** — Multiple Link Loom instances with distributed locking
- **Load Balancing** — nginx/HAProxy for API distribution
- **High Availability** — PostgreSQL replication, MongoDB sharding

---

## 📈 Roadmap

### Phase 1 (Foundation)
- [x] DAG engine with state machine
- [x] Cron scheduling
- [x] HTTP callbacks for task execution
- [x] PostgreSQL + MongoDB persistence
- [x] Basic REST API

### Phase 2 (Clients)
- [ ] Web UI (React dashboard)
- [ ] Desktop App (Electron)
- [ ] CLI Tool (Go/Rust)
- [ ] SDK libraries (Java, Python, Go, JS)

### Phase 3 (Enterprise)
- [ ] Event-based triggers (Kafka, HTTP webhooks)
- [ ] Advanced retry strategies (circuit breaker, timeout policies)
- [ ] Distributed tracing (OpenTelemetry)
- [ ] Multi-tenancy support
- [ ] Custom resource definitions (CRDs) for Kubernetes

---

## 🤝 Contributing

Contributions are welcome! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

### Development Setup
```bash
# Clone repo
git clone https://github.com/yourusername/link-loom-scheduler.git

# Build server
cd server && mvn clean install

# Build clients
cd ../clients/web-ui && npm install && npm start

# Run tests
mvn test
```

---

## 📝 License

This project is licensed under the **MIT License** — see [LICENSE](LICENSE) file for details.

---

## 💬 Support

- **Issues** — Report bugs on [GitHub Issues](https://github.com/yourusername/link-loom-scheduler/issues)
- **Discussions** — Join our [GitHub Discussions](https://github.com/yourusername/link-loom-scheduler/discussions)
- **Documentation** — See [docs/](docs/) folder
- **Email** — support@linkloom.dev (if applicable)

---

## 🙌 Acknowledgments

Link Loom is inspired by:
- [Apache Airflow](https://airflow.apache.org/) — DAG-based workflow orchestration
- [Temporal](https://temporal.io/) — Distributed workflow engine
- [Argo Workflows](https://argoproj.github.io/argo-workflows/) — Kubernetes-native workflows
- [Jenkins](https://www.jenkins.io/) — CI/CD automation

---

## 📞 Get Started

1. ⭐ **Star this repository** if you find it useful
2. 📖 **Read the [Quick Start](#-quick-start)** guide
3. 🔗 **Check out [Examples](examples/)** folder
4. 🐛 **Report issues** or suggest improvements
5. 🤝 **Contribute** code or documentation

---

<div align="center">

**[Website](https://linkloom.dev)** • **[Documentation](docs/)** • **[Discord Community](#)** • **[Twitter](#)**

Made with ❤️ by the Link Loom Team

</div>