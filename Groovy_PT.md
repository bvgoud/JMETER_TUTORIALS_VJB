Here's a structured Groovy-for-JMeter learning path, Basic → Advanced, built specifically for interview prep — language concepts paired with JMeter-relevant examples.

---

# LEVEL 1: BASIC GROOVY SYNTAX (in JMeter context)

## 1.1 Variables & Dynamic Typing
```groovy
def name = "Vijay"           // dynamic typing, type inferred
String explicitName = "Vijay" // explicit typing also allowed
int count = 5
def price = 99.99

log.info("Name: ${name}, Count: ${count}")
```
**Interview point:** Groovy is dynamically typed but supports optional static typing — unlike Java, `def` lets the type be inferred at runtime.

## 1.2 String Interpolation (GString)
```groovy
def orderId = vars.get("orderId")
def msg = "Order ${orderId} processed at ${new Date()}"
log.info(msg)

// vs plain Java-style concatenation (works too, but less idiomatic)
def msg2 = "Order " + orderId + " processed"
```
**Interview point:** Double quotes `"..."` support interpolation (GString); single quotes `'...'` are plain Java strings, no interpolation.

## 1.3 Reading/Writing JMeter Variables
```groovy
// vars = thread-local (per virtual user)
vars.put("token", "abc123")
def token = vars.get("token")

// props = JVM-wide (shared across all threads)
props.put("globalCounter", 0)
def counter = props.get("globalCounter")
```
**Interview point:** This distinction is asked in almost every JMeter scripting interview — memorize it cold.

## 1.4 Basic Conditionals
```groovy
def statusCode = prev.getResponseCode()

if (statusCode == "200") {
    log.info("Success")
} else if (statusCode == "401") {
    log.warn("Unauthorized - token may have expired")
} else {
    log.error("Unexpected status: ${statusCode}")
}
```

## 1.5 Loops
```groovy
// for loop
for (int i = 0; i < 5; i++) {
    log.info("Iteration: ${i}")
}

// each (Groovy idiomatic way — preferred over for-loop)
(1..5).each { i ->
    log.info("Iteration: ${i}")
}
```
**Interview point:** Groovy favors closures (`.each`, `.collect`, `.find`) over traditional for-loops — shows you write idiomatic Groovy, not "Java with Groovy syntax."

---

# LEVEL 2: INTERMEDIATE — COLLECTIONS & CLOSURES

## 2.1 Lists
```groovy
def statusList = ["ACTIVE", "PENDING", "SUSPENDED"]
def randomStatus = statusList[new Random().nextInt(statusList.size())]

// filtering
def activeOnly = statusList.findAll { it == "ACTIVE" }

// mapping/transforming
def upperCased = statusList.collect { it.toUpperCase() }
```

## 2.2 Maps (very common for JSON payload building)
```groovy
def order = [
    orderId: "ORD-001",
    customerId: vars.get("customerId"),
    status: "CREATED"
]

log.info("Order status: ${order.status}")   // dot access
log.info("Order status: ${order['status']}") // bracket access, same result
```

## 2.3 Closures
```groovy
// A closure is a reusable block of code, like a lambda
def calculateLatency = { startTime, endTime ->
    return endTime - startTime
}

def latency = calculateLatency(1000L, 1500L)
log.info("Latency: ${latency}ms")
```
**Interview point:** Closures are core to Groovy — used heavily in `.each`, `.collect`, `.findAll`. Being able to explain "a closure is basically a portable, callable block of code with its own scope" is a strong differentiator.

## 2.4 JSON Parsing (JsonSlurper) — used constantly in Post-Processors/Assertions
```groovy
import groovy.json.JsonSlurper

def response = prev.getResponseDataAsString()
def json = new JsonSlurper().parseText(response)

def orderId = json.orderId
def firstItem = json.items[0].sku
```

## 2.5 JSON Building (JsonOutput) — used constantly in Pre-Processors
```groovy
import groovy.json.JsonOutput

def payload = [
    orderId: "ORD-001",
    items: [[sku: "SKU1", qty: 2], [sku: "SKU2", qty: 1]]
]
def jsonString = JsonOutput.toJson(payload)
vars.put("requestBody", jsonString)
```

---

# LEVEL 3: ADVANCED — CONCURRENCY, THREAD SAFETY, PERFORMANCE

## 3.1 Thread-Safe Shared State (props + synchronized collections)
```groovy
// Unsafe (race condition risk under concurrent threads):
// props.put("list", new ArrayList())

// Safe:
if (props.get("sharedList") == null) {
    props.put("sharedList", Collections.synchronizedList(new ArrayList()))
}
def sharedList = props.get("sharedList")
sharedList.add(vars.get("orderId"))
```
**Interview point:** This is a classic "prove you understand concurrency" question — explain why plain `ArrayList`/`HashMap` in `props` is dangerous under multi-threaded JMeter execution, and why you wrap with `Collections.synchronizedX()` or use `java.util.concurrent` classes.

## 3.2 Atomic Counters (better than synchronized for simple counters)
```groovy
import java.util.concurrent.atomic.AtomicInteger

if (props.get("orderCounter") == null) {
    props.put("orderCounter", new AtomicInteger(0))
}
def counter = props.get("orderCounter")
def nextVal = counter.incrementAndGet()
vars.put("uniqueOrderId", "ORD-" + nextVal)
```
**Interview point:** `AtomicInteger` avoids lock contention overhead compared to `synchronized` blocks — matters at high concurrency, a nuance senior candidates are expected to know.

## 3.3 ConcurrentHashMap for Cross-Thread Data Sharing
```groovy
import java.util.concurrent.ConcurrentHashMap

if (props.get("sentTimestamps") == null) {
    props.put("sentTimestamps", new ConcurrentHashMap())
}
def sentMap = props.get("sentTimestamps")
sentMap.put(vars.get("correlationId"), System.currentTimeMillis())
```
**Interview point:** `ConcurrentHashMap` is preferred over `Collections.synchronizedMap(new HashMap())` in high-throughput scenarios because it uses fine-grained locking rather than locking the entire map on every access — worth explicitly mentioning as a performance-aware choice.

## 3.4 Script Compilation Caching Awareness
```groovy
// JMeter caches compiled Groovy scripts by default (since script compilation is expensive)
// Bad practice: writing scripts with logic that changes structurally per iteration
// Good practice: keep script structure static, only data/variables change

// Check "Cache compiled script if available" checkbox in JSR223 elements — always keep ON
```
**Interview point:** Explaining that JMeter/Groovy caches compiled scripts (and why leaving that checkbox enabled matters for injector CPU efficiency at scale) is a senior-level performance-of-the-tool-itself insight, not just performance-of-the-system-under-test.

## 3.5 Exception Handling & Graceful Failure
```groovy
try {
    def json = new groovy.json.JsonSlurper().parseText(prev.getResponseDataAsString())
    vars.put("orderId", json.orderId.toString())
} catch (Exception e) {
    log.error("Failed to parse response for correlation: ${e.message}")
    vars.put("orderId", "PARSE_ERROR")
    AssertionResult.setFailure(true)
    AssertionResult.setFailureMessage("JSON parse failure: ${e.message}")
}
```
**Interview point:** Shows defensive scripting — malformed/unexpected responses shouldn't crash the whole test run; they should be caught, logged, and flagged as a controlled failure.

## 3.6 Working with Java Interop (calling Java classes directly)
```groovy
// HMAC signature (already covered) is a good example, here's another: date formatting
import java.time.LocalDateTime
import java.time.format.DateTimeFormatter

def now = LocalDateTime.now()
def formatted = now.format(DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ss"))
vars.put("requestTimestamp", formatted)
```
**Interview point:** Groovy runs on the JVM and has full access to any Java library — a strong point to make: "Groovy in JMeter isn't a limited scripting sandbox, it's full Java interop with more concise syntax."

## 3.7 Custom Reusable Functions via Shared Groovy Files (BSF/JSR223 with external script)
```groovy
// Instead of duplicating logic across many JSR223 elements, load an external .groovy file:
// File: /scripts/CommonUtils.groovy contains reusable methods

def utils = new GroovyShell().parse(new File("/mnt/jmeter/scripts/CommonUtils.groovy"))
def signature = utils.generateHmac(vars.get("apiKey"), vars.get("apiSecret"))
vars.put("reqSignature", signature)
```
**Interview point:** Very senior-level pattern — externalizing common logic (signature gen, payload builders) into shared files instead of copy-pasting the same Groovy block into 20 different JSR223 elements across a large test suite. Shows framework-thinking, not just scripting.

## 3.8 Custom Sampler Result Manipulation
```groovy
// Overriding what counts as "success" based on business logic, not just HTTP code
def response = prev.getResponseDataAsString()
def json = new groovy.json.JsonSlurper().parseText(response)

if (prev.getResponseCode() == "200" && json.orderStatus == "FAILED_BUSINESS_RULE") {
    // HTTP succeeded but business logic failed — override JMeter's success flag
    prev.setSuccessful(false)
    prev.setResponseMessage("Business rule failure: ${json.failureReason}")
}
```
**Interview point:** Distinguishes HTTP-level success from business-level success — a subtle but important senior concept: "200 OK doesn't always mean the transaction actually succeeded."

---

# LEVEL 4: EXPERT — PATTERNS SENIOR/LEAD ROLES ARE EXPECTED TO KNOW

## 4.1 Building a Reusable "Correlation Framework" Pattern
```groovy
// A generic utility used across many samplers to store/retrieve values with expiry awareness
class CorrelationStore {
    static void store(props, String key, def value) {
        def map = props.get("correlationStore") ?: [:]
        map[key] = [value: value, timestamp: System.currentTimeMillis()]
        props.put("correlationStore", map)
    }

    static def retrieve(props, String key, long maxAgeMs = 60000) {
        def map = props.get("correlationStore")
        if (map == null || !map.containsKey(key)) return null
        def entry = map[key]
        if (System.currentTimeMillis() - entry.timestamp > maxAgeMs) return null // expired
        return entry.value
    }
}

// usage inside a JSR223 element:
CorrelationStore.store(props, vars.get("correlationId"), System.currentTimeMillis())
```
**Interview point:** This is architect-level thinking — building a small reusable "framework" inside your test suite rather than one-off scripts scattered everywhere. Strong differentiator for a Lead-level interview.

## 4.2 Statistical Percentile Calculation Mid-Script (custom real-time SLA check)
```groovy
def latencyList = props.get("latencyList") ?: Collections.synchronizedList(new ArrayList())
props.put("latencyList", latencyList)
latencyList.add(prev.getTime())

if (latencyList.size() % 100 == 0) {  // check every 100 samples
    def sorted = latencyList.sort()
    def p95Index = (int)(sorted.size() * 0.95)
    def p95 = sorted[p95Index]
    log.info("Current running p95 latency: ${p95}ms (sample size: ${sorted.size()})")
    
    if (p95 > 2000) {
        log.warn("SLA BREACH ALERT: p95 latency ${p95}ms exceeds 2000ms threshold")
    }
}
```
**Interview point:** Shows you can build **live SLA monitoring logic inside the test itself**, not just wait for the post-test Aggregate Report — useful for long soak tests where you want early warning, not a 4-hour-later discovery.

## 4.3 Dynamic Test Behavior Based on External Feature Flags
```groovy
// Simulate reading a feature-flag/config service to adjust test behavior mid-run
import groovy.json.JsonSlurper

if (props.get("featureFlags") == null || (ctx.getThreadNum() == 0 && vars.getIteration() % 50 == 0)) {
    def flagResponse = new URL("http://config-service/flags").getText()
    def flags = new JsonSlurper().parseText(flagResponse)
    props.put("featureFlags", flags)
}

def flags = props.get("featureFlags")
if (flags?.newBillingFlow == true) {
    vars.put("billingEndpoint", "/v2/billing/create")
} else {
    vars.put("billingEndpoint", "/v1/billing/create")
}
```
**Interview point:** Real production systems increasingly use feature flags — showing you can build a test that adapts to a flag state rather than hardcoding one path demonstrates awareness of modern deployment patterns (canary/progressive rollout testing).

## 4.4 Custom Exit Criteria — Stopping Test Early on Critical Failure Threshold
```groovy
def failCount = props.get("criticalFailCount") ?: new java.util.concurrent.atomic.AtomicInteger(0)
props.put("criticalFailCount", failCount)

if (prev.getResponseCode() == "500") {
    def current = failCount.incrementAndGet()
    if (current > 50) {
        log.error("Critical failure threshold exceeded (50x HTTP 500) — stopping test")
        ctx.getEngine().stopTest()  // graceful stop, lets listeners flush
        // or ctx.getEngine().stopTest(true) for abrupt stop
    }
}
```
**Interview point:** Senior-level "fail fast" pattern — don't let a 4-hour test run to completion generating useless data if the system is clearly broken in the first 5 minutes. Shows cost/time-awareness in test execution strategy.

---

# QUICK REFERENCE — Groovy vs Java differences worth knowing cold

| Concept | Java | Groovy |
|---|---|---|
| Type declaration | Mandatory | Optional (`def`) |
| String interpolation | `"x" + var` | `"${var}"` |
| Getters/setters | `obj.getX()` | `obj.x` (auto) |
| Null checks | `if (x != null)` | `x?.method()` (safe navigation) |
| Equality | `.equals()` | `==` (calls equals automatically) |
| Closures | Lambdas (Java 8+) | Native, more flexible |
| Semicolons | Required | Optional |

## Null-safe operator example (very handy in JMeter scripting)
```groovy
def customerName = json?.customer?.name ?: "UNKNOWN"
// if json, customer, or name is null at any point, returns "UNKNOWN" instead of throwing NullPointerException
```
**Interview point:** The `?.` (safe navigation) and `?:` (Elvis operator) combo is a Groovy-specific defensive-coding idiom worth demonstrating — avoids verbose null-checking chains.

---
----------------------------------------------------------------------------------------

# JSR223 Groovy — JMeter Scripting Reference
## Correlation | Parameterization | Multi-Order Responses | Data Extraction

---

## 1. Correlation

Correlation = pulling a dynamic value out of one response and injecting it into the next request (orderId, token, sessionId, txnRef).

### 1.1 JSON correlation with JsonSlurper (preferred over regex for JSON)

```groovy
// JSR223 PostProcessor — attached to "Create Order" sampler
import groovy.json.JsonSlurper

def response = prev.getResponseDataAsString()
def json = new JsonSlurper().parseText(response)

def orderId = json.orderId
def txnRef  = json.transaction.referenceNumber

vars.put("orderId", orderId)
vars.put("txnRef", txnRef)

log.info("Correlated orderId=" + orderId + " txnRef=" + txnRef)
```

Use `${orderId}` in the next sampler's body/path.

### 1.2 Regex-based correlation (when response isn't clean JSON — HTML, mixed logs)

```groovy
def response = prev.getResponseDataAsString()
def matcher = response =~ /"sessionToken"\s*:\s*"([a-zA-Z0-9\-]+)"/

if (matcher.find()) {
    vars.put("sessionToken", matcher.group(1))
} else {
    log.error("sessionToken not found in response")
    vars.put("sessionToken", "NOT_FOUND")
}
```

### 1.3 Header correlation (auth token, correlation-id from response headers)

```groovy
def headers = prev.getResponseHeaders()
def matcher = headers =~ /X-Correlation-Id:\s*(\S+)/
if (matcher.find()) {
    vars.put("correlationId", matcher.group(1).trim())
}
```

### 1.4 Correlation with fallback / retry-safe defaults

```groovy
// Avoids polluting vars with null when an intermittent 4xx/5xx happens
import groovy.json.JsonSlurper

if (prev.getResponseCode() == "200") {
    def json = new JsonSlurper().parseText(prev.getResponseDataAsString())
    vars.put("provisioningId", json?.provisioningId ?: "MISSING")
} else {
    log.warn("Non-200 on provisioning call: " + prev.getResponseCode())
    vars.put("provisioningId", "MISSING")
}
```

---

## 2. Parameterization

Beyond CSV Data Set Config — dynamic, computed, or randomized parameterization in Groovy.

### 2.1 Random test data generation (JSR223 PreProcessor)

```groovy
import java.util.UUID

// Unique MSISDN-style subscriber number per thread iteration
def msisdn = "601" + (10000000 + new Random().nextInt(89999999))
vars.put("msisdn", msisdn)

// Unique idempotency/transaction key
vars.put("txnId", UUID.randomUUID().toString())

// Random order type from a fixed set — simulates realistic order mix
def orderTypes = ["NEW_CONNECTION", "PORT_IN", "PLAN_CHANGE", "RECONTRACT"]
vars.put("orderType", orderTypes[new Random().nextInt(orderTypes.size())])
```

### 2.2 Weighted parameterization (realistic traffic mix, e.g. 70% GET / 30% POST style ratios)

```groovy
def rand = new Random().nextInt(100)
def orderType
if (rand < 70) {
    orderType = "PLAN_CHANGE"       // 70%
} else if (rand < 90) {
    orderType = "NEW_CONNECTION"    // 20%
} else {
    orderType = "PORT_IN"           // 10%
}
vars.put("orderType", orderType)
```

### 2.3 Reading and cycling through a CSV inside Groovy (when CSV Data Set Config's thread-sharing doesn't fit)

```groovy
import java.nio.file.Files
import java.nio.file.Paths

// Load once per thread group start, cache in props (shared across threads)
if (props.get("customerData") == null) {
    def lines = Files.readAllLines(Paths.get("/opt/jmeter/data/customers.csv"))
    props.put("customerData", lines)
}

def data = props.get("customerData")
def row = data[ctx.getThreadNum() % data.size()]
def cols = row.split(",")
vars.put("customerId", cols[0])
vars.put("planCode", cols[1])
```

### 2.4 Date/time parameterization (order effective dates, TTL windows)

```groovy
import java.time.LocalDateTime
import java.time.format.DateTimeFormatter

def now = LocalDateTime.now()
def effectiveDate = now.plusDays(1)
def fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ss")

vars.put("orderTimestamp", now.format(fmt))
vars.put("effectiveDate", effectiveDate.format(fmt))
```

---

## 3. Handling Multiple Order Responses (arrays / batch order journeys)

Common in OSS/BSS: a single API call returns an array of orders, or you fan out N sub-orders and need to track each independently.

### 3.1 Parsing a JSON array of orders and looping

```groovy
import groovy.json.JsonSlurper

def json = new JsonSlurper().parseText(prev.getResponseDataAsString())
def orders = json.orders   // assume: { "orders": [ {...}, {...} ] }

log.info("Total orders returned: " + orders.size())

orders.eachWithIndex { order, idx ->
    vars.put("order_${idx}_id", order.orderId)
    vars.put("order_${idx}_status", order.status)
    vars.put("order_${idx}_type", order.orderType)
}

vars.put("orderCount", orders.size().toString())
```

Downstream, a **ForEach Controller** or a **While Controller** driven by `orderCount` iterates `order_0_id`, `order_1_id`, etc.

### 3.2 Filtering multi-order response for specific status before proceeding (gate pattern)

```groovy
import groovy.json.JsonSlurper

def json = new JsonSlurper().parseText(prev.getResponseDataAsString())
def pendingOrders = json.orders.findAll { it.status == "PENDING_PROVISIONING" }

if (pendingOrders.isEmpty()) {
    log.warn("No pending orders — skipping provisioning check")
    vars.put("skipProvisioning", "true")
} else {
    vars.put("skipProvisioning", "false")
    vars.put("pendingCount", pendingOrders.size().toString())
    // stash the first pending order's ID for the next sampler
    vars.put("firstPendingOrderId", pendingOrders[0].orderId)
}
```
Pair with an **If Controller**: `${skipProvisioning} == "false"`.

### 3.3 Aggregating results across a batch (building a JSON array to POST as a bulk request)

```groovy
import groovy.json.JsonBuilder

def orderIds = (1..5).collect { vars.get("order_${it}_id") }.findAll { it != null }

def payload = new JsonBuilder()
payload {
    batchId vars.get("txnId")
    orderReferences orderIds
}
vars.put("bulkOrderPayload", payload.toString())
```
Use `${bulkOrderPayload}` as the raw body of a bulk-status sampler.

### 3.4 Correlating N sub-orders from a split/fan-out response (e.g. bundle order → device + SIM + plan sub-orders)

```groovy
import groovy.json.JsonSlurper

def json = new JsonSlurper().parseText(prev.getResponseDataAsString())

json.subOrders.each { sub ->
    switch (sub.componentType) {
        case "DEVICE":
            vars.put("deviceOrderId", sub.orderId)
            break
        case "SIM":
            vars.put("simOrderId", sub.orderId)
            break
        case "PLAN":
            vars.put("planOrderId", sub.orderId)
            break
    }
}
```

---

## 4. Data-Related Response Processing

### 4.1 Nested field extraction with null-safe navigation

```groovy
import groovy.json.JsonSlurper

def json = new JsonSlurper().parseText(prev.getResponseDataAsString())

// Safe navigation (?.) avoids NPE if a nested key is missing/optional
def billingCycle = json?.subscriber?.billing?.cycleType ?: "UNKNOWN"
def deviceImei    = json?.order?.device?.imei ?: "N/A"

vars.put("billingCycle", billingCycle)
vars.put("deviceImei", deviceImei)
```

### 4.2 Response validation as a scripted Assertion (JSR223 Assertion)

```groovy
import groovy.json.JsonSlurper

def json = new JsonSlurper().parseText(prev.getResponseDataAsString())

if (json.status != "SUCCESS") {
    AssertionResult.setFailure(true)
    AssertionResult.setFailureMessage("Order failed. Status: " + json.status +
        " | Reason: " + (json.failureReason ?: "not provided"))
}

if (json.orderId == null || json.orderId.toString().trim().isEmpty()) {
    AssertionResult.setFailure(true)
    AssertionResult.setFailureMessage("orderId missing in response")
}
```
Note: in a JSR223 Assertion, `AssertionResult` is injected automatically — don't redeclare it.

### 4.3 Comparing response field against expected reference data (data-driven verification)

```groovy
import groovy.json.JsonSlurper

def json = new JsonSlurper().parseText(prev.getResponseDataAsString())
def expectedPlan = vars.get("planCode")   // came from CSV/parameterization
def actualPlan = json.subscriber.planCode

if (actualPlan != expectedPlan) {
    log.error("Plan mismatch. Expected=" + expectedPlan + " Actual=" + actualPlan)
    vars.put("planMismatchFlag", "true")
} else {
    vars.put("planMismatchFlag", "false")
}
```

### 4.4 Extracting and summing numeric fields across an array (e.g. total charges across order lines)

```groovy
import groovy.json.JsonSlurper

def json = new JsonSlurper().parseText(prev.getResponseDataAsString())
def totalCharge = json.orderLines.sum { it.charge as BigDecimal }

vars.put("totalCharge", totalCharge.toString())
log.info("Computed total charge across ${json.orderLines.size()} lines: " + totalCharge)
```

### 4.5 Response time / SLA check scripted inline (custom pass/fail beyond Duration Assertion)

```groovy
def elapsed = prev.getTime()
def slaThresholdMs = 3000

if (elapsed > slaThresholdMs) {
    prev.setSuccessful(false)
    prev.setResponseMessage("SLA breach: ${elapsed}ms > ${slaThresholdMs}ms threshold")
}
```

### 4.6 Writing extracted data to an external file (for offline analysis / audit trail)

```groovy
import groovy.json.JsonSlurper

def json = new JsonSlurper().parseText(prev.getResponseDataAsString())
def line = "${System.currentTimeMillis()},${json.orderId},${json.status},${prev.getTime()}\n"

new File("/opt/jmeter/results/order_audit.csv").append(line)
```
Caution: file I/O from every thread adds contention at high concurrency — prefer buffering or Backend Listener (InfluxDB) for real load tests; use this pattern mainly in low-thread diagnostic runs.

---

## Quick Reference — Object Cheat Sheet

| Object | Scope | Common use |
|---|---|---|
| `vars` | Thread-local | get/put correlated values, parameterized data |
| `props` | JVM-global (shared across threads) | shared static data, counters, cached CSV loads |
| `prev` | Current sampler result | `getResponseDataAsString()`, `getResponseCode()`, `getTime()`, `setSuccessful()` |
| `ctx` | Thread context | `getThreadNum()`, `getVariables()` |
| `log` | Jmeter logger | `log.info/warn/error` — goes to jmeter.log |
| `AssertionResult` | JSR223 Assertion only | `setFailure()`, `setFailureMessage()` |

**Performance tip:** always favor `JsonSlurper`/`JsonBuilder` over manual regex for JSON — it's both cleaner and avoids catastrophic backtracking regex can hit on large payloads. Cache compiled scripts (Script Compilation Cache: enable "Cache compiled script if available" checkbox in JSR223 elements) to avoid re-parsing Groovy on every iteration at scale.


# JSR223 Sampler — Simulating 3PP (Third-Party Provider) Dummy Responses in JMeter

Use case: downstream systems (credit bureau, billing engine, SIM provisioning, device inventory,
payment gateway) aren't available, rate-limited, or too fragile for full load — so you stub them
with a JSR223 Sampler that *acts like* the 3PP, returning realistic JSON with controllable
latency, error rate, and payload shape. This keeps your pipeline (Kafka producers, downstream
listeners, correlation logic) exercised end-to-end without hammering the real system.

---

## 1. Basic dummy JSON response (JSR223 Sampler, not PostProcessor)

```groovy
import groovy.json.JsonBuilder

def orderId = vars.get("orderId") ?: "ORD-UNKNOWN"

def response = new JsonBuilder()
response {
    orderId orderId
    status "SUCCESS"
    provisioningRef "PROV-" + System.currentTimeMillis()
    timestamp new Date().format("yyyy-MM-dd'T'HH:mm:ss")
}

SampleResult.setResponseData(response.toString(), "UTF-8")
SampleResult.setResponseCode("200")
SampleResult.setSuccessful(true)
SampleResult.setResponseMessage("OK")
```
Note: in a JSR223 **Sampler**, the injected result object is `SampleResult` (capital S), not `prev`.
`prev` is only valid in PostProcessors/Assertions referencing the *previous* sampler.

---

## 2. Simulating realistic network latency (so downstream timers/SLA logic behave normally)

```groovy
import groovy.json.JsonBuilder

// Simulate 3PP latency distribution — most calls fast, some slow (long tail)
def rand = new Random()
def latencyMs = (rand.nextInt(100) < 90) ?
    (150 + rand.nextInt(200)) :      // 90% of calls: 150-350ms
    (1000 + rand.nextInt(2000))      // 10% of calls: 1-3s (simulates 3PP slowness)

Thread.sleep(latencyMs)

def response = new JsonBuilder()
response {
    creditCheckResult "APPROVED"
    latencyMs latencyMs
}

SampleResult.setResponseData(response.toString(), "UTF-8")
SampleResult.setResponseCode("200")
SampleResult.setSuccessful(true)
```
Caution: `Thread.sleep` inside a sampler blocks the thread for that duration — factor this into
your thread group math the same as you would real 3PP latency (don't let it distort throughput
calcs unexpectedly).

---

## 3. Simulating error rates / partial failures (chaos-style dummy for negative-path testing)

```groovy
import groovy.json.JsonBuilder

def rand = new Random().nextInt(100)
def response = new JsonBuilder()

if (rand < 85) {
    // 85% success
    response {
        status "SUCCESS"
        billingAccountId "BA-" + (100000 + new Random().nextInt(899999))
    }
    SampleResult.setResponseCode("200")
    SampleResult.setSuccessful(true)

} else if (rand < 95) {
    // 10% business-rule failure (still HTTP 200 in many billing APIs — status is in payload)
    response {
        status "FAILED"
        errorCode "BILLING_ACCOUNT_LOCKED"
        message "Account under dispute hold"
    }
    SampleResult.setResponseCode("200")
    SampleResult.setSuccessful(true)   // functionally valid response, just a business failure

} else {
    // 5% hard failure (simulates 3PP outage/timeout)
    response {
        status "ERROR"
        message "Upstream billing service unavailable"
    }
    SampleResult.setResponseCode("503")
    SampleResult.setSuccessful(false)
}

SampleResult.setResponseData(response.toString(), "UTF-8")
```
This is the pattern to use when you need your **error-handling / retry logic** (Kafka DLQ,
compensation flow, alerting) exercised under load, not just the happy path.

---

## 4. Request-aware dummy — reading the incoming request body and echoing/deriving fields

Useful when the next sampler in the chain depends on values from *this* request (e.g. provisioning
dummy must reflect the plan code that was actually submitted).

```groovy
import groovy.json.JsonSlurper
import groovy.json.JsonBuilder

// SampleResult.getSamplerData() gives you what THIS sampler is about to "send"
// if you've put the outgoing payload in the sampler's Body Data field
def requestBody = sampler.getArguments()?.getArgument(0)?.getValue() ?: vars.get("requestPayload")

def reqJson = new JsonSlurper().parseText(requestBody)
def planCode = reqJson.planCode

def response = new JsonBuilder()
response {
    orderId vars.get("orderId")
    provisionedPlan planCode
    status "PROVISIONED"
    activationDate new Date().format("yyyy-MM-dd")
}

SampleResult.setResponseData(response.toString(), "UTF-8")
SampleResult.setResponseCode("200")
SampleResult.setSuccessful(true)
```
Simpler alternative: since you likely built the request from `vars` in the first place (per the
parameterization patterns), just reuse `vars.get("planCode")` directly instead of re-parsing —
cheaper and avoids depending on sampler internals that vary by sampler type.

---

## 5. Dummy response with array payload (simulating a 3PP that returns multiple records — e.g. device inventory lookup)

```groovy
import groovy.json.JsonBuilder

def skus = ["SKU-IPH15-BLK", "SKU-SGS24-WHT", "SKU-PIX9-BLU"]
def stock = skus.collect { sku ->
    [ sku: sku, available: new Random().nextBoolean(), qty: new Random().nextInt(50) ]
}

def response = new JsonBuilder()
response {
    warehouseId "WH-KL-01"
    items stock
}

SampleResult.setResponseData(response.toString(), "UTF-8")
SampleResult.setResponseCode("200")
SampleResult.setSuccessful(true)
```

---

## 6. Stateful dummy — simulating a 3PP that changes behavior across calls (e.g. first call PENDING, later poll returns COMPLETED)

Common for async provisioning/status-polling flows. Use `props` (JVM-shared) keyed by orderId to
track fake "state" across a thread's poll loop.

```groovy
import groovy.json.JsonBuilder

def orderId = vars.get("orderId")
def key = "poll_count_" + orderId

def pollCount = (props.get(key) ?: "0") as Integer
pollCount++
props.put(key, pollCount.toString())

def response = new JsonBuilder()
def status = (pollCount < 3) ? "IN_PROGRESS" : "COMPLETED"

response {
    orderId orderId
    status status
    pollAttempt pollCount
}

SampleResult.setResponseData(response.toString(), "UTF-8")
SampleResult.setResponseCode("200")
SampleResult.setSuccessful(true)

// downstream While Controller can loop on: ${status} != "COMPLETED"  (extract status via PostProcessor)
```
Remember to clean up `props` keys when the order completes (`props.remove(key)`), or a long
soak/endurance run will leak memory-held state across thousands of order IDs.

---

## 7. Dummy response driven by a lookup table (deterministic test-data-driven mocking, not random)

Useful when you need **reproducible** results tied to specific test input (e.g. specific MSISDN
ranges always map to specific credit-check outcomes, for regression-style scenarios).

```groovy
import groovy.json.JsonBuilder

def msisdn = vars.get("msisdn")
def lastDigit = msisdn[-1] as Integer

def creditResult
switch (lastDigit) {
    case { it in 0..6 }: creditResult = "APPROVED"; break
    case 7:              creditResult = "REFERRED"; break
    case 8:              creditResult = "DECLINED"; break
    default:              creditResult = "APPROVED"
}

def response = new JsonBuilder()
response {
    msisdn msisdn
    creditResult creditResult
}

SampleResult.setResponseData(response.toString(), "UTF-8")
SampleResult.setResponseCode("200")
SampleResult.setSuccessful(true)
```

---

## Quick Reference — JSR223 Sampler-specific objects

| Object | Notes |
|---|---|
| `SampleResult` | The result object you're building (capital S — this is the sampler's own result, unlike `prev`) |
| `SampleResult.setResponseData(str, "UTF-8")` | Sets the mock response body |
| `SampleResult.setResponseCode("200")` | Sets HTTP-style status code shown in listeners |
| `SampleResult.setSuccessful(true/false)` | Drives pass/fail in reports and assertions downstream |
| `SampleResult.setLatency(ms)` | Optionally override reported latency separate from `Thread.sleep` wall time |
| `sampler` | Reference to the JSR223 Sampler element itself (rarely needed beyond `getArguments()`) |

**Design note:** keep the dummy sampler's response *shape* identical to the real 3PP's contract
(same field names, nesting, status enums). That way your correlation/assertion/PostProcessor
scripts downstream don't need a separate code path for "mock mode" vs "real mode" — you just swap
the sampler out when the real system becomes available for integration-level runs.
