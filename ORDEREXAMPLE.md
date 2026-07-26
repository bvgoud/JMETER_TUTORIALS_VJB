Here's a complete end-to-end API performance test for a telco **Order Journey** (new connection/SIM order to activation) — full hierarchy, correlation logic, and the reasoning to present it in an interview.

---

## PROJECT CONTEXT

**Scenario:** A customer orders a new mobile connection via a digital channel (app/web). The journey spans multiple REST APIs across CRM, Inventory, Billing, and Network Activation systems — each a separate microservice, chained together by order state.

**Order Journey APIs (typical flow):**
1. `POST /auth/login` — customer/agent authentication
2. `GET /coverage/check?address=` — serviceability check
3. `POST /catalog/plans` — fetch eligible plans
4. `POST /order/create` — create order (returns `orderId`)
5. `POST /inventory/reserve` — reserve SIM/device (`orderId` → returns `simNumber`)
6. `POST /billing/account/create` — create billing account
7. `POST /order/submit` — submit order for provisioning
8. `GET /order/status/{orderId}` — poll until `ACTIVATED`
9. `POST /notification/send` — SMS/email confirmation

**Goal:** Validate the full order journey holds up under concurrent load (e.g., 200 simultaneous orders during a promo launch), with SLA on total journey time and no cross-step data corruption.

---

## FULL HIERARCHY MODEL

```
TEST PLAN: "Telco Order Journey - API Performance Test"
│
├── User Defined Variables
│     base.url = https://api-perf.telco.com
│     auth.username = ${__P(username,perfuser1)}
│     env = PERF
│
├── setUp Thread Group (once, before load)
│   └── HTTP Request: Health check on all downstream services (CRM/Inventory/Billing/Network)
│         Assertion: all return 200 before proceeding
│
├── Thread Group: "Order Journey - Concurrent Customers"
│   │  (Concurrency Thread Group: 200 users, ramp-up 120s, hold 20min)
│   │
│   ├── Config Elements
│   │     HTTP Request Defaults → base.url, protocol=https
│   │     HTTP Header Manager → Content-Type: application/json
│   │     HTTP Cookie Manager → clear each iteration
│   │     CSV Data Set Config → customerId, address, deviceType (unique per thread)
│   │
│   ├── Once Only Controller: "Authentication" (runs once per thread, not per loop)
│   │   └── HTTP Request: POST /auth/login
│   │         Body: {"username":"${auth.username}","password":"${__P(password)}"}
│   │         Post-Processor: JSON Extractor → authToken ($.token)
│   │         Assertion: Response Assertion (contains "token")
│   │         → sets Authorization: Bearer ${authToken} for all subsequent calls
│   │
│   ├── Transaction Controller: "Step 1 - Coverage Check"
│   │   └── HTTP Request: GET /coverage/check?address=${address}
│   │         Assertion: JSON Assertion → $.serviceable == true
│   │         Duration Assertion: <1000ms
│   │
│   ├── Transaction Controller: "Step 2 - Plan Selection"
│   │   └── HTTP Request: GET /catalog/plans?type=${deviceType}
│   │         Post-Processor: JSON Extractor → selectedPlanId ($.plans[0].planId)
│   │         Assertion: Response Assertion (plans array not empty)
│   │
│   ├── Transaction Controller: "Step 3 - Create Order"
│   │   └── HTTP Request: POST /order/create
│   │         Body: {"customerId":"${customerId}","planId":"${selectedPlanId}"}
│   │         Pre-Processor: JSR223 → add idempotency key header
│   │         Post-Processor: JSON Extractor → orderId ($.orderId)
│   │         Assertion: JSON Assertion → $.orderStatus == "CREATED"
│   │
│   ├── Transaction Controller: "Step 4 - Reserve Inventory"
│   │   └── HTTP Request: POST /inventory/reserve
│   │         Body: {"orderId":"${orderId}","deviceType":"${deviceType}"}
│   │         Post-Processor: JSON Extractor → simNumber ($.simNumber)
│   │         If Controller: retry once if $.stockStatus == "PENDING" (JSR223 condition)
│   │
│   ├── Transaction Controller: "Step 5 - Create Billing Account"
│   │   └── HTTP Request: POST /billing/account/create
│   │         Body: {"orderId":"${orderId}","customerId":"${customerId}"}
│   │         Post-Processor: JSON Extractor → billingAccountId
│   │         Assertion: Duration Assertion <2000ms (billing SLA)
│   │
│   ├── Transaction Controller: "Step 6 - Submit Order for Provisioning"
│   │   └── HTTP Request: POST /order/submit
│   │         Body: {"orderId":"${orderId}","simNumber":"${simNumber}","billingAccountId":"${billingAccountId}"}
│   │         Assertion: JSON Assertion → $.orderStatus == "SUBMITTED"
│   │
│   ├── While Controller: "Step 7 - Poll Order Status" (condition: status != ACTIVATED, max 10 tries)
│   │   ├── HTTP Request: GET /order/status/${orderId}
│   │   │     Post-Processor: JSON Extractor → currentStatus ($.status)
│   │   ├── JSR223 PostProcessor: increment poll counter, log status
│   │   └── Constant Timer: 3000ms between polls
│   │
│   ├── Transaction Controller: "Step 8 - Validate Activation"
│   │   └── JSR223 Assertion: fail if currentStatus != "ACTIVATED" after max polls
│   │         Also logs total journey time (order-create timestamp → activation timestamp)
│   │
│   ├── JDBC Request: "Cross-check OSS DB order record" (config: OSS DB connection)
│   │     JSR223 Assertion: DB orderStatus matches API-reported ACTIVATED status
│   │
│   └── Listeners
│         Simple Data Writer → order_journey_results.jtl
│         Aggregate Report → per-transaction-controller stats (each step's own SLA)
│         Backend Listener → Grafana (live journey completion rate + step-wise latency)
│
├── tearDown Thread Group (once, after load)
│   └── JDBC Request: cleanup test orders (delete/archive test data by customerId prefix)
│
└── PerfMon Metrics Collector
      CRM, Inventory, Billing, Network Activation server nodes — correlate with Dynatrace
```

---

## KEY SCRIPT DETAILS

### JSR223 PreProcessor — Idempotency Key (Step 3)
```groovy
// Order creation APIs often require idempotency keys to prevent duplicate orders on retry
def idempotencyKey = UUID.randomUUID().toString()
vars.put("idempotencyKey", idempotencyKey)
// Referenced in Header Manager as: Idempotency-Key: ${idempotencyKey}
```

### JSR223 PostProcessor — Total Journey Time Tracking
```groovy
// Capture order creation time at Step 3
if (vars.get("orderCreateTimestamp") == null) {
    vars.put("orderCreateTimestamp", System.currentTimeMillis().toString())
}
```
Then at Step 8 (activation confirmed):
```groovy
def createTime = vars.get("orderCreateTimestamp").toLong()
def activationTime = System.currentTimeMillis()
def totalJourneyMs = activationTime - createTime

vars.put("totalJourneyTimeMs", totalJourneyMs.toString())
log.info("Order ${vars.get('orderId')} completed in ${totalJourneyMs}ms")

// Track against SLA
if (totalJourneyMs > 30000) {
    log.warn("Order ${vars.get('orderId')} exceeded 30s journey SLA")
}
```

### If Controller condition — Inventory Retry (Step 4)
```groovy
// If Controller "Condition" field (JavaScript-style expression JMeter evaluates)
"${stockStatus}" == "PENDING"
```

### JSR223 Assertion — Final Activation + DB Cross-Check
```groovy
def apiStatus = vars.get("currentStatus")
def dbStatus = vars.get("dbOrderStatus") // from JDBC extraction

if (apiStatus != "ACTIVATED") {
    AssertionResult.setFailure(true)
    AssertionResult.setFailureMessage("Order ${vars.get('orderId')} did not activate — final status: ${apiStatus}")
}

if (apiStatus != dbStatus) {
    AssertionResult.setFailure(true)
    AssertionResult.setFailureMessage("API/DB status mismatch — API: ${apiStatus}, DB: ${dbStatus}")
}
```

---

## METRICS TO REPORT

| Metric | Target | Measured via |
|---|---|---|
| End-to-end order journey time | <30 sec p95 | JSR223 timestamp delta |
| Per-step latency (each Transaction Controller) | Coverage <1s, Billing <2s, etc. | Aggregate Report per TC |
| Order activation success rate | >99.5% | JSR223 Assertion pass rate |
| Concurrent order handling | 200 simultaneous, no failures | Thread Group config + error % |
| Inventory reservation conflicts | 0 duplicate SIM assignments | Shared props Set check (like earlier race-condition pattern) |
| DB/API consistency | 100% match | JDBC cross-check assertion |

---

## WHY THIS DESIGN (interview narrative)

1. **Once Only Controller for auth** — token fetched once per thread, not re-fetched every iteration, mirrors real session behavior and avoids skewing results with unnecessary auth overhead
2. **Transaction Controller per business step** — gives step-wise SLA visibility (you can point to exactly which microservice is the bottleneck — e.g., "Billing account creation is the slowest step at 1.8s avg")
3. **While Controller for async activation polling** — order activation isn't synchronous in real systems (network provisioning takes time), so polling with a max-retry cap mirrors production reality instead of assuming instant response
4. **Correlation chain (orderId → simNumber → billingAccountId)** — demonstrates you understand realistic multi-system order orchestration, not just isolated endpoint testing
5. **JDBC cross-check at the end** — validates the *system of record* actually reflects what the API reported, catching silent inconsistencies API-only testing would miss
6. **Idempotency key** — shows awareness of a real production concern (duplicate order prevention under retry/network-flake conditions), a detail that signals senior-level API testing maturity

---

This is the structure to describe when asked "walk me through an API performance test you designed for an order journey" — it shows correlation, multi-system orchestration, async polling, step-wise SLA breakdown, and data-integrity validation, not just a single endpoint load test.

Want the **mock interview Q&A sheet** now, covering both this Order Journey project and the Kafka project together, so you have rehearsed answers ready for both?Here's a complete end-to-end API performance test for a telco **Order Journey** (new connection/SIM order to activation) — full hierarchy, correlation logic, and the reasoning to present it in an interview.

---

## PROJECT CONTEXT

**Scenario:** A customer orders a new mobile connection via a digital channel (app/web). The journey spans multiple REST APIs across CRM, Inventory, Billing, and Network Activation systems — each a separate microservice, chained together by order state.

**Order Journey APIs (typical flow):**
1. `POST /auth/login` — customer/agent authentication
2. `GET /coverage/check?address=` — serviceability check
3. `POST /catalog/plans` — fetch eligible plans
4. `POST /order/create` — create order (returns `orderId`)
5. `POST /inventory/reserve` — reserve SIM/device (returns `simNumber`)
6. `POST /billing/account/create` — create billing account
7. `POST /order/submit` — submit order for provisioning
8. `GET /order/status/{orderId}` — poll until `ACTIVATED`
9. `POST /notification/send` — SMS/email confirmation

**Goal:** Validate the full journey holds up under concurrent load (e.g., 200 simultaneous orders during a promo launch), with SLA on total journey time and no cross-step data corruption.

---

## FULL HIERARCHY MODEL

```
TEST PLAN: "Telco Order Journey - API Performance Test"
│
├── User Defined Variables
│     base.url, env=PERF
│
├── setUp Thread Group
│   └── HTTP Request: Health check on CRM/Inventory/Billing/Network → assert all 200
│
├── Thread Group: "Order Journey - Concurrent Customers" (200 users, ramp 120s, hold 20min)
│   │
│   ├── Config: HTTP Request Defaults, Header Manager, Cookie Manager, CSV Data Set
│   │
│   ├── Once Only Controller: Login → extract authToken
│   │
│   ├── TC "Step1 Coverage Check" → GET /coverage/check
│   ├── TC "Step2 Plan Selection" → GET /catalog/plans → extract selectedPlanId
│   ├── TC "Step3 Create Order" → POST /order/create → extract orderId
│   ├── TC "Step4 Reserve Inventory" → POST /inventory/reserve → extract simNumber
│   ├── TC "Step5 Create Billing Account" → POST /billing/account/create
│   ├── TC "Step6 Submit Order" → POST /order/submit
│   ├── While Controller "Step7 Poll Status" → GET /order/status/{orderId} (max 10 tries, 3s timer)
│   ├── TC "Step8 Validate Activation" → JSR223 Assertion (status check + journey time)
│   ├── JDBC Request: cross-check OSS DB order record
│   │
│   └── Listeners: Simple Data Writer, Aggregate Report (per TC), Backend Listener → Grafana
│
├── tearDown Thread Group → cleanup test orders
│
└── PerfMon Metrics Collector → CRM/Inventory/Billing/Network nodes, correlated with Dynatrace
```

---

## KEY SCRIPTS

**Idempotency key (PreProcessor, Step 3):**
```groovy
vars.put("idempotencyKey", UUID.randomUUID().toString())
```

**Total journey time (PostProcessor):**
```groovy
if (vars.get("orderCreateTimestamp") == null) {
    vars.put("orderCreateTimestamp", System.currentTimeMillis().toString())
}
```
At activation:
```groovy
def totalMs = System.currentTimeMillis() - vars.get("orderCreateTimestamp").toLong()
vars.put("totalJourneyTimeMs", totalMs.toString())
if (totalMs > 30000) log.warn("Order ${vars.get('orderId')} exceeded 30s SLA")
```

**Final activation + DB cross-check (Assertion):**
```groovy
def apiStatus = vars.get("currentStatus")
def dbStatus = vars.get("dbOrderStatus")

if (apiStatus != "ACTIVATED") {
    AssertionResult.setFailure(true)
    AssertionResult.setFailureMessage("Order ${vars.get('orderId')} not activated — status: ${apiStatus}")
}
if (apiStatus != dbStatus) {
    AssertionResult.setFailure(true)
    AssertionResult.setFailureMessage("API/DB mismatch — API: ${apiStatus}, DB: ${dbStatus}")
}
```

---

## METRICS TO REPORT

| Metric | Target | Measured via |
|---|---|---|
| End-to-end journey time | <30s p95 | JSR223 timestamp delta |
| Per-step latency | Coverage <1s, Billing <2s | Aggregate Report per Transaction Controller |
| Activation success rate | >99.5% | Assertion pass rate |
| Concurrent order handling | 200 simultaneous, no failures | Thread Group + error % |
| SIM reservation conflicts | 0 duplicates | Shared `props` Set check |
| API/DB consistency | 100% match | JDBC cross-check assertion |

---

## WHY THIS DESIGN (interview narrative)

1. **Once Only Controller for auth** — token fetched once per thread, mirrors real session reuse
2. **Transaction Controller per step** — isolates which exact microservice is the bottleneck
3. **While Controller for polling** — activation is async in reality; fixed max-retry avoids infinite loops
4. **Correlation chain** (orderId → simNumber → billingAccountId) — shows real multi-system orchestration understanding
5. **JDBC cross-check** — catches silent API/DB inconsistency that pure API testing misses
6. **Idempotency key** — signals awareness of production-grade retry/duplicate-prevention concerns

This is the structure to describe when asked to "walk through an API performance test for an order journey" — it demonstrates correlation, multi-system chaining, async polling, step-wise SLA breakdown, and data-integrity validation.

Want the **mock interview Q&A sheet** now, covering both this Order Journey project and the earlier Kafka project together, so you have rehearsed answers ready for both?
