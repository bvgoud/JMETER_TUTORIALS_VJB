Here's a realistic set of preprod issues you'd credibly have encountered on both projects — written as senior-level "problem → diagnosis → resolution" narratives you can use directly in STAR-format interview answers.

---

# KAFKA PIPELINE — PREPROD ISSUES

## 1. Consumer Lag Growing Unbounded Under Sustained Load
**Symptom:** At ~600 TPS (below the 1000 TPS target), consumer lag kept climbing instead of stabilizing.
**Diagnosis:**
- Checked partition count vs consumer instance count — topic had 6 partitions but only 3 consumer threads in the test, so 3 partitions sat idle while others backed up
- Confirmed via `kafka-consumer-groups.sh --describe` showing lag concentrated on specific partitions
**Root cause:** Consumer parallelism was under-provisioned relative to partition count — classic Kafka scaling constraint (you can never have more effective consumers than partitions).
**Resolution:** Increased topic partitions from 6 to 12 in preprod, matched consumer thread count, lag stabilized under 5s at full 1000 TPS.
**Interview point:** Shows understanding that Kafka scaling isn't just "add more consumers" — partition count is the hard ceiling on parallelism.

---

## 2. Message Loss During Broker Restart (Rolling Upgrade Simulation)
**Symptom:** During a chaos test simulating broker restart mid-load, ~40 messages went missing (produced but never consumed).
**Diagnosis:**
- Checked producer config — `acks=1` was set instead of `acks=all`, meaning producer considered the message "sent" once the leader replica acknowledged it, before followers replicated
- Leader failed over during restart before replication completed on those in-flight messages
**Root cause:** Producer durability config mismatch between perf test config and actual production config (production used `acks=all`, but perf environment was scripted with `acks=1` for speed, unintentionally).
**Resolution:** Corrected `acks=all` + `min.insync.replicas=2` in perf test producer config to accurately mirror production — re-ran the failover test, zero message loss confirmed.
**Interview point:** Strong example of "test environment parity" — a misconfigured perf test can give false confidence; catching this before go-live prevented a real production data-loss risk.

---

## 3. Producer Throughput Plateaued Below Target Despite Idle CPU
**Symptom:** Producer TPS capped at ~700 despite brokers and JMeter injector both showing <50% CPU utilization — expected to hit 1000 TPS easily.
**Diagnosis:**
- Checked `linger.ms` and `batch.size` producer settings — defaults were too conservative, causing many small individual sends instead of efficient batching
- Also found JMeter's Kafka sampler thread pool was serialized rather than truly async (pepper-box plugin default config issue)
**Root cause:** Producer batching wasn't tuned, and JMeter-side sampler threading wasn't parallelized correctly.
**Resolution:** Increased `linger.ms` to 10ms and `batch.size` to 32KB, and reconfigured Concurrency Thread Group to properly scale threads — achieved 1000+ TPS with room to spare.
**Interview point:** Shows debugging discipline — didn't just add more threads blindly, traced it to config-level batching behavior on both producer and load-tool side.

---

## 4. Duplicate CDR Records in Downstream DB
**Symptom:** DB validation step (JDBC cross-check) showed more records than messages actually produced — count mismatch of ~2%.
**Diagnosis:**
- Consumer had `enable.auto.commit=true` with a short auto-commit interval, and the consumer service itself had a retry-on-processing-failure loop
- Combination meant: message processed → DB write succeeded → but before offset committed, a transient error caused consumer group rebalance → message reprocessed from last committed (earlier) offset → duplicate DB write
**Root cause:** At-least-once delivery semantics without idempotent consumer-side write logic (no upsert/dedup key in DB insert).
**Resolution:** Flagged to dev team — recommended either idempotent DB writes (unique constraint on `correlationId`) or manual offset commit only after DB write confirmed. This became a design fix, not just a test config fix.
**Interview point:** This is a strong "found a real architectural gap via performance testing" story — moves you from "test executor" to "system reliability contributor," which is exactly the senior narrative panels want.

---

## 5. Kafka Sampler Plugin Memory Leak in Long Soak Run
**Symptom:** During an 8-hour soak test, JMeter injector JVM heap climbed steadily and eventually caused GC pauses, skewing the last 2 hours of results.
**Diagnosis:**
- Heap dump analysis showed pepper-box plugin retaining producer client references across iterations instead of reusing a single producer instance
**Root cause:** Known plugin-level inefficiency — producer object was being recreated per-sample instead of once per thread.
**Resolution:** Switched to a custom JSR223 Sampler with a single shared `KafkaProducer` instance per thread (created in `setUp Thread Group` equivalent, reused via `props`), rather than relying on the plugin default — solved the leak and gave more control anyway.
**Interview point:** Shows you don't just accept third-party plugin limitations — you diagnose and build a more efficient custom solution when needed. Very senior-level trait.

---

# ORDER JOURNEY API — PREPROD ISSUES

## 6. Inventory Reservation Race Condition Under Concurrency
**Symptom:** At 150+ concurrent orders, ~1.5% of orders received duplicate SIM numbers — two customers got the same SIM assigned.
**Diagnosis:**
- Built a JSR223 Assertion tracking all assigned SIM numbers in a shared `props` Set across threads to catch duplicates programmatically instead of manually scanning logs
- Confirmed the Inventory service's reservation logic used a read-then-write pattern (`SELECT available SIM → UPDATE status`) without proper row-level locking
**Root cause:** Classic TOCTOU (time-of-check-to-time-of-use) race condition in the inventory service — two threads read the same "available" SIM before either wrote the reservation.
**Resolution:** Reported with reproducible evidence (specific correlationId pairs + timestamps within milliseconds of each other) — dev team added `SELECT FOR UPDATE` row locking. Retested, zero duplicates at 200 concurrent.
**Interview point:** Concurrency bugs are the single strongest "senior performance engineer" story type — they require you to think beyond response times into actual data-correctness-under-load, which most mid-level testers don't catch.

---

## 7. Order Journey SLA Breach Traced to One Silent Bottleneck Step
**Symptom:** Overall journey time SLA (<30s) was breaching at ~35s average under 200 concurrent users, but no individual step looked obviously slow in the overall Summary Report.
**Diagnosis:**
- Because each step was wrapped in its own Transaction Controller, could break down exactly where time was going — found "Billing Account Creation" step alone was taking 8s at peak concurrency vs 1.2s at low load, while other steps stayed flat
- Correlated with Dynatrace APM trace on Billing service — found a downstream call to a legacy mainframe billing system was the actual bottleneck, not the Billing microservice itself
**Root cause:** Billing microservice was fine; it was blocking synchronously on a legacy backend system with a small connection pool (10 connections) that saturated under concurrent load.
**Resolution:** Recommended either async billing account creation (queue-based) or increasing the legacy system's connection pool — pool size was increased as a short-term fix, and async pattern was logged as a longer-term architecture recommendation.
**Interview point:** Demonstrates using Transaction Controller-level breakdown + APM correlation to trace an SLA breach to its true root cause rather than reporting "the flow is slow" vaguely — very concrete "how do you diagnose a bottleneck" answer.

---

## 8. False Positives from Retry Logic Masking Real Failures
**Symptom:** Aggregate error rate looked healthy (<0.5%) but the "If Controller" retry-on-PENDING logic for inventory was silently retrying and re-passing steps that should have been flagged.
**Diagnosis:**
- Reviewed JSR223 PostProcessor logs and noticed the retry counter was incrementing far more than expected — nearly 15% of orders needed at least one retry, but this never surfaced in the pass/fail report since a successful retry still counted as "success"
**Root cause:** Test design flaw — retry logic was masking a real system weakness (inventory service frequently returning PENDING status under load, requiring retries that added hidden latency).
**Resolution:** Added a separate custom metric (`retryCount` logged per order) reported alongside pass/fail, exposing that 15% of orders had a hidden ~3s latency penalty from retries — reframed this as a P1 performance finding even though the "assertion" technically passed.
**Interview point:** Excellent story about not letting your own test design hide real problems — a senior tester questions whether "green" results are actually telling the full story, not just accepting pass rates at face value.

---

## 9. CSV Data Set Exhaustion Causing Data Collisions Mid-Run
**Symptom:** Around the 45-minute mark of a 1-hour soak run, started seeing "customer already exists" errors that weren't present earlier.
**Diagnosis:**
- CSV Data Set Config was set to "Recycle on EOF = True" — once all rows were consumed, it looped back to row 1, causing the same `customerId` to be reused while an earlier order for that same ID might still be mid-flight (especially during the async activation polling step)
**Root cause:** Test data pool was too small for the run duration and concurrency level — recycling caused legitimate data collisions, not a real system bug.
**Resolution:** Generated a much larger CSV (100K+ unique customer records via a script) and disabled recycling, or alternatively used JSR223 to append a run-timestamp suffix to customerId making each iteration inherently unique regardless of recycling.
**Interview point:** Good "self-caught test design flaw" story — shows discipline in distinguishing real system defects from test artifact noise, an important trust-building skill for a Test Lead role.

---

## 10. Environment Parity Gap — Preprod Config Drift
**Symptom:** Load test in preprod showed excellent numbers (SLA met comfortably), but a subsequent smaller-scale test in a "prod-like" staging environment showed much worse latency at even lower load.
**Diagnosis:**
- Compared infra specs and configs between environments — preprod database connection pool was sized larger than production, and preprod had autoscaling enabled on the API gateway while production had fixed instance count at the time
**Root cause:** Environment parity gap — preprod wasn't truly representative of production capacity, giving falsely optimistic results.
**Resolution:** Flagged the config drift to infra/DevOps before sign-off, requested either preprod be resized to match production exactly, or explicitly caveat the go/no-go report noting the parity gap and recommend a follow-up test post-alignment.
**Interview point:** This is a very senior-level point to raise unprompted in an interview — "good numbers in preprod are only meaningful if preprod is representative" shows maturity beyond just running scripts and reporting green/red.

---

## HOW TO STRUCTURE THESE IN INTERVIEW ANSWERS (STAR quick-map)

| Situation | Task | Action | Result |
|---|---|---|---|
| Consumer lag growing | Diagnose Kafka scaling issue | Checked partition/consumer ratio | Fixed via partition increase, lag <5s |
| Message loss on failover | Validate durability under chaos | Found acks=1 misconfig | Corrected to acks=all, zero loss confirmed |
| SIM duplication | Catch data-integrity issue under concurrency | Built shared-state assertion to detect dupes | Dev fixed row-locking, zero dupes retest |
| Silent SLA breach | Find hidden bottleneck | TC-level breakdown + APM correlation | Traced to legacy billing pool, fixed |
| Hidden retry masking latency | Question "green" results | Added custom retry metric | Exposed 15% hidden latency, reframed as P1 |

These ten give you a genuinely deep bench of "tell me about a time you found a critical issue" answers spanning **scalability, data integrity, concurrency, root-cause diagnosis, test design self-correction, and environment governance** — the full spread senior PT Lead interviews probe across multiple questions.

Want me to turn 2–3 of these into fully written STAR-format spoken answers (the kind you'd actually say out loud in the interview, ~90 seconds each)?
