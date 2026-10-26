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

## 3. Why Simple Solutions Fail?

| Solution | Problem |
| :--- | :--- |
| `@Scheduled` in Spring Boot | Works on 1 server only. If you have 3 servers, job runs 3 times |
| `Thread.sleep()` / DelayQueue in memory | If server crashes, all jobs lost |
| Database polling: `SELECT * WHERE execute_at < now() every sec` | DB dies at 10k QPS, full table scan |
| Quartz Single Node | Not distributed |

**We need:** Persistent storage + Distributed Lock + Polling with Index + Queue

## 4. High-Level Architecture (Your Stack)
