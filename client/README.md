# ScrapeFleet

> A distributed web scraping and crawling platform built to explore load balancing, job queues, worker systems, concurrency, and fault tolerance.

## Overview

**ScrapeFleet** is a distributed web scraper where users submit URLs and the system distributes scraping jobs across multiple worker instances.

The project is designed less as a traditional CRUD application and more as a **distributed-systems learning project**.

The main engineering goal is to understand what happens when a workload is split across multiple machines or processes, how workers communicate, and how the system behaves when individual workers become unavailable.

---

## Core Architecture

```text
                         ┌───────────────┐
                         │    SvelteKit  │
                         │   Dashboard   │
                         └───────┬───────┘
                                 │
                                 ▼
                         ┌───────────────┐
                         │   Go / Fiber  │
                         │      API      │
                         └───────┬───────┘
                                 │
                    ┌────────────┴────────────┐
                    ▼                         ▼
              ┌───────────┐             ┌──────────┐
              │   Redis   │             │ Postgres │
              │   Queue   │             │  / Neon  │
              └─────┬─────┘             └──────────┘
                    │
                    ▼
             ┌───────────────┐
             │ Load Balancer │
             └───────┬───────┘
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
      ┌────────┐ ┌────────┐ ┌────────┐
      │Worker 1│ │Worker 2│ │Worker 3│
      │  Go    │ │  Go    │ │  Go    │
      └────────┘ └────────┘ └────────┘
          │          │          │
          └──────────┼──────────┘
                     ▼
                  Internet
                     │
              ┌──────┴──────┐
              ▼             ▼
          Website A      Website B
```

### Important distinction

The project uses both a **job queue** and a **load balancer**, but they solve different problems.

**Job queue:**

> Which worker should process this job?

**Load balancer:**

> Which available server should receive this HTTP request?

This distinction is an important part of the project's learning objectives.

---

# Features

## URL Scraping

Users can submit one or multiple URLs.

Example:

```text
https://example.com
https://example.org
https://news.ycombinator.com
```

The system creates an individual job for each URL.

---

## Page Metadata Extraction

For every successfully scraped page, collect:

- URL
- HTTP status code
- response time
- content type
- page title
- description
- HTML size
- text size
- word count
- image count
- link count

Example result:

```text
URL: https://example.com

Status: 200
Response Time: 142ms
Content Type: text/html
HTML Size: 1.2 KB

Title:
Example Domain

Links: 1
Images: 0
Words: 22
```

---

# Job System

Every URL becomes a job.

A job follows this lifecycle:

```text
PENDING
   │
   ▼
PROCESSING
   │
   ├───────────────┐
   ▼               ▼
COMPLETED        FAILED
```

Example:

```json
{
	"id": "job-001",
	"url": "https://example.com",
	"status": "completed",
	"worker_id": "worker-2",
	"status_code": 200,
	"response_time_ms": 142
}
```

---

# Worker System

Workers are independent processes responsible for executing scraping jobs.

Example:

```bash
./worker --port 4000 --id worker-1
./worker --port 4001 --id worker-2
./worker --port 4002 --id worker-3
```

A worker:

1. Receives or claims a job.
2. Fetches the URL.
3. Parses the response.
4. Extracts metadata.
5. Stores the result.
6. Reports completion.
7. Requests another job.

---

# Distributed Job Processing

Suppose a crawl contains 100 URLs.

Instead of processing everything sequentially:

```text
Worker 1
   │
   ├── URL 1
   ├── URL 2
   ├── URL 3
   ├── ...
   └── URL 100
```

The queue distributes jobs:

```text
                    Redis
                      │
           ┌──────────┼──────────┐
           ▼          ▼          ▼
        Worker 1   Worker 2   Worker 3
           │          │          │
        Job 1      Job 2      Job 3
        Job 4      Job 5      Job 6
        Job 7      Job 8      Job 9
```

When Worker 1 finishes:

```text
Worker 1 → Job 10
```

The worker can immediately continue processing.

---

# Fault Tolerance

Workers can fail.

For example:

```text
Worker 1 ──────────── Online
Worker 2 ──────────── CRASHED
Worker 3 ──────────── Online
```

If Worker 2 was processing a job, the system should eventually detect that the job has become abandoned.

The job can then be returned to the queue:

```text
Worker 2
   │
   X
   │
   ▼
Job timeout
   │
   ▼
Return job to queue
   │
   ▼
Worker 3
```

This prevents a single worker failure from permanently losing a job.

---

# Job Leases

When a worker claims a job, it receives a temporary lease.

Example:

```text
Job:
    ID: job-42
    Worker: worker-2
    Status: processing
    Started: 12:01:00
    Lease: 30 seconds
```

If Worker 2 successfully completes the job:

```text
COMPLETED
```

If Worker 2 disappears:

```text
30 seconds pass
        │
        ▼
Lease expires
        │
        ▼
Job becomes PENDING
```

Another worker can then process it.

---

# Retry System

Temporary failures should not immediately become permanent failures.

Example:

```text
Attempt 1 → FAILED
Attempt 2 → FAILED
Attempt 3 → SUCCESS
```

Use exponential backoff:

```text
1 second
2 seconds
4 seconds
8 seconds
```

The system should have a configurable maximum retry count.

Example:

```text
max_retries = 3
```

After the maximum number of attempts:

```text
FAILED
```

---

# Rate Limiting

The scraper should avoid sending excessive requests to the same website.

Example:

```text
example.com → 2 requests/sec
example.org → 2 requests/sec
github.com  → 1 request/sec
```

Workers should also have a concurrency limit.

Example:

```text
Worker concurrency = 5
```

This means one worker cannot process more than five scraping tasks simultaneously.

The scraper should respect applicable:

- `robots.txt`
- website terms
- rate limits
- access restrictions

The project should not be designed to bypass anti-bot systems or access controls.

---

# Load Balancer

The project includes a custom Go load balancer.

Example:

```text
                 Load Balancer
                 :3000
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
       :4000       :4001       :4002
      Worker 1    Worker 2    Worker 3
```

The load balancer should support:

- round-robin distribution
- health checks
- worker registration
- worker removal
- request forwarding
- failure detection
- basic request logging

Example request distribution:

```text
Request 1 → Worker 1
Request 2 → Worker 2
Request 3 → Worker 3
Request 4 → Worker 1
Request 5 → Worker 2
```

---

# Health Checks

Each worker exposes:

```http
GET /health
```

Example response:

```json
{
	"status": "healthy",
	"worker_id": "worker-2"
}
```

The load balancer periodically checks workers.

If a worker stops responding:

```text
Before:

LB
├── Worker 1
├── Worker 2
└── Worker 3


Worker 2 crashes


After:

LB
├── Worker 1
└── Worker 3
```

When Worker 2 becomes healthy again, it can be added back into the pool.

---

# Monitoring Dashboard

The frontend should provide basic system monitoring.

Example:

```text
┌──────────────────────────────────────────────────┐
│                SCRAPEFLEET                       │
├──────────────────────────────────────────────────┤
│                                                  │
│ Workers          3                              │
│ Active Jobs      7                              │
│ Completed        1,293                          │
│ Failed           34                             │
│                                                  │
├──────────────────────────────────────────────────┤
│ Worker Performance                               │
│                                                  │
│ Worker 1    421 jobs    83 jobs/s               │
│ Worker 2    436 jobs    91 jobs/s               │
│ Worker 3    436 jobs    87 jobs/s               │
│                                                  │
├──────────────────────────────────────────────────┤
│ Average Response Time: 183ms                    │
│ Success Rate:          97.4%                    │
│ Throughput:             261 jobs/s              │
│ Queue Depth:            18                      │
│                                                  │
└──────────────────────────────────────────────────┘
```

---

# Metrics

The system should collect metrics such as:

### Worker metrics

- jobs completed
- jobs failed
- active jobs
- worker uptime
- worker health
- average processing time

### Scraper metrics

- requests per second
- average response time
- HTTP status distribution
- successful requests
- failed requests
- retry count

### Queue metrics

- queue depth
- pending jobs
- processing jobs
- completed jobs
- failed jobs
- average waiting time

### System metrics

- CPU usage
- memory usage
- netwo
