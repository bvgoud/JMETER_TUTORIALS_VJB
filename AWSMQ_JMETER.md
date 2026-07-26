Here's a complete AWS MQ (Amazon MQ) performance testing reference with JMeter — covering both engine types (ActiveMQ and RabbitMQ), hierarchy model, and telco-relevant scenarios.

---

## AWS MQ BACKGROUND (quick context for notebook)

Amazon MQ is a managed message broker supporting two engines:
- **ActiveMQ** (supports JMS, OpenWire, AMQP, MQTT, STOMP protocols)
- **RabbitMQ** (AMQP 0-9-1 protocol)

For telecom, Amazon MQ (ActiveMQ) is commonly used for legacy JMS-based integration — order management systems, billing event queues, notification dispatch — where systems weren't built for Kafka but need reliable queuing.

---

## JMETER ELEMENTS FOR AMAZON MQ

### For ActiveMQ (JMS-based) — built-in JMeter support
- **JMS Point-to-Point Sampler** (Queue-based)
- **JMS Publisher** / **JMS Subscriber** (Topic-based, pub-sub)
- **JMS Connection Configuration** (Config Element — broker URL, connection factory)

### For RabbitMQ (AMQP-based) — requires plugin
- **AMQP Publisher / AMQP Consumer** (via jpgc AMQP plugin, since core JMeter has no native AMQP support)

---

## FULL HIERARCHY MODEL — ActiveMQ Queue-Based Order Notification Testing

```
TEST PLAN: "AWS MQ (ActiveMQ) - Order Notification Queue Performance Test"
│
├── User Defined Variables
│     mq.broker.url = ssl://b-xxxx.mq.ap-southeast-1.amazonaws.com:61617
│     mq.queue.name = order-notification-queue
│     mq.username = ${__P(mquser)}
│     mq.password = ${__P(mqpass)}
│
├── setUp Thread Group
│   └── JSR223 Sampler: "Pre-Test Queue Check"
│         - Connect via JMS, check queue depth = 0 (clean baseline)
│         - Validate broker connectivity/SSL handshake succeeds
│
├── Thread Group 1: "JMS Producer Load - Order Events"
│   │  (Concurrency Thread Group + Constant Throughput Timer)
│   │
│   ├── Config Elements
│   │     JMS Connection Configuration
│   │       - Connection Factory: ActiveMQSslConnectionFactory (AWS MQ requires SSL)
│   │       - Provider URL: ${mq.broker.url}
│   │       - JNDI initial context factory (or direct connection factory class)
│   │       - Username/Password: ${mq.username}/${mq.password}
│   │
│   ├── Loop Controller
│   │   │
│   │   ├── JSR223 PreProcessor: "Build Order Notification Message"
│   │   │     (dynamic JSON/XML payload — shown below)
│   │   │
│   │   ├── JMS Point-to-Point Sampler
│   │   │     Communication Style: Producer
│   │   │     Destination: ${mq.queue.name}
│   │   │     Message: ${orderNotificationPayload}
│   │   │     Message properties: correlationId header set
│   │   │
│   │   └── Response Assertion: check send confirmation (no JMS exception)
│   │
│   └── Listeners: Simple Data Writer, jp@gc TPS listener, Backend Listener → Grafana
│
├── Thread Group 2: "JMS Consumer Load - Order Processing Service Simulation"
│   │  (parallel, simulates consumer read rate)
│   │
│   ├── JMS Point-to-Point Sampler
│   │     Communication Style: Consumer (Receiver)
│   │     Destination: ${mq.queue.name}
│   │     Timeout: 5000ms
│   │
│   ├── JSR223 PostProcessor: "Calculate End-to-End Latency"
│   │     (match consumed correlationId against producer-side sentTimestamp via props)
│   │
│   └── JSR223 Assertion: validate message content integrity + no duplicate consumption
│
├── tearDown Thread Group
│   └── JSR223 Sampler: purge test queue, log final queue depth (should return to 0)
│
└── PerfMon Metrics Collector
      AWS MQ broker CPU/memory/queue depth via CloudWatch-correlated Backend Listener
```

---

## KEY SCRIPT DETAILS

### JSR223 PreProcessor — Build Order Notification Payload
```groovy
import groovy.json.JsonOutput

def correlationId = UUID.randomUUID().toString()
def payload = [
    correlationId: correlationId,
    orderId: "ORD-" + (100000..999999).next(),
    eventType: "ORDER_ACTIVATED",
    timestamp: new Date().format("yyyy-MM-dd'T'HH:mm:ss'Z'")
]

vars.put("correlationId", correlationId)
vars.put("orderNotificationPayload", JsonOutput.toJson(payload))

// track sent time for latency calc (shared across Thread Groups)
def sentMap = props.get("mqSentTimestamps")
if (sentMap == null) {
    sentMap = Collections.synchronizedMap(new HashMap())
    props.put("mqSentTimestamps", sentMap)
}
sentMap.put(correlationId, System.currentTimeMillis())
```

### JSR223 PostProcessor — Consumer-Side Latency Calculation
```groovy
import groovy.json.JsonSlurper

def receivedMsg = prev.getResponseDataAsString()
def parsed = new JsonSlurper().parseText(receivedMsg)
def correlationId = parsed.correlationId

def sentMap = props.get("mqSentTimestamps")
if (sentMap != null && sentMap.containsKey(correlationId)) {
    def latency = System.currentTimeMillis() - sentMap.get(correlationId)
    log.info("MQ E2E latency for ${correlationId}: ${latency}ms")
    vars.put("mqLatencyMs", latency.toString())
}
```

### JSR223 Assertion — Duplicate Consumption Check (important for JMS — at-least-once delivery risk)
```groovy
def receivedIds = props.get("mqReceivedIds")
if (receivedIds == null) {
    receivedIds = Collections.synchronizedSet(new HashSet())
    props.put("mqReceivedIds", receivedIds)
}

def currentId = vars.get("correlationId")
if (!receivedIds.add(currentId)) {
    AssertionResult.setFailure(true)
    AssertionResult.setFailureMessage("Duplicate message consumed: ${currentId}")
}
```

---

## COMMON PREPROD ISSUES SPECIFIC TO AWS MQ

**1. SSL Handshake Failures**
- **Issue:** AWS MQ enforces SSL/TLS by default (`ssl://` not `tcp://`); scripts built against on-prem ActiveMQ using plain `tcp://` failed to connect
- **Fix:** Used `ActiveMQSslConnectionFactory`, imported AWS MQ's broker certificate into JMeter's JVM truststore (`-Djavax.net.ssl.trustStore`)

**2. Connection Pool Exhaustion Under Load**
- **Issue:** JMS Connection Configuration created a new connection per sampler instead of reusing — at high concurrency, hit AWS MQ's max connection limit (broker-tier dependent)
- **Fix:** Set JMS Connection Config to reuse a shared connection per thread (not per request) — configured "Number of connections" appropriately relative to instance limits, and requested a larger broker instance tier for load testing

**3. Queue Depth Growing Despite Consumers Running**
- **Issue:** Similar to Kafka partition scenario — ActiveMQ queue is inherently single-queue (unlike Kafka's partitioned model), so consumer scaling only helps up to a point; found the bottleneck was actual message processing time downstream, not JMeter's consumer read rate
- **Fix:** Confirmed via broker-side CloudWatch metrics (`QueueSize`, `ConsumerCount`) that the queue itself wasn't the bottleneck — it was the downstream order-processing service; redirected performance focus there

**4. Active/Standby Broker Failover Not Transparent**
- **Issue:** AWS MQ (ActiveMQ) in active/standby HA mode caused a ~30-second connection interruption during simulated failover — producer threads threw JMS exceptions instead of reconnecting automatically
- **Fix:** Added JMS failover connection string with retry logic (`failover:(ssl://broker1:61617,ssl://broker2:61617)`) so the client-side driver handles reconnection automatically — validated zero message loss with `acks`-equivalent JMS persistent delivery mode

---

## METRICS TO REPORT

| Metric | Target | Source |
|---|---|---|
| Producer throughput | Matches business event rate (e.g., 200 msg/sec) | jp@gc TPS listener |
| Consumer processing latency | <2 sec p95 | JSR223 latency calc |
| Queue depth under sustained load | Stable, not growing | CloudWatch QueueSize metric |
| Message loss | 0% | JSR223 duplicate/missing check |
| Failover recovery time | <30 sec, zero message loss | Chaos test + assertion |
| Broker CPU/memory | Within safe operating range | PerfMon/CloudWatch correlation |

---

## AWS MQ vs KAFKA — quick differentiation (good interview talking point)

| Aspect | Amazon MQ (ActiveMQ/JMS) | Kafka |
|---|---|---|
| Model | Traditional queue/topic (JMS) | Distributed log with partitions |
| Scaling | Vertical (bigger broker instance) | Horizontal (more partitions/brokers) |
| Delivery | At-least-once (transactional options) | At-least-once (configurable) |
| Use case fit | Legacy JMS integration, simpler ordering guarantees | High-throughput event streaming, replay capability |
| Telecom use | Legacy OSS/BSS system integration, notification dispatch | CDR pipelines, real-time event streaming |

If you're citing this in an interview: "I chose JMS Sampler with connection reuse and failover strings for Amazon MQ because unlike Kafka's partition-based horizontal scaling, ActiveMQ scaling is more vertical — so my test design focused on validating broker instance sizing and HA failover behavior rather than partition-parallelism tuning."

Want the **RabbitMQ/AMQP variant** next (different plugin, different config), or should we move to a **mock interview Q&A sheet** consolidating Kafka + Order Journey + AWS MQ into one rehearsal document?
