# Getting Started with Link Loom

Welcome to Link Loom! This guide will help you understand the basics and get started quickly.

---

## 📚 Understanding Link Loom

### What is Link Loom?

Link Loom is a **centralized workflow orchestration service** that helps you:

1. **Schedule** tasks automatically (using cron expressions)
2. **Manage** task dependencies (DAG — Directed Acyclic Graph)
3. **Execute** tasks reliably across different systems
4. **Monitor** everything from a single dashboard

### Why Use Link Loom?

| Scenario | Link Loom |
|----------|-----------|
| Run daily reports automatically | ✅ Yes |
| Orchestrate multi-step data pipelines | ✅ Yes |
| Manage CI/CD workflows | ✅ Yes |
| Coordinate microservices | ✅ Yes |
| Handle task retries automatically | ✅ Yes |
| Monitor all executions in one place | ✅ Yes |

### Quick Analogy

Think of Link Loom as a **smart scheduler for your tasks**:

- **Without Link Loom**: You use cron, bash scripts, ad-hoc monitoring
- **With Link Loom**: You define workflows in YAML, let Link Loom handle scheduling, retries, and monitoring

---

## 🏗️ Architecture Overview

### Three Parts

```
1. Link Loom Server (the orchestrator)
   └─> Decides WHEN to run and WHAT order

2. Your Backend Services (the workers)
   └─> Actually EXECUTE the tasks

3. User Interfaces (the dashboards)
   └─> You MONITOR what's happening
```

### How It Works

```
┌─────────────────────────────────┐
│    Your application             │
│  (Spring Boot, Flask, etc.)     │
│                                 │
│  @PostMapping("/api/tasks/..") │
│  public TaskResult fetch() {    │
│    // Your code here            │
│  }                              │
└──────────────┬──────────────────┘
               │ ▲
               │ │ HTTP callback
        POST / │ \ Response
        /tasks │ 
               │
    ┌──────────▼──────────────┐
    │ Link Loom Server        │
    │                         │
    │ - Scheduler (cron)      │
    │ - DAG Engine            │
    │ - Executor              │
    │ - REST API              │
    └─────────────┬───────────┘
                  │
        ┌─────────┼─────────┐
        │         │         │
        ▼         ▼         ▼
    ┌─────────┐ ┌───────┐ ┌────┐
    │ Web UI  │ │Desktop│ │ CLI│
    │(React)  │ │(Elec.)│ │    │
    └─────────┘ └───────┘ └────┘
```

---

## 🚀 Getting Started in 5 Minutes

### Step 1: Start Link Loom

```bash
# Clone the repository
git clone https://github.com/yourusername/link-loom-scheduler.git
cd link-loom-scheduler

# Start with Docker Compose
docker-compose up -d

# Wait for services to be ready
docker-compose logs -f
```

**Services ready when you see:**
- Link Loom Server: http://localhost:8080
- Web UI: http://localhost:3000
- Database: localhost:5432, localhost:27017

### Step 2: Create Your First Workflow

Create a file `workflow.yaml`:

```yaml
id: hello-world
name: Hello World Workflow
schedule: "*/5 * * * *"  # Every 5 minutes
tasks:
  - id: greet
    type: http
    method: POST
    url: http://localhost:8888/api/tasks/greet
```

### Step 3: Register the Workflow

```bash
curl -X POST http://localhost:8080/api/workflows \
  -H "Content-Type: application/yaml" \
  -d @workflow.yaml
```

### Step 4: Create a Simple Backend Service

Create `your-backend.java`:

```java
@RestController
@RequestMapping("/api/tasks")
public class TaskController {
    
    @PostMapping("/greet")
    public Map<String, Object> greet(@RequestBody Map<String, Object> request) {
        System.out.println("Hello from Link Loom!");
        
        return Map.of(
            "status", "SUCCESS",
            "message", "Greeting executed at " + System.currentTimeMillis()
        );
    }
}
```

Run your backend on port 8888:
```bash
java -jar your-backend.jar --server.port=8888
```

### Step 5: Watch It Execute

Open http://localhost:3000 and see your workflow execute every 5 minutes!

---

## 📖 Core Concepts

### Workflow

A **workflow** is a collection of tasks with dependencies.

```yaml
id: data-pipeline
name: Data Processing Pipeline
schedule: "0 2 * * *"  # Daily at 2 AM
tasks:
  # ... tasks go here
```

**Key properties:**
- `id` — Unique identifier
- `name` — Human-readable name
- `schedule` — Cron expression (when to run)
- `tasks` — List of tasks

### Task

A **task** is a unit of work.

```yaml
- id: fetch_data
  type: http
  method: POST
  url: http://your-backend:8080/api/tasks/fetch
  timeout: 5m
  maxAttempts: 3
  dependsOn: []  # No dependencies
```

**Key properties:**
- `id` — Unique identifier within workflow
- `type` — http, shell, java
- `dependsOn` — Which tasks must complete first
- `timeout` — How long before giving up
- `maxAttempts` — How many times to retry

### Execution

An **execution** is one run of a workflow.

```
Workflow "daily-report" defined once
Execution 1: 2024-01-14 at 03:00 ✅ SUCCESS
Execution 2: 2024-01-15 at 03:00 ✅ SUCCESS
Execution 3: 2024-01-16 at 03:00 ❌ FAILED (retry...)
```

### DAG (Directed Acyclic Graph)

A **DAG** shows task dependencies visually:

```
fetch_data ──┐
             ├─► process_data ──┐
validate ────┘                  ├─► send_report
                                │
             retry_email ────────┘
```

**Rules:**
- Tasks can depend on other tasks
- No circular dependencies allowed
- Can execute independent tasks in parallel

---

## 🔄 Common Workflows

### Example 1: ETL Pipeline

```yaml
id: daily-etl
name: Daily ETL Pipeline
schedule: "0 2 * * *"
tasks:
  - id: extract
    type: http
    method: POST
    url: http://backend:8080/api/etl/extract
    
  - id: transform
    type: http
    method: POST
    url: http://backend:8080/api/etl/transform
    dependsOn: [extract]
    
  - id: load
    type: http
    method: POST
    url: http://backend:8080/api/etl/load
    dependsOn: [transform]
```

### Example 2: Parallel Tasks

```yaml
id: parallel-work
name: Parallel Processing
tasks:
  - id: task_a
    type: http
    method: POST
    url: http://backend:8080/api/tasks/a
    
  - id: task_b
    type: http
    method: POST
    url: http://backend:8080/api/tasks/b
    
  - id: task_c
    type: http
    method: POST
    url: http://backend:8080/api/tasks/c
    dependsOn: [task_a, task_b]  # Wait for both
```

Task A and B run in parallel, then C runs after both complete.

### Example 3: Conditional Retries

```yaml
id: resilient-workflow
name: Workflow with Retries
tasks:
  - id: fetch
    type: http
    method: POST
    url: http://backend:8080/api/tasks/fetch
    maxAttempts: 3
    retryStrategy: exponential  # 1s, 2s, 4s delays
    timeout: 30s
```

---

## 🎮 Using the Web UI

### Dashboard

The **Dashboard** shows:
- All workflows
- Status (enabled/disabled)
- Next scheduled time
- Last execution result

### Workflow Details

Click a workflow to see:
- **DAG Graph** — Visual dependencies
- **Execution History** — Past runs
- **Properties** — YAML definition
- **Trigger** button — Run manually

### Execution Details

Click an execution to see:
- **Timeline** — Which tasks ran when
- **Task Results** — Success/failure for each
- **Logs** — Output from each task
- **Retry Info** — How many retries

---

## 💻 Using the CLI

```bash
# List all workflows
loom workflow list

# Get details of a workflow
loom workflow describe daily-report

# Trigger a workflow manually
loom workflow trigger daily-report

# List executions
loom execution list --workflow daily-report

# View logs (real-time)
loom execution logs exec-123 --follow

# Watch all workflows (live updates)
loom watch
```

---

## 🔧 Implementing Task Handlers

Your backend services need **task handlers** — endpoints that Link Loom calls.

### Java (Spring Boot)

```java
@RestController
@RequestMapping("/api/tasks")
public class MyTaskHandler {
    
    @PostMapping("/fetch-data")
    public TaskResult fetchData(@RequestBody TaskExecutionRequest request) {
        // Get input from Link Loom
        Map<String, Object> input = request.getInput();
        String executionId = request.getExecutionId();
        
        // Do your work
        List<String> data = fetchFromDatabase();
        
        // Return result
        return new TaskResult(
            "SUCCESS",  // status
            Map.of("records", data),  // output
            "Fetched " + data.size() + " records"  // logs
        );
    }
    
    @PostMapping("/process-data")
    public TaskResult processData(@RequestBody TaskExecutionRequest request) {
        try {
            Map<String, Object> input = request.getInput();
            // Process the data from fetch_data
            processAndStore(input);
            
            return new TaskResult("SUCCESS", Map.of(), "Processing complete");
        } catch (Exception e) {
            // Link Loom will retry automatically
            return new TaskResult("FAILED", Map.of(), e.getMessage());
        }
    }
}
```

### Python (Flask)

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/api/tasks/fetch-data', methods=['POST'])
def fetch_data():
    data = request.json
    execution_id = data['executionId']
    
    # Your logic
    records = fetch_from_db()
    
    return jsonify({
        'status': 'SUCCESS',
        'output': {'records': records},
        'logs': f'Fetched {len(records)} records'
    })

if __name__ == '__main__':
    app.run(port=8080)
```

---

## 🚨 Troubleshooting

### Workflow Not Executing

1. **Check if workflow is registered**
   ```bash
   loom workflow list
   loom workflow describe your-workflow-id
   ```

2. **Check if schedule is correct**
   ```bash
   # Cron syntax: minute hour day month weekday
   "0 3 * * *"  # 3 AM every day ✅
   "*/5 * * * *"  # Every 5 minutes ✅
   ```

3. **Check backend connectivity**
   ```bash
   curl -X POST http://your-backend:8080/api/tasks/your-task \
     -H "Content-Type: application/json" \
     -d '{"executionId":"test"}'
   ```

### Task Failing

1. **Check execution logs**
   ```bash
   loom execution logs exec-123 --follow
   ```

2. **Check task handler endpoint**
   ```bash
   # Is it responding correctly?
   curl http://your-backend:8080/api/tasks/your-task
   ```

3. **Check timeout settings**
   ```yaml
   tasks:
     - id: my_task
       timeout: 5m  # Increase if needed
   ```

### Performance Issues

1. **Check execution timeline**
    - Visit Web UI → Execution Details → Timeline
    - Identify bottleneck tasks

2. **Optimize task handlers**
    - Profile your endpoints
    - Optimize database queries
    - Add parallel tasks where possible

---

## 📚 Next Steps

1. **Read the Documentation**
    - [Architecture Guide](docs/architecture.md)
    - [API Reference](docs/api-spec.md)
    - [Deployment Guide](docs/deployment.md)

2. **Explore Examples**
    - [ETL Pipeline Example](examples/etl-pipeline/)
    - [CI/CD Workflow Example](examples/cicd-workflow/)
    - [Data Science Pipeline](examples/ml-pipeline/)

3. **Join the Community**
    - [GitHub Discussions](https://github.com/yourusername/link-loom-scheduler/discussions)
    - [Discord Community](#)
    - [Twitter](#)

4. **Contribute**
    - See [CONTRIBUTING.md](CONTRIBUTING.md)
    - Report bugs
    - Submit feature requests

---

## 🆘 Getting Help

### Resources

- **Documentation**: https://linkloom.dev/docs
- **GitHub Issues**: https://github.com/yourusername/link-loom-scheduler/issues
- **Email**: support@linkloom.dev

### Common Questions

**Q: Can I use Link Loom with Docker?**  
A: Yes! Use `docker-compose up` or see [Deployment Guide](docs/deployment.md).

**Q: What happens if my backend service crashes?**  
A: Link Loom will retry the task. Configure `maxAttempts` and `retryStrategy`.

**Q: Can multiple workflows run in parallel?**  
A: Yes! Link Loom can execute independent workflows concurrently.

**Q: How do I deploy Link Loom to production?**  
A: See [Deployment Guide](docs/deployment.md) for Kubernetes setup.

---

## 🎉 You're Ready!

You now understand Link Loom basics. Start by:

1. Running `docker-compose up`
2. Creating your first workflow
3. Implementing task handlers
4. Monitoring in Web UI

Happy orchestrating! 🚀

---

<div align="center">

**[Back to README](README.md)** • **[Documentation](docs/)** • **[Contributing](CONTRIBUTING.md)**

</div>