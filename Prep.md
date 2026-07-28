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
