# Interview Prep — Performance Test Lead Panel
### Areas covered: Self-intro | NFR/Resiliency | JMeter | Java coding | Dynatrace/JVM

---

## 1. Self-Introduction (30–45 sec, then expand if asked)

> "I'm Vijay, a Performance Test Lead with 10+ years in QA, 8+ years focused specifically on Performance Engineering. 

> My core stack is LoadRunner (VuGen, Controller, Analysis — HTTP, Web Services, Java, Citrix, MQ, Oracle protocols) and JMeter for distributed load testing, paired with Dynatrace and Datadog for APM, and ELK/Kibana for log correlation. I've also worked extensively with Kafka on the production side — queue performance, consumer lag, offset management — since a lot of our provisioning workflows were event-driven.

> Beyond scripting and execution, I own the full performance engineering lifecycle: NFR gathering, test strategy/planning, scenario design, execution, bottleneck analysis with APM tools, and defect triage down to root cause — including recovering 15+ Severity 1/2 defects across engagements. I'm ISTQB Performance Testing and Dynatrace Associate certified, and I hold a JMeter certification as well.

> I'm currently on notice, wrapping up on 28th August, and looking for a Performance Test Lead/Manager role — open to relocating anywhere in India."

**Tip:** They will steer you immediately into stress/endurance from this — have that transition ready (see Section 8).

---

## 2. NFR vs Performance Testing — how they differ

- **NFR (Non-Functional Requirements) discussion** happens *upstream*, at design/architecture stage. It's about defining the **targets**: expected concurrent users, TPS, response time SLAs per transaction, throughput, scalability targets, availability %, failover/recovery time (RTO/RPO), data volume growth, infrastructure sizing. It's a conversation with architects, business, and infra teams — output is a signed-off NFR document.
- **Performance testing** is the *execution/validation* phase — you take those NFRs and design test scenarios (load, stress, endurance, spike, resiliency) to **prove or disprove** the system meets them, using tools like JMeter/LoadRunner, and validate against APM data (Dynatrace/Datadog).
- One-liner distinction: **NFR = what "good" looks like (targets); Performance testing = whether the system actually delivers it (evidence).**
- Resiliency is a subset of NFR too — often stated as "system should degrade gracefully / recover within X seconds if a dependency fails" — and performance/resiliency testing is what proves that.

---

## 3. Resiliency Test Plan Document — have you worked on one? What does it include?

Yes — frame it as: *"I've contributed to and authored resiliency/failure-injection test plans as part of NFR sign-off for critical provisioning and billing flows."*

**Typical contents of a resiliency test plan:**
1. **Scope & objective** — which application/flow, which layers are in scope (web, app, mid-tier/integration, DB, external systems)
2. **In-scope failure scenarios** — server crash, network latency/partition, DB connection pool exhaustion, service timeout, dependency/API failure, disk full, CPU/memory starvation, pod/container kill (if Kubernetes)
3. **Entry/exit criteria** — when testing can start (env stability, baseline performance established) and when it's considered complete
4. **Test environment & topology diagram** — layers, load balancers, clustering/failover setup
5. **Steady-state hypothesis** — the expected normal behavior/metrics baseline before injecting failure (this is core to chaos engineering practice)
6. **Failure injection method/tooling** — e.g., manual (kill process, block port via iptables), or tools like Chaos Monkey, Gremlin, Istio fault injection, AWS FIS
7. **Monitoring & observability plan** — which dashboards/tools (Dynatrace, Datadog, ELK) and which metrics to watch during injection (error rate, latency, retries, circuit breaker state)
8. **Expected vs actual behavior / recovery criteria** — e.g., "system should failover within 30s with <1% transaction loss"
9. **Rollback plan** — how to restore the failed component
10. **Roles & responsibilities**, **risk & mitigation**, **schedule**
11. **Defect/observation log template** and **sign-off section**

---

## 4. Resiliency Testing Across Layers (Web → App → Mid-tier → DB)

Structure your answer as a **layered strategy**, not one big test:

- **Web/Load balancer layer:** Kill one web/app node while load is running → verify LB detects it (health check) and reroutes traffic without dropping active sessions; check sticky-session behavior if applicable.
- **App server layer:** Simulate app server thread pool exhaustion or high CPU/memory (using a background load or tools like stress-ng); observe whether requests queue gracefully, timeout as configured, or throw 5xx; check auto-scaling triggers if cloud-native.
- **Mid-tier / integration layer (MQ, Kafka, APIs):** Kill a Kafka broker or introduce consumer lag; verify producer retries, dead-letter handling, no message loss; for synchronous mid-tier calls, inject latency/timeout and verify circuit breaker (Hystrix/Resilience4j) trips and falls back correctly instead of cascading failure.
- **DB layer:** Simulate connection pool exhaustion, kill a DB replica, or introduce query latency; verify connection retry logic, read-replica failover, and that the app doesn't hang indefinitely (timeout configs honored).
- **Cross-cutting:** At each layer, correlate with APM (Dynatrace/Datadog service flow, PurePath) to pinpoint exactly where the failure surfaces and how it propagates — this is the key value-add of layered design: you're isolating **blast radius** one layer at a time rather than testing everything simultaneously, so root cause is unambiguous.

**Design principle to state explicitly:** "I design scenarios bottom-up in isolation first (single layer, single failure) to get a clean signal, then combine failures (e.g., app server degradation + DB latency together) to simulate realistic cascading scenarios once individual layers are proven resilient."

---

## 5. External System Resiliency (e.g., Payment Gateway redirect flow)

Key challenge they're testing: you usually **don't have injection access** to a third-party system.

Approach to describe:
1. **Contract/SLA review first** — check the payment gateway's published SLA (uptime, timeout behavior, retry policy) — this sets your expectation baseline.
2. **Simulate the failure at your boundary, not theirs** — since you can't kill their servers, use a **service virtualization / stub / mock** (WireMock, SoapUI mock service, or a sandbox/UAT endpoint the payment provider gives) to simulate: timeout, slow response, HTTP 5xx, malformed response, connection refused.
3. **Validate your application's handling of each simulated failure:**
   - Does it timeout within the configured limit rather than hang?
   - Does it retry (and is retry idempotent — important for payments, to avoid double charging)?
   - Does it show a proper user-facing error/fallback instead of a blank/crashed redirect?
   - On return-to-origin after payment, does it correctly reconcile state if the callback is delayed or lost (webhook/callback failure scenario)?
4. **If sandbox access is available**, coordinate with the payment provider to run limited fault-injection tests in their UAT/sandbox environment jointly.
5. **Monitor via APM on your side** end-to-end trace, so even though you can't see inside their system, you can measure exactly how long you waited and how your system reacted.

One-liner if pressed: *"For a system I don't own, resiliency testing shifts from 'can I break it' to 'can I prove my application degrades gracefully when it fails' — using stubs/mocks to simulate the failure modes and validating timeout, retry, and fallback behavior on my side."*

---

## 6. Types of Failure Injections You've Conducted

Have a concrete list ready (pick ones consistent with your background):
- Server/process kill (app node, one instance in a cluster)
- Network latency injection / packet loss between layers
- Service timeout simulation (mocked slow dependency)
- Database connection pool exhaustion
- Kafka broker/consumer failure — killed a broker, verified rebalancing and no message loss; introduced consumer lag and validated alerting
- CPU/memory saturation on app server (resource starvation)
- Disk space exhaustion on a log-heavy service
- Dependency/API failure via mocked 5xx or connection refused
- (If Kubernetes/cloud-native) pod kill / node drain to test auto-healing

---

## 7. Issues Identified from Resiliency/Performance Testing

Have 2–3 concrete, defensible examples ready, e.g.:
- Connection pool not releasing connections on timeout → pool exhaustion under sustained load → traced via Dynatrace to a non-closed DB connection in exception path.
- Retry storm — a downstream timeout caused synchronous retries without backoff, amplifying load on an already-struggling service (classic cascading failure) — recommended exponential backoff + circuit breaker.
- Kafka consumer lag growing unbounded during a load spike because consumer parallelism wasn't scaled with partition count — identified via consumer lag metrics.
- Session stickiness breaking on node failover, causing user logouts — found during LB failover injection.

**Frame every issue as: symptom → tool used to isolate it → root cause → recommendation.** This is the pattern panels want to hear.

---

## 8. Load vs Stress vs Endurance vs Spike — definitions to have crisp

| Type | Purpose | How you set it up |
|---|---|---|
| **Load test** | Validate system behaves within SLA at **expected/normal production load** (e.g., 1000 concurrent users) | Ramp-up gradually to target user count, hold steady state, measure response time/throughput/error rate against NFR targets |
| **Stress test** | Find the **breaking point** — push beyond expected load until failure/degradation | Incrementally increase load beyond normal (e.g., 1000 → 1500 → 2000...) until response time degrades sharply or errors spike; identify the ceiling and failure mode |
| **Endurance (Soak) test** | Detect issues that only appear **over time** — memory leaks, connection leaks, log/disk growth, DB growth-related slowdown | Run at moderate/expected load for an extended duration (8–24+ hrs), monitor trend lines (not just peak numbers) — GC behavior, heap growth, thread count over time |
| **Spike test** | Validate behavior under **sudden, sharp bursts** of load (e.g., flash sale, campaign launch) | Ramp up very fast to a high user count, hold briefly, drop suddenly — check if system recovers cleanly after the drop, not just whether it survives the peak |

**For your 1000-user load test example**, describe it step-by-step:
1. Confirm NFR: e.g., 1000 concurrent users, response time <3s at 95th percentile, error rate <1%.
2. Design realistic think-time and transaction mix (based on production traffic pattern, not uniform).
3. Ramp-up strategy: gradual ramp-up (e.g., 100 users every 30s) rather than all at once, to mimic real user arrival and avoid a false-positive bottleneck from ramp shock.
4. Correlate JMeter results with server-side APM (Dynatrace) during the run — CPU, memory, GC, DB query time.
5. Compare against baseline/NFR, report throughput, response time percentiles (avg is not enough — always look at p90/p95/p99), error rate, and resource utilization.
6. If SLA breached, drill into APM traces to find the bottleneck.

---

## 9. JMeter Architecture (be ready to draw/describe verbally)

- **Test Plan** is the root — contains everything.
- **Thread Group** — defines number of virtual users (threads), ramp-up period, loop count/duration. Multiple thread groups can simulate different user profiles in one test.
- **Samplers** (HTTP Request, JDBC Request, etc.) — the actual requests sent.
- **Logic Controllers** (If, While, Loop, Module, Transaction) — control execution flow.
- **Config Elements** (HTTP Request Defaults, CSV Data Set Config, HTTP Header Manager) — shared configuration.
- **Pre/Post Processors** — e.g., extracting a token via Regex/JSON Extractor post-processor, or manipulating a request pre-processor.
- **Assertions** — validate response (Response Assertion, JSON Assertion, Duration Assertion).
- **Timers** — Constant Timer, Uniform Random Timer — control pacing/think-time.
- **Listeners** — result collection (View Results Tree for debug only, Summary Report/Aggregate Report for actual runs — avoid heavy listeners during real load).
- **Engine model:** JMeter is Java-based; each thread = one simulated user running independently, executing the sampler tree top-down per iteration.

---

## 10. Simulating 7000 Users in JMeter — Distributed Testing Steps

A single JMeter machine typically maxes out well before thousands of heavy users due to JVM/thread and OS resource limits, so:

1. **Estimate capacity per load generator** — based on protocol (HTTP is lighter than heavy JDBC/Java samplers) and payload size, decide how many threads one machine can sustain (rule of thumb, do a capacity test first — don't assume a number).
2. **Set up Distributed Testing (master-slave / remote testing):**
   - One **controller (master)** machine running the JMeter GUI/CLI that orchestrates.
   - Multiple **load generator (slave/worker)** machines running `jmeter-server`.
   - Configure `remote_hosts` in `jmeter.properties` on the master with the IPs of all worker machines.
   - Run in **non-GUI mode** with `-r` (run remote) or `-R` flag pointing to worker IPs.
3. **Distribute the 7000 users** across however many worker nodes you provision (e.g., 7 workers × 1000 users each), each worker running headless (`-n` non-GUI mode) for lower overhead.
4. **Aggregate results** back at the controller, or better — push results to a results server / InfluxDB+Grafana / BlazeMeter-style backend listener for real-time aggregation instead of pulling huge local result files.
5. **Key considerations to mention:** ensure all machines are time-synced (NTP) for accurate correlation, use `-Xms`/`-Xmx` JVM heap tuning on each worker, disable unnecessary listeners on workers, and validate network bandwidth between load generators and target isn't itself a bottleneck (test agents should not be under-resourced or you get false results).
6. **Cloud/CI alternative:** if using Jenkins/cloud infra, spin up multiple containerized JMeter workers (e.g., via Docker/Kubernetes) as an alternative to static VMs, for elastic scaling.

---

## 11. If Controller vs While Controller

- **If Controller:** executes its child elements **once, conditionally** — based on a boolean expression evaluated at that point in the loop iteration. Use when a decision is a **one-time branch** (e.g., "only hit this endpoint if user type == premium").
- **While Controller:** executes its child elements **repeatedly** as long as a condition remains true — re-evaluated after each loop. Use when you need to **poll for a state change**.

**Use their exact report-generation example as your canned answer:**
> "A good example is asynchronous report generation. You submit a 'generate report' request, and the backend may take 3–5 minutes. Since the next transaction depends on the report being ready, you can't just wait with a fixed pause. So I use a While Controller with the condition `${status}` != 'Completed'. Inside, I place a 'check status' request plus a Regex/JSON post-processor to extract the current status (pending / in-progress / completed) into a variable, and a short Timer to avoid hammering the server on every poll. The While loop keeps polling until status == Completed, then execution proceeds to the next transaction. I'd also add a max-iteration safety counter to avoid an infinite loop if something goes wrong."

---

## 12. Module Controller / Test Fragments

Yes — describe the use case: *"I use Test Fragments combined with the Module Controller to avoid duplicating common reusable flows — like a login sequence or a token-refresh flow — across multiple thread groups/scenarios. I define the fragment once outside the main thread group tree, then reference it via Module Controller wherever needed. This keeps the test plan maintainable — a change to login logic only needs to happen in one place."*

## 13. setUp Thread Group / tearDown Thread Group

Yes — *"setUp Thread Group runs before the main thread groups start — I use it for one-time setup like authentication token generation, test data seeding, or environment warm-up. tearDown Thread Group runs after all main thread groups finish — I use it for cleanup, like deleting test data created during the run or invalidating sessions/tokens, so the environment is left clean for the next run."*

## 14. Jenkins Integration

*"Yes, I've integrated JMeter execution into Jenkins pipelines — running JMeter in non-GUI mode (`-n -t testplan.jmx -l results.jtl`) triggered as a pipeline stage, often parameterized (thread count, duration, target env) via Jenkins parameters. Results (JTL) are then published using the Performance plugin or converted to HTML dashboard reports for trend comparison across builds, and I've set threshold-based build failure (e.g., fail build if error rate > X% or p95 > SLA) to gate releases automatically."* If you haven't done full CI integration end-to-end, be honest: *"I've primarily run in non-GUI mode manually/scheduled; I have working knowledge of Jenkins pipeline integration and threshold-based gating."*

---

## 15. SQL & Java Comfort (be direct — say yes and demonstrate)

**SQL — be ready to write, live, without hesitation:**
```sql
SELECT * FROM orders WHERE status = 'PENDING';
UPDATE orders SET status = 'COMPLETED' WHERE order_id = 1001;
INSERT INTO orders (order_id, customer_id, status) VALUES (1002, 501, 'NEW');
```

**Java logic — Random numbers 80 to 100, print min to max (sorted ascending), no duplicates repeated in wrong order:**
```java
import java.util.*;

public class RandomRange {
    public static void main(String[] args) {
        Random rand = new Random();
        List<Integer> numbers = new ArrayList<>();

        // Generate 10 random numbers between 80 and 100 (inclusive)
        for (int i = 0; i < 10; i++) {
            int value = 80 + rand.nextInt(21); // 21 = (100-80+1)
            numbers.add(value);
        }

        System.out.println("Generated: " + numbers);

        Collections.sort(numbers); // ascending: min -> max
        System.out.println("Min: " + numbers.get(0));
        System.out.println("Max: " + numbers.get(numbers.size() - 1));
        System.out.println("Sorted (min to max): " + numbers);
    }
}
```
*Talk through it out loud as you type: generate → store in a List → sort → print min/first and max/last.*

**Simulate 1000 API calls — the logic they want (say this out loud, don't over-engineer):**
```java
import java.net.HttpURLConnection;
import java.net.URL;
import java.util.concurrent.*;

public class ApiCallSimulator {
    public static void main(String[] args) throws Exception {
        String urlString = "https://api.example.com/endpoint"; // target URL - defined explicitly / from config
        int totalCalls = 1000;

        ExecutorService executor = Executors.newFixedThreadPool(50); // controls concurrency

        for (int i = 0; i < totalCalls; i++) {
            executor.submit(() -> {
                try {
                    URL url = new URL(urlString);
                    HttpURLConnection conn = (HttpURLConnection) url.openConnection();
                    conn.setRequestMethod("GET");
                    int responseCode = conn.getResponseCode();
                    System.out.println("Response: " + responseCode);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            });
        }
        executor.shutdown();
        executor.awaitTermination(5, TimeUnit.MINUTES);
    }
}
```
**Key talking points they're fishing for:**
- **Imports needed:** `java.net.HttpURLConnection`, `java.net.URL`, and a concurrency utility (`ExecutorService`/`Executors` from `java.util.concurrent`) since 1000 sequential calls would be far too slow and unrealistic.
- **Loop:** a simple `for` loop bounded exactly at 1000 (`i < 1000`) — explicitly state "not exceeding 1000" ties to using a **fixed bound**, not a while(true).
- **URL definition:** the endpoint should be defined explicitly (hardcoded string or read from a config/properties file) — say clearly *"I'd define it as a constant or pull it from an external config so it's not hardcoded per environment."*
- **Concurrency control:** use a thread pool (`ExecutorService`) rather than spawning 1000 raw threads, to control load and avoid resource exhaustion — this is the "smart" answer they're listening for.
- **Ramp-up = 10 sec for 1000 users:** if ramp-up is very short (10s) for 1000 users, that means ~100 users/sec arrival rate — a sharp, near-simultaneous ramp. This can cause a **spike/thundering-herd effect**: connection pool exhaustion, CPU spike, possible false-positive errors that aren't representative of real gradual user arrival. Correct answer: *"A 10-second ramp-up for 1000 users is very aggressive — it behaves more like a spike test than a load test, and could produce misleading bottleneck signals not reflective of real-world gradual arrival."*

---

## 16. Core Java Concepts — Quick-Fire Answers

- **Array vs ArrayList:** Array is fixed-size, can hold primitives or objects, faster (less overhead). ArrayList is resizable (backed by a dynamic array internally), part of Collections framework, only holds objects (autoboxes primitives), has built-in methods (add/remove/sort).
- **Collection vs Collections:** `Collection` is an **interface** (root of the collections hierarchy — List, Set, Queue extend it). `Collections` is a **utility class** with static helper methods (`Collections.sort()`, `Collections.reverse()`, `Collections.unmodifiableList()`, etc.).
- **List vs Set vs Map:** `List` — ordered, allows duplicates, index-based access (ArrayList, LinkedList). `Set` — no duplicates, no guaranteed order (HashSet) or sorted (TreeSet) or insertion-order (LinkedHashSet). `Map` — key-value pairs, keys unique (HashMap, TreeMap, LinkedHashMap).
- **Method Overloading vs Overriding:** Overloading = same method name, different parameter list, **same class**, resolved at **compile time** (compile-time/static polymorphism). Overriding = subclass redefines a superclass method with the **same signature**, resolved at **runtime** (runtime/dynamic polymorphism), enables polymorphic behavior.
- **Iterator vs ListIterator:** `Iterator` — traverses any Collection, **forward-only**, supports `remove()`. `ListIterator` — only for `List`, traverses **both directions** (forward/backward), supports `add()`, `set()`, and `remove()`, and gives index positions.
- **"Are you sure normal Iterator can't remove?"** — Correction to have ready: a plain `Iterator` **can** remove — via `iterator.remove()` — that's actually the **safe** way to remove while iterating (avoids `ConcurrentModificationException`). What it **cannot** do is `add()` — that requires `ListIterator`. So the accurate statement is: *"Iterator supports safe removal via `remove()`, but not addition — for insertion during traversal you need ListIterator."*
- **Eliminate duplicates from a list (live coding):**
```java
List<Integer> withDuplicates = Arrays.asList(80, 85, 85, 90, 92, 90, 100);
List<Integer> unique = new ArrayList<>(new LinkedHashSet<>(withDuplicates)); // preserves order
System.out.println(unique);
// Or simply: Set<Integer> uniqueSet = new HashSet<>(withDuplicates);
```
- **Encapsulation:** Bundling data (fields) and the methods that operate on it into a single unit (class), while restricting direct access to internal state — typically via `private` fields with `public` getters/setters. Purpose: control/validate how data is accessed or modified, and hide implementation detail from the outside world.

---

## 17. Dynatrace & JVM Performance Analysis

- **Why Dynatrace helps in performance testing:** it gives automatic, code-level, end-to-end transaction tracing (PurePath) without manual instrumentation — so instead of just knowing "response time degraded," you can see exactly which service/method/DB call in the chain caused it. It also auto-detects topology (Smartscape) and baselines normal behavior, flagging anomalies automatically (AI-based root cause, Davis AI).
- **How you analyze JVM performance / what tools:**
  - **Dynatrace** — heap usage, GC suspension time/frequency, thread pool utilization, deadlocks, out-of-memory patterns, directly correlated to a specific load test time window.
  - Also mention: **JVM built-in tools** as a fallback/cross-check — `jstat` (GC stats), `jstack` (thread dump for deadlock/blocking analysis), `jmap`/heap dump analysis (memory leak investigation via Eclipse MAT), and **VisualVM** for a lighter-weight local view.
  - **What you look for:** rising old-gen heap usage that doesn't drop after GC (memory leak signature), long/frequent GC pauses correlating with response time spikes, thread pool saturation (all worker threads busy → requests queueing), and CPU vs GC time ratio.

---

## 18. Closing / "What's your expectation of the role" type question

If asked (e.g., "is this purely JMeter + basic Java or is it more coding-heavy?"), a good response:
> "I'm comfortable either way — I have solid working knowledge of Java for scripting logic, correlation, and custom samplers, and I'm equally comfortable with SQL. I see performance engineering roles increasingly requiring that blend — being able to read/adapt code, not just record-and-playback — so I'd welcome a role where I get to build on that, whether that's deeper Java-based test framework work or CI/CD-integrated performance engineering."

---

## Quick Reminders Before the Panel
- Have a **live online compiler** (e.g., programiz.com, replit) tab open — they explicitly asked you to code live.
- Keep SQL/Java answers **concise and correct** over clever — they're checking fundamentals, not cleverness.
- For every "have you done X" question, answer **honestly** but bridge to adjacent real experience if you haven't done it exactly (e.g., Jenkins integration).
- Reuse the **symptom → tool → root cause → fix** structure for every "issue you found" question — it signals seniority.













-------------------------------------------------------- 


# Interview Prep — Hierarchy / Mental-Model Format
### Goal: don't memorize sentences — memorize the TREE. Walk down the branches live and the answer builds itself.

---

## 1. SELF-INTRODUCTION

```
WHO AM I
├── Role: Performance Test Lead, 10+ yrs QA / 8+ yrs Performance
├── Last engagement: TCS → CelcomDigi (Malaysia) — Telecom OSS/BSS
│   └── Domain: Order Management → Provisioning → Billing
├── Tools (2 buckets)
│   ├── Execution: LoadRunner (VuGen/Controller/Analysis), JMeter
│   └── Observability: Dynatrace, Datadog, ELK/Kibana
├── Extra depth: Kafka (queue perf, consumer lag, offsets)
├── What I own end-to-end
│   NFR → Strategy → Scenario design → Execution → APM analysis → Defect triage
│   └── Proof point: 15+ Sev1/2 defects recovered
├── Certifications: ISTQB Perf, Dynatrace Associate, JMeter Certified
└── Current status: Notice period, LWD 28-Aug, open to relocate (India)
```
**How to speak it:** walk top to bottom, one branch = one sentence.

---

## 2. NFR vs PERFORMANCE TESTING

```
SYSTEM LIFECYCLE
├── STAGE 1: NFR (upstream / design phase)
│   ├── Who's in the room: Architects, Business, Infra
│   ├── Defines: TPS, response-time SLA, concurrency, availability%, RTO/RPO
│   └── Output: signed-off NFR document (the TARGET)
│
└── STAGE 2: Performance Testing (downstream / validation phase)
    ├── Input: the NFR document
    ├── Action: design load/stress/endurance/spike/resiliency scenarios
    ├── Tools: JMeter/LoadRunner (execution) + Dynatrace/Datadog (proof)
    └── Output: evidence — did the system meet the target? (the PROOF)
```
**One-liner anchor:** NFR = target. Performance testing = proof.

---

## 3. RESILIENCY TEST PLAN DOCUMENT

```
RESILIENCY TEST PLAN
├── 1. Scope — which app, which layers (web/app/mid-tier/DB/external)
├── 2. Failure scenarios in-scope — crash, latency, pool exhaustion, timeout, disk full
├── 3. Entry/Exit criteria — when to start / when "done"
├── 4. Topology — architecture diagram, LB/clustering setup
├── 5. Steady-state baseline — "normal" metrics BEFORE injecting anything
├── 6. Injection method/tooling — manual kill, Chaos Monkey, Gremlin, Istio fault
├── 7. Monitoring plan — which dashboards, which metrics (error%, latency, retries)
├── 8. Recovery criteria — e.g. "failover <30s, <1% txn loss"
├── 9. Rollback plan
├── 10. Roles + risk + schedule
└── 11. Defect log + sign-off
```
**Memory trick:** it's just a normal test plan (scope→criteria→execution→reporting) with one extra idea injected: **steady-state baseline before you break anything.**

---

## 4. LAYERED RESILIENCY TESTING (Web → App → Mid-tier → DB)

```
REQUEST PATH = TEST DESIGN PATH (test one layer at a time, then combine)
│
├── LAYER 1: Web / Load Balancer
│   ├── Inject: kill one node under load
│   └── Verify: LB health-check reroutes, no session drop
│
├── LAYER 2: App Server
│   ├── Inject: CPU/memory saturation, thread pool exhaustion
│   └── Verify: graceful queueing/timeout, auto-scale trigger
│
├── LAYER 3: Mid-tier (MQ / Kafka / API integrations)
│   ├── Inject: kill Kafka broker / API timeout
│   └── Verify: rebalancing, no msg loss, circuit breaker trips + fallback
│
├── LAYER 4: Database
│   ├── Inject: connection pool exhaustion, kill replica
│   └── Verify: retry logic, failover, timeouts honored (no infinite hang)
│
└── CROSS-CUTTING: Dynatrace PurePath at every layer
    → pinpoints exactly WHERE failure surfaces (blast radius isolation)

SEQUENCE RULE: isolate each layer first (clean signal) → THEN combine
              (e.g. app degradation + DB latency together) for cascading realism
```

---

## 5. EXTERNAL SYSTEM RESILIENCY (Payment Gateway example)

```
PROBLEM: I don't own the external system → can't inject failure directly

SOLUTION TREE
├── Step 1: Review provider SLA (uptime, timeout, retry policy) → sets expectation
├── Step 2: Simulate failure AT MY BOUNDARY (not theirs)
│   └── Tool: WireMock / SoapUI mock / provider sandbox
│   └── Simulate: timeout, 5xx, malformed response, connection refused
├── Step 3: Validate MY application's reaction
│   ├── Does it timeout within configured limit?
│   ├── Does it retry — and is retry IDEMPOTENT (no double-charge)?
│   ├── Does it show proper fallback, not a blank/crashed screen?
│   └── On return-callback delay/loss — does state reconcile correctly?
├── Step 4 (if possible): joint fault-injection in provider's UAT/sandbox
└── Step 5: End-to-end APM trace on my side (measure wait time + reaction)

REFRAME: "Can I break it" → "Can I prove MY system degrades gracefully"
```

---

## 6. FAILURE INJECTIONS CONDUCTED (just a checklist to recite)

```
INFRA LAYER        → node/process kill, CPU/memory saturation, disk exhaustion
NETWORK LAYER       → latency injection, packet loss
APPLICATION LAYER   → service timeout (mocked slow dependency)
DATA LAYER          → DB connection pool exhaustion
MESSAGING LAYER     → Kafka broker kill (rebalance check), consumer lag injection
EXTERNAL LAYER      → mocked 5xx / connection refused from dependency
(CLOUD-NATIVE, if asked) → pod kill / node drain → auto-heal check
```

---

## 7. ISSUES IDENTIFIED — use ONE repeatable pattern

```
PATTERN (use for every "issue" question, just swap the details):
SYMPTOM → TOOL USED TO ISOLATE → ROOT CAUSE → FIX/RECOMMENDATION

Example 1:
  Pool exhaustion under load → Dynatrace trace → connection not closed in
  exception path → fixed connection handling

Example 2:
  Cascading failure under downstream timeout → traced retry pattern →
  synchronous retries with no backoff → added exponential backoff + circuit breaker

Example 3:
  Kafka lag growing during spike → consumer lag metric → parallelism not
  scaled with partitions → scaled consumer group
```

---

## 8. LOAD / STRESS / ENDURANCE / SPIKE

```
AXIS 1: HOW MUCH load?         AXIS 2: HOW LONG?        AXIS 3: HOW SUDDEN?
│
├── LOAD TEST     = expected load, normal duration   → validates SLA compliance
├── STRESS TEST   = load PUSHED beyond expected       → finds the breaking point
├── ENDURANCE     = expected load, LONG duration (8-24h+) → finds leaks over time
└── SPIKE TEST    = sudden sharp burst, brief hold     → tests recovery after burst
```

**1000-user load test — walk the flow, not a script:**
```
Confirm NFR target (users, SLA, error%)
   ↓
Design realistic transaction mix + think-time (from prod pattern)
   ↓
Ramp-up gradually (not all-at-once) — mimic real arrival
   ↓
Run + correlate with APM live (CPU/mem/GC/DB)
   ↓
Compare vs NFR → report p90/p95/p99 (never just average)
   ↓
If breached → drill into APM trace for bottleneck
```

---

## 9. JMETER ARCHITECTURE

```
TEST PLAN (root)
└── THREAD GROUP (defines: users, ramp-up, loop count)
    ├── CONFIG ELEMENTS (shared config: defaults, CSV data, headers)
    ├── SAMPLERS (the actual request: HTTP/JDBC)
    │   ├── PRE-PROCESSOR (prep data before request)
    │   ├── (request fires)
    │   ├── POST-PROCESSOR (extract data: Regex/JSON Extractor)
    │   └── ASSERTIONS (validate response)
    ├── LOGIC CONTROLLERS (If/While/Loop/Module/Transaction — control flow)
    ├── TIMERS (Constant/Uniform Random — pacing)
    └── LISTENERS (results: Summary/Aggregate for real runs, avoid heavy ones)

ENGINE MODEL: JMeter = Java-based. Each thread = 1 independent virtual user,
runs the sampler tree top-down, per iteration.
```

---

## 10. SIMULATING 7000 USERS (Distributed Testing)

```
WHY: 1 machine can't sustain 1000s of heavy users (JVM/OS limits)

STEP 1: Capacity-test ONE machine first → find its safe max threads
STEP 2: Set up Master–Worker topology
   ├── MASTER: orchestrates (jmeter.properties → remote_hosts = worker IPs)
   └── WORKERS: run jmeter-server, headless/non-GUI (-n)
STEP 3: Split 7000 across N workers (e.g. 7 × 1000)
STEP 4: Run non-GUI + remote flag → aggregate at master
   (better: push to InfluxDB+Grafana instead of local files)
STEP 5: Watch for:
   ├── NTP time-sync across machines (correlation accuracy)
   ├── JVM heap tuning per worker (-Xms/-Xmx)
   ├── Disable heavy listeners on workers
   └── Confirm workers themselves aren't the bottleneck
STEP 6 (bonus): containerize workers (Docker/K8s) for elastic scaling
```

---

## 11. IF CONTROLLER vs WHILE CONTROLLER

```
IF CONTROLLER          →  ONE-TIME branch, evaluated once per iteration
                           e.g. "only call this endpoint IF userType == premium"

WHILE CONTROLLER        →  REPEATED, re-evaluated every loop
                           use case: POLLING for a state change

REPORT-GENERATION EXAMPLE (their canned scenario) — walk this flow:
  Submit "generate report" request
     ↓
  While ( status != "Completed" ):
     ├── Send "check status" request
     ├── Post-processor extracts status → variable (Regex/JSON Extractor)
     ├── Short Timer (avoid hammering server)
     └── pending/in-progress → loop again | completed → exit loop
     ↓
  Proceed to next transaction
  (add max-iteration safety counter to prevent infinite loop)
```

---

## 12–13. MODULE CONTROLLER + setUp/tearDown THREAD GROUP

```
MODULE CONTROLLER + TEST FRAGMENT
  Fragment defined ONCE (outside main tree) → referenced via Module Controller
  Use case: reusable flows (login, token refresh)
  Benefit: one place to update → maintainability

setUp THREAD GROUP          tearDown THREAD GROUP
  runs BEFORE main groups  →  runs AFTER main groups finish
  use: token gen, data seed →  use: cleanup, invalidate sessions/data
```

---

## 14. JENKINS INTEGRATION

```
EXECUTION MODE
  Standard practice = non-GUI mode (-n -t plan.jmx -l results.jtl)
     ↓
PIPELINE STAGE
  Triggered as Jenkins stage, parameterized (threads/duration/env)
     ↓
RESULTS
  JTL → Performance plugin / HTML dashboard → trend comparison across builds
     ↓
GATE
  Threshold-based build failure (error% > X, or p95 > SLA) → auto-gate release
```
*(If not done end-to-end: keep the tree but stop before "GATE" — say "primarily non-GUI execution, working knowledge of the pipeline/gating piece.")*

---

## 15. SQL & JAVA COMFORT

```
SQL — 3 core statements, no hesitation:
  SELECT * FROM orders WHERE status = 'PENDING';
  UPDATE orders SET status = 'COMPLETED' WHERE order_id = 1001;
  INSERT INTO orders (order_id, customer_id, status) VALUES (1002, 501, 'NEW');

JAVA LOGIC #1 — Random 80-100, print min→max
  Generate 10 randoms in range → store in List → sort → print first/last
  [Random → List<Integer> → Collections.sort() → get(0) / get(size-1)]

JAVA LOGIC #2 — Simulate 1000 API calls
  IMPORTS: java.net.HttpURLConnection, java.net.URL, java.util.concurrent.*
     ↓
  LOOP: for (i < 1000)  — fixed bound, NOT while(true)
     ↓
  CONCURRENCY: ExecutorService fixed thread pool (NOT 1000 raw threads)
     ↓
  URL: defined explicitly — constant or external config (not hardcoded)
     ↓
  RAMP-UP TRAP: 10 sec ramp for 1000 users = ~100 users/sec = near-simultaneous
     → behaves like a SPIKE test, not load test → risk of thundering-herd,
       false-positive errors from unrealistic ramp shock
```

---

## 16. CORE JAVA — PAIRED CONTRASTS (memorize as pairs, not definitions)

```
Array           vs  ArrayList
fixed size          resizable (dynamic array inside)
primitive+object     objects only (autobox)
faster, less overhead  Collections framework, built-in methods

Collection      vs  Collections
INTERFACE (root of List/Set/Queue)   UTILITY CLASS (static helpers: sort/reverse)

List            vs  Set             vs  Map
ordered, dup OK      no dups            key-value, unique keys
index-based          Hash/Tree/Linked   Hash/Tree/LinkedHashMap

Overloading     vs  Overriding
same class            subclass redefines
diff parameters       same signature
compile-time (static) runtime (dynamic) polymorphism

Iterator        vs  ListIterator
any Collection        List only
forward-only           forward + backward
remove() only           add()/set()/remove() + index access

CORRECTION TRAP: "iterator can't remove?"
  → FALSE. iterator.remove() IS the safe way to remove while iterating
    (avoids ConcurrentModificationException)
  → What it CAN'T do is add() — that needs ListIterator

Duplicate removal (live code):
  new ArrayList<>(new LinkedHashSet<>(listWithDuplicates))  // order preserved

Encapsulation
  private fields + public getters/setters
  → controls/validates access, hides internal implementation
```

---

## 17. DYNATRACE & JVM PERFORMANCE ANALYSIS

```
WHY DYNATRACE HELPS
├── PurePath = automatic code-level, end-to-end trace (no manual instrumentation)
│   → pinpoints WHICH service/method/DB call caused the degradation
├── Smartscape = auto topology detection
└── Davis AI = automatic anomaly detection + root cause

JVM ANALYSIS
├── PRIMARY TOOL: Dynatrace
│   → heap usage, GC suspension time/frequency, thread pool util, deadlocks
├── FALLBACK/CROSS-CHECK: JVM native tools
│   ├── jstat  → GC stats
│   ├── jstack → thread dump (deadlock/blocking)
│   ├── jmap + Eclipse MAT → heap dump / memory leak analysis
│   └── VisualVM → lightweight local view
└── WHAT TO LOOK FOR
    ├── old-gen heap not dropping after GC → memory leak signature
    ├── long/frequent GC pauses ↔ response-time spikes (correlate!)
    └── thread pool saturation → requests queueing
```

---

## 18. CLOSING / ROLE-FIT QUESTION

```
STANCE: comfortable either way (JMeter+basic Java OR more coding-heavy)
REASON: solid working Java (scripting, correlation, custom samplers) + SQL
TREND: performance roles increasingly need "read/adapt code" not just record-playback
CLOSE: welcome the growth, whichever direction the role leans
```

---

## HOW TO USE THIS DOCUMENT
1. Don't read the sentences — **trace the tree with your finger/eyes** and speak each branch as one sentence.
2. Practice by covering the doc and redrawing each tree from memory on paper — if you can redraw it, you can explain it under pressure.
3. The **SYMPTOM → TOOL → ROOT CAUSE → FIX** pattern (Section 7) and the **layer-by-layer** pattern (Section 4) are reusable skeletons — almost any resiliency/issue question can be answered by dropping details into one of these two shapes.
4. Live coding: don't memorize the code — memorize the **step tree** (imports → loop → concurrency → data source), the code will follow naturally as you narrate it.



-----------------------------------------------------------------

# Complete Hierarchy Coverage — Every Question, In Transcript Order
### Including the repeated Java basics block — explained fully both times it appears

---

# PANEL ROUND 1 (Krishnamoorthy, Chelladurai)

### Q1. Introduce yourself
```
WHO AM I
├── Role: Performance Test Lead, 10+ yrs QA / 8+ yrs Performance
├── Last engagement: TCS → CelcomDigi (Malaysia) — Telecom OSS/BSS
│   └── Domain: Order Management → Provisioning → Billing
├── Tools (2 buckets)
│   ├── Execution: LoadRunner (VuGen/Controller/Analysis), JMeter
│   └── Observability: Dynatrace, Datadog, ELK/Kibana
├── Extra depth: Kafka (queue perf, consumer lag, offsets)
├── Ownership: NFR → Strategy → Scenario design → Execution → APM → Defect triage
│   └── Proof point: 15+ Sev1/2 defects recovered
├── Certifications: ISTQB Perf, Dynatrace Associate, JMeter Certified
└── Status: Notice period, LWD 28-Aug, open to relocate (India)
```

### Q2. NFR vs Performance testing — how do they differ?
```
SYSTEM LIFECYCLE
├── STAGE 1: NFR (upstream/design)
│   ├── Who: Architects, Business, Infra
│   ├── Defines: TPS, response-time SLA, concurrency, availability%, RTO/RPO
│   └── Output: signed-off NFR doc = the TARGET
└── STAGE 2: Performance Testing (downstream/validation)
    ├── Input: the NFR
    ├── Action: design load/stress/endurance/spike/resiliency scenarios
    ├── Tools: JMeter/LoadRunner + Dynatrace/Datadog
    └── Output: evidence = the PROOF
```
**Anchor line:** NFR = target, Performance testing = proof.

### Q3. Have you worked on a resiliency test plan document?
```
ANSWER: Yes.
CONTEXT: authored/contributed to resiliency & failure-injection test plans
         as part of NFR sign-off for critical provisioning/billing flows (CelcomDigi)
```

### Q4. What information does that plan include?
```
RESILIENCY TEST PLAN CONTENTS
├── 1. Scope — app + layers in scope (web/app/mid-tier/DB/external)
├── 2. Failure scenarios in-scope — crash, latency, pool exhaustion, timeout
├── 3. Entry/Exit criteria
├── 4. Topology diagram — LB/clustering setup
├── 5. Steady-state baseline — "normal" metrics BEFORE injection
├── 6. Injection method/tooling — manual kill / Chaos Monkey / Gremlin / Istio
├── 7. Monitoring plan — dashboards + metrics (error%, latency, retries)
├── 8. Recovery criteria — e.g. "failover <30s, <1% txn loss"
├── 9. Rollback plan
├── 10. Roles + risk + schedule
└── 11. Defect log + sign-off
```

### Q5. New application, client wants layer-by-layer failure injection — how do you test it?
```
APPROACH
├── Map the full request path first (web→app→mid-tier→DB→external)
├── Design scenarios LAYER BY LAYER, not all at once
│   → gets a clean signal on where failure surfaces
├── At each layer: baseline steady-state → inject ONE failure → observe
├── Correlate every injection with APM (Dynatrace/Datadog) to trace propagation
└── ONLY AFTER each layer is proven resilient individually →
    combine failures (e.g. app degradation + DB latency together)
    to simulate realistic cascading scenarios
```

### Q6. Web server, app server, mid-tier, DB — how do you design scenarios across all layers?
```
LAYER-BY-LAYER DESIGN
├── WEB / LOAD BALANCER
│   ├── Inject: kill one node under load
│   └── Verify: LB health-check reroutes, no session drop, sticky-session behavior
├── APP SERVER
│   ├── Inject: CPU/memory saturation, thread pool exhaustion
│   └── Verify: graceful queueing/timeout, auto-scale trigger if cloud-native
├── MID-TIER (MQ/Kafka/APIs)
│   ├── Inject: kill Kafka broker / inject latency-timeout on API call
│   └── Verify: rebalancing + no msg loss / circuit breaker trips + fallback
├── DATABASE
│   ├── Inject: connection pool exhaustion, kill replica, query latency
│   └── Verify: retry logic, read-replica failover, timeouts honored (no hang)
└── CROSS-CUTTING: Dynatrace PurePath at every layer
    → isolates blast radius, pinpoints exact propagation path
```

### Q7 & Q8. External system (payment gateway redirect) — how do you verify it's resilient?
```
CHALLENGE: no injection access to third-party system
SOLUTION
├── Step 1: Review provider SLA (uptime, timeout, retry policy)
├── Step 2: Simulate failure AT MY BOUNDARY
│   └── Tool: WireMock / SoapUI mock / provider sandbox
│   └── Simulate: timeout, 5xx, malformed response, connection refused
├── Step 3: Validate MY app's reaction
│   ├── Timeout honored?
│   ├── Retry idempotent (no double-charge)?
│   ├── Proper fallback, not blank/crashed screen?
│   └── Delayed/lost callback → does state reconcile correctly?
├── Step 4 (if possible): joint fault-injection in provider's sandbox
└── Step 5: End-to-end APM trace on my side (measure wait + reaction)

REFRAME: "can I break it" → "can I prove MY system degrades gracefully"
```

### Q9. What failure injections have you conducted?
```
INFRA        → node/process kill, CPU/memory saturation, disk exhaustion
NETWORK      → latency injection, packet loss
APPLICATION  → service timeout (mocked slow dependency)
DATA         → DB connection pool exhaustion
MESSAGING    → Kafka broker kill (rebalance check), consumer lag injection
EXTERNAL     → mocked 5xx / connection refused
```

### Q10. Issues you've identified for your application?
```
PATTERN: SYMPTOM → TOOL → ROOT CAUSE → FIX
├── Pool exhaustion under load → Dynatrace trace → connection not closed
│   in exception path → fixed connection handling
├── Cascading failure → traced retry pattern → sync retries, no backoff
│   → added exponential backoff + circuit breaker
└── Kafka lag growing during spike → consumer lag metric → parallelism
    not scaled with partitions → scaled consumer group
```

### Q11. How comfortable are you with JMeter?
```
ANSWER: Very comfortable — one of my 2 primary tools alongside LoadRunner
USE: distributed load testing, complex scripting/correlation, controllers,
     production performance validation test plans
```

### Q12. Different types of performance testing conducted?
```
LOAD      → validate SLA at expected/normal concurrency
STRESS    → push beyond expected load → find breaking point
ENDURANCE → moderate load, long duration (8-24h+) → find leaks over time
SPIKE     → sudden sharp burst, brief hold → test recovery
PLUS: resiliency/failure-injection layered on top of these
```

### Q13 & Q14. If Controller vs While Controller (+ report generation example)
```
IF CONTROLLER    → ONE-TIME branch, evaluated once per iteration
                    e.g. "only call endpoint IF userType == premium"
WHILE CONTROLLER → REPEATED, re-evaluated every loop → used for POLLING

REPORT-GENERATION FLOW (their exact scenario):
  Submit "generate report" request
     ↓
  While (status != "Completed"):
     ├── Send "check status" request
     ├── Post-processor extracts status → variable (Regex/JSON Extractor)
     ├── Short Timer (avoid hammering server)
     └── pending/in-progress → loop again | completed → exit
     ↓
  Proceed to next transaction
  (+ max-iteration safety counter to prevent infinite loop)
```

### Q15. Have you used Module Controller / Test Fragment?
```
ANSWER: Yes
STRUCTURE: Fragment defined ONCE (outside main tree) → referenced via
           Module Controller wherever needed
USE CASE: reusable flows — login, token refresh
BENEFIT: one place to update → maintainability
```

### Q16. Have you used setUp / tearDown Thread Group?
```
setUp THREAD GROUP           tearDown THREAD GROUP
runs BEFORE main groups   →  runs AFTER main groups finish
use: token gen, data seed →  use: cleanup, invalidate sessions/data
```

### Q17. Integrated JMeter with Jenkins, or running non-GUI mode?
```
EXECUTION MODE: standard practice = non-GUI (-n -t plan.jmx -l results.jtl)
     ↓
PIPELINE STAGE: triggered in Jenkins, parameterized (threads/duration/env)
     ↓
RESULTS: JTL → Performance plugin / HTML dashboard → trend comparison
     ↓
GATE: threshold-based build failure (error% > X, p95 > SLA) → auto-gate
```

### Q18. Performance issues identified for your application?
```
(Reuse Q10 pattern, add a fresh example if asked again)
Session stickiness breaking on node failover → LB injection test →
  root cause: session affinity misconfig → fixed sticky-session handling
```

### Q19, Q20, Q21. SQL and Java comfort — for JMeter scripting / resiliency logic
```
ANSWER: Very comfortable with both
SQL USE: JDBC samplers, test data setup/validation
JAVA/GROOVY USE: JSR223 samplers/pre-post-processors — custom correlation,
                 dynamic data generation, conditional flow
RESILIENCY TIE-IN: status-polling logic, response validation scripting
```

### Q22. Write select, update, insert statements
```sql
SELECT * FROM orders WHERE status = 'PENDING';
UPDATE orders SET status = 'COMPLETED' WHERE order_id = 1001;
INSERT INTO orders (order_id, customer_id, status) VALUES (1002, 501, 'NEW');
```

### Q23. Java logic — 10 random numbers 80-100, print min to max
```
FLOW: Random → List<Integer> → Collections.sort() → get(0)/get(size-1)
```
```java
import java.util.*;
public class RandomRange {
    public static void main(String[] args) {
        Random rand = new Random();
        List<Integer> numbers = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            numbers.add(80 + rand.nextInt(21)); // 80-100 inclusive
        }
        Collections.sort(numbers);
        System.out.println("Min: " + numbers.get(0));
        System.out.println("Max: " + numbers.get(numbers.size() - 1));
    }
}
```

---

## JAVA BASICS BLOCK — FIRST OCCURRENCE (Q24–Q32)

### Q24. How good are you in Java — okay to ask basic questions?
```
ANSWER: "Sure, happy to go through them."
```

### Q25. Array vs ArrayList
```
Array           vs  ArrayList
fixed size          resizable (dynamic array inside)
primitive+object     objects only (autobox)
faster, less overhead  Collections framework, built-in methods (add/remove/sort)
```

### Q26. Collection vs Collections
```
Collection (interface)              Collections (utility class)
root of List/Set/Queue hierarchy    static helper methods
                                     (sort, reverse, unmodifiableList)
```

### Q27. List vs Set vs Map
```
List                 Set                     Map
ordered, dup OK       no duplicates           key-value pairs, unique keys
index-based access    Hash(no order)/         HashMap / TreeMap /
(ArrayList,           Tree(sorted)/           LinkedHashMap
 LinkedList)           LinkedHash(insertion)
```

### Q28. Method overloading vs overriding
```
Overloading                    Overriding
same class                     subclass redefines superclass method
different parameter list       same signature
compile-time (static poly)     runtime (dynamic poly)
```

### Q29. Iterator vs ListIterator — any idea?
```
Iterator              vs  ListIterator
any Collection             List only
forward-only                forward + backward
supports remove()            supports add()/set()/remove() + index position
```

### Q30. "That is something called list iterator and iterator" — same question restated
```
(Same answer as Q29 — restate to confirm understanding)
Iterator = universal, one-directional, remove-only.
ListIterator = List-specific, bidirectional, full CRUD (add/set/remove) + index.
```

### Q31. "Are you sure a normal Iterator can't remove?" — correction trap
```
CORRECTION: FALSE — Iterator CAN remove, via iterator.remove()
            → this is actually the SAFE way to remove while iterating
              (avoids ConcurrentModificationException)
WHAT IT CAN'T DO: add() — that specifically requires ListIterator
```

### Q32. Open an online compiler — eliminate duplicates
```
FLOW: List with duplicates → wrap in LinkedHashSet (dedupes, keeps order)
      → wrap back in ArrayList
```
```java
List<Integer> withDuplicates = Arrays.asList(80, 85, 85, 90, 92, 90, 100);
List<Integer> unique = new ArrayList<>(new LinkedHashSet<>(withDuplicates));
System.out.println(unique);
```

---

## JAVA BASICS BLOCK — SECOND OCCURRENCE (repeated verbatim in transcript)
### Explained again in full — same core answers, said fresh the second time

### Q24 (again). How good are you in Java — okay to ask basic questions?
```
ANSWER: "Yes, go ahead — happy to walk through them."
```
*(Say it slightly differently from the first time — panels notice canned repetition.)*

### Q25 (again). Array vs ArrayList
```
Core distinction to lead with: SIZE FLEXIBILITY.
Array = fixed length, set at creation, can hold primitives directly.
ArrayList = grows/shrinks dynamically, backed by an array internally,
            only holds objects (wraps primitives via autoboxing),
            gives you utility methods (add, remove, contains, sort) for free.
```

### Q26 (again). Collection vs Collections
```
Core distinction to lead with: INTERFACE vs CLASS.
Collection = the interface — the contract that List, Set, Queue implement.
Collections = a helper class full of static methods that OPERATE ON
              collection objects (sorting, reversing, making them read-only).
```

### Q27 (again). List vs Set vs Map
```
Core distinction to lead with: DUPLICATES + STRUCTURE.
List  = sequence, duplicates allowed, retrieve by index.
Set   = uniqueness enforced, no index, differs by implementation ordering
        (HashSet = no guarantee, LinkedHashSet = insertion order, TreeSet = sorted).
Map   = pairs, not single values — every entry is a key mapped to a value,
        keys must be unique, values can repeat.
```

### Q28 (again). Method overloading vs overriding
```
Core distinction to lead with: WHEN it's resolved.
Overloading = decided by the COMPILER, based on which parameter list matches —
              same method name, same class, different signature.
Overriding  = decided at RUNTIME, based on the actual object type —
              subclass provides its own version of a parent's method,
              same exact signature. This is what makes polymorphism work.
```

### Q29 (again). Iterator vs ListIterator
```
Core distinction to lead with: DIRECTION + SCOPE.
Iterator     = works on any Collection, only moves forward, minimal API.
ListIterator = only works on List, moves BOTH directions, and gives you
               the ability to modify the list during traversal
               (add, set, remove) plus tells you the current index.
```

### Q30 (again). List Iterator and Iterator — restated once more
```
Same concept restated for confirmation:
Think of Iterator as the "basic" reader — one-way, read/delete.
ListIterator is the "upgraded" version, only available for Lists —
two-way, and lets you insert/replace as you go, not just delete.
```

### Q31 (again). "Are you sure normal Iterator can't remove?"
```
Restate the correction firmly the second time, don't hedge:
Iterator DOES support removal — that's actually one of its main purposes,
to let you delete elements safely while looping, without triggering
a ConcurrentModificationException. The ONE thing it cannot do is insert
new elements mid-traversal — that capability is exclusive to ListIterator.
```

### Q32 (again). Open compiler — eliminate duplicates
```
Same logic, say it as a clean narration this time:
"I'll take the list, drop it into a LinkedHashSet — that automatically
removes duplicates while keeping the original order — then wrap it back
into an ArrayList if I need List-specific behavior afterward."
```
```java
List<Integer> nums = Arrays.asList(80, 85, 85, 90, 92, 90, 100);
List<Integer> deduped = new ArrayList<>(new LinkedHashSet<>(nums));
System.out.println(deduped); // [80, 85, 90, 92, 100]
```

---

# PANEL ROUND 2

### Q1. Glimpse of your past experience — performance, Java, etc.
```
(Condensed version of Round 1 Q1 — 20 seconds)
10+ yrs QA / 8+ perf → CelcomDigi telecom OSS/BSS via TCS →
LoadRunner + JMeter core → Dynatrace/Datadog APM → Kafka + SQL backend →
working Java for scripting → ISTQB/Dynatrace/JMeter certified
```

### Q2. Did you work on endurance and spike too?
```
ANSWER: Yes.
Load + Stress = most frequent
Endurance/Soak = extended duration runs → catch memory/connection leaks
Spike = sudden burst scenarios → e.g. campaign launches
```

### Q3. Differences between load, stress, endurance
```
LOAD      → expected/normal concurrency, validate against SLA
STRESS    → incrementally push PAST expected load → find the ceiling/failure mode
ENDURANCE → moderate load sustained 8-24h+ → watch TREND LINES not peak numbers
            (GC behavior, heap growth, thread count over time)
```

### Q4. Example load test — 1000 users, how would you handle it?
```
Confirm NFR target (users, SLA, error%)
   ↓
Design realistic transaction mix + think-time (from prod pattern)
   ↓
Ramp-up gradually (e.g. 100 users/30s) — mimic real arrival
   ↓
Run + correlate live with APM (CPU/mem/GC/DB query time)
   ↓
Compare vs NFR → report p90/p95/p99 (never just average) + error rate
   ↓
If breached → drill into APM trace for bottleneck
```

### Q5 & Q6. Worked with JMeter — explain the architecture
```
TEST PLAN (root)
└── THREAD GROUP (users, ramp-up, loop count)
    ├── CONFIG ELEMENTS (defaults, CSV data, headers)
    ├── SAMPLERS (HTTP/JDBC — the actual request)
    │   ├── PRE-PROCESSOR
    │   ├── (request fires)
    │   ├── POST-PROCESSOR (Regex/JSON Extractor)
    │   └── ASSERTIONS
    ├── LOGIC CONTROLLERS (If/While/Loop/Module/Transaction)
    ├── TIMERS (pacing)
    └── LISTENERS (Summary/Aggregate for real runs)

ENGINE: Java-based, each thread = 1 independent virtual user,
runs sampler tree top-down per iteration.
```

### Q7. Simulate 7000 users — distributed testing steps
```
STEP 1: Capacity-test ONE machine → find its safe max threads
STEP 2: Master–Worker topology
   ├── MASTER: remote_hosts config in jmeter.properties
   └── WORKERS: run jmeter-server, headless (-n)
STEP 3: Split 7000 across N workers (e.g. 7 × 1000)
STEP 4: Run non-GUI + remote flag → aggregate at master
   (better: InfluxDB+Grafana backend listener)
STEP 5: Watch: NTP time-sync, JVM heap tuning, disable heavy listeners
         on workers, confirm workers aren't themselves the bottleneck
STEP 6 (bonus): containerize workers (Docker/K8s) for elastic scaling
```

### Q8. Java program to simulate 1000 API calls — logic, loops, imports
```
IMPORTS: java.net.HttpURLConnection, java.net.URL, java.util.concurrent.*
LOOP: for (i < 1000) — fixed bound, not while(true)
CONCURRENCY: ExecutorService fixed thread pool (not 1000 raw threads)
```

### Q9. "You're requesting 4000 API calls" — HTTP URL connection import, loop within 1000
```
NOTE: panel shifted the number mid-question (4000 → but "not exceed 1000") —
      answer the LOGIC, not a fixed literal number:
IMPORT: java.net.HttpURLConnection (already imported per Q8)
LOOP: bound the for-loop to whatever the target count is (parameterize it,
      e.g. `int totalCalls = 1000;` as a variable, not hardcoded in the loop) —
      this way the same logic answers "1000" or "4000" cleanly.
URL AWARENESS: yes — each call needs a target endpoint, defined explicitly
      (see Q10).
```

### Q10. How would you know which URL to call — did you define it?
```
ANSWER: Yes — defined explicitly, either as a constant string or pulled
        from an external config/properties file, so it's not hardcoded
        per environment (dev/QA/prod endpoints differ).
```

```java
import java.net.HttpURLConnection;
import java.net.URL;
import java.util.concurrent.*;

public class ApiCallSimulator {
    public static void main(String[] args) throws Exception {
        String urlString = "https://api.example.com/endpoint"; // explicit/config-driven
        int totalCalls = 1000; // parameterized, reusable for any count
        ExecutorService executor = Executors.newFixedThreadPool(50);
        for (int i = 0; i < totalCalls; i++) {
            executor.submit(() -> {
                try {
                    URL url = new URL(urlString);
                    HttpURLConnection conn = (HttpURLConnection) url.openConnection();
                    conn.setRequestMethod("GET");
                    System.out.println("Response: " + conn.getResponseCode());
                } catch (Exception e) { e.printStackTrace(); }
            });
        }
        executor.shutdown();
        executor.awaitTermination(5, TimeUnit.MINUTES);
    }
}
```

### Q11. If ramp-up is 10 seconds for 1000 users, what will happen?
```
CALC: 1000 users / 10 sec ≈ 100 users/sec arrival rate = near-simultaneous
EFFECT: behaves like a SPIKE test, not a realistic load test
RISK: thundering-herd effect — connection pool exhaustion, sharp CPU spike,
      errors that are ARTIFACTS of unrealistic ramp, not real capacity issues
CONCLUSION: misleading bottleneck signal vs a gradual, production-realistic ramp
```

### Q12. What is encapsulation?
```
DEFINITION: bundling data (fields) + methods that operate on it, into
            one unit (class)
MECHANISM: private fields + public getters/setters
PURPOSE: control/validate how data is accessed or modified,
         hide internal implementation from the outside world
```

### Q13. Experience with Jenkins — integrated performance tests into it?
```
(Same tree as Round 1 Q17)
EXECUTION: non-GUI mode standard practice
     ↓
PIPELINE STAGE: parameterized (threads/duration/env)
     ↓
RESULTS: JTL → Performance plugin/HTML dashboard → trend comparison
     ↓
GATE: threshold-based build failure (error% / p95 SLA breach)
```

### Q14. Dynatrace — why is it helpful in performance testing?
```
PurePath  → automatic, code-level, end-to-end trace, no manual instrumentation
            → pinpoints WHICH service/method/DB call caused degradation
Smartscape → auto topology detection
Davis AI   → automatic anomaly detection + root cause
VALUE: cuts root-cause analysis time vs manually correlating logs/metrics
```

### Q15. How do you analyze JVM performance — what tools?
```
PRIMARY: Dynatrace → heap usage, GC suspension time/frequency,
         thread pool utilization, deadlocks, OOM patterns
FALLBACK/CROSS-CHECK: JVM native tools
  ├── jstat → GC stats
  ├── jstack → thread dump (deadlock/blocking)
  ├── jmap + Eclipse MAT → heap dump / memory leak analysis
  └── VisualVM → lightweight local view
WATCH FOR: old-gen heap not dropping after GC (leak signature),
           long/frequent GC pauses correlating with response-time spikes,
           thread pool saturation (queueing)
```

### Q16. Expectation for the role — JMeter with basic Java, or purely coding?
```
STANCE: comfortable either way
REASON: solid working Java (scripting, correlation, custom samplers) + SQL
TREND: performance roles increasingly need "read/adapt code" not just
       record-and-playback
CLOSE: welcome the growth, whichever direction the role leans
```

### Q17. "You'll have more challenging work that will enhance your capabilities."
```
NOT A QUESTION — acknowledge positively:
"That sounds great, I'm looking for exactly that kind of growth."
```

---

## USAGE NOTE
Every tree above maps to its exact question number in your transcript, in the exact order they appear — including the Java basics block explained fully **twice**, phrased slightly differently the second time so it doesn't sound rehearsed if the panel circles back. Trace the branches out loud; don't recite the sentences underneath them.
