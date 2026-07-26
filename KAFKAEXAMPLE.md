Here's a complete end-to-end Kafka performance testing project for a telco use case (CDR/Event pipeline), structured exactly as you'd build and present it — full JMeter hierarchy plus the reasoning behind each layer.

---

## PROJECT CONTEXT (what we're testing)

**Scenario:** Telecom subscriber events (call start/end, SMS, data session, top-up) are produced to Kafka by network/OSS systems, consumed by billing, CDR-archival, and fraud-detection services downstream. We're performance-testing the **producer side** (ingestion capacity) and **consumer lag behavior** under sustained and peak load — a very realistic Performance Test Lead deliverable.

**Goal:** Validate the pipeline sustains **1000 events/sec** during busy hour with consumer lag staying under 5 seconds, and no message loss.

---

## FULL HIERARCHY MODEL

```
TEST PLAN: "Telco CDR Kafka Pipeline - Performance Test"
│
├── User Defined Variables
│     kafka.bootstrap.servers = broker1:9092,broker2:9092
│     kafka.topic = cdr-events
│     kafka.consumer.group = perf-test-cdr-consumers
│
├── setUp Thread Group (runs once, before load)
│   └── JSR223 Sampler: "Pre-Test Validation"
│         - Check topic exists & partition count
│         - Check consumer group lag = 0 (clean baseline)
│         - Log broker cluster health via Admin API
│
├── Thread Group 1: "Kafka Producer Load - CDR Events"
│   │  (Concurrency Thread Group + Throughput Shaping Timer)
│   │
│   ├── Config Elements
│   │     Kafka Connection Config (jpgc/pepper-box plugin)
│   │       - bootstrap.servers = ${kafka.bootstrap.servers}
│   │       - key.serializer = StringSerializer
│   │       - value.serializer = StringSerializer
│   │       - acks = all  (durability — matches production config)
│   │       - compression.type = snappy
│   │
│   ├── Throughput Shaping Timer
│   │     0 → 1000 TPS over 300s (ramp)
│   │     1000 TPS hold for 1800s (busy hour)
│   │     1000 → 100 TPS over 300s (taper)
│   │
│   ├── Logic Controller: Loop Controller (infinite, controlled by Thread Group duration)
│   │   │
│   │   ├── JSR223 PreProcessor: "Build CDR Event Payload"
│   │   │     (dynamic JSON generation — shown below)
│   │   │
│   │   ├── Kafka Producer Sampler (pepper-box plugin)
│   │   │     Topic: ${kafka.topic}
│   │   │     Key: ${correlationId}
│   │   │     Value: ${cdrPayload}
│   │   │
│   │   ├── JSR223 PostProcessor: "Track Sent Timestamp"
│   │   │     - vars.put("sentTimestamp", System.currentTimeMillis())
│   │   │     - stored keyed by correlationId for later consumer-side latency match
│   │   │
│   │   └── Response Assertion
│   │         - Check producer ack received (no exception in response)
│   │
│   └── Listeners
│         Simple Data Writer → producer_results.jtl
│         jp@gc Transactions per Second → compare actual vs 1000 TPS target
│         Backend Listener → Grafana (live producer TPS + error rate)
│
├── Thread Group 2: "Kafka Consumer Lag Monitor" (parallel to Thread Group 1)
│   │  (runs concurrently, lower thread count — 5 threads polling)
│   │
│   ├── Config Elements
│   │     Kafka Consumer Config
│   │       - group.id = ${kafka.consumer.group}
│   │       - auto.offset.reset = latest
│   │       - enable.auto.commit = false (manual commit — matches real consumer behavior)
│   │
│   ├── Loop Controller
│   │   │
│   │   ├── Kafka Consumer Sampler (pepper-box or custom JSR223)
│   │   │     Polls topic, reads available messages
│   │   │
│   │   ├── JSR223 PostProcessor: "Calculate End-to-End Latency"
│   │   │     (matches consumed message's correlationId against sentTimestamp)
│   │   │
│   │   └── JSR223 Assertion: "Validate No Message Loss"
│   │         - Track received correlationIds in a shared props Set
│   │         - Compare against expected count from producer side
│   │
│   └── Listeners
│         jp@gc Response Times Over Time → consumer processing latency
│         Backend Listener → Grafana (consumer lag panel via PerfMon/JMX metrics)
│
├── Thread Group 3: "Downstream DB Validation" (lighter thread count, periodic)
│   │
│   ├── JDBC Connection Config → CDR archival DB
│   │
│   ├── Loop Controller (polls every 30s)
│   │   │
│   │   ├── JDBC Request: "Check CDR record count matches Kafka produced count"
│   │   │
│   │   └── JSR223 Assertion: "Validate settlement completeness"
│   │         - Compare Kafka-produced count vs DB-persisted count
│   │         - Flag if gap exceeds acceptable async-processing window
│   │
│   └── Listener: Simple Data Writer → db_validation_results.jtl
│
├── tearDown Thread Group (runs once, after all load)
│   └── JSR223 Sampler: "Post-Test Cleanup & Report"
│         - Fetch final consumer lag via Kafka Admin API
│         - Log total messages produced vs consumed vs DB-persisted
│         - Optionally purge test topic data (short-retention approach)
│
└── PerfMon Metrics Collector (Listener, test-plan level)
      Monitors broker nodes: CPU, memory, disk I/O, network throughput
      Pairs with Dynatrace/Datadog dashboards for correlated view
```

---

## KEY SCRIPT DETAILS

### JSR223 PreProcessor — Build CDR Event Payload
```groovy
import groovy.json.JsonOutput

def correlationId = UUID.randomUUID().toString()
def callTypes = ["VOICE", "SMS", "DATA"]

def event = [
    correlationId: correlationId,
    subscriberId: "SUB" + (100000..999999).next(),
    callType: callTypes[new Random().nextInt(callTypes.size())],
    duration: (10..600).next(),
    startTime: new Date().format("yyyy-MM-dd'T'HH:mm:ss'Z'"),
    cellTowerId: "CELL-" + (1..500).next()
]

vars.put("correlationId", correlationId)
vars.put("cdrPayload", JsonOutput.toJson(event))
```

### JSR223 PostProcessor — Producer-Side Timestamp Tracking
```groovy
def correlationId = vars.get("correlationId")
def sentTime = System.currentTimeMillis()

// Store in shared props map since consumer runs in a different Thread Group
def sentMap = props.get("sentTimestamps")
if (sentMap == null) {
    sentMap = Collections.synchronizedMap(new HashMap())
    props.put("sentTimestamps", sentMap)
}
sentMap.put(correlationId, sentTime)
```

### JSR223 PostProcessor — Consumer-Side Latency Calculation
```groovy
def receivedCorrelationId = vars.get("consumedCorrelationId")
def sentMap = props.get("sentTimestamps")

if (sentMap != null && sentMap.containsKey(receivedCorrelationId)) {
    def sentTime = sentMap.get(receivedCorrelationId)
    def latency = System.currentTimeMillis() - sentTime
    vars.put("e2eLatencyMs", latency.toString())
    log.info("E2E latency for ${receivedCorrelationId}: ${latency}ms")
} else {
    log.warn("No matching sent timestamp found for ${receivedCorrelationId}")
}
```

### JSR223 Assertion — Message Loss Detection
```groovy
def receivedIds = props.get("receivedCorrelationIds")
if (receivedIds == null) {
    receivedIds = Collections.synchronizedSet(new HashSet())
    props.put("receivedCorrelationIds", receivedIds)
}

def currentId = vars.get("consumedCorrelationId")
if (receivedIds.contains(currentId)) {
    AssertionResult.setFailure(true)
    AssertionResult.setFailureMessage("Duplicate message detected: ${currentId}")
} else {
    receivedIds.add(currentId)
}
```

---

## METRICS TO REPORT (what a senior PT Lead presents at the end)

| Metric | Target (this project) | Source |
|---|---|---|
| Producer throughput | 1000 events/sec sustained | jp@gc TPS listener |
| Producer error rate | <0.1% | Response Assertion failures |
| Consumer lag | <5 sec at peak | Kafka Admin API / Burrow / PerfMon |
| End-to-end latency (produce→consume) | <2 sec p95 | JSR223 latency calc, Aggregate Report |
| Message loss | 0% | JSR223 Assertion duplicate/missing check |
| DB settlement completeness | 100% within 30s window | JDBC cross-check |
| Broker resource utilization | CPU <70%, disk I/O headroom | PerfMon + Dynatrace correlation |

---

## WHY THIS DESIGN (interview narrative)

1. **Separate Thread Groups for producer/consumer** — mirrors real architecture where producers and consumers are independent systems; lets you scale each side's load independently
2. **`props` for cross-thread-group data sharing** — necessary because `vars` is thread-local and producer/consumer run in different Thread Groups entirely
3. **Correlation ID pattern** — the backbone of measuring true end-to-end latency in an async, decoupled system — can't just measure JMeter's own request/response time since Kafka is fire-and-forget
4. **Downstream DB validation as a separate parallel Thread Group** — tests the *entire* pipeline's data integrity, not just the messaging layer in isolation — this is what makes it a true "pipeline test" instead of just a "Kafka broker load test"
5. **Throughput Shaping Timer with busy-hour curve** — models realistic telco traffic rather than flat artificial load
6. **PerfMon + Backend Listener to Grafana** — ties JMeter-side load metrics to broker-side infra metrics in one correlated view, matching your existing Dynatrace/Datadog/Grafana toolchain

---

This structure — producer load, consumer lag monitoring, downstream DB validation, all correlated via a shared ID and reported against SLA-based metrics — is exactly the kind of project you'd describe end-to-end in a Performance Test Lead interview when asked "walk me through a Kafka performance test you've designed."

Want a **one-page project summary doc** (like a mini case study — objective, approach, tools, results, challenges) that you could literally use as a resume bullet expansion or interview STAR-format answer?
