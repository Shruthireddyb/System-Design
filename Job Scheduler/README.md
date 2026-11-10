# Distributed Job Scheduler 

> Scalable Job Scheduler like Quartz Cluster, AWS EventBridge, Sidekiq, Cron - Handles 100M jobs, 50k jobs/sec, Exactly Once Execution

## 1. Overview

A system to schedule jobs to run at a specific time in future. 
Examples: 
- Swiggy: "Send reminder if payment not done in 10 mins"
- Flipkart: "Cancel order if not paid in 15 mins"
- Notification: "Send birthday email at 9 AM"
- Payment: "Retry failed payment after 5, 10, 30 mins (exponential backoff)"

**Core Challenge:** Simple `Thread.sleep()` or `@Scheduled` fails in distributed system. Need persistence, exactly-once execution, scalability, and handling server crashes.

## 2. Requirements

### Functional
- Schedule a job: `POST /schedule {execute_at: "2026-10-27T10:00:00", payload, type}`
- Types: ONE_TIME, RECURRING (Cron: `0 9 * *`), DELAYED (delay 10 mins)
- Cancel job: `DELETE /jobs/{id}`
- Retry with backoff: If job fails, retry 3 times
- Dashboard: View job status (PENDING, RUNNING, SUCCESS, FAILED)

### Non-Functional
- Scale: 100M scheduled jobs
- Accuracy: Job should execute within 1 sec of scheduled time (not 10 mins late)
- Availability: 99.99% - No job should be missed if server crashes
- Exactly Once: Job should execute exactly once, even with multiple servers
- Throughput: 50k jobs/sec execution

## Design Decisions

| Decision / Pattern | Used In Job Scheduler For | Why This Pattern / Choice? | SOLID / Design Principle |
| :--- | :--- | :--- | :--- |
| **Command** | Job - `EmailJob`, `ReportJob`, `BackupJob`, `CleanupJob` all implement `Job.execute()` | Each job is an object - can be queued, retried, logged, cancelled, scheduled | **OCP + SRP** - New job = new class, no change in scheduler |
| **Strategy** | Retry Strategy - Fixed Delay, Exponential Backoff, No Retry, Custom Retry | Retry logic changes by job type - Payment job retries differently than Email job | **OCP** - Switch strategy at runtime |
| **Strategy** | Scheduling Strategy - Cron, Fixed Rate, Fixed Delay, One-Time | Same scheduler supports all trigger types with same interface | **OCP + SRP** |
| **Factory** | Creating Job - `JobFactory.create(type)` - Email, Report, Backup, Webhook | Client sends type string, Factory decides which Job class to create | **SRP + OCP** - Creation logic in one place |
| **Factory** | Creating Trigger - `TriggerFactory.create(cron / fixedRate / oneTime)` | Parse cron expression and return correct Trigger object | **OCP** |
| **Singleton** | Scheduler Instance, DB Connection, Redis Client, Thread Pool, Config Manager | Only one scheduler should run in one node, avoid duplicate execution | **SRP** - One instance controls one shared resource |
| **Observer** | Job Events - Job Started, Job Success, Job Failed, Job Retried, Job Timeout | One event notifies many - Logger, Metrics, Email Alert, Slack, Dashboard | **OCP + SRP** - Sender and receivers decoupled |
| **Builder** | Building `JobDetail` - JobId, JobType, Payload, RetryCount, Timeout, Priority, Cron | Job has 10+ optional fields, avoid `new Job(a,b,c,d,e,f,null)` | **SRP** - Construction separate from representation |
| **Adapter** | Executor - Local ThreadPool, Kubernetes Job, AWS Lambda, External API | Our code uses one `JobExecutor` interface, Adapter converts to vendor format | **OCP + DIP** - Depend on abstraction, not concrete executor |
| **Decorator** | Job Wrapper - Logging + Metrics + Timeout + Retry + Circuit Breaker | Add features dynamically like `Retry(Timeout(Logging(Job)))` without changing base Job | **OCP + SRP** - Each feature in its own class |
| **Repository** | `JobRepository`, `JobExecutionRepository`, `JobHistoryRepository` | Separate DB logic from business logic, easy to switch PostgreSQL to MongoDB | **SRP + DIP** - Business logic doesn't depend on DB |
| **Facade** | `SchedulerFacade.scheduleJob()` | One method hides - Validate, Save to DB, Add to Queue, Create Trigger, Notify | **SRP** - Client sees one simple interface |
| **State Pattern** | Job State - PENDING, RUNNING, SUCCESS, FAILED, RETRYING, CANCELLED, TIMEOUT | Each state has its own behaviour, no big `if-else` for status | **OCP + SRP** |
| **Template Method** | `BaseJob.execute()` - Pre-check -> Execute -> Post-log -> Update DB | Common flow fixed, only `doExecute()` changes per job | **OCP** - Reuse common code |
| **Microservices** | Scheduler Service, Worker Service, API Service, Monitoring Service | Scale Workers 10x during peak, Scheduler stays stable | **SRP** - One service, one responsibility |
