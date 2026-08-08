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


Here's the consolidated, advanced-level master hierarchy — everything covered so far tied together with the deeper execution-engine mechanics that senior interviews actually probe.

---

# JMETER COMPLETE ELEMENT HIERARCHY — ADVANCED MODEL

## 1. TOP-LEVEL STRUCTURE

```
TEST PLAN
│
├── Test Plan level properties
│     - "Run Thread Groups consecutively" (sequential vs parallel TG execution)
│     - "Functional Test Mode" (NEVER enable in load tests — logs full response, kills throughput)
│     - Classpath (custom jars — JDBC drivers, Kafka client libs)
│     - User Defined Variables (Test Plan scope — visible everywhere)
│
├── Non-Test Elements (WorkBench-adjacent, setup-only)
│     HTTP(S) Test Script Recorder, HTTP Mirror Server, Property Display, Sample
│
├── setUp Thread Group
├── Thread Group(s) [1..N] — can run sequentially or parallel
├── tearDown Thread Group
```

---

## 2. THREAD GROUP LAYER (execution containers)

```
Thread Group Types:
├── Standard Thread Group          → fixed threads/ramp-up/loop
├── setUp / tearDown Thread Group  → lifecycle hooks
└── jpgc Custom Thread Groups
    ├── Stepping Thread Group      → gradual step load
    ├── Ultimate Thread Group      → arbitrary load shape table
    ├── Arrivals Thread Group      → arrival-rate driven (open model)
    ├── Free-Form Arrivals TG      → irregular arrival pattern
    └── Concurrency Thread Group   → target concurrency + Throughput Shaping Timer pairing
```
Here are concrete config examples for each — values you can literally set in the GUI, plus a telco use case for each.

---

## 1. Stepping Thread Group

**Config:**
- This Group Will Start = `100` threads
- First, Wait for = `10` sec
- Then Start = `10` threads every `30` sec
- Startup time = `10` sec (time to start each step's batch)
- Hold load for = `120` sec (steady time before next step)
- Then Stop = `10` threads every `20` sec
- Thread Lifetime — Startup, then hold `600` sec total, Shutdown `10` threads every `20` sec

**Meaning:** Starts small, adds 10 users every 30 seconds until reaching 100, holds, then winds down the same way.

**Telco use case:** Finding the breaking point of the **Order Provisioning API** — start at 10 concurrent orders, step up by 10 every 30s, watch response times/error rate at each step until SLA breaches (e.g., response time crosses 2s) → gives you the exact capacity ceiling before go-live.

---

## 2. Ultimate Thread Group

**Config (table rows):**

| Start Threads | Initial Delay (s) | Startup Time (s) | Hold Load (s) | Shutdown Time (s) |
|---|---|---|---|---|
| 50 | 0 | 60 | 300 | 60 |
| 150 | 360 | 120 | 600 | 60 |
| 30 | 1140 | 30 | 300 | 30 |

**Meaning:** Row 1 ramps to 50 users over 60s, holds 5 min. Row 2 (starting at the 360s mark) ramps to an additional 150 users, holds 10 min (this is your peak/spike). Row 3 tapers down to a light 30-user tail.

**Telco use case:** Simulating a **promotional recharge campaign launch** — normal baseline traffic (50 users), then a sudden marketing SMS blast drives a spike to 150 concurrent users hitting the Recharge API, then traffic settles to a long tail as the promo winds down. Custom, non-linear shape — exactly what Ultimate TG is built for.

---

## 3. Arrivals Thread Group

**Config:**
- Target Rate = `200` arrivals per `minute`
- Ramp-up time = `60` sec
- Steady state = `600` sec
- Ramp-down time = `60` sec
- Processing threads: engine auto-calculates based on response time (open model — doesn't need manual thread math)

**Meaning:** You specify "I want 200 new virtual users arriving per minute" — JMeter figures out how many threads it needs to sustain that arrival rate, regardless of how slow the backend responds.

**Telco use case:** Modeling **CDR event arrival rate from the network** — real CDRs don't wait for JMeter's previous "thread" to finish before the next one exists; they arrive independently based on actual call/SMS/data activity. Arrivals TG matches this real-world open-system behavior far better than a closed Standard Thread Group would.

---

## 4. Free-Form Arrivals Thread Group

**Config (table, irregular arrival pattern):**

| Start RPS | End RPS | Duration (s) |
|---|---|---|
| 10 | 10 | 300 (quiet night traffic) |
| 10 | 500 | 60 (sudden morning surge) |
| 500 | 500 | 1800 (busy hour) |
| 500 | 50 | 300 (drop after lunch) |
| 50 | 400 | 120 (evening spike — promo notification burst) |
| 400 | 20 | 600 (night taper) |

**Telco use case:** Replaying a **realistic 24-hour subscriber traffic curve** captured from actual production logs (low overnight, morning commute spike, midday steady, evening promo burst, late-night taper) — irregular, not a clean linear ramp, which is exactly what Free-Form Arrivals is designed to replicate versus the standard Arrivals TG's simpler consistent ramp.

---

## 5. Concurrency Thread Group (+ Throughput Shaping Timer)

**Concurrency Thread Group config:**
- Target Concurrency = `300`
- Ramp-up Time = `120` sec
- Ramp-up Steps Count = `10`
- Hold Target Rate Time = `1800` sec

**Paired Throughput Shaping Timer config (graph):**

| Time (s) | Target TPS |
|---|---|
| 0 | 50 |
| 300 | 1000 |
| 1800 | 1000 |
| 2100 | 100 |

**Meaning:** Concurrency TG supplies enough threads (up to 300) to physically be capable of hitting the load; Throughput Shaping Timer is the one actually **dictating the real TPS target over time** — the two work together, TG provides capacity, Timer provides the precise pacing.

**Telco use case:** **Real-time charging (OCS) busy-hour test** — SLA requires sustaining exactly 1000 TPS during the 30-minute busy hour window, ramping up over 5 minutes and tapering after. This is the most accurate way to hit a *precise* throughput target (not just "however many threads happen to produce"), which matters enormously for revenue-critical charging systems where both under-testing and over-testing give misleading capacity numbers.

---

## Quick comparison table for your notebook

| Thread Group | Controls by | Best for |
|---|---|---|
| Stepping | Thread count, stepped | Finding breakpoint/capacity ceiling |
| Ultimate | Custom table (threads over time) | Arbitrary/custom shapes, spike simulation |
| Arrivals | Arrival rate (open model) | Matching real independent-arrival systems (CDRs) |
| Free-Form Arrivals | Irregular rate table | Replaying real, non-linear production traffic curves |
| Concurrency + Throughput Shaping Timer | Precise TPS over time | SLA-driven exact throughput targets (charging, billing) |

For your ExperienceCompany-style work, **Concurrency TG + Throughput Shaping Timer** is the one you'll want to be able to configure live/whiteboard in an interview — it's the most commonly asked "design a busy-hour load test" scenario at senior level.
**Advanced concept — Closed vs Open system models:**
- Standard/Concurrency TG = **closed model** (fixed thread pool, new request only after previous completes — thread "waits")
- Arrivals TG = **open model** (new virtual users arrive at a rate regardless of whether previous ones finished — mimics real internet traffic better)
- **Interview point:** "Closed models under-report the effect of slow responses because threads simply queue; open models expose true system saturation the way production traffic actually behaves." This distinction is a strong senior-level answer to "what's wrong with traditional load testing models?"

---

## 3. FULL SCOPE + EXECUTION ORDER (the part interviews probe hardest)

```
Within ANY scope (Thread Group, Controller, or Sampler level),
JMeter's internal execution order per iteration is:

1. Configuration Elements     (merge top-down, most specific wins on conflict)
2. Pre-Processors             (in tree order, top to bottom)
3. Timers                     (in tree order — if multiple, delays ADD UP)
4. SAMPLER EXECUTES            (the actual request/response)
5. Post-Processors             (in tree order)
6. Assertions                  (in tree order — ANY failure = sample marked failed)
7. Listeners                   (record/display result)

This cycle repeats for EVERY sampler.
Controllers (If/Loop/While/etc.) wrap around this cycle to control WHICH
samplers run and HOW MANY TIMES — they don't have their own position 
in the above sequence, they just gate access to it.
```

**Advanced scope inheritance rule:**
```
Element placed at:
  Thread Group level     → applies to ALL samplers in that Thread Group
  Controller level        → applies to all samplers nested inside that controller
  Sampler level (child)   → applies ONLY to that one sampler

Config Elements: MERGE (e.g., two Header Managers combine, don't override
                  unless same key exists — most specific/nested wins on conflict)
Timers:          ADD UP if multiple exist in scope (two 1000ms timers = 2000ms delay)
Assertions:      ALL must pass; any one failing fails the whole sample
Pre/Post-Proc:   Execute in tree ORDER (top to bottom), not by type grouping
```
**Interview point:** This exact mechanic — "config elements merge, timers add, assertions AND together, processors run in document order" — is a very commonly asked "explain JMeter's execution model" question at senior level.

---

## 4. FULL SAMPLER MODEL

```
SAMPLERS (generate real traffic)
│
├── HTTP/Web
│   └── HTTP Request                        [90% of usage]
│
├── Web Services
│   └── SOAP/XML-RPC Request
│
├── Database
│   └── JDBC Request
│
├── Messaging
│   ├── JMS Point-to-Point / Publisher / Subscriber   (Amazon MQ / ActiveMQ)
│   ├── Kafka Producer/Consumer Sampler (plugin — pepper-box/Kafkameter)
│   └── AMQP Publisher/Consumer (plugin — RabbitMQ)
│
├── Protocol-specific
│   ├── TCP Sampler
│   ├── LDAP Request / LDAP Extended Request
│   ├── FTP Request
│   └── Mail Reader Sampler
│
├── Java
│   └── Java Request (custom sampler class)
│
├── Scripting
│   ├── JSR223 Sampler       [preferred]
│   ├── BeanShell Sampler    [legacy, avoid]
│   └── OS Process Sampler
│
├── Flow/Debug
│   ├── Debug Sampler
│   ├── Test Action / Flow Control Action
│   └── Access Log Sampler
│
└── jpgc Plugin Samplers
    ├── Dummy Sampler          → test assertions/logic without real traffic
    ├── WebSocket Sampler
    └── GraphQL Sampler (3rd party)
```
Here are concrete config examples for each sampler category — same style as the Thread Group examples, with sample field values and a telco use case for each.

---

## HTTP / Web

### HTTP Request
**Config:**
- Method: `POST`
- Path: `/api/v2/order/create`
- Body Data: `{"customerId":"${customerId}","planId":"${planId}"}`
- Content-Type: `application/json` (via Header Manager)

**Telco use case:** Submitting a new connection order — the backbone sampler of nearly every telco test.

---

## Web Services

### SOAP/XML-RPC Request
**Config:**
- URL: `https://legacy-billing.telco.com/BillingService`
- SOAPAction: `urn:createInvoice`
- XML Data:
```xml
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <createInvoice>
      <customerId>${customerId}</customerId>
      <amount>${billAmount}</amount>
    </createInvoice>
  </soap:Body>
</soap:Envelope>
```
**Telco use case:** Legacy mainframe-based billing systems that still expose SOAP interfaces — common in older telco OSS stacks even alongside modern REST APIs.

---

## Database

### JDBC Request
**Config:**
- Variable Name (from JDBC Connection Config): `oss_db_pool`
- Query Type: `Select Statement`
- Query: `SELECT order_status FROM orders WHERE order_id = ?`
- Parameter values: `${orderId}`
- Parameter types: `VARCHAR`
- Result variable name: `dbOrderStatus`

**Telco use case:** Cross-checking order activation status directly in the OSS database — used constantly in your data-integrity validation scripts (API-says-ACTIVATED vs DB-says-ACTIVATED check).

---

## Messaging

### JMS Point-to-Point Sampler
**Config:**
- Communication Style: `Producer`
- JNDI Connection Factory: `ConnectionFactory`
- Destination: `order-notification-queue`
- Content: `${orderNotificationPayload}`
- Message properties: `correlationId=${correlationId}`

**Telco use case:** Amazon MQ / ActiveMQ order notification dispatch (covered in your earlier AWS MQ project).

### Kafka Producer Sampler (pepper-box plugin)
**Config:**
- Kafka Broker: `broker1:9092,broker2:9092`
- Topic Name: `cdr-events`
- Key: `${correlationId}`
- Message: `${cdrPayload}`

**Telco use case:** CDR event ingestion pipeline (your Kafka project).

### AMQP Publisher (jpgc plugin)
**Config:**
- Host: `rabbitmq-broker.telco.com`
- Port: `5671` (SSL)
- Exchange: `subscriber-events`
- Routing Key: `subscriber.suspended`
- Message content: `${suspensionEventPayload}`

**Telco use case:** RabbitMQ-based subscriber lifecycle event publishing (suspend/reactivate flows).

---

## Protocol-specific

### TCP Sampler
**Config:**
- Server Name/IP: `hlr-simulator.telco.com`
- Port: `9999`
- Text to Send: `MSISDN=${msisdn};ACTION=LOCATION_UPDATE\n`
- Re-use connection: `True`

**Telco use case:** Testing HLR/HSS network element interfaces that use raw TCP-based custom binary/text protocols instead of HTTP.

### LDAP Request
**Config:**
- Servername: `directory.telco.com`
- Port: `636`
- Search Base: `ou=subscribers,dc=telco,dc=com`
- Search Filter: `(msisdn=${msisdn})`

**Telco use case:** Subscriber directory lookup validation during authentication flows.

### FTP Request
**Config:**
- Server: `cdr-archive.telco.com`
- Remote File: `/exports/cdr_batch_${timestamp}.csv`
- Get(RETR)/Put(STOR): `Put`
- Local File: `/data/generated_cdr.csv`

**Telco use case:** Legacy CDR batch file transfer to a downstream billing mediation system.

### Mail Reader Sampler
**Config:**
- Server Type: `IMAP`
- Server: `imap.telco.com`
- Folder: `INBOX`
- Num messages: `1`
- Delete: `False`

**Telco use case:** Validating OTP/activation-confirmation emails were actually sent and received during a self-care portal registration flow.

---

## Java

### Java Request
**Config:**
- Classname: `com.telco.perf.CustomProvisioningSampler` (implements JMeter's `JavaSamplerClient`)
- Parameters: `subscriberId=${subscriberId};action=PROVISION`

**Telco use case:** Wrapping an internal Java SDK/client library that talks to a proprietary network element interface not exposed via HTTP/REST.

---

## Scripting

### JSR223 Sampler
**Config (Groovy):**
```groovy
SampleResult.setSamplerData("Custom balance check")
def startTime = System.currentTimeMillis()
def balance = new Random().nextInt(1000)
Thread.sleep(30)
SampleResult.setStampAndTime(startTime, System.currentTimeMillis() - startTime)
SampleResult.setResponseData("Balance: ${balance}", "UTF-8")
SampleResult.setSuccessful(balance >= 0)
```
**Telco use case:** Synthetic balance-check simulation for load-shaping tests where no real endpoint exists yet (pre-integration environment).

### OS Process Sampler
**Config:**
- Command: `/scripts/check_broker_health.sh`
- Arguments: `--broker=kafka-broker1`
- Timeout: `5000` ms

**Telco use case:** Running a local shell script mid-test to check Kafka broker health via CLI tools, correlating with load spikes.

---

## Flow/Debug

### Debug Sampler
**Config:**
- Display JMeter variables: `True`
- Display JMeter properties: `True`

**Telco use case:** Debugging why `${orderId}` isn't populating correctly mid-script — shows exact variable state at that point in the tree.

### Test Action / Flow Control Action
**Config:**
- Action: `Pause`
- Duration: `5000` ms
- (or Action: `Stop Thread` / `Stop Test Now`)

**Telco use case:** Forcing a deliberate pause between provisioning steps to simulate a manual approval delay in an enterprise order flow.

### Access Log Sampler
**Config:**
- Log File: `/logs/prod_access_2026.log`
- Parser Class Name: `TCLogParser`

**Telco use case:** Replaying real captured production traffic patterns from web server logs — rare, but useful for a "genuine traffic shape" replay test.

---

## jpgc Plugin Samplers

### Dummy Sampler
**Config:**
- Response Data: `{"status":"SUCCESS","orderId":"TEST123"}`
- Response Code: `200`
- Response Time (ms): `250`
- Simulate data transfer: `True`

**Telco use case:** Validating your Assertion/Extractor logic works correctly before the real Order API is even available — decouples script-logic testing from environment availability.

### WebSocket Sampler
**Config:**
- Connection: `wss://notifications.telco.com/live`
- Request data: `{"subscribe":"orderStatus","orderId":"${orderId}"}`
- Read Timeout: `6000` ms

**Telco use case:** Testing live order-status push notifications on a self-care portal (real-time provisioning status updates without polling).

### GraphQL Sampler (3rd party)
**Config:**
- Endpoint: `https://api.telco.com/graphql`
- Query:
```graphql
query { subscriber(msisdn: "${msisdn}") { plan { name, dataBalance } } }
```
**Telco use case:** Modern digital-channel backend exposing subscriber data via GraphQL instead of REST — increasingly common in newer telco digital platforms.

---

## Quick reference table

| Sampler | Protocol/Use | Telco example |
|---|---|---|
| HTTP Request | REST/HTTP | Order creation |
| SOAP Request | XML/SOAP | Legacy billing invoice |
| JDBC Request | SQL | DB order-status cross-check |
| JMS P2P | Amazon MQ/ActiveMQ | Order notification queue |
| Kafka Sampler | Kafka | CDR event ingestion |
| AMQP Publisher | RabbitMQ | Subscriber lifecycle events |
| TCP Sampler | Raw TCP | HLR/HSS simulator |
| LDAP Request | LDAP | Subscriber directory lookup |
| FTP Request | FTP | CDR batch file transfer |
| Mail Reader | IMAP/POP3 | OTP email validation |
| Java Request | Custom Java class | Proprietary network element SDK |
| JSR223 Sampler | Groovy script | Synthetic balance check |
| OS Process Sampler | Shell command | Broker health check script |
| Debug Sampler | N/A | Variable state debugging |
| Dummy Sampler | N/A | Test assertions before API is ready |
| WebSocket Sampler | WebSocket | Live order-status push |
| GraphQL Sampler | GraphQL | Modern subscriber data API |

Given your ExperienceCompany background, **HTTP Request, JDBC Request, Kafka Sampler, and JSR223 Sampler** remain your highest-frequency samplers — the rest (TCP, LDAP, FTP, JMS/AMQP) are the "I know these exist and when to reach for them" tier that's still worth being able to speak to fluently if probed.
---

## 5. CONFIG ELEMENTS — MERGE HIERARCHY

```
CONFIG ELEMENTS (set defaults, don't send traffic)
│
├── HTTP Request Defaults        → server/port/path template
├── HTTP Header Manager          → merges across scope levels
├── HTTP Cookie Manager          → session handling (per-thread by default)
├── HTTP Authorization Manager   → Basic/NTLM/Digest creds
├── CSV Data Set Config          → data-driven, scope determines sharing behavior
├── User Defined Variables       → static constants, Test Plan or TG scope
├── JDBC Connection Configuration→ connection pool for JDBC Request
├── Kafka/JMS/AMQP Connection Config (plugin) → broker/queue setup
├── Random Variable / Counter    → dynamic unique value generation
└── Keystore Configuration       → mTLS client-cert scenarios
```

**Advanced CSV Data Set nuance:**
```
Scope placement determines sharing:
  Test Plan level        → single shared file pointer across ALL Thread Groups
  Thread Group level      → separate pointer PER Thread Group (common bug source)
  "Sharing mode" setting  → All threads / Current thread group / Current thread
```
This exact config was the root cause discussed earlier (producer/consumer mismatch) — worth having as a rehearsed example.

Here are concrete config examples for each, plus the merge-behavior mechanics that make this category tricky in interviews.

---

## HTTP Request Defaults
**Config:**
- Protocol: `https`
- Server Name: `api-perf.telco.com`
- Port: `443`
- Path: (leave blank — set per-request)

**Merge behavior:** Any HTTP Request under its scope inherits these values automatically unless the sampler itself explicitly overrides a field (e.g., a sampler pointing to a different server for one specific call).

**Telco use case:** Set once at Test Plan level so all 9 Order Journey API calls (auth, coverage, catalog, order, inventory, billing...) don't repeat the same server/port.

---

## HTTP Header Manager
**Config:**
- `Content-Type: application/json`
- `Accept: application/json`
- `X-Channel-Id: DIGITAL_APP`

**Merge behavior — the tricky one:**
```
Thread Group level Header Manager:
  Content-Type: application/json
  X-Channel-Id: DIGITAL_APP

Sampler level Header Manager (inside one specific HTTP Request):
  Authorization: Bearer ${authToken}

Result sent on that request = ALL THREE headers merged:
  Content-Type: application/json
  X-Channel-Id: DIGITAL_APP
  Authorization: Bearer ${authToken}
```
If the **same header key** exists at both levels (e.g., `Content-Type` at both TG and sampler level), duplicates get sent — this was the exact bug example covered earlier ("duplicate header rejected by API").

**Telco use case:** Global headers (channel ID, content-type) at Thread Group level; per-call auth token only where needed.

---

## HTTP Cookie Manager
**Config:**
- Clear cookies each iteration: `True`
- Cookie Policy: `standard`

**Merge behavior:** Not really "merged" — it's per-thread by default, meaning each virtual user maintains its own independent cookie jar (correctly simulating separate real users, not one shared session).

**Telco use case:** Self-care portal login — each simulated subscriber gets isolated session cookies just like real independent users would.

---

## HTTP Authorization Manager
**Config:**
- Base URL: `https://api-perf.telco.com`
- Username: `${username}`
- Password: `${password}`
- Mechanism: `BASIC`

**Telco use case:** Legacy partner/B2B API that still uses Basic Auth instead of OAuth token-based auth (common with older interconnect/roaming partner integrations).

---

## CSV Data Set Config
**Config:**
- Filename: `/data/subscribers.csv`
- Variable Names: `msisdn,customerId,planCode`
- Delimiter: `,`
- Recycle on EOF: `False`
- Stop thread on EOF: `True`
- Sharing mode: `All threads`

**Merge/scope behavior (the classic interview trap):**
```
Placed at Test Plan level:
  → ONE shared file pointer, all Thread Groups pull from the same
    row sequence (no duplicate rows across Producer/Consumer TGs)

Placed inside Thread Group 1 only:
  → Only threads in TG1 read it; TG2 has NO access to these variables
    at all (not "shared, just separate" — literally not visible)

Placed separately in BOTH Thread Group 1 AND Thread Group 2:
  → TWO independent pointers reading the SAME file from the start —
    this is what caused the producer/consumer mismatch bug covered earlier
```

**Telco use case:** Single `subscribers.csv` with 50,000 unique MSISDNs — Test Plan-level placement ensures Producer and Consumer Thread Groups in the Kafka project reference the same subscriber per logical test flow.

---

## User Defined Variables
**Config:**
- `env = PERF`
- `baseUrl = https://api-perf.telco.com`
- `apiVersion = v2`

**Merge behavior:** Evaluated **once** at test start (static), not per-iteration. If placed at Test Plan level, visible everywhere; if placed inside a Thread Group, only visible within that group and below.

**Telco use case:** Environment-level constants that never change during a run — API base URL, environment tag for reporting.

---

## JDBC Connection Configuration
**Config:**
- Variable Name for created pool: `oss_db_pool`
- Database URL: `jdbc:mysql://oss-db-perf.telco.com:3306/orders`
- JDBC Driver class: `com.mysql.cj.jdbc.Driver`
- Username / Password: `${dbUser}` / `${dbPass}`
- Max Number of Connections: `20`
- Pool Exhausted Action: `WAIT`

**Merge behavior:** Referenced by name (`oss_db_pool`) — any JDBC Request anywhere in scope pointing to that pool name shares the same connection pool, regardless of where in the tree the JDBC Request sits, as long as it's within scope of this Config Element.

**Telco use case:** Cross-checking order/billing status in OSS DB — pool sized to `20` to avoid exhausting DB connections while still supporting concurrent validation threads.

---

## Kafka/JMS/AMQP Connection Config (plugin)
**Config (Kafka example):**
- Kafka Brokers: `broker1:9092,broker2:9092`
- Key Serializer: `StringSerializer`
- Value Serializer: `StringSerializer`
- Additional Properties: `acks=all;compression.type=snappy;linger.ms=10`

**Telco use case:** CDR pipeline producer config — `acks=all` intentionally matches production durability settings (from the earlier preprod issue where `acks=1` caused a false "message loss" scare).

---

## Random Variable
**Config:**
- Variable Name: `randomMsisdn`
- Output Format: `601########` (# = random digit)
- Minimum/Maximum: `100000000` / `999999999`
- Per Thread: `True`

**Telco use case:** Generating unique-looking MSISDNs per iteration for a SIM activation load test, avoiding fixed CSV dataset exhaustion issues covered earlier.

---

## Counter
**Config:**
- Start: `1`
- Increment: `1`
- Reference Name: `orderSeq`
- Track counter independently for each user: `False` (global sequence across all threads)

**Telco use case:** Sequential unique order reference numbers (`ORD-00001`, `ORD-00002`...) matching how real order-numbering systems increment.

---

## Keystore Configuration
**Config:**
- Keystore file: `/certs/client-cert.p12`
- Password: `${keystorePass}`
- Variable name for aliases: `client_cert_alias`

**Telco use case:** mTLS-secured API endpoint — e.g., partner interconnect API requiring client-certificate authentication in addition to standard token auth.

---

## SUMMARY — Merge behavior cheat table

| Config Element | Merge behavior | Common trap |
|---|---|---|
| HTTP Request Defaults | Sampler overrides win | None major |
| Header Manager | ALL levels merge/combine | Duplicate headers if same key repeated at multiple levels |
| Cookie Manager | Per-thread isolated (not merged) | Forgetting "clear each iteration" causes stale session bleed |
| CSV Data Set Config | Scope determines visibility, NOT auto-shared | Placing separately per Thread Group causes desync |
| User Defined Variables | Static, evaluated once | Assuming it updates per-iteration (it doesn't — use Counter/Random instead) |
| JDBC Connection Config | Shared by pool name across scope | Undersized pool causes connection exhaustion under load |

**One-line interview answer:** *"Config Elements merge top-down by scope — a child-level config generally overrides or adds to a parent-level one, except Header Manager, which combines headers from every level in scope, so duplicate keys at different levels is a common real bug I've hit and fixed."*

---

## 6. LOGIC CONTROLLERS — FLOW CONTROL LAYER

```
LOGIC CONTROLLERS
│
├── Grouping/Organization
│   ├── Transaction Controller   → groups + reports combined response time
│   └── Simple Controller        → pure folder, no logic
│
├── Conditional
│   ├── If Controller
│   └── Switch Controller
│
├── Looping
│   ├── Loop Controller           (fixed count)
│   ├── While Controller          (condition-based)
│   └── ForEach Controller        (iterate extracted variable list)
│
├── Randomization/Distribution
│   ├── Random Controller         (picks ONE child)
│   ├── Random Order Controller   (all children, random order)
│   └── Throughput Controller     (% or fixed-count execution split)
│
├── Reuse
│   ├── Module Controller         (same test plan)
│   ├── Include Controller        (external .jmx)
│   └── Test Fragment             (reusable block, must be called)
│
├── Concurrency-sensitive
│   ├── Interleave Controller     (round-robin between children)
│   ├── Once Only Controller      (first iteration only)
│   └── Critical Section Controller (mutex — one thread at a time)
│
└── Recording
    └── Recording Controller
```
Here are concrete config examples for each, with a telco use case per controller.

---

## Grouping/Organization

### Transaction Controller
**Config:**
- Name: `Create Order`
- Generate parent sample: `True`
- Include duration of timer and pre-post processors in generated sample: `False`

**Telco use case:** Wraps `POST /order/create` + `GET /order/confirm/{orderId}` into one reported "Create Order" business step (already covered in detail earlier).

### Simple Controller
**Config:**
- Name: `Order Journey - Digital Channel` (just a folder, no settings to configure)

**Telco use case:** Organizing a huge script visually — grouping "Login Flow," "Order Flow," "Billing Flow" as labeled folders so the tree is readable, with zero effect on execution.

---

## Conditional

### If Controller
**Config:**
- Condition: `"${stockStatus}" == "PENDING"`
- Evaluate for all children: `True`

**Telco use case:** Retry inventory reservation only if SIM stock status comes back as "PENDING" instead of immediately failing (covered in the Order Journey project).

### Switch Controller
**Config:**
- Switch Value: `${subscriberType}`
- Children named to match: `0` (Prepaid), `1` (Postpaid), `2` (Enterprise)

**Telco use case:** Route each simulated subscriber through a completely different provisioning flow depending on type — Prepaid has instant activation, Postpaid needs credit check, Enterprise needs approval workflow.

---

## Looping

### Loop Controller
**Config:**
- Loop Count: `3`

**Telco use case:** Repeat "Check Balance" API call 3 times within one order flow to simulate a user checking status multiple times before confirming payment.

### While Controller
**Config:**
- Condition: `"${currentStatus}" != "ACTIVATED"`

**Telco use case:** Poll `GET /order/status/{orderId}` repeatedly until activation completes (covered in Order Journey's Step 7 polling).

### ForEach Controller
**Config:**
- Input variable prefix: `simCandidate` (from an earlier extractor that captured multiple matches: `simCandidate_1`, `simCandidate_2`, etc.)
- Output variable name: `currentSim`
- Start/End index: leave blank (process all)

**Telco use case:** Inventory API returns a list of 5 available SIM numbers for a device type — loop through each one and validate its status individually before selecting one.

---

## Randomization/Distribution

### Random Controller
**Config:**
- (no numeric config — just contains multiple child samplers/controllers)
```
Random Controller: "Browse Behavior"
  ├── HTTP Request: View Plans Page
  ├── HTTP Request: View FAQ Page
  └── HTTP Request: View Offers Page
```
Only ONE of these three runs per iteration, picked randomly.

**Telco use case:** Simulating varied self-care portal browsing — not every user views the same page before purchasing.

### Random Order Controller
**Config:**
- (no numeric config)
```
Random Order Controller: "Pre-Purchase Checks"
  ├── HTTP Request: Check Coverage
  ├── HTTP Request: Check Device Compatibility
  └── HTTP Request: Check Promo Eligibility
```
All three run every iteration, but in randomized order each time.

**Telco use case:** These 3 checks have no strict dependency on each other — randomizing execution order avoids artificially uniform caching/sequencing patterns skewing results.

### Throughput Controller
**Config:**
- Style: `Percent Executions`
- Throughput: `70` (%)

Second sibling Throughput Controller set to `30`%:
```
Throughput Controller: "Standard Checkout" (70%)
Throughput Controller: "Express Checkout" (30%)
```

**Telco use case:** Modeling real subscriber behavior split — 70% of customers use standard multi-step checkout, 30% use express one-click checkout (matches actual production traffic distribution, not an artificial 50/50 split).

---

## Reuse

### Module Controller
**Config:**
- Module to Run: points to `Test Fragment: "Standard Login Flow"` (defined elsewhere in same test plan)

**Telco use case:** Reusing the exact same login sequence across "Order Journey" and "Billing Inquiry" test scripts within the same `.jmx`, without duplicating the login samplers twice.

### Include Controller
**Config:**
- Filename: `/scripts/common/auth-fragment.jmx`

**Telco use case:** A separate, standalone `.jmx` file containing the shared authentication flow — reused across multiple *different* test plan files (Order Journey, Billing, Self-Care Portal), not just within one file.

### Test Fragment
**Config:**
- (holds samplers, not executed unless called by Module/Include Controller)
```
Test Fragment: "Standard Login Flow"
  ├── HTTP Request: POST /auth/login
  └── JSON Extractor: authToken
```

**Telco use case:** The reusable block itself that Module Controller points to.

---

## Concurrency-sensitive

### Interleave Controller
**Config:**
- Ignore sub-controller blocks: `False`
```
Interleave Controller: "Data Center Routing"
  ├── HTTP Request: Call via Data Center A
  └── HTTP Request: Call via Data Center B
```
Iteration 1 → Data Center A, Iteration 2 → Data Center B, Iteration 3 → A again...

**Telco use case:** Alternating traffic between two geo-redundant OSS data centers to validate both are handling load correctly (active-active DR validation).

### Once Only Controller
**Config:**
- (no settings — just runs children on first iteration only)
```
Once Only Controller: "Login"
  └── HTTP Request: POST /auth/login
```

**Telco use case:** Login happens once per virtual user (thread), then subsequent loop iterations skip straight to "Browse/Order" steps — matches real session-based behavior.

### Critical Section Controller
**Config:**
- Lock Name: `msisdn-reservation-lock`

**Telco use case:** Forces MSISDN/number reservation to happen one thread at a time — used deliberately in a **controlled** test to prove the system handles concurrency correctly (or, removed deliberately in a **stress** test to expose the race condition, as covered in the earlier duplicate-SIM-number bug).

---

## Recording

### Recording Controller
**Config:**
- (no settings — just a landing container)

**Telco use case:** Target destination for the HTTP(S) Test Script Recorder when capturing a new Order Journey flow from a browser session (already covered).

---

## QUICK REFERENCE TABLE

| Controller | Config style | Telco example |
|---|---|---|
| Transaction Controller | Name + parent sample toggle | Group order-creation calls |
| Simple Controller | Just a folder | Organize script sections |
| If Controller | Boolean condition | Retry on PENDING stock status |
| Switch Controller | Variable-matched branches | Prepaid/Postpaid/Enterprise routing |
| Loop Controller | Fixed count | Repeat balance check 3x |
| While Controller | Condition string | Poll order status until ACTIVATED |
| ForEach Controller | Variable prefix | Loop through returned SIM candidates |
| Random Controller | Children, picks 1 | Varied browse behavior |
| Random Order Controller | Children, random order | Independent pre-purchase checks |
| Throughput Controller | % split | 70/30 standard vs express checkout |
| Module Controller | Points to Test Fragment (same file) | Reuse login within one script |
| Include Controller | Points to external .jmx | Reuse login across multiple scripts |
| Interleave Controller | Alternates children | DR data center routing validation |
| Once Only Controller | First iteration only | Login once per thread |
| Critical Section Controller | Lock name | Number reservation concurrency test |
| Recording Controller | Landing zone | Script recorder target |

For your OSS/BSS interviews, the ones worth being able to whiteboard fluently are: **Transaction Controller, If Controller, While Controller, ForEach Controller, Once Only Controller, and Throughput Controller** — that combination covers nearly every realistic telco flow-control scenario you've built across the Kafka, Order Journey, and AWS MQ projects.


---

## 7. TIMERS — PACING LAYER

```
TIMERS
│
├── Constant Timer
├── Uniform Random Timer
├── Gaussian Random Timer
├── Constant Throughput Timer
├── Precise Throughput Timer      [preferred over Constant — better distribution]
├── Synchronizing Timer            (batch release / spike simulation)
├── JSR223 Timer                   (custom/adaptive pacing)
└── jpgc Throughput Shaping Timer  (TPS changes over time — paired with Concurrency TG)
```

**Advanced pacing formula (interview-favorite):**
```
Threads needed = Target Throughput (req/sec) × Average Response Time (sec)

Example: 
  Target = 100 req/sec
  Avg response time = 2 sec
  Threads needed ≈ 100 × 2 = 200 concurrent threads

This is Little's Law applied to load test sizing — 
L = λ × W (Users = Arrival Rate × Wait Time)
```
This is a frequently asked "how do you calculate required thread count for a TPS target" question — you already have Little's Law in your prep, worth explicitly linking it here.

Here are concrete config examples for each timer, in the same format as the other sections — with telco use cases.

---

## Constant Timer
**Config:**
- Thread Delay: `3000` ms

**Telco use case:** Fixed 3-second pause between "View Plan Details" and "Confirm Purchase" steps — simulates a user reading plan terms before committing.

---

## Uniform Random Timer
**Config:**
- Random Delay Maximum: `2000` ms
- Constant Delay Offset: `1000` ms

**Result:** Delay ranges 1000–3000ms, different for each thread/iteration.

**Telco use case:** Browsing between self-care portal pages (Plans → Offers → FAQs) — no two subscribers pause for the exact same duration, avoiding artificially synchronized traffic bursts.

---

## Gaussian Random Timer
**Config:**
- Deviation: `500` ms
- Constant Delay Offset: `2000` ms

**Result:** Most delays cluster around 2000ms ± 500ms (bell curve), occasional outliers.

**Telco use case:** Simulating realistic think-time on the "Enter Payment Details" form during recharge — most users take ~2s, a few take noticeably longer (fumbling with card details), matching real human behavior better than flat random.

---

## Constant Throughput Timer
**Config:**
- Target throughput: `600` samples/minute (= 10 req/sec)
- Calculate Throughput based on: `all active threads in current thread group`

**Telco use case:** SLA states "Recharge API must sustain 600 transactions/min" — set enough threads (e.g., 60), let this timer throttle actual fire rate to exactly 600/min regardless of thread count.

---

## Precise Throughput Timer
**Config:**
- Target throughput (per unit): `600`
- Throughput unit: `requests per minute`
- Test duration: `1800` sec (30 min)
- Allowed throughput surplus: `1.0`

**Telco use case:** Same Recharge SLA target as above, but for a longer 30-minute steady-state soak — Precise Throughput Timer distributes requests more evenly across the full duration using probability-based scheduling instead of "catch-up" bursts.

---

## Synchronizing Timer
**Config:**
- Number of Simultaneous Users to Group by: `500`
- Timeout in ms: `5000`

**Telco use case:** Simulating **midnight batch billing cycle trigger** — 500 postpaid subscriber billing jobs all need to fire at the exact same instant (midnight cutover), not trickle in gradually. Synchronizing Timer holds all 500 threads until ready, then releases simultaneously.

---

## JSR223 Timer
**Config (Groovy):**
```groovy
def hour = new Date().getHours()
if (hour >= 9 && hour <= 11) {
    return 300   // simulated busy hour - aggressive pacing
} else {
    return 3000  // off-peak - relaxed pacing
}
```

**Telco use case:** Adaptive pacing that mimics real subscriber activity patterns changing throughout a simulated business day within a single long-running test, rather than one fixed rate for the entire duration.

---

## jpgc Throughput Shaping Timer
**Config (graph/table):**

| Time (s) | Target TPS |
|---|---|
| 0 | 50 |
| 300 | 1000 |
| 1800 | 1000 |
| 2100 | 100 |

**Telco use case:** Real-time OCS charging busy-hour test — ramps from 50 to 1000 TPS over 5 minutes, holds 1000 TPS for 25 minutes (busy hour), tapers to 100 TPS over 5 minutes. Paired with Concurrency Thread Group, this is your go-to for SLA-driven charging/billing throughput validation.

---

## QUICK REFERENCE TABLE

| Timer | Sample config | Telco use case |
|---|---|---|
| Constant Timer | 3000 ms | Fixed think-time before purchase confirm |
| Uniform Random Timer | Offset 1000, Max 2000 | Portal page browsing |
| Gaussian Random Timer | Offset 2000, Deviation 500 | Realistic payment-form think-time |
| Constant Throughput Timer | 600/min | Basic SLA-rate enforcement |
| Precise Throughput Timer | 600/min, 1800s duration | Steady 30-min soak at fixed TPS |
| Synchronizing Timer | 500 users, 5000ms timeout | Midnight batch billing trigger |
| JSR223 Timer | Time-of-day adaptive delay | Simulated daily traffic pattern shift |
| Throughput Shaping Timer (jpgc) | 50→1000→1000→100 TPS | OCS busy-hour charging SLA test |

For interviews, the pairing to master fluently is **Precise Throughput Timer** (simple fixed-TPS SLA tests) and **Throughput Shaping Timer + Concurrency Thread Group** (shaped busy-hour curves) — these two cover almost every real telco throughput scenario you'd be asked to design live.



---

## 8. PRE/POST-PROCESSORS + ASSERTIONS — VALIDATION LAYER

```
PRE-PROCESSORS (before sampler)
├── HTTP URL Re-writing Modifier
├── User Parameters
├── HTML Link Parser
├── JSR223 PreProcessor          [primary tool: signatures, dynamic headers, payload build]
└── RegEx User Parameters

POST-PROCESSORS (after sampler — extraction/correlation)
├── Regular Expression Extractor  [universal]
├── JSON Extractor                [structured JSON — preferred over regex for JSON]
├── JSON JMESPath Extractor       [complex nested JSON]
├── XPath/XPath2 Extractor        [XML/SOAP]
├── CSS/JQuery Extractor          [HTML]
├── Boundary Extractor            [fast, simple substring]
└── JSR223 PostProcessor          [complex/custom extraction + calculations]

ASSERTIONS (pass/fail validation)
├── Response Assertion            [text/regex match — most used]
├── Size Assertion
├── JSON Assertion / JMESPath Assertion
├── XPath/XPath2 Assertion
├── Duration Assertion            [SLA response-time enforcement]
├── Compare Assertion             [regression baseline comparison]
└── JSR223 Assertion              [custom business-rule / cross-field validation]
```
Here are concrete config examples for each, in the same format — with telco use cases.

---

# PRE-PROCESSORS (before sampler)

## HTTP URL Re-writing Modifier
**Config:**
- Session Argument Name: `JSESSIONID`
- Path Extension: `False`
- Cache Session Id: `True`

**Telco use case:** Legacy self-care portal that tracks session via URL parameter (`;jsessionid=xyz` in the path) instead of cookies — rare today but still found in older telco web portals built pre-2010.

---

## User Parameters
**Config:**
- Names: `deviceType`, `channel`
- Thread 1: `SMARTPHONE`, `APP`
- Thread 2: `TABLET`, `WEB`

**Telco use case:** Assigning a small fixed set of device/channel combinations directly in-line without needing a full CSV file for a quick, small-scale device-compatibility test.

---

## JSR223 PreProcessor
**Config (Groovy):**
```groovy
def timestamp = System.currentTimeMillis().toString()
vars.put("reqTimestamp", timestamp)

def rawString = vars.get("apiKey") + timestamp
vars.put("reqSignature", rawString.digest("SHA-256"))
```

**Telco use case:** Generating a fresh signature/timestamp header before every OCS charging API call — required for request authenticity validation.

---

# POST-PROCESSORS (after sampler — extraction/correlation)

## Regular Expression Extractor
**Config:**
- Apply to: `Main sample`
- Field to check: `Body`
- Reference Name: `orderId`
- Regular Expression: `"orderId":"(.*?)"`
- Template: `$1$`
- Match No.: `1`

**Telco use case:** Extracting `orderId` from a plain-text or non-standard-JSON response from a legacy order-creation endpoint.

---

## JSON Extractor
**Config:**
- Names of created variables: `authToken`
- JSON Path expressions: `$.token`
- Match numbers: `1`
- Default values: `TOKEN_NOT_FOUND`

**Telco use case:** Extracting the auth token from `POST /auth/login` response — your most-used extractor across nearly every telco API flow.

---

## JSON JMESPath Extractor
**Config:**
- Variable name: `activePlanNames`
- JMESPath expression: `plans[?status=='ACTIVE'].name`

**Telco use case:** Pulling only the names of currently active plans from a subscriber's plan list response that contains a mix of active/expired/suspended plans — regex or basic JSONPath can't easily do this conditional filtering.

---

## XPath / XPath2 Extractor
**Config:**
- Reference Name: `invoiceAmount`
- XPath query: `//createInvoiceResponse/amount/text()`

**Telco use case:** Extracting invoice amount from a legacy SOAP billing system's XML response.

---

## Boundary Extractor
**Config:**
- Reference Name: `sessionToken`
- Left Boundary: `"token":"`
- Right Boundary: `"`

**Telco use case:** Fast, lightweight extraction alternative to regex when the token has clean, predictable surrounding text — slightly faster processing at very high thread counts.

---

## JSR223 PostProcessor
**Config (Groovy):**
```groovy
import groovy.json.JsonSlurper

def response = new JsonSlurper().parseText(prev.getResponseDataAsString())
vars.put("orderStatus", response.status ?: "UNKNOWN")

def kafkaEventTime = vars.get("kafkaEventTimestamp")
if (kafkaEventTime != null) {
    def latency = System.currentTimeMillis() - kafkaEventTime.toLong()
    vars.put("settlementLatencyMs", latency.toString())
}
```

**Telco use case:** Calculating custom settlement latency between a Kafka order event and DB persistence confirmation (from your Kafka project).

---

# ASSERTIONS (pass/fail validation)

## Response Assertion
**Config:**
- Field to test: `Text Response`
- Pattern Matching Rule: `Contains`
- Pattern: `"status":"ACTIVATED"`

**Telco use case:** Basic content check confirming order activation succeeded — used everywhere as the first line of validation.

---

## Size Assertion
**Config:**
- Size: `100` bytes minimum (or exact range)
- Comparison type: `Not less than`

**Telco use case:** Catching a truncated CDR export response that returns 200 OK but with an unexpectedly small/empty body due to a backend timeout mid-stream.

---

## JSON Assertion
**Config:**
- Assert JSON Path exists: `$.billingAccountId`
- Additionally assert value: `True`
- Expected Value: (regex, e.g.) `BA-[0-9]+`

**Telco use case:** Validating billing account creation response contains a properly formatted billing account ID, not just any string.

---

## XPath / XPath2 Assertion
**Config:**
- XPath: `//createInvoiceResponse/status[text()='SUCCESS']`

**Telco use case:** Legacy SOAP billing invoice creation — confirming the XML response explicitly reports SUCCESS.

---

## Duration Assertion
**Config:**
- Duration in milliseconds: `2000`

**Telco use case:** Enforcing OCS real-time charging SLA — any charging request over 2 seconds is flagged as a failure directly in the script, not just noted in post-run reports.

---

## Compare Assertion
**Config:**
- Compare content: `True`
- Compare time: `False`

**Telco use case:** Regression testing — comparing today's `GET /catalog/plans` response against a saved baseline response from last release to catch unintended API contract changes.

---

## JSR223 Assertion
**Config (Groovy):**
```groovy
def expectedAmount = vars.get("expectedBillAmount").toDouble()
def actualAmount = vars.get("actualBillAmount").toDouble()

if (Math.abs(expectedAmount - actualAmount) > 0.01) {
    AssertionResult.setFailure(true)
    AssertionResult.setFailureMessage("Billing mismatch — expected: ${expectedAmount}, actual: ${actualAmount}")
}
```

**Telco use case:** Cross-field billing validation — comparing DB-expected amount against API-returned invoice amount (already covered as a core example).

---

## QUICK REFERENCE TABLE

| Element | Sample config | Telco use case |
|---|---|---|
| HTTP URL Re-writing Modifier | Session arg name | Legacy URL-based session portal |
| User Parameters | Per-thread inline values | Small fixed device/channel test set |
| JSR223 PreProcessor | HMAC signature gen | OCS charging request signing |
| Regular Expression Extractor | `"orderId":"(.*?)"` | Legacy/plain-text order ID extraction |
| JSON Extractor | `$.token` | Auth token extraction |
| JMESPath Extractor | `plans[?status=='ACTIVE']` | Filtered active-plan extraction |
| XPath Extractor | `//amount/text()` | SOAP invoice amount |
| Boundary Extractor | `"token":"` / `"` | Fast lightweight token extraction |
| JSR223 PostProcessor | Custom latency calc | Kafka-to-DB settlement latency |
| Response Assertion | Contains "ACTIVATED" | Basic status content check |
| Size Assertion | Min 100 bytes | Detect truncated CDR export |
| JSON Assertion | `$.billingAccountId` regex match | Billing account ID format check |
| XPath Assertion | Status = SUCCESS | SOAP invoice validation |
| Duration Assertion | <2000ms | Real-time charging SLA |
| Compare Assertion | Content compare | Plan catalog regression check |
| JSR223 Assertion | Cross-field billing check | API vs DB amount reconciliation |

For interviews, the combo to have most fluent is: **JSON Extractor + JSON Assertion** (everyday REST validation), **Duration Assertion** (SLA enforcement), and **JSR223 Assertion** (business-rule/cross-system validation) — that trio covers the vast majority of real telco validation scenarios across your Order Journey, Kafka, and AWS MQ projects.
---

## 9. LISTENERS — REPORTING LAYER

```
LISTENERS
│
├── Debug/UI (NEVER use during real load — memory heavy)
│   ├── View Results Tree
│   ├── View Results in Table
│   └── Graph Results
│
├── Summary/Aggregate
│   ├── Summary Report
│   ├── Aggregate Report          [percentiles — SLA reporting standard]
│   └── Aggregate Graph
│
├── File-based (REAL LOAD TESTS)
│   └── Simple Data Writer        → writes to .jtl, minimal overhead
│
├── Backend
│   └── Backend Listener          → live stream to InfluxDB/Grafana/Datadog
│
└── jpgc Custom Listeners
    ├── Transactions per Second
    ├── Response Times Over Time
    ├── Active Threads Over Time
    ├── Response Codes per Second
    ├── Response Latencies Over Time
    └── PerfMon Metrics Collector  → server-side CPU/mem via PerfMon Agent
```

**Post-test reporting command (always expected knowledge):**
```bash
jmeter -n -t script.jmx -l results.jtl -e -o /report-output-folder
```
`-n` non-GUI, `-t` test file, `-l` results log, `-e -o` auto-generate HTML Dashboard Report.

Here are concrete config examples for each listener, in the same format, with telco use cases and the practical run-time guidance that ties them together.

---

## Debug/UI Listeners (debugging only — never in real load runs)

### View Results Tree
**Config:**
- Log/Display Only: `Errors` (checkbox, to reduce memory footprint even during debugging)

**Telco use case:** Debugging why the "Reserve Inventory" step returns a 400 — inspect full request/response body, headers, and extractor results for that one sampler, thread count = 1.

### View Results in Table
**Config:**
- Columns shown: default (Sample #, Time, Label, Status, Response Time)

**Telco use case:** Quick tabular scan during script-build phase to confirm each of the 9 Order Journey steps returns 200, before scaling to real load.

### Graph Results
**Config:** (no meaningful settings beyond display toggles)

**Telco use case:** Live visual demo to a stakeholder during a small-scale dry run — never used in the actual load test itself due to memory overhead.

---

## Summary/Aggregate Listeners

### Summary Report
**Config:** (auto-populates: Label, # Samples, Avg, Min, Max, Error%, Throughput)

**Telco use case:** Quick post-run sanity check — "did Recharge API error rate stay under 1%?"

### Aggregate Report
**Config:** (adds Median, 90th/95th/99th percentile columns)

**Telco use case:** SLA sign-off report — "OCS charging p95 must be <200ms" — this is the listener you'd screenshot for a go/no-go decision meeting.

### Aggregate Graph
**Config:**
- Graph Type: bar chart of selected metric (Average/Median/90th percentile) across all Transaction Controller labels

**Telco use case:** Visual comparison across all 9 Order Journey steps in one glance — instantly spot that "Billing Account Creation" bar is 3x taller than the others.

---

## File-based Listener (REAL LOAD TESTS)

### Simple Data Writer
**Config:**
- Filename: `/results/order_journey_run1.jtl`
- Configure → check only: `Save Response Code`, `Save Response Message`, `Save Thread Name`, `Save Success`
- Uncheck: `Save Response Data` (keeps file size manageable at scale — don't log full bodies for thousands of samples)

**Telco use case:** The only listener enabled during the actual 200-concurrent-user, 20-minute Order Journey load run. Everything else gets disabled to protect JMeter injector performance.

---

## Backend Listener

### Backend Listener
**Config:**
- Implementation: `org.apache.jmeter.visualizers.backend.influxdb.InfluxdbBackendListenerClient`
- influxdbUrl: `http://influx-host:8086/write?db=jmeter`
- application: `TelcoOrderJourney`
- measurement: `jmeter`
- summaryOnly: `False` (per-sample granularity for live dashboard)

**Telco use case:** Streams live metrics to Grafana during the run — your team watches TPS/error-rate/response-time dashboards in real time instead of waiting for post-test .jtl analysis. Given your Dynatrace/Datadog stack, this is the listener you'd configure most often.

---

## jpgc Custom Listeners

### Transactions per Second
**Config:** (no fields — just visual, reads from the .jtl/live stream)

**Telco use case:** Compare actual achieved TPS against your Throughput Shaping Timer's 1000 TPS target during the OCS charging busy-hour test — visually confirms whether the system kept pace or fell behind at any point in the curve.

### Response Times Over Time
**Config:** (Aggregation granularity: e.g., every 5 sec)

**Telco use case:** Spot exactly when in the 30-min busy-hour test response times started degrading — correlate that timestamp with PerfMon CPU spike on the OCS server.

### Active Threads Over Time
**Config:** (auto-reads Thread Group data)

**Telco use case:** Visually confirm your Stepping/Ultimate Thread Group actually executed the shape you configured — verify the 10-users-every-30-seconds step pattern really happened as designed, not just trust the config blindly.

### Response Codes per Second
**Config:** (auto-categorizes by response code)

**Telco use case:** Pinpoint the exact second 500 errors started spiking during the Order Journey load test — narrows root-cause investigation window drastically versus scanning the whole 20-minute run.

### Response Latencies Over Time
**Config:** (measures time-to-first-byte specifically, separate from full response time)

**Telco use case:** Distinguishing network/connection latency from actual server processing time — useful when diagnosing whether a slowdown is network-layer or application-layer.

### PerfMon Metrics Collector
**Config:**
- PerfMon Server Agent: `oss-app-server:4444`, `billing-server:4444`
- Metrics: CPU, Memory, Disk I/O, Network

**Telco use case:** Correlate JMeter-side load spikes directly with server-side CPU/memory inside JMeter's own report — no need to cross-reference a separate Dynatrace dashboard manually.

---

## POST-TEST REPORTING (what you generate after the run)

**Command:**
```bash
jmeter -n -t order_journey.jmx -l results/order_journey_run1.jtl -e -o /reports/order_journey_html
```
`-n` = non-GUI mode, `-t` = test plan, `-l` = results log, `-e -o` = auto-generate the HTML Dashboard Report (statistics table, response time graphs, percentiles, all from the `.jtl`).

**Telco use case:** This is the actual deliverable you'd attach to a performance test sign-off email — full dashboard with percentile graphs, generated from the lean `.jtl` file rather than any live listener.

---

## QUICK REFERENCE TABLE

| Listener | Sample config | Telco use case |
|---|---|---|
| View Results Tree | Errors only | Debug single failing sampler |
| View Results in Table | Default columns | Quick script-build sanity check |
| Graph Results | N/A | Live demo only, never real load |
| Summary Report | Auto | Quick post-run health check |
| Aggregate Report | Adds percentiles | SLA sign-off reporting |
| Aggregate Graph | Bar chart per label | Visual bottleneck comparison across steps |
| Simple Data Writer | Minimal fields, no response data | **Always-on for real load runs** |
| Backend Listener | InfluxDB/Grafana endpoint | Live dashboard during run |
| jp@gc TPS | N/A | Compare actual vs target TPS |
| jp@gc Response Times Over Time | 5s granularity | Pinpoint degradation onset |
| jp@gc Active Threads Over Time | Auto | Verify load shape executed correctly |
| jp@gc Response Codes/Sec | Auto | Pinpoint error spike timing |
| jp@gc PerfMon Metrics Collector | Agent host:port list | Correlate load with server resources |

**Golden rule to state in interviews:** *"During script development I use View Results Tree and Summary Report for debugging. For the actual load run, I disable every UI listener and keep only Simple Data Writer plus Backend Listener to Grafana — then generate the full HTML Dashboard Report from the .jtl file afterward for the official SLA sign-off document."*


---

## 10. DISTRIBUTED TESTING HIERARCHY (advanced — often skipped, always asked at Lead level)

```
                    ┌─────────────────┐
                    │  Controller/     │
                    │  Master Node     │  (runs jmeter -n -t plan.jmx -r)
                    └────────┬─────────┘
                             │ RMI protocol
              ┌──────────────┼──────────────┐
              ▼               ▼              ▼
        ┌──────────┐   ┌──────────┐   ┌──────────┐
        │ Injector 1│   │Injector 2│   │Injector 3│   (jmeter-server on each)
        │(slave/agent)│ │(slave)   │   │(slave)   │
        └──────────┘   └──────────┘   └──────────┘
              │               │              │
              ▼               ▼              ▼
                    Target System Under Test
```

**Key config points:**
- `jmeter.properties`: `remote_hosts=injector1_ip,injector2_ip,injector3_ip`
- Each injector runs `jmeter-server` and must have identical `.jmx`, test data (CSV), and JMeter version
- **Critical gotcha (already flagged earlier)**: Ultimate/Stepping Thread Groups don't auto-divide thread counts across injectors — each injector runs the FULL configured thread count independently, so you must manually divide (e.g., 300 total threads ÷ 3 injectors = 100 threads configured per injector's effective share)
- CSV Data Set Config: use unique files per injector, or "Sharing mode: Current thread group" carefully to avoid data collision across distributed nodes
- Results aggregate back to controller node only if using `-r` (remote start) properly configured

**Interview point:** "I always validate that injector CPU stays below 70-80% during a run — if the injector itself is CPU-bound, response time measurements become unreliable because the injector can't accurately timestamp requests/responses under its own resource pressure." This is a strong "load generator sizing" answer.

Here's the full distributed testing setup with concrete configs, commands, and telco-scale scenarios — this is the section Lead-level interviews weight heavily since it separates scripters from architects.

---

## ARCHITECTURE RECAP

```
                    ┌─────────────────┐
                    │  Controller/     │
                    │  Master Node     │
                    └────────┬─────────┘
                             │ RMI (Java Remote Method Invocation)
              ┌──────────────┼──────────────┐
              ▼               ▼              ▼
        ┌──────────┐   ┌──────────┐   ┌──────────┐
        │ Injector 1│   │Injector 2│   │Injector 3│
        └──────────┘   └──────────┘   └──────────┘
              │               │              │
              ▼               ▼              ▼
                    Target System Under Test
```

---

## STEP-BY-STEP SETUP (with actual config values)

### 1. On each Injector/Slave machine

**Config: `jmeter.properties` (or `user.properties`) on each slave**
```properties
server_port=1099
server.rmi.localport=50000
```

**Start the slave listener:**
```bash
./jmeter-server -Djava.rmi.server.hostname=10.0.1.11
```
(Run this identical command on each injector — 10.0.1.11, 10.0.1.12, 10.0.1.13 etc., each with its own IP)

**Requirements per injector (commonly asked "what do you check before scaling out"):**
- Identical JMeter version as master
- Identical `.jmx` test plan file and CSV data files copied locally
- Identical Java version
- Sufficient open file descriptors (`ulimit -n 65535`) — default OS limits choke high-concurrency HTTP connections
- Firewall allows RMI ports (1099 + dynamic high ports, or fix with `server.rmi.localport`)

---

### 2. On the Master/Controller machine

**Config: `jmeter.properties` on master**
```properties
remote_hosts=10.0.1.11,10.0.1.12,10.0.1.13
client.rmi.localport=4000
```

**Run command (non-GUI, distributed, remote start on all):**
```bash
jmeter -n -t order_journey.jmx -r -l results/master_results.jtl -e -o /reports/order_journey_html
```
`-r` = run on all servers defined in `remote_hosts`. Alternative `-R host1,host2,host3` lets you specify a subset without editing properties file.

**Run on specific injectors only (partial scale-out for a quick test):**
```bash
jmeter -n -t order_journey.jmx -R 10.0.1.11,10.0.1.12 -l results.jtl
```

---

## KEY THREAD-COUNT MATH (the actual Lead-level gotcha)

**Scenario:** You need 900 total concurrent users for an OCS charging test, split across 3 injectors.

**Wrong assumption:** Set Thread Group to 900 threads, run distributed → JMeter will NOT auto-divide this.

**What actually happens:**
```
Thread Group configured: 900 threads
Distributed across 3 injectors
Result: EACH injector runs 900 threads = 2700 total threads generated
```

**Correct approach:**
```
Total target = 900 users
Injectors = 3
Threads per injector = 900 / 3 = 300

Set Thread Group to: 300 threads
Run distributed across 3 injectors
Result: 300 × 3 = 900 total — correct
```

This exact math is a favorite Lead-interview trap question: *"If you need 1000 users across 4 injectors, what do you configure in the Thread Group?"* → Answer: **250**, not 1000.

---

## FULL TELCO EXAMPLE: OCS Charging Busy-Hour Distributed Test

**Requirement:** 1000 TPS busy-hour load, single injector maxes out around 350-400 TPS realistically (network/CPU limits), so scale to 3 injectors.

```
Master node: coordinates + aggregates results
├── Injector 1 (10.0.1.11) → Concurrency TG: 100 threads, target ~340 TPS
├── Injector 2 (10.0.1.12) → Concurrency TG: 100 threads, target ~330 TPS
└── Injector 3 (10.0.1.13) → Concurrency TG: 100 threads, target ~330 TPS

Combined effective: ~1000 TPS at the OCS endpoint
```

**Throughput Shaping Timer** on each injector is configured with the **same relative curve** (e.g., each targets ~1/3 of the total 1000 TPS shape), not the full 1000 on each.

---

## CSV DATA SET ACROSS DISTRIBUTED INJECTORS (common bug source)

**Problem:** All 3 injectors reading the same `subscribers.csv` from row 1 → same MSISDNs get used across injectors → data collision (two injectors reserve the same MSISDN simultaneously, false "duplicate" bug that isn't really a system bug).

**Fix — split the CSV per injector:**
```bash
split -n 3 subscribers.csv subscribers_part_
# creates subscribers_part_aa, subscribers_part_ab, subscribers_part_ac
```
Copy `subscribers_part_aa` to Injector 1, `_ab` to Injector 2, `_ac` to Injector 3 — each references its own local file by the same relative filename in the `.jmx`.

**Telco use case:** Exactly this pattern was needed for the Order Journey distributed load test — without splitting, you'd get false MSISDN-collision failures that look like the race-condition bug covered earlier, but are actually a test-data-design flaw, not a system bug. Being able to distinguish these two is a strong senior/lead talking point.

---

## RESULTS AGGREGATION

- Master node collects results from all injectors via RMI back-channel automatically when using `-r`/`-R`
- Single `master_results.jtl` file contains combined data from all injectors — Aggregate Report/HTML Dashboard on the master reflects the TRUE combined system behavior
- **Gotcha:** if network between master and injectors is unstable, results can be delayed/lost in transit — for very high-volume tests, some teams instead configure each injector to write its own local `.jtl` and merge them post-test via script, avoiding real-time RMI result-streaming overhead entirely

**Merge command (alternative approach for very high volume):**
```bash
# On each injector: jmeter -n -t plan.jmx -l local_results.jtl (no -r, run locally)
# Then merge:
cat injector1_results.jtl injector2_results.jtl injector3_results.jtl > combined.jtl
# Re-sort by timestamp if needed, then generate report:
jmeter -g combined.jtl -o /reports/combined_html
```
**Telco use case:** Used this pattern for the CDR ingestion Kafka producer test at very high TPS — RMI result-streaming itself was adding measurable overhead to injector CPU at 1000+ TPS, so switched to local-write + post-merge.

---

## MONITORING INJECTOR HEALTH DURING THE RUN (critical Lead-level discipline)

**Watch on each injector during the run:**
```bash
# CPU/memory
top -b -n 1 | head -20

# Open file descriptors (HTTP connection limit check)
lsof -p <jmeter_pid> | wc -l

# Network connections in TIME_WAIT (can exhaust ephemeral ports at high load)
netstat -an | grep TIME_WAIT | wc -l
```

**Interview point:** *"I always monitor injector-side CPU, memory, and TIME_WAIT socket count during a distributed run — if the injector itself gets resource-constrained, response time measurements become unreliable because the injector can't accurately timestamp requests under its own load. I've seen false 'server is slow' conclusions that were actually 'my own load generator was CPU-starved.'"*

---

## SUMMARY TABLE

| Aspect | Config/Command | Telco relevance |
|---|---|---|
| Slave startup | `./jmeter-server -Djava.rmi.server.hostname=IP` | Each injector must be reachable via RMI |
| Master config | `remote_hosts=ip1,ip2,ip3` | Defines injector pool |
| Distributed run | `jmeter -n -t plan.jmx -r -l results.jtl` | Starts test on all injectors |
| Thread math | Total ÷ injector count | 900 users ÷ 3 injectors = 300 threads each |
| CSV split | `split -n 3 file.csv` | Avoid data collision across injectors |
| Result merge (alt) | `cat *.jtl > combined.jtl` | High-TPS tests where RMI streaming adds overhead |
| Injector health | `top`, `lsof`, `netstat TIME_WAIT` | Validate injector isn't the bottleneck |

**One-line interview closer:** *"For high-TPS telco tests like OCS charging at 1000+ TPS, a single injector can't realistically generate that load without becoming the bottleneck itself — so I scale out across 3-4 injectors, manually divide the thread/TPS math per injector since JMeter doesn't auto-split, split CSV test data to avoid cross-injector collisions, and actively monitor injector-side CPU/file-descriptors/TIME_WAIT sockets throughout the run to make sure I'm measuring the system under test, not my own load generator's limits."*


---

## 11. THE MASTER MENTAL MODEL (say this out loud in interviews)

```
"JMeter's element hierarchy follows a simple mental model:

 - Thread Group defines WHO (how many users, what shape of load)
 - Config Elements define DEFAULTS (what every request starts with)
 - Controllers define FLOW (order, conditions, repetition)
 - Timers define PACE (how fast/slow requests fire)
 - Samplers are the ONLY thing that generates real traffic
 - Pre-Processors prepare data JUST BEFORE the request
 - Post-Processors extract data JUST AFTER the response
 - Assertions validate CORRECTNESS, not just connectivity
 - Listeners record/display, and should be minimal during actual load

 Everything except Thread Groups and Samplers follows a SCOPE rule:
 placed at a parent level, it applies to everything below it;
 placed inside a sampler, it applies only there."
```

---

## QUICK-FIRE INTERVIEW ANSWERS TABLE

| Question | One-line answer |
|---|---|
| Execution order within scope? | Config → PreProcessor → Timer → Sampler → PostProcessor → Assertion → Listener |
| vars vs props? | vars = thread-local, props = JVM-wide shared |
| Closed vs open model? | Closed = fixed thread waits; Open = arrival-rate independent of completion |
| Thread count formula? | Little's Law: Threads = Throughput × Response Time |
| Why JSR223 over BeanShell? | Compiled + cached, much faster at scale |
| CSV shared across Thread Groups? | Only if scoped at Test Plan level, not per-TG |
| Distributed testing thread math? | Each injector runs FULL configured threads — divide manually |
| Real load test listener setup? | Simple Data Writer only; disable all UI listeners |
| How to enforce SLA in-script? | Duration Assertion (per-sample) + JSR223 running-percentile check (aggregate) |

This is your complete, tie-everything-together reference sheet — the one to review the morning of the interview, since it consolidates every category we've built into a single execution-order + scope + distributed-architecture mental model that senior interviewers specifically probe for.
