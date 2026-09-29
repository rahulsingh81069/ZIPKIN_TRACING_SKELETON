# Distributed Tracing (Zipkin) — Reusable Skeleton & Trace-Reading Guide

> Use this whenever a new multi-service project needs distributed tracing.

---

## Part 1 — Setup

### 1. Run Zipkin itself (not committed to Git)

**Theory:**
GitHub rejects files >100MB. If you accidentally commit the zipkin.jar file, you'll need to remove it from Git history. The lesson here is to add it to .gitignore immediately after download to prevent this issue.

**Code:**

```bash
curl -sSL https://zipkin.io/quickstart.sh | bash -s
java -jar zipkin.jar
```

**Clean up if already committed:**

```bash
echo "zipkin.jar" >> .gitignore
git reset --soft origin/main
# Then add .gitignore and recommit
```

---

### 2. Dependencies — every service that appears in a trace

**Theory:**
Every microservice that participates in a distributed trace needs the Actuator and Zipkin starter dependencies. These provide the instrumentation to generate and report trace spans to the Zipkin server.

**Code:**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-zipkin</artifactId>
</dependency>
```

**Critical Version Gotcha:**
- **Boot 3.x:** Requires two separate manual dependencies: `micrometer-tracing-bridge-brave` + `zipkin-reporter-brave`
- **Boot 4.x:** Folded both into `spring-boot-starter-zipkin`

Adding the old pair to Boot 4.x means tracing silently never activates—no error at all, making this easy to miss.

---

### 3. Dependency — ONLY on services making outgoing Feign calls

**Theory:**
Step 2 makes a service generate its own traces. However, it does NOT automatically forward trace context to services it calls via Feign. Without this dependency, every downstream service in the call chain shows a disconnected trace with a DIFFERENT Trace ID—even though logically it's one request. The feign-micrometer dependency bridges this gap by instrumenting Feign clients to propagate trace context.

You can verify this works by comparing FULL Trace IDs (not just the prefix) across services. Brave's 128-bit IDs embed a timestamp in the high bits, so same-second unrelated traces may share a prefix by coincidence.

**Code:**

```xml
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-micrometer</artifactId>
</dependency>
```

---

### 4. Properties (in centralized config repo, per service)

**Theory:**
- `sampling.probability=1.0` traces 100% of requests (fine for learning; use 0.1 or lower in production to reduce overhead)
- `management.tracing.propagation.type=B3` ensures W3C Trace Context is propagated across services
- `logging.pattern.correlation` adds trace and span IDs to every log line, making it easy to correlate logs with traces in Zipkin

**Critical Version Gotcha:**
- **Boot 3.x:** Property was `management.zipkin.tracing.endpoint`
- **Boot 4.x:** Renamed to `management.tracing.export.zipkin.endpoint`

The old key is silently ignored with no warning—tracing just does nothing. Always check current official docs when something "should work" but doesn't.

**Code:**

```properties
management.tracing.sampling.probability=1.0
management.tracing.export.zipkin.endpoint=http://localhost:9411/api/v2/spans
management.tracing.propagation.type=B3
logging.pattern.correlation=[${spring.application.name:},%X{traceId:-},%X{spanId:-}]
```

---

### 5. Startup order

**Theory:**
Services must start in a specific order to ensure all dependencies (service discovery, config, tracing backend) are available before services attempt to register or emit traces.

**Order:**

```
1. Zipkin (java -jar zipkin.jar)
2. Config Server
3. Eureka Server
4. Leaf services (user-service, product-service, ...)
5. Services calling other services (order-service, ...)
6. API Gateway
```

---

### 6. Verify in this order (don't jump to the UI)

**Theory:**
An empty services list typically indicates a configuration or dependency problem, not a UI issue. Verify that services are actually reporting spans before troubleshooting the web interface. When using the Zipkin UI, always select service names from the dropdown rather than typing them—typing can leave leading spaces that cause silent query failures.

**Code:**

```bash
curl http://localhost:9411/api/v2/services
```

**Expected Response (example):**

```json
["order-service", "user-service", "product-service"]
```

**If empty [] is returned:**
This indicates a config/dependency problem. Fix the configuration before accessing the Zipkin web UI.

**UI Tip:**
Select serviceName FROM THE DROPDOWN, not by typing. Typing can leave a leading space (`?serviceName=+order-service`), which silently returns zero results.

---

## Part 2 — How to Read a Trace

### Step A — Reading the search-results list

**Theory:**
The search results table shows high-level metadata about each trace. Understanding these columns helps you quickly identify which trace corresponds to your request and spot obvious performance issues.

**Column Meanings:**

| Column | Meaning |
|--------|---------|
| **Root** | Which service + method + path started this trace |
| **Start Time** | Timestamp of trace initiation; cross-reference with your action history if unsure |
| **Spans** | How many steps the request broke into (indicates microservice hops) |
| **Duration** | Total end-to-end time for the whole request |
| **Warning icon** | Indicates something inside this trace errored |

**Diagnostic Clues:**
A missing or generic path (e.g., "http get" instead of "http get /api/users/{id}") is itself a clue—usually means the request never matched a Controller mapping. Common causes:
- Security rejected it (403) BEFORE routing could identify the endpoint template
- Request hit an undefined path like "/"

---

### Step B — Opening a trace, top summary

**Theory:**
The summary header confirms whether multiple microservices are genuinely connected in ONE trace. This is enabled by the feign-micrometer dependency. Always compare the FULL Trace ID across services to verify connection, never just the first few characters.

**Example Summary:**

```
Duration 309.202ms | Services 3 | Total Spans 9 | Trace ID 6a99e3aa83f1e74ea5405f77330702eb
```

**What it tells you:**
- `Services > 1` confirms multiple microservices are genuinely connected in one trace
- The full Trace ID should appear in every span from every service in the chain

---

### Step C — Reading the waterfall (indentation = causality)

**Theory:**
The waterfall view shows the chronological and causal relationship between spans. Indentation indicates "happened because of the span above," not just time order. A span's duration compared to its parent's duration reveals whether time was spent on network overhead or actual application logic.

**Example (order-creation request):**

```
order-service: http post /api/orders                 -> 309.202ms  (root span, whole request)
  order-service: http get                             -> 128.771ms (Feign call OUT to user-service)
    user-service: http get /api/users/{id}             -> 44.608ms  (user-service's handling)
      security filterchain before                       -> 31.143ms  (JwtAuthFilter runs here)
      authorize request                                 -> 472us    (role/permission check)
      secured request                                    -> 11.905ms (actual Controller->Service->Repo)
      security filterchain after                          -> 577us
  order-service: http get                             -> 48.071ms  (Feign call OUT to product-service)
    product-service: http get /api/products/{id}        -> 26.428ms
```

**Reading Rules:**

1. **Indentation shows causality**, not just chronological order
2. **A span appearing twice under the same parent** = separate outgoing Feign calls, auto-created by feign-micrometer
3. **Compare child duration to parent duration** to isolate network overhead vs actual work:
   - Example: user-service call took 128.771ms from order-service's perspective
   - But user-service itself only spent 44.608ms
   - The difference (~84ms) is network round-trip, not application logic

---

### Step D — Automatic Spring Security spans (you never wrote this code)

**Theory:**
Spring Security's `ObservationFilterChainDecorator` (Micrometer integration) automatically creates spans for each stage of request processing. These spans help you identify where authentication and authorization overhead occurs, and where your actual business logic runs.

**Auto-Generated Spans:**

```
security filterchain before
  → your custom filters (e.g., JwtAuthFilter)
  → token extraction/validation, setting SecurityContextHolder

authorize request
  → Spring's AuthorizationFilter
  → evaluates your SecurityConfig rules (.hasRole(...), .authenticated())

secured request
  → your actual Controller → Service → Repository code
  → the real business logic

security filterchain after
  → cleanup as response exits the filter chain
```

**Performance Analysis:**
- **If "secured request" dominates duration:** Bottleneck is in business logic or database queries
- **If filter/authorize spans are unexpectedly large:** Something in the authentication/authorization itself is slow

---

### Step E — Reading an error trace

**Theory:**
When a request fails, look for the "error" tag under the failing span, not just the red icon. Different error signatures point to different root causes. Cross-referencing the trace's Start Time with your action history (Postman, browser logs) confirms which request actually failed.

**Common Error Signatures:**

**Signature 1: Missing or Invalid Token**

```
error: Access is denied
authorities: [ROLE_ANONYMOUS]
decision: [granted=false]
```

**Diagnosis:** Request had no valid token; Spring Security treated it as anonymous and hit a path requiring authentication. Common cause: a browser tab auto-opened (e.g., Codespaces port globe icon) hitting bare root path "/" with zero headers.

**Signature 2: Path Not Found**

```
404-style failure
no error tag
generic/missing path in root span name (e.g., "http get" instead of specific path)
```

**Diagnosis:** The path never matched any `@RequestMapping`. Check your controller mappings.

**Debugging Step:**
Always correlate the trace's exact Start Time against your own action history (Postman History, browser dev tools, etc.) to confirm which request it actually was.

---

## Master Debugging Checklist (in order)

**Step 1: Services Reporting?**

```bash
curl http://localhost:9411/api/v2/services
```

If empty `[]` is returned, proceed to Step 2. Otherwise, skip to Step 5.

---

**Step 2: Check pom.xml**

Ensure you have the correct starter for your Spring Boot version:

```xml
<!-- Correct for Boot 4.x -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-zipkin</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<!-- NOT correct for Boot 4.x (Boot 3.x style) -->
<!-- Do NOT use both of these: -->
<!-- micrometer-tracing-bridge-brave -->
<!-- zipkin-reporter-brave -->
```

---

**Step 3: Check Config Property Name**

Verify the property name matches your Boot major version:

```properties
# Boot 4.x (CORRECT)
management.tracing.export.zipkin.endpoint=http://localhost:9411/api/v2/spans

# Boot 3.x (old, will be silently ignored on Boot 4.x)
# management.zipkin.tracing.endpoint=http://localhost:9411/api/v2/spans
```

---

**Step 4: Restart Services**

Confirm the service was actually RESTARTED after any config or dependency change. Configuration changes don't take effect without a restart.

---

**Step 5: Services Appear but Never Connected?**

Add `feign-micrometer` to the calling service only:

```xml
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-micrometer</artifactId>
</dependency>
```

Verify by comparing FULL Trace IDs—they should match across services in the same logical request.

---

**Step 6: Use Zipkin UI Correctly**

When viewing traces in the Zipkin web UI, always select service names from the dropdown, never by typing. Typing can leave leading spaces that cause silent failures.

---

**End of Guide**
