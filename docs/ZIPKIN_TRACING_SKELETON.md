
# Distributed Tracing (Zipkin) — Reusable Skeleton & Trace-Reading Guide

> Use this whenever a new multi-service project needs distributed tracing.

---

## Part 1 — Setup

### 1. Run Zipkin itself (not committed to Git)
curl -sSL https://zipkin.io/quickstart.sh | bash -s
java -jar zipkin.jar

Add to .gitignore immediately: echo "zipkin.jar" >> .gitignore
Lesson: GitHub rejects files >100MB. If already committed:
git reset --soft origin/main -> unstage jar -> add to .gitignore -> recommit clean.

### 2. Dependencies — every service that appears in a trace
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-zipkin</artifactId>
</dependency>

CRITICAL VERSION GOTCHA: Boot 3.x used two separate manual dependencies
(micrometer-tracing-bridge-brave + zipkin-reporter-brave). Boot 4.x folded
both into spring-boot-starter-zipkin above. Adding the old pair on Boot 4.x
means tracing silently never activates — no error at all.

### 3. Dependency — ONLY on services making outgoing Feign calls
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-micrometer</artifactId>
</dependency>

Why separate: step 2 makes a service generate ITS OWN traces. It does NOT
forward trace context to services it calls via Feign. Without this,
every service shows a disconnected trace with a DIFFERENT Trace ID —
even though logically it's one request. Confirmed by comparing FULL
Trace IDs (not just prefix — Brave's 128-bit IDs embed a timestamp in
the high bits, so same-second unrelated traces share a prefix by pure
coincidence).

### 4. Properties (in centralized config repo, per service)
management.tracing.sampling.probability=1.0
management.tracing.export.zipkin.endpoint=http://localhost:9411/api/v2/spans
management.tracing.propagation.type=B3
logging.pattern.correlation=[${spring.application.name:},%X{traceId:-},%X{spanId:-}]

VERSION GOTCHA AGAIN: Boot 3.x property was management.zipkin.tracing.endpoint.
Boot 4.x renamed it to management.tracing.export.zipkin.endpoint. Old key is
silently ignored - no warning, tracing just does nothing. Check current
official docs when something "should work" and doesn't.

sampling.probability=1.0 = trace 100% (fine for learning; lower in real
production, e.g. 0.1). logging.pattern.correlation prints
[service-name,traceId,spanId] into every log line for cross-referencing.

### 5. Startup order
1. Zipkin (java -jar zipkin.jar)
2. Config Server
3. Eureka Server
4. Leaf services (user-service, product-service, ...)
5. Services calling other services (order-service, ...)
6. API Gateway

### 6. Verify in this order (don't jump to the UI)
curl http://localhost:9411/api/v2/services

Empty [] = nothing reported yet — a config/dependency problem, not a UI
problem. Fix before touching the Zipkin web UI.

Once services appear, select serviceName FROM THE DROPDOWN, not by typing
- typing can leave a leading space (?serviceName=+order-service), which
silently returns zero results.

---

## Part 2 — How to Read a Trace

### Step A — Reading the search-results list

Column meanings:
- Root: which service + method + path started this trace
- Start Time: cross-reference with your own action history if unsure
- Spans: how many steps the request broke into
- Duration: total time for the whole request
- Warning icon: something inside this trace errored

A missing/generic path (e.g. "http get" instead of "http get /api/users/{id}")
is itself a clue — usually means the request never matched a Controller
mapping, often because security rejected it (403) BEFORE routing could
identify the endpoint template, or it hit an undefined path like "/".

### Step B — Opening a trace, top summary

Duration 309.202ms | Services 3 | Total Spans 9 | Trace ID 6a99e3aa83f1e74ea5405f77330702eb

"Services > 1" confirms multiple microservices are genuinely connected in
ONE trace (enabled by feign-micrometer). Always compare the FULL Trace ID
across services, never just the first few characters.

### Step C — Reading the waterfall (indentation = causality)

Example (order-creation request):
order-service: http post /api/orders                 -> 309.202ms  (root span, whole request)
  order-service: http get                             -> 128.771ms (Feign call OUT to user-service)
    user-service: http get /api/users/{id}             -> 44.608ms  (user-service's handling)
      security filterchain before                       -> 31.143ms  (JwtAuthFilter runs here)
      authorize request                                 -> 472us    (role/permission check)
      secured request                                    -> 11.905ms (actual Controller->Service->Repo)
      security filterchain after                          -> 577us
  order-service: http get                             -> 48.071ms  (Feign call OUT to product-service)
    product-service: http get /api/products/{id}        -> 26.428ms

Rules:
- Indentation = "happened because of the span above," not just chronological.
- A span appearing twice under the same parent = separate outgoing Feign
  calls, auto-created by feign-micrometer.
- Compare a span's duration to its PARENT's duration to see network/overhead
  vs actual work. Example: user-service call took 128.771ms as seen from
  order-service, but user-service itself only spent 44.608ms — the rest
  (~84ms) is network round-trip, not application logic.

### Step D — Automatic Spring Security spans (you never wrote this code)

Generated automatically by Spring Security's ObservationFilterChainDecorator
(Micrometer integration):

- "security filterchain before" -> your custom filters (e.g. JwtAuthFilter) -
  token extraction/validation, setting SecurityContextHolder
- "authorize request" -> Spring's AuthorizationFilter - evaluates your
  SecurityConfig rules (.hasRole(...), .authenticated())
- "secured request" -> your actual Controller -> Service -> Repository code
  - the real business logic
- "security filterchain after" -> cleanup as response exits the filter chain

If "secured request" dominates duration, bottleneck is business logic/DB.
If the filter/authorize spans are unexpectedly large, something in auth
itself is slow.

### Step E — Reading an error trace

Look for the "error" tag under the failing span, not just the red icon.
Signatures seen in this project:
- error: Access is denied + authorities: [ROLE_ANONYMOUS] +
  decision: [granted=false] -> request had no valid token, Spring Security
  treated it as anonymous, hit a path requiring authentication. Common
  cause: a browser tab auto-opened (e.g. Codespaces port globe icon)
  hitting bare root path "/" with zero headers.
- 404-style failure, no error tag, generic/missing path in root span name
  -> the path never matched any @RequestMapping.

Always correlate the trace's exact Start Time against your own action
history (Postman History, etc.) to confirm which request it actually was.

---

## Master Debugging Checklist (in order)

1. curl http://localhost:9411/api/v2/services — anything reporting at all?
2. If empty: check pom.xml for exact starter (spring-boot-starter-zipkin,
   not the old manual pair) and spring-boot-starter-actuator.
3. Check config repo property name matches your Boot major version exactly.
4. Confirm the service was actually RESTARTED after any config/dependency
   change.
5. If services appear individually but never connected: add feign-micrometer
   to the calling service only, verify by comparing FULL Trace IDs.
6. Use the Zipkin service-name filter via dropdown, never by typing.
