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








