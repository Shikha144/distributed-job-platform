# Distributed Job Platform

**Distributed Job Scheduling & Task Execution Platform**

A backend platform for submitting, scheduling, processing, and monitoring background jobs through a REST API. It includes priority-based scheduling, asynchronous worker execution, retry handling, job-state management, authentication, rate limiting, event auditing, and monitoring.

**Author:** Shikha Sharma

---

## Overview

Distributed Job Platform is a Spring Boot application that allows authenticated users to create and manage background jobs. Jobs are stored in PostgreSQL, scheduled for execution, and processed asynchronously by a bounded worker pool.

The platform supports job priorities, scheduled execution, retry handling with exponential backoff, idempotent job creation, role-based access control, and administrative recovery of jobs that reach the DEAD state.

Kafka is used to publish job lifecycle events for auditing. Redis supports API rate limiting, while Prometheus and Grafana provide application monitoring and visualization.

The backend is implemented as a modular monolith, with a React frontend and a Docker Compose environment for running the supporting services.

## Key Features

- **Job scheduling:** Create jobs for immediate or future execution.
- **Priority-based dispatch:** Jobs are selected according to priority and scheduled time.
- **Asynchronous execution:** A bounded worker pool processes jobs in the background.
- **Job lifecycle management:** Tracks jobs through nine defined states.
- **Automatic retries:** Failed jobs can be retried using exponential backoff.
- **Dead-letter handling:** Jobs that exhaust their retries move to the DEAD state.
- **Administrative recovery:** Administrators can recover eligible DEAD jobs.
- **Idempotent job creation:** Supports idempotency keys to prevent duplicate job creation from repeated requests.
- **Concurrency protection:** Uses atomic database updates to prevent multiple workers from claiming the same queued job at the same time.
- **Authentication and authorization:** JWT-based authentication with USER and ADMIN roles.
- **API rate limiting:** Redis-based sliding-window rate limiting.
- **Lifecycle event auditing:** Kafka events are consumed and persisted in an audit table.
- **Monitoring:** Prometheus metrics and a provisioned Grafana dashboard.

## Technology Stack

| Component | Technology |
|---|---|
| Backend | Java 17, Spring Boot 3.2.5 |
| API | Spring Web MVC, REST |
| Security | Spring Security, JWT, BCrypt |
| Database | PostgreSQL 16 |
| Cache and rate limiting | Redis 7 |
| Messaging and event auditing | Apache Kafka 3.6 |
| Frontend | React 18, Vite 5 |
| Styling | Tailwind CSS 3.4 |
| Monitoring | Prometheus, Grafana |
| Build tool | Maven |
| Containerization | Docker, Docker Compose |
| Testing | JUnit 5, Mockito |

## Architecture

The application uses a modular monolith architecture. The Spring Boot backend contains the scheduling, job management, worker execution, security, and event-processing modules.

```text
                    React Frontend
                         |
                         | REST API
                         v
                 Spring Boot Backend
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
      PostgreSQL       Redis           Kafka
      Job storage      Rate limits     Lifecycle events
          |
          v
      Job Scheduler
          |
          v
    Priority Dispatch
          |
          v
     Worker Pool
          |
          v
     Job Handlers
          |
          v
     Job Results
          
          Monitoring:
          Prometheus -> Grafana
```

### Main Components

- **Spring Boot Backend:** Exposes REST APIs and coordinates job management, scheduling, execution, and security.
- **PostgreSQL:** Stores jobs, job attempts, results, idempotency records, users, and audit events.
- **Redis:** Supports per-user API rate limiting.
- **Kafka:** Propagates job lifecycle events to the audit consumer.
- **Worker Pool:** Executes jobs asynchronously using a bounded thread pool.
- **React Frontend:** Provides the client-side application.
- **Prometheus and Grafana:** Collect and visualize application metrics.

**Architecture note:** PostgreSQL is the source of truth for job state. Kafka is used for lifecycle event propagation and auditing; it is not the task queue.

## Job Lifecycle

The platform uses a nine-state job lifecycle:

| State | Description |
|---|---|
| CREATED | Job has been created. |
| SCHEDULED | Job is waiting for its scheduled execution time. |
| QUEUED | Job is ready for dispatch. |
| RUNNING | A worker has claimed the job and started execution. |
| SUCCESS | Job completed successfully. |
| FAILED | Job execution failed. |
| RETRYING | Job is waiting for another execution attempt. |
| DEAD | Job has exhausted its retry attempts. |
| CANCELLED | Job has been cancelled. |

The state machine validates allowed transitions and prevents invalid lifecycle changes.

## Job Execution and Reliability

### Priority-based scheduling

Jobs are dispatched according to their priority and scheduled time. The configured priority levels are:

- HIGH
- MEDIUM
- LOW

### Worker execution

A bounded `ThreadPoolExecutor` processes jobs asynchronously. The configured worker pool has five threads and an in-memory queue capacity of 100.

Workers claim jobs through an atomic conditional database update. Only one worker can successfully change a job from QUEUED to RUNNING for a given claim operation.

### Retry mechanism

When an execution fails, the platform can retry the job using exponential backoff:

`min(1000 × 2^retryCount, 30000)` milliseconds.

Jobs that exhaust their configured retry attempts can enter the DEAD state. Administrators can use the recovery endpoint to return eligible DEAD jobs to the queue.

### Execution semantics

The platform provides **at-least-once execution semantics**, not exactly-once execution. The in-memory dispatch queue is not durable; PostgreSQL retains the job state and is used to rediscover queued work.

## Security

The application includes:

- JWT authentication using HS256.
- BCrypt password hashing.
- USER and ADMIN roles.
- Stateless authentication.
- Owner-scoped job access.
- Administrative endpoint protection.
- Registration that creates USER accounts rather than allowing users to assign themselves ADMIN privileges.

Requests for jobs that do not belong to the authenticated user are handled without exposing another user's job records.

## API Reference

### Authentication

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/auth/register` | Register a user |
| POST | `/api/auth/login` | Authenticate and obtain a JWT |
| GET | `/api/auth/me` | Get the authenticated user's information |

### Job Management

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/jobs` | Create a job |
| GET | `/api/jobs` | List the authenticated user's jobs |
| GET | `/api/jobs/{id}` | Get job details |
| POST | `/api/jobs/{id}/cancel` | Cancel a job |
| POST | `/api/jobs/{id}/retry` | Retry a job |
| GET | `/api/jobs/{id}/attempts` | Get job execution attempts |
| GET | `/api/jobs/{id}/result` | Get the job result |

### Administrative APIs

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/jobs/{id}/recover` | Recover an eligible DEAD job |
| GET | `/api/jobs/admin/jobs` | List jobs through the administrative API |

Administrative endpoints require the ADMIN role.

## Monitoring and Observability

The application uses Micrometer and Prometheus to expose metrics through:

`/actuator/prometheus`

Prometheus scrapes application metrics, and Grafana provides a provisioned dashboard named **Job Platform Overview** with ten panels.

Default service ports:

| Service | Port |
|---|---:|
| Spring Boot Backend | 8080 |
| React Vite Development Server | 5173 |
| Prometheus | 9090 |
| Grafana | 3001 |

## Getting Started

### Prerequisites

Install the following tools:

- Java 17
- Maven
- Node.js and npm
- Docker
- Docker Compose

### 1. Clone the repository

```bash
git clone https://github.com/Shikha144/distributed-job-platform.git
cd distributed-job-platform
```

### 2. Build the backend

```bash
mvn clean package -DskipTests
```

The Dockerfile expects the packaged JAR to exist before the application image is built.

### 3. Start the application stack

```bash
docker compose up -d --build
```

This starts the application and its supporting services, including PostgreSQL, Redis, Kafka, Prometheus, and Grafana.

### 4. Start the frontend locally

Open another terminal:

```bash
cd frontend
npm install
npm run dev
```

The Vite development server runs on port `5173`.

### 5. Access the services

- Backend API: `http://localhost:8080`
- Frontend: `http://localhost:5173`
- Prometheus: `http://localhost:9090`
- Grafana: `http://localhost:3001`

Use the credentials configured for your local development environment. Change default development credentials and secrets before using the application outside a local demo environment.

## Testing

Run the backend unit and integration tests:

```bash
mvn test
```

Run the end-to-end verification script against a running Docker environment:

```bash
python e2e_verify.py
```

The attached technical report records 128 automated test methods passing across eight test classes. It also documents 15 runtime verification scenarios. These are results recorded in the project material, not a new test run performed as part of this README update.

## Database

The platform uses PostgreSQL to persist:

- `jobs`
- `job_attempts`
- `job_results`
- `idempotency_records`
- `job_audit_events`
- `users`

PostgreSQL is the authoritative store for job state and execution history.

## Known Limitations

The following limitations are documented in the project implementation:

- The backend is a modular monolith rather than a multi-instance distributed deployment.
- The scheduler has no distributed leader election or cluster-wide lock.
- The dispatch queue is in memory and is not independently durable.
- Execution is at-least-once, not exactly-once.
- Job handlers are simulated implementations rather than integrations with external services.
- Timeout handling uses cooperative interruption and does not guarantee forced termination.
- A process crash can leave a job in RUNNING without an automatic stale-job recovery sweeper.
- Kafka and Redis infrastructure are configured for the project's demo environment and do not include production-grade transport security.
- The scheduler interval configuration has a unit mismatch: the configured property is named in seconds, but `fixedDelayString` interprets the supplied value as milliseconds. The default value of `5` therefore results in a 5 ms delay, not 5 seconds.

These limitations should be considered before adapting the project for production use.

## Future Improvements

Potential areas for further development include:

- Distributed scheduler coordination.
- Durable dispatch and recovery mechanisms.
- Stale RUNNING job detection and recovery.
- Production-grade infrastructure security.
- Real integrations for job handlers.
- Stronger timeout and cancellation guarantees.
- Additional deployment and scalability options.

## Author

**Shikha Sharma**

GitHub: [Shikha144](https://github.com/Shikha144)

**Repository:** [Distributed Job Platform](https://github.com/Shikha144/distributed-job-platform)

---

