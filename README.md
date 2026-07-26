# JMETER_TUTORIALS_VJB
JMETER_TUTORIALS

Here's the full landscape of Thread Groups in JMeter — built-in and plugin (jpgc) — with what each is actually used for.

## Built-in Thread Groups

**1. Thread Group (standard)**
- Basic: threads, ramp-up, loop count (or infinite), optional scheduler (duration/startup delay)
- Use: simple load profiles, baseline/smoke tests, anything where you just need N users hitting a flow M times

**2. setUp Thread Group**
- Runs before all regular thread groups start, regardless of position in the plan
- Use: pre-test setup — auth token generation, test data seeding, cache warm-up, environment health check

**3. tearDown Thread Group**
- Runs after all regular thread groups finish
- Use: cleanup — deleting test data, closing sessions, releasing reserved resources

Standard TG's weakness: you can't do non-linear ramp-up, can't hold steady state cleanly with fine control, and can't easily do spike/step patterns without workarounds (Synchronizing Timer, JSR223 hacks). That's where jpgc plugins come in.

## jmeter-plugins (jpgc) Thread Groups — via Custom Thread Groups plugin

**4. Stepping Thread Group**
- Adds users in steps (e.g., 10 users every 30s until 100), holds, then steps down
- Use: gradual ramp-up load tests, finding the breakpoint where response times degrade — most commonly used jpgc TG for realistic capacity testing

**5. Ultimate Thread Group**
- Fully custom load shape via a table: (start threads, initial delay, startup time, hold time, shutdown time) rows, chainable
- Use: complex, arbitrary load curves — spike tests, sawtooth patterns, business-hour traffic simulation. Most flexible option; often paired with the jpgc graph to visualize the shape before running

**6. Arrivals Thread Group**
- Defines load by arrival rate (requests/sec you want arriving) rather than thread count — engine calculates threads needed dynamically
- Use: modeling systems where throughput/arrival rate is the real-world spec (e.g., "we get 500 orders/min"), rather than guessing at user counts. Closer to open-model load testing (like Gatling's default) vs JMeter's default closed model

**7. Free-Form Arrivals Thread Group**
- Same arrival-rate concept as above but with a free-form table for irregular arrival patterns over time
- Use: traffic replaying real production arrival patterns that don't fit a clean ramp/step shape

**8. Concurrency Thread Group**
- Maintains a target concurrent thread count using a specified ramp-up strategy, and importantly, works with **Throughput Shaping Timer** to hold precise throughput rather than just thread count
- Use: paired almost always with bzm's Throughput Shaping Timer (different plugin, "Throughput Shaping Timer" under jpgc) when the actual goal is "sustain X req/s" rather than "sustain X users" — very common in your kind of telecom OSS/BSS load profiling where SLAs are throughput-based, not user-count-based

## Practical mapping for performance test design

| Goal | Thread Group to use |
|---|---|
| Simple functional/load smoke | Standard Thread Group |
| Token/session prep before load | setUp Thread Group |
| Post-test data cleanup | tearDown Thread Group |
| Find breaking point / soak with steps | Stepping Thread Group |
| Arbitrary custom load shape / spike test | Ultimate Thread Group |
| Model by arrival rate (open system) | Arrivals / Free-Form Arrivals TG |
| Hold precise TPS/throughput target | Concurrency TG + Throughput Shaping Timer |

One thing worth flagging since you work with distributed JMeter: Ultimate and Stepping Thread Groups don't distribute cleanly across remote injectors by default the way standard TG does — you need to account for per-injector thread math manually, since each injector runs the full shape independently unless you divide counts yourself.

In JMeter's element hierarchy, "Non-Test Elements" is a distinct, narrow category — separate from Thread Groups, Samplers, Listeners, etc. These live on the WorkBench (or can be added under Test Plan → Add → Non-Test Elements). It's a much smaller list than Thread Groups, and plugins barely touch this category since it's tool/utility-focused, not test-flow-focused.

## Built-in Non-Test Elements

**1. HTTP(S) Test Script Recorder**
- Acts as a proxy server that captures browser/app traffic and converts it into HTTP Samplers automatically
- Use: bootstrapping scripts for complex flows (multi-step OSS/BSS provisioning journeys, e.g.) instead of hand-building every sampler. Almost always your starting point for a new script before you clean it up manually

**2. HTTP Mirror Server**
- Spins up a local mirror that echoes back whatever HTTP request it receives
- Use: niche — testing your own JMeter config/assertions/extractors against a known predictable response, or verifying proxy/recorder setup, without hitting a real target system

**3. Property Display**
- Displays current JMeter properties (JVM args, jmeter.properties values, custom -J properties passed at runtime) in a simple table
- Use: debugging property overrides — confirming a `-Jthreads=50` passed via command line actually landed, or checking `jmeter.properties` values mid-session

**4. Sample**
- A bare generic placeholder element, mostly a legacy leftover
- Use: essentially none in modern practice; rarely used today

## Plugins and Non-Test Elements

This is the short answer: **jpgc (JMeter Plugins) doesn't really extend the Non-Test Elements category.** Almost every jpgc addition — Custom Thread Groups, Custom Listeners (graphs), Custom Samplers, Throughput Shaping Timer, PerfMon Server Agent — is added into other categories (Thread Groups, Listeners, Samplers, Timers, Config Elements), not Non-Test Elements. The category itself is essentially frozen at those 4 core items across JMeter versions.

The one adjacent thing worth knowing since you use Dynatrace/Datadog/Grafana alongside JMeter: **PerfMon Server Agent** (a standalone jar you run on the SUT, separate from JMeter itself) technically isn't a JMeter element at all — it's an external monitoring daemon that the **PerfMon Metrics Collector Listener** (a jpgc plugin, added under Listeners) connects to for CPU/memory/disk stats. People sometimes mentally file it under "non-test element" because it runs outside the test plan, but strictly in JMeter's own taxonomy it's not part of that category.

## Practical summary

| Element | Category | Real-world use |
|---|---|---|
| HTTP(S) Test Script Recorder | Non-Test | Script bootstrapping from captured traffic |
| HTTP Mirror Server | Non-Test | Local echo target for debugging test logic |
| Property Display | Non-Test | Verify runtime property overrides |
| Sample | Non-Test | Legacy, effectively unused |

Given your stack, the Recorder is really the only one you'd reach for regularly — for OSS/BSS flows with complex session/auth handshakes, recording via proxy and then parameterizing/correlating manually is usually faster than hand-writing every sampler.

Here's the full setup and workflow, end to end.

## 1. Add the Recorder

- Right-click **Test Plan** → **Add → Non-Test Elements → HTTP(S) Test Script Recorder**
- This also auto-adds a **Recording Controller** under Test Plan (if not already present) — that's where captured samplers land, not directly under Thread Group

## 2. Configure the Recorder itself

Key fields in the Recorder pane:
- **Port**: default 8888 — change only if it's in use
- **HTTPS Domains**: leave blank unless restricting to specific domains
- **Target Controller**: point this to the Recording Controller you want captured requests to nest under
- **Grouping**: usually "Store 1st sampler of each group only" or "Don't group samplers" depending on whether you want each redirect/resource as separate samplers — for correlation work, keep it granular initially, group later
- **Capture HTTP Headers**: enable if you need header-based correlation (auth tokens, session headers) — for your OSS/BSS work, keep this ON, you'll almost always need it
- **Add Assertions**: leave off during recording, add manually after cleanup
- **Regex matching for URL Patterns to Include/Exclude**: this is where you filter noise

## 3. Filter static content (important)

Under **URL Patterns to Exclude**, add regex to skip junk:
```
.*\.js
.*\.css
.*\.png
.*\.jpg
.*\.gif
.*\.ico
.*\.woff.*
```
Without this, your recording fills with CSS/JS/image requests that add zero value to a backend load test.

## 4. Install the JMeter root CA certificate (for HTTPS)

- Click **Start** on the Recorder — this auto-generates `ApacheJMeterTemporaryRootCA.crt` in your JMeter `bin/` folder
- Import that cert into your browser's trusted root store (or OS keychain), or use a dedicated recording browser profile with the cert imported — don't do this on your regular daily-use browser profile for security hygiene
- Without this step, every HTTPS site will throw certificate warnings and some sites will simply refuse to load

## 5. Configure browser proxy

- Set browser (or system) proxy to `127.0.0.1` (or JMeter's machine IP if remote) on the port you set in step 2 (e.g. 8888)
- Easiest: use a dedicated Firefox profile with **FoxyProxy** or manual proxy settings — avoids polluting your main browsing session and avoids proxy conflicts with corporate VPN/proxy setups (relevant if you're recording against a client VPN-gated environment like CelcomDigi's)

## 6. Start recording

- Click **Start** in the Recorder panel (JMeter will prompt to trust the cert on first launch if not done)
- Navigate through the application flow in the browser exactly as an end user would — login, transaction steps, logout
- Watch the Recording Controller populate live with samplers as you click through

## 7. Stop and clean up

- Click **Stop**
- Now the real work starts — raw recordings are messy:
  - **Correlate dynamic values**: session IDs, CSRF tokens, auth tokens — replace hardcoded captured values with extractors (Regular Expression Extractor / JSON Extractor / Boundary Extractor) referencing prior response
  - **Parameterize**: swap hardcoded test data (customer IDs, MSISDNs, order refs) with CSV Data Set Config variables
  - **Remove redundant samplers**: duplicate polling calls, analytics beacons, favicon requests that slipped past your filter
  - **Add Response Assertions**: recorder doesn't add these — you add them post-hoc to validate correctness, not just status code
  - **Group logically**: use Transaction Controllers to wrap each business step (e.g., "Login," "Search Subscriber," "Submit Order") for meaningful response-time reporting later

## 8. Move out of Recording Controller

- Once cleaned, either leave scripts under the Recording Controller (fine for reuse) or cut/paste into your actual Thread Group structure — Recording Controller itself doesn't execute during a real load test if it's outside the active Thread Group, so don't forget this step or your "recorded" script silently does nothing when you hit Run

## Common gotchas worth flagging for you specifically

- **SSO/OAuth flows** (common in telecom OSS/BSS with SAML/OAuth-gated portals): the recorder often captures redirect chains messily — you'll likely need to manually rebuild the token exchange rather than trust the raw capture
- **Mobile app backends**: if you're recording API traffic from a mobile app instead of browser, point the app's proxy settings to JMeter the same way, but cert installation differs by OS (Android requires installing as a user cert and, on Android 7+, apps targeting API 24+ ignore user certs unless the app explicitly opts in — a real blocker you may hit if the OSS/BSS system has a companion app)
- **WebSocket/long-polling traffic**: the standard recorder doesn't capture these natively — you'd need the jpgc WebSocket Sampler plugin scripted manually instead

Here's a clean, simple breakdown you can copy into your notebook — organized by category so it's easy to write out.

## What is a Sampler?
A Sampler sends an actual request to the server and waits for response. It's the only element that generates real traffic — everything else (Timers, Assertions, Extractors) just supports it.

---

## BUILT-IN SAMPLERS

**Web / HTTP**
- **HTTP Request** — sends HTTP/HTTPS requests (GET, POST, PUT, DELETE etc.) → 90% of all real-world usage, your bread and butter

**Web Services**
- **SOAP/XML-RPC Request** — legacy SOAP calls (older telecom OSS systems still use this)
- **HTTP Request (supports SOAP body too, more common now)**

**Java**
- **Java Request** — calls a custom Java class implementing JMeter's sampler interface → used for custom in-house protocol testing

**Database**
- **JDBC Request** — runs SQL queries directly against DB → used for backend/DB load testing, or validating data post-transaction

**Messaging / Queue**
- **JMS Point-to-Point** — sends/receives via JMS queue
- **JMS Publisher** / **JMS Subscriber** — pub-sub messaging (Topic-based)
→ relevant if OSS/BSS system uses JMS-based order queues

**FTP**
- **FTP Request** — upload/download files via FTP → rare, used for legacy file-transfer-based provisioning systems

**TCP**
- **TCP Sampler** — raw TCP socket requests → used for custom binary protocols, non-HTTP systems

**LDAP**
- **LDAP Request** / **LDAP Extended Request** — directory lookups/auth → used for subscriber/user directory validation in telecom

**Mail**
- **Mail Reader Sampler** — reads from POP3/IMAP mailbox → used to validate OTP/notification emails sent by system under test

**OS Process**
- **OS Process Sampler** — runs a native OS command/script → used for local system checks, invoking external scripts mid-test

**Scripting**
- **JSR223 Sampler** — runs Groovy/Java/JS code as a sampler → used for custom logic, calling APIs not natively supported, complex transformations
- **BeanShell Sampler** — older scripting sampler (JSR223 replaced this, avoid using now)
- **BSF Sampler** — even older, deprecated, avoid

**Test Fragment / Flow**
- **Test Action** — pause or stop thread/test (not real traffic, control action)
- **Debug Sampler** — outputs JMeter variables/properties as fake response → used to debug what variables are holding at that point in script
- **Flow Control Action** — same as Test Action, newer name in recent JMeter versions

**Access Log**
- **Access Log Sampler** — replays traffic from a web server access log file → rare, used to replay real production traffic patterns

---

## PLUGIN SAMPLERS (jpgc / third-party)

**Custom Samplers (jpgc)**
- **Dummy Sampler** — fakes a response without sending real traffic → used to test assertions/extractors/listeners logic without hitting real server

**WebSocket Sampler (jpgc)**
- Handles WebSocket connections (open/send/close) → used for chat, real-time notification, live-status systems (relevant if OSS portal has live order-status push)

**MQTT Sampler (jpgc / emqx plugin)**
- Publishes/subscribes MQTT messages → used for IoT/device telemetry testing

**gRPC Sampler (third-party plugin)**
- Sends gRPC requests → used for modern microservices using gRPC instead of REST

**Kafka Sampler (Kafkameter or pepper-box plugin)**
- Produces/consumes Kafka messages directly as a sampler → very relevant to you, since you already work on Kafka queue performance — this lets you load-test producer/consumer throughput directly in JMeter instead of external scripts

**GraphQL Sampler (third-party plugin)**
- Sends GraphQL queries/mutations → used if backend exposes GraphQL APIs instead of REST

---

## SIMPLE TABLE FOR YOUR NOTEBOOK

| Sampler | Protocol | When to use |
|---|---|---|
| HTTP Request | HTTP/HTTPS | 90% of web/API load testing |
| JDBC Request | SQL/DB | DB-level load or data validation |
| JMS P2P/Pub-Sub | Messaging | Queue-based systems |
| TCP Sampler | Raw TCP | Custom binary protocols |
| LDAP Request | LDAP | Directory/auth lookups |
| JSR223 Sampler | Script | Custom logic, unsupported protocols |
| Debug Sampler | N/A | Debugging variables |
| Dummy Sampler (jpgc) | N/A | Testing script logic without real calls |
| WebSocket Sampler (jpgc) | WebSocket | Real-time/push systems |
| Kafka Sampler (plugin) | Kafka | Kafka producer/consumer load testing |
| gRPC Sampler (plugin) | gRPC | Microservices using gRPC |
| GraphQL Sampler (plugin) | GraphQL | GraphQL API testing |

Given your background, **HTTP Request, JDBC Request, JSR223 Sampler, and Kafka Sampler** are the ones you'll realistically use most in telecom OSS/BSS work — worth starring those in your notes.

Here's the same notebook-style breakdown for Config Elements.

## What is a Config Element?
A Config Element doesn't send any traffic itself — it sets up default values, variables, or connection settings that Samplers use. It runs before samplers execute, in top-down/scope order.

---

## BUILT-IN CONFIG ELEMENTS

**HTTP related**
- **HTTP Request Defaults** — sets common values (server name, port, protocol, path prefix) once, so you don't repeat them in every HTTP Request → almost always the first thing you add in any HTTP test plan
- **HTTP Header Manager** — sets common headers (Content-Type, Authorization, custom headers) sent with every request → essential for token-based auth, API testing
- **HTTP Cookie Manager** — technically listed separately in JMeter but functions as config — manages session cookies automatically across requests → needed for any login-based flow
- **HTTP Authorization Manager** — stores Basic/NTLM/Digest auth credentials → used when app uses basic auth instead of token-based

**Data-driven testing**
- **CSV Data Set Config** — reads data from a CSV file and feeds a new row to each thread/iteration → your most-used element for parameterizing test data (customer IDs, MSISDNs, order refs)
- **User Defined Variables** — sets static variables once at test-plan level → good for constants (base URL, environment name, API version) that don't change per iteration

**Database**
- **JDBC Connection Configuration** — defines DB connection pool (driver, URL, credentials) used by JDBC Request samplers → required before any JDBC Request works

**Java / Keystore**
- **Java Request Defaults** — same idea as HTTP Defaults but for Java Sampler
- **Keystore Configuration** — used for client-certificate-based HTTPS testing (mutual TLS) → relevant if OSS/BSS system uses mTLS for API security

**FTP**
- **FTP Request Defaults** — default server/port/path for FTP Sampler

**LDAP**
- **LDAP Request Defaults** — default LDAP connection settings

**Random / Counter**
- **Random Variable** — generates a random number/string into a variable → used for unique test data (random order IDs to avoid DB collisions)
- **Counter** — increments a number each iteration → used for sequential unique IDs across threads

**TCP**
- **TCP Sampler Config** — default host/port/timeout for TCP Sampler

---

## PLUGIN CONFIG ELEMENTS

**jpgc — Custom Config**
- **Throughput Shaping Timer** (technically a Timer, but functions like a config for pacing) — often mentioned alongside Concurrency Thread Group for TPS-based load shaping (already covered earlier)

**Kafka-related (Kafkameter / pepper-box)**
- **Kafka Connection Config** — sets broker list, topic, serializer settings → pairs with Kafka Sampler for your Kafka load work

**Redis (third-party plugin)**
- **Redis Data Set Config** — pulls test data dynamically from a Redis cache instead of static CSV → used when test data needs to be shared/live across distributed injectors

**Database Connection pooling alt**
- Some third-party plugins offer **NoSQL Config Elements** (MongoDB, Cassandra) → used for load-testing NoSQL-backed systems directly, rare but exists in telecom billing/CDR systems

---

## SIMPLE TABLE FOR YOUR NOTEBOOK

| Config Element | Purpose | When to use |
|---|---|---|
| HTTP Request Defaults | Common server/port/path | Every HTTP test plan |
| HTTP Header Manager | Common headers | Auth tokens, content-type |
| HTTP Cookie Manager | Session handling | Login-based flows |
| HTTP Authorization Manager | Basic/NTLM/Digest auth | Non-token auth systems |
| CSV Data Set Config | Data-driven testing | Parameterize with real data |
| User Defined Variables | Static constants | Base URL, env name |
| JDBC Connection Config | DB connection pool | Before any JDBC Request |
| Keystore Configuration | mTLS/client cert | Secured API endpoints |
| Random Variable | Random unique values | Avoid data collision |
| Counter | Sequential unique values | Unique IDs per thread/loop |
| Kafka Connection Config (plugin) | Kafka broker/topic setup | Kafka producer/consumer load |
| Redis Data Set Config (plugin) | Live shared test data | Distributed load, live data pool |

For your OSS/BSS + Kafka background, the ones you'll use in nearly every script are: **HTTP Request Defaults, HTTP Header Manager, HTTP Cookie Manager, CSV Data Set Config, User Defined Variables**, plus **Kafka Connection Config** whenever you're doing queue-side testing.


Here are practical, ready-to-use examples for each — values you can literally copy into the fields.

---

## 1. Constant Timer
**Config:** Thread Delay = `3000` (ms)
**Meaning:** Every thread waits exactly 3 seconds before firing next sampler.
**Use case:** Login page → wait 3s (simulating user reading screen) → click Submit.

---

## 2. Uniform Random Timer
**Config:**
- Random Delay Maximum = `2000` (ms)
- Constant Delay Offset = `1000` (ms)

**Meaning:** Delay = 1000ms (always) + random value between 0–2000ms → total delay ranges 1000ms to 3000ms.
**Use case:** Browsing between product pages — no two users wait the exact same time, avoids unrealistic synced traffic spikes.

---

## 3. Gaussian Random Timer
**Config:**
- Deviation = `500` (ms)
- Constant Delay Offset = `2000` (ms)

**Meaning:** Delay centers around 2000ms, with most values falling within ±500ms (occasionally more, rarely much more — bell curve).
**Use case:** Simulating natural human think-time on a form-filling page — most users take ~2s, few take much longer/shorter.

---

## 4. Constant Throughput Timer
**Config:**
- Target throughput = `600` (samples/minute = 10 requests/sec)
- Calculate Throughput based on = **"all active threads in current thread group"**

**Meaning:** JMeter auto-inserts delays so total requests across all threads stay at ~10 req/sec, regardless of thread count.
**Use case:** SLA says "system must handle 600 order submissions/min" — set threads to any reasonable number (e.g., 50), let this timer throttle to exactly 600/min.

---

## 5. Precise Throughput Timer
**Config:**
- Target throughput (per unit) = `600`
- Throughput unit = **requests per minute**
- Test duration = `1800` (seconds = 30 min)
- Random seed = leave default (0)
- Allowed throughput surplus = `1.0`

**Meaning:** Same goal as Constant Throughput Timer (600/min) but distributes requests more evenly using a proper probability model instead of just "catch up if behind."
**Use case:** Same SLA target as above, but for longer/steadier test runs where accuracy over time matters more (e.g., 30-min soak at fixed TPS).

---

## 6. Synchronizing Timer
**Config:**
- Number of Simultaneous Users to Group by = `50`
- Timeout in ms = `5000`

**Meaning:** JMeter holds threads (up to 50) until all 50 are ready and waiting, then releases them at the exact same instant. If 50 aren't ready within 5000ms, releases whatever's waiting anyway (timeout safety).
**Use case:** Simulating a flash-sale or billing-cycle-triggered burst — e.g., 50 subscribers all get provisioned at midnight simultaneously.

---

## 7. JSR223 Timer
**Config (Groovy script in the script box):**
```groovy
// Longer delay during simulated "peak hours", shorter otherwise
def hour = new Date().getHours()
if (hour >= 9 && hour <= 11) {
    return 500   // busy hour - shorter delay = higher load
} else {
    return 3000  // off-peak - longer delay = lighter load
}
```
**Meaning:** Delay value is calculated dynamically by code, not fixed.
**Use case:** Modeling variable pacing based on time-of-day, previous response time, or a variable extracted earlier in the script (e.g., wait longer if last response was slow — adaptive throttling).

---

## 8. Throughput Shaping Timer (jpgc plugin)
**Config (in the Timer's graph/table, added via Composite Graph):**

| Start TPS | End TPS | Duration |
|---|---|---|
| 0 | 100 | 300s (ramp up) |
| 100 | 100 | 600s (hold peak) |
| 100 | 10 | 300s (ramp down) |

**Meaning:** Throughput climbs from 0→100 TPS over 5 min, holds steady at 100 TPS for 10 min, then tapers down to 10 TPS over 5 min.
**Use case:** Simulating a realistic telecom busy-hour curve — e.g., morning provisioning surge, sustained peak, evening taper — paired with **Concurrency Thread Group** to supply enough threads to hit that shape.

---

## Quick copy-paste reference table for notebook

| Timer | Sample values |
|---|---|
| Constant Timer | 3000 ms |
| Uniform Random Timer | Offset 1000, Max 2000 |
| Gaussian Random Timer | Offset 2000, Deviation 500 |
| Constant Throughput Timer | 600 samples/min |
| Precise Throughput Timer | 600/min, duration 1800s |
| Synchronizing Timer | 50 users, timeout 5000ms |
| JSR223 Timer | Custom Groovy delay logic |
| Throughput Shaping Timer | 0→100→100→10 TPS over 300/600/300s |



Here's the notebook-style breakdown for Listeners.

## What is a Listener?
A Listener collects and displays the results of a test run — response times, status codes, throughput, errors. It doesn't affect what gets sent, only how results are shown/saved. **Important rule to note**: Listeners are heavy on memory/CPU — always disable them during actual load runs and only enable during script debugging/validation. For real load tests, log to file and analyze after.

---

## BUILT-IN LISTENERS

**Table/Tree view (debugging)**
- **View Results Tree** — shows request/response in full detail (headers, body, timing) per sample → best for debugging a script line-by-line, but extremely memory-heavy — never use in load tests, only in single-thread debug runs
- **View Results in Table** — tabular summary (response code, time, bytes) per sample → lighter than Results Tree, still mainly for debugging

**Summary / Aggregate stats**
- **Summary Report** — table of stats (avg, min, max, error%, throughput) per sampler name → your main go-to for quick pass/fail review after a run
- **Aggregate Report** — same as Summary Report but adds median, 90/95/99th percentile → this is the one you actually want for SLA reporting since percentiles matter more than averages in performance work
- **Aggregate Graph** — same data as Aggregate Report but as a bar graph → useful for quick visual comparison across samplers

**Graphs**
- **Graph Results** — plots response time over time, live → visually intuitive during a demo, but very resource-heavy, avoid in real runs
- **Response Time Graph** — cleaner response-time-only line graph
- **Response Time Distribution** — distribution histogram of response times → helpful to check if response times are consistent or highly variable

**Throughput-related**
- **Transactions per Second (TPS) Listener** — Wait, this is actually available as a jpgc feature (see below), not built-in — noting this so you don't confuse it

**Error-specific**
- **Assertion Results** — shows pass/fail detail for each Assertion applied → used when debugging why a specific assertion is failing

**File-based (most important for real load tests)**
- **Simple Data Writer** — writes raw results directly to a `.jtl` file without any live UI/graphs → this is what you actually use in real load tests — enable this, disable everything else visual, then analyze the `.jtl` afterward

**Backend**
- **Backend Listener** — streams live results to an external monitoring backend (e.g., InfluxDB + Grafana, or Datadog) in real-time → this is the big one for you given your Grafana/Datadog stack — lets you watch a live dashboard during the run instead of waiting for post-test report

---

## PLUGIN LISTENERS (jpgc — Custom Listeners plugin)

- **jp@gc - Response Times Over Time** — graph of response time trend across the run duration
- **jp@gc - Transactions per Second** — graph of actual TPS achieved over time → directly compare against your target TPS from Throughput Shaping Timer
- **jp@gc - Active Threads Over Time** — graph of concurrent thread count over time → useful to visually confirm your ramp-up/step shape actually executed as designed
- **jp@gc - Response Codes per Second** — graph of status codes (200, 500, etc.) over time → quickly spot when errors started spiking
- **jp@gc - Bytes Throughput Over Time** — network throughput graph → useful for bandwidth-constrained scenarios
- **jp@gc - Response Latencies Over Time** — latency (time to first byte) specifically, separate from full response time
- **jp@gc - PerfMon Metrics Collector** — displays CPU/memory/disk stats from the PerfMon Server Agent running on the SUT → pairs with your Dynatrace/Datadog monitoring to correlate JMeter load with server-side resource usage directly inside JMeter itself

---

## SIMPLE TABLE FOR YOUR NOTEBOOK

| Listener | Purpose | When to use |
|---|---|---|
| View Results Tree | Full request/response detail | Script debugging only |
| View Results in Table | Tabular per-sample detail | Light debugging |
| Summary Report | Basic stats (avg/min/max/error%) | Quick post-run check |
| Aggregate Report | Adds percentiles (90/95/99th) | SLA reporting — main one |
| Aggregate Graph | Bar graph of aggregate stats | Visual comparison |
| Response Time Graph | Live response time trend | Demo/visual only |
| Response Time Distribution | Histogram of response times | Consistency check |
| Assertion Results | Pass/fail detail per assertion | Debugging failed assertions |
| Simple Data Writer | Writes raw results to .jtl file | **Real load tests — always use this** |
| Backend Listener | Streams live data to Grafana/Datadog/InfluxDB | Live dashboard during run |
| jp@gc Transactions per Second | TPS achieved over time | Compare vs target TPS |
| jp@gc Active Threads Over Time | Thread ramp shape verification | Confirm load shape executed correctly |
| jp@gc Response Codes per Second | Error spike timing | Spot when failures started |
| jp@gc PerfMon Metrics Collector | Server CPU/mem/disk in JMeter | Correlate load with server resources |

**Golden rule for your actual test runs**: disable all UI listeners (Results Tree, Graph Results, View Results in Table) → keep only **Simple Data Writer** (writes to `.jtl`) and optionally **Backend Listener** (if streaming to Grafana live) → generate the **HTML Dashboard Report** from the `.jtl` file after the run completes (`jmeter -g results.jtl -o /report-folder`) for the full percentile/graph breakdown.

Given your Dynatrace/Datadog/Grafana background, **Backend Listener + PerfMon Metrics Collector** is the combo most worth mastering — it lets you correlate JMeter-side load metrics with server-side resource metrics in one place instead of cross-referencing two separate tools after the fact.

Here's the notebook-style breakdown for Assertions.

## What is an Assertion?
An Assertion checks whether the response is actually correct — not just "did I get a response" but "is this the right response." Without assertions, JMeter only flags HTTP-level failures (like 500 errors) — it won't catch a "200 OK" response that silently returns wrong or broken data. This is where real functional validation happens inside a performance test.

---

## BUILT-IN ASSERTIONS

**Response content checks**
- **Response Assertion** — checks response body/headers/code against text, regex, or "contains/matches/equals" pattern → your most-used assertion, e.g., checking response contains "SUCCESS" or doesn't contain "error"
- **Size Assertion** — checks response size (bytes) is within expected range → used to catch truncated/incomplete responses that still return 200 OK

**Structured data checks**
- **JSON Assertion** — validates a JSON path exists and optionally matches an expected value → used heavily in modern REST API testing, e.g., checking `$.orderStatus` equals `"CONFIRMED"`
- **JSON JMESPath Assertion** — same idea as JSON Assertion but uses JMESPath query syntax (more powerful filtering/expressions) → used for complex nested JSON validation
- **XPath Assertion / XPath2 Assertion** — validates XML response structure/value using XPath → used for SOAP/XML-based legacy telecom systems
- **XML Assertion** — simply checks the response is well-formed XML → basic structural check, not value-based

**Timing checks**
- **Duration Assertion** — fails the sample if response time exceeds a set threshold (ms) → used to enforce SLA response-time limits directly in the script (e.g., fail if >2000ms), separate from just reporting slow times later

**Protocol-specific**
- **HTML Assertion** — validates response HTML against W3C/JTidy standards → rare, used for strict HTML-compliance checking, not typical in API/backend testing
- **SOAP Assertion / SMIME Assertion** — SOAP-specific and email-signature-specific checks → niche, legacy SOAP systems only

**Comparison**
- **Compare Assertion** — compares response of a sampler against a previous "baseline" response → used in regression-style testing, verifying response hasn't unexpectedly changed between test runs

**Scripted**
- **JSR223 Assertion** — write custom Groovy logic to decide pass/fail → used when validation logic is too complex for standard assertions (e.g., cross-checking two different response fields against each other, or a calculated business rule)

**BeanShell Assertion**
- Older scripting version → avoid, use JSR223 Assertion instead

---

## Assertion helper (not an assertion itself, but pairs with it)

- **Assertion Results (Listener)** — displays pass/fail detail for assertions, already covered under Listeners

---

## PLUGIN ASSERTIONS

Assertions are mostly a core JMeter category — jpgc doesn't add much here since community focus went into Thread Groups/Listeners/Samplers instead. The one notable addition:

- **JSON Assertion (jpgc's older plugin version, now merged into core JMeter)** — historically was a plugin, now built-in as of modern JMeter, so no separate install needed anymore

---

## SIMPLE TABLE FOR YOUR NOTEBOOK

| Assertion | Checks | When to use |
|---|---|---|
| Response Assertion | Text/regex in body/headers/code | Most common — general content validation |
| Size Assertion | Response size in bytes | Catch truncated/incomplete responses |
| JSON Assertion | JSON path value | REST API field validation |
| JSON JMESPath Assertion | Complex JSON queries | Nested/filtered JSON validation |
| XPath / XPath2 Assertion | XML structure/value via XPath | SOAP/XML legacy systems |
| XML Assertion | Well-formed XML check | Basic XML structure check |
| Duration Assertion | Response time threshold | Enforce SLA time limits in-script |
| HTML Assertion | HTML standard compliance | Rare, strict HTML validation |
| SOAP Assertion | SOAP-specific checks | Legacy SOAP systems |
| Compare Assertion | Compares against baseline response | Regression testing |
| JSR223 Assertion | Custom scripted pass/fail logic | Complex/custom business rule validation |

**Scope rule to note down** (same as Timers): an assertion applies to whatever sampler it's placed under/inside. Drop it inside one HTTP Request → only checks that one. Drop it at Thread Group level → checks every sampler in that group.

For your OSS/BSS + REST/JSON work, the ones you'll use constantly are: **Response Assertion, JSON Assertion, and Duration Assertion** — that trio covers content correctness, structured data correctness, and SLA timing in nearly every script.



Here's the notebook-style breakdown for Pre-Processors.

## What is a Pre-Processor?
A Pre-Processor runs code/logic **before** a sampler fires — it modifies or prepares something (a variable, a request field, the sampler itself) right before the request goes out. Think of it as "setup work done just-in-time, per request" — different from Config Elements which set values once at a higher scope.

---

## BUILT-IN PRE-PROCESSORS

**HTTP specific**
- **HTTP URL Re-writing Modifier** — rewrites URLs to carry session ID in the URL path instead of cookies → used for legacy apps that do URL-based session tracking instead of cookie-based (rare today, but some old telecom portals still do this)

**Parameter/data manipulation**
- **User Parameters** — sets per-thread variable values (different value per simulated user) right before the sampler → used to assign each thread its own set of values inline in the tree, alternative to CSV Data Set Config for small fixed data sets

**Sampler modification**
- **HTML Link Parser** — automatically parses HTML response and modifies next request's URL if needed → older technique from pre-correlation-extractor days, rarely used now (Extractors do this job better)

**Scripting**
- **JSR223 PreProcessor** — write custom Groovy/Java logic before the sampler runs → this is the big one you'll actually use — dynamic header generation, custom auth signature calculation, request body manipulation, timestamp injection, encryption/hashing before sending
- **BeanShell PreProcessor** — older scripting version, avoid, use JSR223 instead

**Sample timing**
- **RegEx User Parameters** — applies regex-extracted values as parameters before the sampler → niche, mostly superseded by Regular Expression Extractor + variable reference

---

## PLUGIN PRE-PROCESSORS

Pre-Processors, like Assertions, are mostly a core-JMeter category — jpgc doesn't add many here. The most notable third-party additions:

- **Custom JSR223 snippets from community libraries** — not really separate elements, just shared Groovy scripts people reuse inside the built-in JSR223 PreProcessor (e.g., HMAC signature generation scripts for API security testing)

There's no major dedicated "plugin pre-processor" ecosystem the way there is for Thread Groups/Listeners — this category is deliberately kept simple in core JMeter, and JSR223 covers basically anything custom you'd need.

---

## SIMPLE TABLE FOR YOUR NOTEBOOK

| Pre-Processor | Purpose | When to use |
|---|---|---|
| HTTP URL Re-writing Modifier | URL-based session tracking | Legacy apps without cookie sessions |
| User Parameters | Per-thread inline variable values | Small fixed data sets, alt to CSV |
| HTML Link Parser | Auto-parse HTML for next URL | Rare, mostly replaced by Extractors |
| JSR223 PreProcessor | Custom scripted logic before request | Dynamic headers, signatures, timestamps, encryption |
| RegEx User Parameters | Regex-based param injection | Niche, mostly replaced by Extractor + variable |

---

## Practical example most relevant to you

Given your OSS/BSS + API work, the classic real-world use of **JSR223 PreProcessor** is generating a request signature or timestamp header that the API requires fresh on every call:

```groovy
// Add current timestamp + a calculated signature header before each request
def timestamp = System.currentTimeMillis().toString()
vars.put("reqTimestamp", timestamp)

def rawString = vars.get("apiKey") + timestamp
def signature = rawString.digest("SHA-256")  // simplified example
vars.put("reqSignature", signature)
```
Then in the HTTP Header Manager, you'd reference `${reqTimestamp}` and `${reqSignature}` — values are freshly generated per-request instead of static.

**Scope rule** (same pattern as Timers/Assertions): a Pre-Processor placed inside a sampler runs only before that sampler; placed at Thread Group level, it runs before every sampler in that group.

Want **Post-Processors/Extractors** next — that's the natural next step since PreProcessor sets things up, but Extractors are what actually pull dynamic values (tokens, IDs) out of responses to use later?It looks like your message came through empty — no worries, I'll continue with what we lined up: **Post-Processors/Extractors**, since that's the natural next step after Pre-Processors.

## What is a Post-Processor?
A Post-Processor runs **after** a sampler gets its response — it pulls (extracts) a value out of that response so you can reuse it in later requests. This is the backbone of **correlation** — without it, dynamic values like session tokens, order IDs, or auth codes stay hardcoded and your script breaks the moment the server generates a new value.

---

## BUILT-IN POST-PROCESSORS (Extractors)

**Text/Regex based**
- **Regular Expression Extractor** — pulls a value out of response body/headers using regex → the classic, most-used extractor, works on any response format (HTML, JSON, XML, plain text)

**Structured data**
- **JSON Extractor** — pulls a value using JSONPath syntax (`$.data.orderId`) → cleaner and safer than regex for JSON APIs, your go-to for REST responses
- **JSON JMESPath Extractor** — same idea, more powerful JMESPath query syntax for complex nested/filtered JSON extraction
- **XPath Extractor / XPath2 Extractor** — pulls a value from XML using XPath → used for SOAP/XML legacy telecom responses
- **CSS/JQuery Extractor** — pulls a value using CSS selector syntax → used for HTML page scraping-style extraction (less common in API testing, more for web-page-heavy apps)

**Simple substring**
- **Boundary Extractor** — pulls value between a "left boundary" and "right boundary" string → simpler and faster than regex when the value has clear fixed text around it (e.g., between `"token":"` and `"`)

**Result-level**
- **Result Status Action Handler** — not really an extractor, technically listed here — controls what happens (stop thread/test) based on sample result → edge case usage

**Scripting**
- **JSR223 PostProcessor** — custom Groovy/Java logic after the response → used when extraction logic is too complex for standard extractors (parsing + transforming + conditional logic together), or when you need to do something with the response besides just extracting (e.g., decode base64, compute a hash, write to external file)
- **BeanShell PostProcessor** — older version, avoid, use JSR223 instead

---

## PLUGIN POST-PROCESSORS

Same pattern as Assertions/Pre-Processors — this is a core JMeter category, jpgc doesn't add dedicated new ones. JSR223 PostProcessor covers custom needs.

---

## SIMPLE TABLE FOR YOUR NOTEBOOK

| Post-Processor | Extracts from | When to use |
|---|---|---|
| Regular Expression Extractor | Any text (HTML/JSON/XML/plain) | Universal — most flexible, most used |
| JSON Extractor | JSON (JSONPath) | Clean REST API extraction |
| JSON JMESPath Extractor | JSON (JMESPath) | Complex/nested JSON queries |
| XPath / XPath2 Extractor | XML (XPath) | SOAP/XML legacy systems |
| CSS/JQuery Extractor | HTML (CSS selector) | Web page scraping style |
| Boundary Extractor | Text between two markers | Fast, simple extraction |
| JSR223 PostProcessor | Custom scripted logic | Complex parsing/transformation |

---

## Practical correlation example (session token)

**Regular Expression Extractor config:**
- Apply to: Main sample
- Field to check: Body
- Reference Name: `sessionToken`
- Regular Expression: `"token":"(.*?)"`
- Template: `$1$`
- Match No.: `1`

Then in your next HTTP Request's header or body, reference it as `${sessionToken}`.

**Same thing with JSON Extractor (cleaner for JSON APIs):**
- Names of created variables: `sessionToken`
- JSON Path expressions: `$.token`
- Match No.: `1`

**Scope rule** (same as always): a Post-Processor placed inside a sampler extracts only from that sampler's response; at Thread Group level it'd apply to every sampler — but in practice, extractors are almost always placed *inside* the specific sampler whose response holds the value you need.

For your OSS/BSS work, **Regular Expression Extractor + JSON Extractor** will cover nearly everything — regex when the API returns plain text/HTML fragments, JSON Extractor when it's clean REST JSON. Use **JSR223 PostProcessor** only when you need to do more than just pull a value (e.g., decode/transform it before storing).

Got it — here's the clean notebook version for Post-Processors (Extractors).

## What is a Post-Processor?
Runs **after** a sampler gets its response, and pulls a value out of it to reuse later. This is the core mechanism for **correlation** — capturing dynamic values like session tokens, order IDs, OTPs so your script doesn't break when the server generates fresh values each run.

---

## BUILT-IN POST-PROCESSORS

| Post-Processor | Extracts from | When to use |
|---|---|---|
| **Regular Expression Extractor** | Any text (HTML/JSON/XML/plain) | Universal — most used, works everywhere |
| **JSON Extractor** | JSON (JSONPath syntax) | Clean, safe REST API field extraction |
| **JSON JMESPath Extractor** | JSON (JMESPath syntax) | Complex/nested/filtered JSON queries |
| **XPath Extractor / XPath2 Extractor** | XML (XPath syntax) | SOAP/XML legacy telecom responses |
| **CSS/JQuery Extractor** | HTML (CSS selector) | Web page scraping, less common in API testing |
| **Boundary Extractor** | Text between two fixed markers | Fast/simple when value has clear surrounding text |
| **JSR223 PostProcessor** | Custom scripted logic | Complex parsing, decoding, hashing, conditional logic |
| **BeanShell PostProcessor** | Same as JSR223, older | Avoid — legacy, slower |

---

## PLUGIN POST-PROCESSORS
No major jpgc additions here — this stays a core JMeter category. JSR223 PostProcessor covers any custom need instead.

---

## Practical example (session token correlation)

**Using Regular Expression Extractor:**
- Field to check: `Body`
- Reference Name: `sessionToken`
- Regex: `"token":"(.*?)"`
- Template: `$1$`
- Match No.: `1`

**Using JSON Extractor (cleaner for JSON APIs):**
- Names of created variables: `sessionToken`
- JSON Path expressions: `$.token`
- Match No.: `1`

Then reuse it anywhere downstream as `${sessionToken}`.

---

**Scope rule** (same pattern as Timers/Assertions/Pre-Processors): place the extractor **inside** the specific sampler whose response holds the value — that's standard practice, rather than at Thread Group level.

For your OSS/BSS work: **Regular Expression Extractor** for messy/plain-text responses, **JSON Extractor** for clean REST JSON — those two will cover almost everything you script.
Here's the notebook-style breakdown for Logic Controllers.

## What is a Logic Controller?
A Logic Controller doesn't send requests itself — it controls the **order, flow, and conditions** under which samplers inside it execute. Think of it as the "if/loop/grouping" logic layer of your test script.

---

## BUILT-IN LOGIC CONTROLLERS

**Grouping/organization**
- **Transaction Controller** — groups multiple samplers into one logical "transaction" and reports combined response time as one → your most-used controller for reporting, e.g., wrap "Login" (which might be 3 HTTP calls) into one Transaction Controller so reports show "Login: 1.2s" instead of 3 separate lines
- **Simple Controller** — just a folder/grouping with no logic, purely organizational → used to visually organize a script (e.g., "Login Flow," "Order Flow") without affecting execution

**Conditional**
- **If Controller** — runs child elements only if a condition (JS/variable expression) is true → used for conditional flows, e.g., only hit "Retry" sampler if previous response failed
- **Switch Controller** — runs one specific child branch based on a variable's value (like a switch/case) → used when you have multiple distinct paths based on a category variable (e.g., different flows per subscriber type: prepaid/postpaid/enterprise)

**Looping**
- **Loop Controller** — repeats child elements a fixed number of times → used for repeating a specific step block inside one iteration
- **While Controller** — repeats child elements while a condition remains true → used when repeat count isn't fixed, depends on a runtime condition (e.g., keep polling order status until it's "COMPLETED")
- **ForEach Controller** — loops through a set of variables (usually from an Extractor that captured multiple matches) → used when a previous response returned a list, and you need to run a sampler once per item (e.g., loop through all returned order IDs and fetch each one's detail)

**Randomization/distribution**
- **Random Controller** — picks one child element at random to execute (not all) → used to simulate varied user behavior — different users take different random paths
- **Random Order Controller** — runs all children but in random order → used when all steps must run but order shouldn't always be identical, more realistic simulation
- **Throughput Controller** — controls what % of iterations, or a fixed number of times, its children run → used to simulate percentage-based behavior splits, e.g., "30% of users go through Express Checkout, 70% go through Standard Checkout"

**Recording**
- **Recording Controller** — target container for the HTTP(S) Test Script Recorder (already covered earlier) → just a landing zone, no logic itself

**Reliability/error handling**
- **Include Controller** — pulls in and runs an entire external `.jmx` test fragment file → used to reuse common flows (like login) across multiple test plans without copy-pasting
- **Module Controller** — runs a Test Fragment defined elsewhere in the *same* test plan → used to reuse a block within one script without duplicating it, keeps script DRY
- **Test Fragment** — not a controller exactly, but the element type Module Controller and Include Controller reference → holds a reusable chunk of test logic, disconnected from the main execution flow unless called

**Interleaving**
- **Interleave Controller** — alternates execution between its children, one per loop iteration (round-robin) → used to alternate between two flows, e.g., iteration 1 hits Server A path, iteration 2 hits Server B path, etc.

**Once-only**
- **Once Only Controller** — runs child elements only on the first iteration of a loop, skips on subsequent ones → used for one-time setup within a thread's lifecycle, e.g., login once, then repeat only the "browse" steps for remaining iterations

**Critical Section**
- **Critical Section Controller** — ensures child elements run one thread at a time (mutex-style lock) → used for testing scenarios requiring serialized access, e.g., validating unique-sequence-number generation isn't broken under concurrency

---

## PLUGIN LOGIC CONTROLLERS

Logic Controllers are also a stable core category — jpgc doesn't significantly extend this. It's considered "complete" in core JMeter for flow-control needs; anything more custom is typically handled via JSR223 Sampler/PreProcessor logic instead.

---

## SIMPLE TABLE FOR YOUR NOTEBOOK

| Controller | Behavior | When to use |
|---|---|---|
| Transaction Controller | Groups samplers, reports combined time | Business-step-level reporting (most used) |
| Simple Controller | Pure grouping, no logic | Script organization only |
| If Controller | Conditional execution | Run steps only if condition true |
| Switch Controller | Branch by variable value | Multi-path flows (subscriber type, etc.) |
| Loop Controller | Fixed repeat count | Repeat a step block N times |
| While Controller | Repeat while condition true | Poll until status changes |
| ForEach Controller | Loop through extracted variable list | Process each item from a prior multi-match extract |
| Random Controller | Runs one random child | Simulate varied user paths |
| Random Order Controller | Runs all children, random order | Realistic non-sequential behavior |
| Throughput Controller | % or fixed-count execution | Percentage-split user behavior |
| Recording Controller | Landing zone for recorder | Script recording only |
| Include Controller | Runs external .jmx fragment | Reuse flow across separate test plans |
| Module Controller | Runs fragment within same plan | Reuse flow within one script |
| Interleave Controller | Alternates children per iteration | Round-robin between paths |
| Once Only Controller | Runs children on first iteration only | One-time setup (e.g., login once) |
| Critical Section Controller | Serializes access (mutex) | Concurrency-sensitive validation |

For your OSS/BSS work, the ones you'll use constantly are: **Transaction Controller** (reporting), **If Controller** (conditional retry/error handling), **ForEach Controller** (processing lists like multiple orders/subscribers), and **Once Only Controller** (login-once-per-thread pattern). **Throughput Controller** is also valuable for modeling realistic traffic splits across different customer journeys (prepaid vs postpaid vs enterprise provisioning, for example).





Here's one clean Groovy example for each JSR223 element type, organized exactly by where they appear in JMeter's right-click **Add** menu.

---

## 1. Add → Sampler → JSR223 Sampler
**Use case:** Custom sampler that doesn't fit HTTP/JDBC — e.g., simulate a delay-based synthetic transaction or call a Java utility class directly.

```groovy
// Simulate a custom "network provisioning check" as a sampler
SampleResult.setSamplerData("Synthetic Provisioning Check")

def startTime = System.currentTimeMillis()

// simulate calling an internal validation routine
def subscriberId = vars.get("subscriberId")
def isValid = subscriberId != null && subscriberId.startsWith("SUB")

Thread.sleep(50) // simulate processing time

def endTime = System.currentTimeMillis()
SampleResult.setStampAndTime(startTime, endTime - startTime)

if (isValid) {
    SampleResult.setResponseData("Provisioning check passed for ${subscriberId}", "UTF-8")
    SampleResult.setSuccessful(true)
    SampleResult.setResponseCode("200")
} else {
    SampleResult.setResponseData("Invalid subscriber ID", "UTF-8")
    SampleResult.setSuccessful(false)
    SampleResult.setResponseCode("400")
}
```
**Why used:** when you need a fully custom "request" that isn't really HTTP/DB/Kafka — a placeholder synthetic check, or a wrapper calling a Java SDK method directly.

---

## 2. Add → Timer → JSR223 Timer
**Use case:** Adaptive pacing based on previous response time (already covered, repeated here in its correct category slot).

```groovy
def lastResponseTime = prev?.getTime() ?: 0

if (lastResponseTime > 3000) {
    log.info("Slow response (${lastResponseTime}ms) — backing off pacing")
    return 5000
} else if (lastResponseTime > 1000) {
    return 2000
} else {
    return 500
}
```
**Why used:** self-throttling pacing model — avoids naive fixed delay when server is already under stress.

---

## 3. Add → Pre Processors → JSR223 PreProcessor
**Use case:** Generate a signed request header (HMAC) before an OCS charging call.

```groovy
import javax.crypto.Mac
import javax.crypto.spec.SecretKeySpec

def apiKey = vars.get("apiKey")
def secret = vars.get("apiSecret")
def timestamp = System.currentTimeMillis().toString()
def payload = apiKey + timestamp

Mac mac = Mac.getInstance("HmacSHA256")
mac.init(new SecretKeySpec(secret.getBytes(), "HmacSHA256"))
def signature = mac.doFinal(payload.getBytes()).encodeBase64().toString()

vars.put("reqTimestamp", timestamp)
vars.put("reqSignature", signature)
```
**Why used:** anything that must be freshly computed right before the sampler fires — signatures, timestamps, encrypted fields.

---

## 4. Add → Post Processors → JSR223 PostProcessor
**Use case:** Extract order status from JSON response and calculate custom settlement latency metric.

```groovy
import groovy.json.JsonSlurper

def response = new JsonSlurper().parseText(prev.getResponseDataAsString())
vars.put("orderStatus", response.status ?: "UNKNOWN")

def kafkaEventTime = vars.get("kafkaEventTimestamp")
if (kafkaEventTime != null) {
    def latency = System.currentTimeMillis() - kafkaEventTime.toLong()
    vars.put("settlementLatencyMs", latency.toString())
    log.info("Order ${vars.get('orderId')} settlement latency: ${latency}ms")
}
```
**Why used:** whenever extraction needs logic beyond a simple path match — calculations, conditional storage, cross-referencing an earlier variable.

---

## 5. Add → Assertions → JSR223 Assertion
**Use case:** Validate billing amount from API matches DB-extracted expected value.

```groovy
def expectedAmount = vars.get("expectedBillAmount").toDouble()
def actualAmount = vars.get("actualBillAmount").toDouble()
def tolerance = 0.01

if (Math.abs(expectedAmount - actualAmount) > tolerance) {
    AssertionResult.setFailure(true)
    AssertionResult.setFailureMessage("Billing mismatch — expected: ${expectedAmount}, actual: ${actualAmount}")
}
```
**Why used:** business-rule validation that's too complex for Response/JSON Assertion — cross-field or cross-source checks.

---

## 6. Add → Listener → JSR223 Listener
**Use case:** Push a custom summary metric to an external file/monitoring endpoint as results stream in, instead of relying only on the .jtl file.

```groovy
// Runs once per sample result as it's collected
if (sampleEvent != null) {
    def result = sampleEvent.getResult()
    def label = result.getSampleLabel()
    def responseTime = result.getTime()
    def success = result.isSuccessful()

    if (!success) {
        def logLine = "${new Date()} | FAILED | ${label} | ${responseTime}ms | ${result.getResponseCode()}\n"
        new File("/tmp/jmeter_failures.log").append(logLine)
    }
}
```
**Why used:** rare, but useful for lightweight custom logging/alerting hooks alongside standard Listeners — e.g., writing only failures to a separate file for quick triage during a long soak test.

---

## Quick reference table

| Menu location | Element | Typical telecom use |
|---|---|---|
| Sampler | JSR223 Sampler | Custom synthetic transaction |
| Timer | JSR223 Timer | Adaptive pacing |
| Pre Processors | JSR223 PreProcessor | HMAC signature, dynamic headers |
| Post Processors | JSR223 PostProcessor | Custom extraction + latency calc |
| Assertions | JSR223 Assertion | Cross-field business validation (billing) |
| Listener | JSR223 Listener | Custom failure logging/alerting |

**Note on Config Elements:** there's no JSR223 Config Element in JMeter's menu — if you need scripted config-like setup (e.g., computing a value once before the whole test), that's typically done via **JSR223 PreProcessor at Thread Group level** combined with **Once Only Controller**, or via **setUp Thread Group** with a JSR223 Sampler inside it.

Want a **combined script walkthrough** next — one full script trace showing a single request moving through PreProcessor → Sampler → PostProcessor → Assertion, so you can explain the complete lifecycle end-to-end in an interview?
