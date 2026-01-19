# TaskFlow - Distributed Job Scheduling Platform

TaskFlow is a fault-tolerant, distributed job scheduling platform designed to reliably execute large volumes of background jobs under concurrency and failure conditions.

Unlike in-memory schedulers or cron-based systems, TaskFlow is designed to operate safely under concurrency, partial failures, and restarts, mirroring real production constraints found in large-scale backend systems.

---

## Key Features

- Durable Job State  
  All job metadata and execution state are persisted in PostgreSQL. Jobs survive process crashes, container restarts, and worker failures.

- Distributed Worker Coordination  
  Multiple stateless worker nodes can run concurrently. Redis-based leasing ensures only one worker executes a job at a time, preventing race conditions.

- Fault Tolerance and Retries  
  Failed jobs are retried using exponential backoff. Transient failures do not result in data loss or duplicate execution.

- Horizontal Scalability  
  Workers are fully stateless and can be scaled horizontally. Throughput increases linearly with additional worker instances.

- Low-Latency Execution Path  
  In-memory coordination via Redis minimizes database contention. PostgreSQL access patterns are restricted to predictable point reads and writes.

---

## System Guarantees

TaskFlow intentionally provides the following execution guarantees:

- **At-least-once execution**  
  A job may be retried after failures, but it will never be lost.

- **Single-worker execution per job**  
  Redis-based leasing ensures that no two workers execute the same job concurrently.

- **Durable state transitions**  
  All job state changes are persisted in PostgreSQL and survive process crashes and restarts.

- **Deterministic retry behavior**  
  Failed jobs follow a predictable exponential backoff policy with bounded retries.

---

## Operational Model

TaskFlow follows a clear separation between **control plane** and **execution plane** responsibilities.

- The **TaskFlow API** acts as the control plane:
  - Accepts job submissions
  - Validates inputs
  - Persists durable job metadata
  - Exposes job lifecycle and status APIs

- The **TaskFlow Workers** act as the execution plane:
  - Poll for executable jobs
  - Compete for distributed leases
  - Execute jobs in isolation
  - Report results back to durable storage

This separation ensures that API availability is not coupled to job execution throughput, enabling independent scaling and failure isolation.

---
```
## High-Level Architecture

Client
  |
  v
TaskFlow API
  |
  +-- PostgreSQL (Durable job state)
  |
  +-- Redis (Leases / locks)
          |
          v
   TaskFlow Workers (N instances)
```
---

## Tech Stack

- Language: Java 17 (LTS)
- Framework: Spring Boot
- Database: PostgreSQL
- Distributed Coordination: Redis
- Build Tool: Maven
- Containerization: Docker and Docker Compose
- IDE: IntelliJ IDEA

---

## Repository Structure

```
taskflow/
├── README.md
├── docker-compose.yml
├── Makefile
├── .gitignore
│
├── docs/
│   ├── architecture.md
│   ├── data-model.md
│   ├── execution-flow.md
│   ├── failure-semantics.md
│   └── demo.md
│
├── infra/
│   ├── postgres/
│   │   └── init/
│   │       └── V1__init_schema.sql
│   └── redis/
│
├── services/
│   ├── taskflow-api/
│   │   ├── Dockerfile
│   │   ├── pom.xml
│   │   └── src/
│   │       ├── main/
│   │       │   ├── java/com/taskflow/api/
│   │       │   │   ├── TaskflowApiApplication.java
│   │       │   │   ├── controller/
│   │       │   │   │   └── JobController.java
│   │       │   │   ├── service/
│   │       │   │   │   └── JobService.java
│   │       │   │   ├── scheduler/
│   │       │   │   │   └── JobDispatcher.java
│   │       │   │   ├── repository/
│   │       │   │   │   └── JobRepository.java
│   │       │   │   ├── model/
│   │       │   │   │   └── JobEntity.java
│   │       │   │   └── config/
│   │       │   │       ├── DataSourceConfig.java
│   │       │   │       └── RedisConfig.java
│   │       │   └── resources/
│   │       │       ├── application.yml
│   │       │       └── db/migration/
│   │       │           └── V1__init_schema.sql
│   │       └── test/
│   │           └── java/
│
│   ├── taskflow-worker/
│   │   ├── Dockerfile
│   │   ├── pom.xml
│   │   └── src/
│   │       ├── main/
│   │       │   ├── java/com/taskflow/worker/
│   │       │   │   ├── TaskflowWorkerApplication.java
│   │       │   │   ├── poller/
│   │       │   │   │   └── JobPoller.java
│   │       │   │   ├── executor/
│   │       │   │   │   └── JobExecutor.java
│   │       │   │   ├── lease/
│   │       │   │   │   └── RedisLeaseManager.java
│   │       │   │   ├── retry/
│   │       │   │   │   └── BackoffCalculator.java
│   │       │   │   ├── repository/
│   │       │   │   │   └── JobRepository.java
│   │       │   │   └── config/
│   │       │   │       ├── RedisConfig.java
│   │       │   │       └── WorkerConfig.java
│   │       │   └── resources/
│   │       │       └── application.yml
│   │       └── test/
│   │           └── java/
│
├── shared/
│   └── taskflow-common/
│       ├── pom.xml
│       └── src/main/java/com/taskflow/common/
│           ├── dto/
│           │   └── JobRequest.java
│           ├── enums/
│           │   └── JobStatus.java
│           └── util/
│               └── TimeUtils.java
│
└── scripts/
    ├── submit-job.sh
    ├── stress-test.sh
    └── cleanup.sh
```

---

## Core Components

TaskFlow API  
Accepts job submissions via REST endpoints, persists job metadata to PostgreSQL, dispatches jobs for execution, and exposes job status and lifecycle information.

TaskFlow Worker  
Polls for pending jobs, acquires distributed leases using Redis, executes jobs deterministically, and handles retries with exponential backoff.

PostgreSQL  
Acts as the durable source of truth for all job state and execution metadata.

Redis  
Provides lightweight, fast coordination primitives to ensure safe execution under concurrency.

---

## Job Lifecycle

1. Job is submitted to the TaskFlow API.
2. Job metadata is persisted in PostgreSQL.
3. Worker polls for available jobs.
4. Worker acquires a Redis lease for the job.
5. Job is executed.
6. Job status is updated to SUCCESS, FAILED, or RETRY.
7. Failed jobs are retried with exponential backoff.

---

## Failure Scenarios Handled

TaskFlow is designed to behave safely under common real-world failure conditions:

- **Worker crash during execution**  
  Redis leases expire automatically, allowing another worker to safely retry the job.

- **API process restart**  
  All job state is recovered from PostgreSQL with no in-memory dependency.

- **Duplicate execution attempts**  
  Lease-based coordination ensures only one active executor per job at any time.

- **Transient database or network failures**  
  Jobs remain in a retriable state and follow deterministic backoff policies.

- **Redis restart**  
  No durable state is lost; leases are re-established safely from PostgreSQL state.

---

## Running Locally

### Prerequisites
- Java 17
- Docker Desktop
- Maven

### Start the system
```bash
docker compose up --build
```

Submit a job:
./scripts/submit-job.sh

Run stress test:
./scripts/stress-test.sh

Cleanup:
./scripts/cleanup.sh

---

## Design Principles

- Correctness over cleverness
- Deterministic execution
- Idempotent operations
- Clear separation of responsibilities
- Production realism over toy abstractions

---

## Design Tradeoffs

- The system prioritizes **correctness and durability** over ultra-low-latency execution.
- At-least-once execution is chosen over exactly-once semantics to avoid distributed transaction complexity.
- Redis is used only for coordination, not as a source of truth, to keep failure recovery simple and predictable.

---

## Non-Goals

This project intentionally does NOT attempt to solve the following problems:

- Exactly-once execution semantics
- Distributed transactions across services
- Workflow orchestration or DAG scheduling
- Long-running streaming workloads
- Cron-based or time-triggered scheduling

These constraints keep the system focused on correctness, observability, and safe concurrent execution under failure.

---

## Why This Project Stands Out

This project demonstrates real distributed systems thinking, safe concurrency handling, practical fault-tolerance mechanisms, and clean scalable backend design aligned with FAANG-level SWE expectations.

The system is intentionally scoped to be deep rather than broad, focusing on the core problems that matter in real production systems.

---

## Future Extensions

Potential extensions that fit naturally into the existing architecture:

- Job prioritization and queue partitioning
- Rate-limited worker pools
- Pluggable execution backends
- Metrics and tracing integration (Prometheus / OpenTelemetry)
- Dead-letter queues for permanently failed jobs

---

## Author

Vidhi Babariya  
Software Engineer - Backend and Distributed Systems


