Here's the master one-page reference tying everything together — structure it in your notebook as a single flow diagram plus execution rules.

## JMETER TEST PLAN — FULL HIERARCHY

```
TEST PLAN
│
├── Non-Test Elements (setup tools, not part of execution)
│   └── HTTP(S) Test Script Recorder, HTTP Mirror Server, Property Display
│
├── User Defined Variables (Config) — constants for whole plan
│
├── setUp Thread Group (runs FIRST, once, before everything)
│   └── Pre-test setup: auth token gen, data seeding
│
├── Thread Group(s) — the main execution units (parallel/sequential)
│   │
│   ├── Config Elements (scope: this Thread Group & below)
│   │     HTTP Request Defaults, HTTP Header Manager, HTTP Cookie Manager,
│   │     CSV Data Set Config, JDBC Connection Config
│   │
│   ├── Timers (scope: this level & below)
│   │     Constant/Uniform/Gaussian Timer, Throughput Timer, Synchronizing Timer
│   │
│   ├── Pre-Processors (scope: this level & below)
│   │     JSR223 PreProcessor, User Parameters
│   │
│   ├── Logic Controllers (organize & control flow)
│   │   │
│   │   ├── Once Only Controller → Login (runs 1st iteration only)
│   │   │     └── HTTP Request: Login
│   │   │           ├── Post-Processor: Extract sessionToken
│   │   │           └── Assertion: Response Assertion (check "SUCCESS")
│   │   │
│   │   └── Transaction Controller: "Place Order"
│   │         └── If Controller (condition: stock available)
│   │               └── HTTP Request: Submit Order
│   │                     ├── Pre-Processor: JSR223 (add signature header)
│   │                     ├── Post-Processor: JSON Extractor (orderId)
│   │                     ├── Assertion: JSON Assertion (status = CONFIRMED)
│   │                     └── Assertion: Duration Assertion (<2000ms)
│   │
│   ├── Samplers (the actual requests — can sit directly or inside Controllers)
│   │     HTTP Request, JDBC Request, Kafka Sampler, JSR223 Sampler
│   │
│   └── Listeners (scope: this level & below — what gets recorded/shown)
│         Simple Data Writer (.jtl file) — ALWAYS ON for real runs
│         Backend Listener (Grafana/Datadog) — optional live dashboard
│         View Results Tree / Summary Report — DEBUG ONLY, disable for load runs
│
├── tearDown Thread Group (runs LAST, once, after everything)
│   └── Cleanup: delete test data, close sessions
│
└── WorkBench (optional scratch space, not saved with .jmx by default)
```

---

## EXECUTION ORDER RULES (write these down — commonly asked in interviews)

1. **setUp Thread Group → regular Thread Groups (parallel) → tearDown Thread Group**
2. Within a Thread Group, elements execute **top-to-bottom**, but scope-based elements (Config, Timer, Pre/Post-Processor, Assertion, Listener) apply to **everything below them at their level**, regardless of visual position
3. **Execution priority at the same scope level**: Config Elements → Pre-Processors → Timers → **Sampler runs** → Post-Processors → Assertions → Listeners
4. Controllers change **flow/order**, not the above priority — they just decide *which* samplers run and *how many times*

---

## SCOPE RULE (the one rule that governs Timers, Assertions, Pre/Post-Processors, Config, Listeners)

> Placed **inside** a sampler → applies to that sampler only.
> Placed **at Thread Group level** (sibling to samplers, not nested inside one) → applies to **every sampler below it in that group**.

---

## QUICK CATEGORY CHEAT-SHEET

| Category | Purpose | Sends traffic? |
|---|---|---|
| Thread Group | Defines users/load pattern | No |
| Config Element | Sets defaults/variables/connections | No |
| Timer | Adds delay/pacing | No |
| Pre-Processor | Prepares data before request | No |
| **Sampler** | **Sends the actual request** | **Yes** |
| Post-Processor | Extracts data from response | No |
| Assertion | Validates response correctness | No |
| Logic Controller | Controls flow/order/conditions | No |
| Listener | Displays/records results | No |
| Non-Test Element | Setup tools (recorder, mirror) | No (recorder captures, doesn't test) |

---

## YOUR REAL-WORLD "GO-TO STACK" (based on everything covered)

- **Thread Group**: Concurrency Thread Group + Throughput Shaping Timer (TPS-driven SLA tests)
- **Config**: HTTP Request Defaults, Header Manager, Cookie Manager, CSV Data Set Config
- **Timer**: Precise Throughput Timer (steady TPS), Gaussian Random Timer (think-time)
- **Pre-Processor**: JSR223 PreProcessor (dynamic headers/signatures)
- **Sampler**: HTTP Request (primary), JDBC Request (DB checks), Kafka Sampler (queue testing)
- **Post-Processor**: JSON Extractor, Regular Expression Extractor
- **Assertion**: Response Assertion, JSON Assertion, Duration Assertion
- **Controller**: Transaction Controller (reporting), Once Only Controller (login pattern), If Controller (conditional retry)
- **Listener**: Simple Data Writer (.jtl) + Backend Listener (Grafana/Datadog) for real runs

This single sheet, plus the individual tables from each earlier topic, should give you a complete manual reference — from "what exists" down to "sample config values" for every element category in JMeter.

Want a **glossary page** next — one-line definitions of terms like correlation, ramp-up, think-time, pacing, percentile, throughput — the vocabulary interviewers often test separately from the tool mechanics itself?
