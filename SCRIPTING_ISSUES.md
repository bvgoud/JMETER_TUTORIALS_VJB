Here's a set of realistic **scripting-phase** issues (build-time, not runtime load issues) — the kind that come up while actually constructing the JMX and prove hands-on scripting depth in an interview.

---

## 1. Correlation Broken by Non-Unique Regex Match
**Symptom:** Regular Expression Extractor was pulling the wrong `orderId` — script worked for the first iteration then broke on subsequent ones.
**Diagnosis:**
- Response body had `orderId` appearing twice — once in a `relatedOrders` array and once in the main object
- Regex `"orderId":"(.*?)"` matched the first occurrence (wrong one) since Match No. was set to 1 without checking result structure
**Root cause:** Regex was too generic and matched an unintended field with the same name.
**Resolution:** Switched to JSON Extractor with a precise JSONPath (`$.order.orderId` instead of a global scan), which is structure-aware rather than pattern-based — eliminated ambiguity entirely.
**Interview point:** Shows why JSON Extractor is preferred over regex for structured responses — regex only "looks" at text, doesn't understand JSON hierarchy.

---

## 2. Dynamic Token Expiring Mid-Script During Long Iterations
**Symptom:** Intermittent 401 errors appearing only in later steps of a long multi-step order journey, not at login.
**Diagnosis:**
- Token had a 60-second expiry; journey with polling (While Controller) sometimes took 90+ seconds due to multiple retry cycles
- Token captured once via Once Only Controller was stale by the time later steps executed
**Root cause:** Token lifetime assumption didn't account for realistic script execution time, especially with async polling steps.
**Resolution:** Added a JSR223 PreProcessor before each Transaction Controller checking token age (`vars.get("tokenIssuedAt")`), triggering a silent re-login sampler if older than 50 seconds — kept the script self-healing instead of hardcoding a single login-once pattern.
**Interview point:** Good example of scripting for real-world token lifetimes, not just assuming static session validity for the whole script duration.

---

## 3. CSV Data Set Config Sharing Across Thread Groups Not Working as Expected
**Symptom:** Producer Thread Group and Consumer Thread Group (Kafka project) both needed the same `subscriberId` values, but each Thread Group's CSV Data Set Config was reading independently, causing mismatched values between producer and consumer sides.
**Diagnosis:** Each Thread Group has its own execution context; a CSV Data Set Config scoped separately per Thread Group reads its own pointer position, not shared.
**Root cause:** Misunderstanding CSV Data Set Config scope — it's not automatically synchronized across parallel Thread Groups.
**Resolution:** Moved the CSV Data Set Config to Test Plan level (not inside a Thread Group), which makes it a single shared file pointer across all Thread Groups reading it — or alternatively, generated the value once and propagated via `props` (shared) instead of relying on CSV in both places.
**Interview point:** Scope/shared-state understanding — a common scripting mistake even at mid-level, worth explicitly calling out that you know the difference.

---

## 4. JSON Body with Special Characters Breaking Request Payload
**Symptom:** Requests with a customer name containing an apostrophe (e.g., "O'Brien") or address with quotes caused malformed JSON and 400 errors — intermittent, only for specific CSV rows.
**Diagnosis:** Raw string concatenation in HTTP Request body wasn't escaping special characters properly.
**Root cause:** Body was built as a plain string template (`{"name":"${customerName}"}`) instead of proper JSON serialization, so unescaped quotes/apostrophes broke the JSON structure.
**Resolution:** Moved payload construction into a JSR223 PreProcessor using `groovy.json.JsonOutput.toJson()` (proper object serialization), eliminating manual string concatenation entirely.
```groovy
def payload = [name: vars.get("customerName"), address: vars.get("address")]
vars.put("requestBody", groovy.json.JsonOutput.toJson(payload))
```
**Interview point:** Classic "why raw string templating for JSON bodies is fragile" lesson — proper serialization avoids an entire class of data-driven bugs.

---

## 5. HTTPS Recording Captured Encrypted/Garbled Bodies
**Symptom:** After recording via HTTP(S) Test Script Recorder, several samplers showed unreadable binary/garbled response and request bodies.
**Diagnosis:** App used gzip/brotli compression by default, and some calls used a mobile-specific binary protocol layer on top of HTTPS.
**Root cause:** Recorder captures raw traffic; compressed/encoded bodies aren't auto-decoded for display unless content-encoding headers are handled correctly.
**Resolution:** Enabled "Content-Encoding" handling in Recorder settings and, for the binary protocol calls, excluded them from recording entirely and hand-built those specific samplers using API documentation instead — recording isn't always reliable for 100% of a flow.
**Interview point:** Shows realistic understanding that recording is a starting point, not a complete solution — especially for non-standard/mobile traffic.

---

## 6. Thread Group Ramp-Up Miscalculation Causing Uneven Load Start
**Symptom:** First few seconds of test showed unrealistic spike in requests despite a configured ramp-up period.
**Diagnosis:** Confused "ramp-up time" with "ramp-up per thread" — with 100 threads and 10s ramp-up, JMeter starts 1 thread every 0.1s, which is correct, but the script also had a Loop Controller with zero inter-iteration delay, so once threads started, they fired requests back-to-back immediately with no pacing.
**Root cause:** Missing Timer at Thread Group level — ramp-up controls *thread start* spacing, not *request* spacing between iterations.
**Resolution:** Added a Uniform Random Timer at Thread Group scope to properly pace requests within each thread's loop, separate from the ramp-up setting.
**Interview point:** A very common scripting misconception worth explicitly clarifying in interviews — ramp-up ≠ request pacing, they're two separate controls.

---

## 7. Variable Not Available Across Threads/Scopes as Expected
**Symptom:** A JSR223 script correctly set a variable in the Producer Thread Group, but the Consumer Thread Group's script showed it as `null`.
**Diagnosis:** Used `vars.put()` which is thread-local — value never crosses Thread Group boundaries or even between threads in the same group.
**Root cause:** Fundamental `vars` vs `props` misunderstanding early in script build.
**Resolution:** Switched to `props.put()`/`props.get()` for anything that needs to be visible across threads/Thread Groups, and used thread-safe collections (`Collections.synchronizedMap`) since multiple threads write concurrently.
**Interview point:** Already covered this distinction earlier, but worth having as a specific "I made this mistake once while scripting and now always check scope first" honesty moment — interviewers value that kind of grounded experience over claiming perfection.

---

## 8. Header Manager Conflict — Duplicate Headers Sent
**Symptom:** API rejected requests with a "duplicate header" error, even though only one Header Manager was visibly added.
**Diagnosis:** Had an HTTP Header Manager at Thread Group level (global headers) AND another one inside a specific HTTP Request sampler (for a one-off header) — both got merged and the `Content-Type` header ended up duplicated instead of overridden.
**Root cause:** Misunderstanding of Header Manager scope merging behavior — nested Header Managers add to, not replace, parent-level headers.
**Resolution:** Consolidated to a single Thread-Group-level Header Manager with common headers, and used JSR223 PreProcessor (`sampler.getHeaderManager().add(...)`) only for genuinely dynamic one-off headers per request, avoiding nested static Header Manager duplication.
**Interview point:** Good detail-level scripting knowledge — shows you understand element merging/scope behavior beyond just "add it and hope."

---

## 9. Parameterized Test Data Causing Unrealistic Cache Hit Rates
**Symptom:** Response times were unrealistically fast (too good) during early test runs.
**Diagnosis:** CSV Data Set Config was configured with "Recycle" and a small dataset (100 rows) reused constantly across thousands of iterations — server-side caching layer was serving cached responses for the same repeated `msisdn`/`customerId` values.
**Root cause:** Insufficiently large/unique test data pool caused the system under test to behave better than it would with real, unique production traffic.
**Resolution:** Generated a much larger, genuinely unique dataset (tens of thousands of rows via a script) to better simulate real-world cache-miss ratios, giving a more honest performance picture.
**Interview point:** Strong "test data realism" story — shows understanding that small/repetitive datasets can produce misleadingly good results, an easy trap for less experienced scripters to fall into.

---

## 10. JSR223 Script Performance Overhead on Load Generator Itself
**Symptom:** JMeter injector CPU usage was unexpectedly high, limiting achievable thread count per injector machine.
**Diagnosis:** Profiled the injector and found heavy JSR223 script usage (complex Groovy JSON parsing/building) in the hot path of every single sampler iteration, executed at high concurrency.
**Root cause:** Overuse of scripting where lighter-weight built-in elements (JSON Extractor, User Defined Variables) would have sufficed — Groovy script compilation/execution carries real overhead at scale.
**Resolution:** Replaced non-essential JSR223 usage with native JMeter elements wherever possible, reserving JSR223 only for logic that genuinely required it (signature generation, cross-field validation) — reduced injector CPU usage meaningfully, allowing higher thread density per machine.
**Interview point:** Excellent "watch your own tool's footprint" story — shows maturity in recognizing that the load generator itself can become the bottleneck if scripts aren't efficient, not just the system under test.

---

## Quick reference table for your notebook

| Issue | Category | Root cause type |
|---|---|---|
| Wrong correlation match | Extractor | Regex too generic |
| Token expiry mid-script | Session handling | Wrong lifetime assumption |
| CSV not shared across TGs | Data/scope | Config Element scope misunderstanding |
| Malformed JSON body | Payload construction | String concat instead of serialization |
| Garbled recorded bodies | Recording | Compression/binary protocol not handled |
| Uneven load spike at start | Thread Group/Timer | Ramp-up vs pacing confusion |
| Variable null across threads | vars vs props | Scope misunderstanding |
| Duplicate headers | Header Manager | Scope merging behavior |
| Unrealistic cache hits | Test data | Dataset too small/repetitive |
| High injector CPU | Script efficiency | Overuse of JSR223 vs native elements |

These are exactly the kind of hands-on, "I built it, it broke, here's how I fixed it" details that separate someone who's read about JMeter from someone who's actually scripted hundreds of hours of it — good material to have ready when asked "tell me about a challenging script you built."

Want the **STAR-format spoken versions** of 2-3 of these next, so you have them ready to say out loud rather than just read from notes?
