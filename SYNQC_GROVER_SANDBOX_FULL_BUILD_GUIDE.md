# SynQc TDS — Grover’s Agent Sandbox Build (End‑to‑End Guide)

## Purpose of This Document

This document is a **complete, ground‑truth record** of the Grover’s Agent sandbox build for the SynQc Temporal Dynamics Series (TDS) hybrid controller.

It is intentionally **long, explicit, and narrative**, because its goal is *repeatability*:
- Future you (or another engineer) should be able to rebuild this sandbox **from scratch**
- No tribal knowledge
- No “it worked on my machine” steps
- No hand‑waving

This is **not** just a Prometheus guide.  
It covers the **entire sandbox**:
- API
- Worker
- Redis
- Grover agent execution
- Queue plumbing
- Metrics
- Prometheus
- Grafana
- Debug failures
- Rules learned the hard way

---

## High‑Level Architecture

**SynQc TDS Sandbox Components**

| Component | Role |
|---------|------|
| API | Receives run requests, validates, persists jobs |
| Redis | Single source of truth for jobs, queues, metrics state |
| Worker | Executes jobs (Grover + others) |
| Grover Agent | Example non‑toy agent proving real execution |
| Prometheus | Metrics collection |
| Grafana | Visualization |
| Web UI | Human interaction layer |

All components run **locally** via Docker Compose.

---

## Design Rules (Established Early — Do Not Violate)

1. **No fake simulators**
2. **No mocked agents**
3. **Real queue, real Redis**
4. **Worker must pull jobs from Redis**
5. **Metrics must reflect reality, not health checks**
6. **Everything must be debuggable from CLI**
7. **No silent Docker overrides**
8. **VS Code must be told EXACTLY which file to edit**
9. **If it’s not reproducible, it’s broken**

---

## Repository Layout (Relevant Parts)

```
synqc-temporal-dynamics-series-hybrid-controller/
├── backend/
│   ├── synqc_backend/
│   │   ├── api.py
│   │   ├── worker.py
│   │   ├── worker_service.py
│   │   ├── run_queue.py
│   │   ├── metrics.py
│   │   ├── redis_client.py
│   │   └── ...
│   ├── Dockerfile
│   └── pyproject.toml
├── web/
├── docker/
│   └── prometheus.yml
├── docker-compose.yml
└── docs/
```

---

## Phase 1 — Initial Sandbox Bring‑Up

### Goal
Bring up all services and confirm they **exist**, not that they work.

### Command
```bash
docker compose up -d --build
```

### Initial Verification
```bash
docker compose ps
curl http://127.0.0.1:8001/health
```

At this stage:
- API was up
- Redis was healthy
- Worker was running
- **Jobs did NOT execute**

This was expected.

---

## Phase 2 — Grover Agent Execution Failure

### Symptom
- `/runs` returned `202 queued`
- Job never progressed
- Redis queue empty
- Worker logs silent

### Root Cause
**Two queue systems existed**:
- Legacy queue (`synqc:q:*`)
- Actual run queue (`synqc:runq:*`)

Grover jobs were going into:
```
synqc:runq:pending
```
Worker was initially polling the wrong queue.

### Diagnosis Commands
```bash
docker compose exec redis redis-cli KEYS 'synqc:*'
docker compose exec redis redis-cli LRANGE synqc:runq:pending 0 10
```

### Fix
Ensure Worker uses **RedisRunQueue**, not legacy queueing helpers.

Once aligned:
```bash
worker logs → “Run succeeded”
```

This was the **first major breakthrough**.

---

## Phase 3 — Redis Metrics Lying

### Symptom
Prometheus showed:
```
synqc_redis_connected = 0
```
but:
```bash
curl /health
```
showed Redis OK.

### Root Cause
Metric logic assumed:
```python
summary["redis_connected"]
```
but actual summary used:
- `redis_ok`
- `ok`
- `connected`

### Fix (metrics.py)
Logic rewritten to probe multiple keys safely.

This made metrics **truthful**.

---

## Phase 4 — Prometheus Could Not Scrape Containers

### Symptom
- Prometheus target DOWN
- Connection refused

### Root Cause
Metrics servers bound to `127.0.0.1` **inside containers**.

### Fix
Bind to all interfaces:
```
0.0.0.0
```

Environment variables:
```
SYNQC_METRICS_BIND_ADDRESS=0.0.0.0
SYNQC_METRICS_WORKER_BIND_ADDRESS=0.0.0.0
```

---

## Phase 5 — Worker Metrics Were Missing

### Symptom
- API metrics OK
- Worker metrics unreachable

### Root Cause
Worker metrics server **never started**.

### Fix
Worker needed explicit startup of Prometheus server:
```python
start_http_server(port, addr)
```

Also:
- Missing `import os` caused infinite restart loop

This was a **hard‑won fix**.

---

## Phase 6 — Docker Override Hallucination

### Critical Lesson

`docker-compose.override.yml` is **automatically merged** by Docker if present.

Even if not tracked in Git.

### Fix
- Delete override files
- Add to `.gitignore`

This single issue caused hours of confusion.

---

## Phase 7 — Grafana Integration

### Local Grafana
- Runs on `http://127.0.0.1:3000`
- Docker container: `grafana/grafana-oss`

### Prometheus Data Source
```
http://prometheus:9090
```

### Result
- Live metrics
- Queue depth
- Run success counts
- Redis connectivity
- Worker health

Grafana now reflects **actual system behavior**.

---

## Phase 8 — Final Validation (Non‑Negotiable)

### Submit Grover Run
```bash
curl -X POST http://127.0.0.1:8001/runs   -H "Content-Type: application/json"   -d '{
    "preset": "hello_quantum_sim",
    "hardware_target": "sim_local",
    "shot_budget": 64
  }'
```

### Confirm
- Redis queue increments
- Worker logs execution
- Run transitions to `succeeded`
- Metrics increment
- Grafana updates

This **proves the sandbox is real**.

---

## Tooling Used

- Docker Desktop
- Docker Compose
- Redis
- Prometheus
- Grafana
- Python 3.12
- VS Code
- Git Bash (Windows)
- curl
- redis-cli

---

## Final Outcome

✅ Grover agent executes  
✅ Jobs flow through Redis correctly  
✅ Worker consumes run queue  
✅ Metrics reflect reality  
✅ Prometheus scrapes API + Worker  
✅ Grafana visualizes live state  
✅ System is reproducible  

This sandbox is now a **gold‑standard reference build**.

---

## Recommendation Going Forward

- Freeze this sandbox as a **known‑good baseline**
- Build MVP UI on top of this
- Never “simplify” queue or metrics paths
- Always validate against Redis, not assumptions

---

**End of Document**
