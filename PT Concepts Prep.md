# ISTQB Performance Testing (CT-PT) — Interview Prep Tree

```
ISTQB PERFORMANCE TESTING
│
├── 1. FUNDAMENTALS OF PERFORMANCE TESTING
│   │
│   ├── 1.1 Purpose of Performance Testing
│   │   ├── Verify system meets performance requirements/SLAs
│   │   ├── Identify bottlenecks before production
│   │   ├── Support capacity planning & sizing decisions
│   │   └── Build confidence for release / go-live
│   │
│   ├── 1.2 Performance-Related Risks
│   │   ├── Poor response time under load
│   │   ├── Resource exhaustion (CPU, memory, disk I/O, network, connections, threads)
│   │   ├── Scalability failure (system can't grow with demand)
│   │   ├── Reliability failure (crashes, memory leaks under sustained load)
│   │   └── Business risk: lost revenue, reputational damage, SLA penalties
│   │
│   ├── 1.3 Categories of Performance Testing (Test Types)
│   │   ├── Load Testing → expected/normal load, validate SLAs
│   │   ├── Stress Testing → beyond capacity, find breaking point
│   │   ├── Spike Testing → sudden sharp increase/decrease in load
│   │   ├── Soak/Endurance Testing → sustained load over long duration → detect leaks, degradation
│   │   ├── Scalability Testing → incrementally increase load/resources, measure how system scales
│   │   ├── Volume Testing → large volumes of data (DB size, queue depth), not necessarily concurrent users
│   │   ├── Concurrency Testing → simultaneous access to shared resources → race conditions, deadlocks
│   │   └── Capacity Testing → determine max users/throughput system can sustain while meeting SLA
│   │
│   └── 1.4 Key Terminology
│       ├── Response Time = time to process a single request
│       ├── Throughput = transactions/requests processed per unit time
│       ├── Latency = network/transmission delay component of response time
│       ├── Think Time = user pause between actions (simulated in scripts)
│       ├── Pacing = controlled delay between iterations
│       ├── Ramp-up/Ramp-down = gradual increase/decrease of virtual users
│       ├── Concurrent Users vs Active Users vs Virtual Users (VUsers)
│       └── Utilization = % of resource capacity consumed (CPU%, memory%)
│
├── 2. PERFORMANCE TESTING THROUGHOUT THE SDLC
│   │
│   ├── 2.1 Why Test Early (Shift-Left)
│   │   ├── Cheaper to fix design-level bottlenecks early
│   │   ├── Unit-level / component-level performance testing (API, DB query level)
│   │   └── Continuous performance testing in CI/CD pipelines
│   │
│   ├── 2.2 Performance Testing in Different Lifecycle Models
│   │   ├── Sequential (Waterfall) → dedicated performance test phase near end
│   │   ├── Iterative/Incremental → performance testing each iteration
│   │   └── Agile → performance testing integrated into sprints, automated in pipeline
│   │
│   └── 2.3 Types of Environments
│       ├── Component/Unit level (isolated code, DB query, microservice)
│       ├── Integration level (multiple components/services)
│       └── System level (full production-like environment)
│
├── 3. PERFORMANCE TESTING TASKS (THE TEST PROCESS)
│   │
│   ├── 3.1 Risk Identification & Analysis
│   │   ├── Identify performance-critical business processes
│   │   ├── Assess probability & impact
│   │   └── Prioritize test scenarios by risk
│   │
│   ├── 3.2 Test Planning
│   │   ├── Define objectives, scope, NFRs (SLAs)
│   │   ├── Identify workload model (business processes, transaction mix, user distribution)
│   │   ├── Define test environment requirements (prod-like sizing)
│   │   ├── Define entry/exit criteria
│   │   └── Define tools, roles, schedule
│   │
│   ├── 3.3 Test Design & Specification
│   │   ├── Workload modeling
│   │   │   ├── Identify key business transactions
│   │   │   ├── Transaction mix / distribution (%)
│   │   │   ├── User arrival rate / concurrency model
│   │   │   └── Data variation strategy (parameterization)
│   │   ├── Define load profiles (steady, ramp, spike, step)
│   │   └── Define acceptance criteria per scenario
│   │
│   ├── 3.4 Test Environment Setup
│   │   ├── Prod-like hardware/software/network topology
│   │   ├── Test data preparation (volume + variety)
│   │   ├── Monitoring/APM tool setup (Dynatrace, Datadog, ELK, Grafana)
│   │   └── Isolate environment from other testing to avoid contention
│   │
│   ├── 3.5 Test Tool Setup / Script Development
│   │   ├── Record or build scripts (JMeter, LoadRunner VuGen)
│   │   ├── Correlation → handle dynamic values (session IDs, tokens, CSRF)
│   │   ├── Parameterization → data-driven inputs
│   │   ├── Add assertions/validations
│   │   └── Configure think time, pacing, timers
│   │
│   ├── 3.6 Test Execution
│   │   ├── Baseline run (single user) → validate script correctness
│   │   ├── Incremental load runs
│   │   ├── Full scenario execution per load profile
│   │   ├── Monitor system resources & app behavior in real time
│   │   └── Capture logs, metrics, errors during run
│   │
│   ├── 3.7 Analysis & Reporting
│   │   ├── Analyze response time percentiles (p90, p95, p99), avg, median
│   │   ├── Analyze throughput vs. target
│   │   ├── Analyze error rate
│   │   ├── Correlate app metrics with infra metrics (CPU/GC/DB waits) to find root cause
│   │   ├── Identify bottleneck layer (app, DB, network, infra)
│   │   └── Report: pass/fail vs SLA, trends, recommendations
│   │
│   └── 3.8 Test Closure
│       ├── Compare actual vs. planned coverage
│       ├── Archive scripts, results, reports for reuse/regression
│       └── Lessons learned / retrospective
│
├── 4. PERFORMANCE TESTING TECHNIQUES & MEASUREMENTS
│   │
│   ├── 4.1 Load Modeling Approaches
│   │   ├── Closed Model → fixed number of virtual users, each waits for response before next request
│   │   └── Open Model → new users arrive at a defined rate regardless of system response (more realistic for web-scale)
│   │
│   ├── 4.2 Statistical Concepts
│   │   ├── Little's Law → L = λ × W (Users = Arrival Rate × Response Time)
│   │   ├── Percentiles vs Average (why p95/p99 matter more than mean — masks outliers)
│   │   └── Standard deviation / variance in response times
│   │
│   ├── 4.3 System Resource Monitoring Metrics
│   │   ├── CPU utilization
│   │   ├── Memory usage / heap / GC pauses
│   │   ├── Disk I/O
│   │   ├── Network I/O / bandwidth
│   │   ├── Thread/connection pool usage
│   │   ├── DB metrics (query time, lock waits, connection pool)
│   │   └── Queue depth (e.g., Kafka consumer lag)
│   │
│   └── 4.4 Bottleneck Analysis Approach
│       ├── Application layer (code inefficiency, thread contention)
│       ├── Database layer (slow queries, indexing, locks)
│       ├── Infrastructure layer (CPU/memory/network saturation)
│       ├── Middleware/integration layer (API gateway, message queue)
│       └── Client-side (browser rendering, network latency) — for web apps
│
├── 5. PERFORMANCE TESTING TOOLS
│   │
│   ├── 5.1 Tool Categories
│   │   ├── Load generation tools → JMeter, LoadRunner, Gatling, k6
│   │   ├── Monitoring/APM tools → Dynatrace, Datadog, AppDynamics, New Relic
│   │   ├── Log analysis tools → ELK/Kibana, Splunk
│   │   └── CI/CD integration tools → Jenkins, GitLab CI
│   │
│   ├── 5.2 Tool Selection Criteria
│   │   ├── Protocol support (HTTP, JDBC, JMS/Kafka, gRPC etc.)
│   │   ├── Scripting/correlation capability
│   │   ├── Scalability of load generation (distributed load)
│   │   ├── Reporting & analytics depth
│   │   ├── Integration with CI/CD & monitoring
│   │   └── Licensing cost vs open-source
│   │
│   └── 5.3 Risks of Tool Usage
│       ├── Load generator itself becomes bottleneck (resource-starved injectors)
│       ├── Network between generator & SUT not representative
│       └── Tool licensing limiting concurrent virtual users
│
└── 6. BEST PRACTICES / SUCCESS FACTORS
    ├── Test in prod-like environment (data volume + infra parity)
    ├── Use realistic workload models, not guesswork
    ├── Isolate variables — one change at a time between test runs
    ├── Always capture a baseline before comparing results
    ├── Correlate every dynamic value — never hardcode session data
    ├── Monitor client + server + network side simultaneously
    ├── Automate performance tests in CI/CD for early regression detection
    └── Report in business terms (SLA compliance) not just raw numbers
```

## Quick-Recall Cheat Sheet (30-second answers)

| Concept | One-liner to say in interview |
|---|---|
| Load vs Stress vs Spike vs Soak | Load = expected traffic validate SLA; Stress = push past capacity to find breaking point; Spike = sudden burst; Soak = sustained duration to catch leaks/degradation |
| Closed vs Open model | Closed = fixed VUsers, wait-and-repeat; Open = arrival-rate driven, users don't wait on each other — more realistic for internet-facing systems |
| Little's Law | Users (L) = Arrival Rate (λ) × Response Time (W) — used to derive required concurrent users from expected throughput |
| Why percentiles over average | Average hides outliers; p95/p99 shows real experience of slowest users, which is what SLAs are usually written against |
| Correlation | Capturing dynamic server-generated values (tokens, session IDs) from response and reusing in subsequent requests so scripts don't fail on replay |
| Think Time vs Pacing | Think time = user's pause between actions within a transaction; Pacing = controls iteration start-to-start time to hit a target throughput |
| Bottleneck triage order | Check app server → DB → infra (CPU/mem/network) → middleware/queue → client-side, correlating APM traces with infra metrics at the same timestamp |
