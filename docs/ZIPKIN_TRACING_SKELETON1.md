# Distributed Tracing (Zipkin) - Code Template + Reading Guide

---

# PART 1 - CODE (copy-paste ready, file locations marked)

### File: .gitignore  (repo root)
```
zipkin.jar
```

### Command: run Zipkin (a downloaded tool, run from anywhere)
```
curl -sSL https://zipkin.io/quickstart.sh | bash -s
java -jar zipkin.jar
```

### File: pom.xml - of EVERY service that should appear in a trace
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

### File: pom.xml - ONLY of the service(s) making outgoing Feign calls (e.g. order-service)
```xml
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-micrometer</artifactId>
</dependency>
```

### File: microservices-config-repo/user-service.properties (same 4 lines also in product-service.properties, order-service.properties)
```properties
management.tracing.sampling.probability=1.0
management.tracing.export.zipkin.endpoint=http://localhost:9411/api/v2/spans
management.tracing.propagation.type=B3
logging.pattern.correlation=[${spring.application.name:},%X{traceId:-},%X{spanId:-}]
```

### Startup order
```
1. java -jar zipkin.jar
2. config-server
3. eureka-server
4. user-service, product-service   (leaf services)
5. order-service                    (calls the leaf services)
6. api-gateway
```

### Verification command
```
curl http://localhost:9411/api/v2/services
```

---

# PART 2 - DOCUMENTATION (why each piece of code above exists)

## Why each file/dependency is where it is

**.gitignore - why:** zipkin.jar is a ~129MB downloaded tool, not your code. GitHub rejects files over 100MB. If already committed: `git reset --soft origin/main` (undo unpushed commits, keep changes staged) then `git reset HEAD <path-to-jar>` (unstage) then add ignore line then recommit clean then push.

**spring-boot-starter-actuator + spring-boot-starter-zipkin - why on every service:** These make a service capable of generating its own trace/span data and sending it to Zipkin. Without them nothing about that service ever appears in Zipkin, regardless of config correctness.

VERSION GOTCHA: Boot 3.x used two separate manual dependencies (micrometer-tracing-bridge-brave + zipkin-reporter-brave). Boot 4.x folded both into the single spring-boot-starter-zipkin starter. Adding the old 3.x pair on a 4.x project doesn't error - it silently does nothing. Check which Boot major version you're on and match dependencies to CURRENT official docs, not an older tutorial.

**feign-micrometer - why only on the calling service:** The dependencies above make a service generate its OWN trace when it receives a request. They do NOT make an outgoing Feign call carry that trace forward in its headers. Without feign-micrometer, order-service generates Trace A, then calls user-service with plain HTTP carrying no trace headers - user-service has no idea it's part of anything and starts a brand new Trace B. Symptom: order-service/user-service/product-service each show a separate trace instead of one connected trace with "Services: 3". feign-micrometer wraps the Feign client to stamp the current trace context onto every outgoing call.

How to verify correctly: compare the FULL Trace ID (all 32 hex chars) across services, not just the first few characters. Brave's 128-bit trace ID embeds a timestamp in its high-order bits, so unrelated traces created in the same second coincidentally share a prefix. Only a full match proves they are the same trace.

**Properties - why per-service, not shared:** Each service reports to Zipkin independently and needs its own copy of these 4 lines alongside its existing Config Server-fetched properties.

VERSION GOTCHA AGAIN: Boot 3.x property key was management.zipkin.tracing.endpoint. Boot 4.x renamed it to management.tracing.export.zipkin.endpoint. The old key doesn't error on Boot 4 - it's silently ignored, so tracing quietly does nothing. This is the single most common "I added everything and it's still not working" cause - always check the property name against your actual Boot version's docs.

sampling.probability=1.0 traces 100% (fine for learning; lower in real production, e.g. 0.1, since tracing every request at scale adds overhead). logging.pattern.correlation is not required for tracing to work - it prints [service-name,traceId,spanId] into every log line so you can grep logs and find the matching Zipkin trace directly.

**Startup order - why it matters:** If order-service starts before Zipkin is reachable, its first requests simply fail to report (app will not crash - reporting failures are non-fatal). Starting Zipkin first avoids a confusing "why isn't it showing up" moment.

**The verification curl - why before the UI:** An empty [] tells you unambiguously that no data arrived at all - a dependency/config problem. Skipping this and going straight to the UI, an empty results screen looks identical whether the real problem is "no data ever arrived" or "your UI filter is wrong" - you will debug the wrong layer.

---

## Part 3 - How to Read an Actual Trace

### Reading the search-results list first

Column meanings:
- Root: which service + method + path started this trace
- Start Time: cross-reference against your own Postman History if unsure
- Spans: how many steps the request broke into
- Duration: total time for the whole request
- Warning icon: something inside this trace errored

A missing/generic path in the root row (e.g. "http get" instead of "http get /api/users/{id}") is itself a clue - usually the request never matched a Controller mapping, often because security rejected it (403) BEFORE routing could identify the endpoint, or it hit an undefined path like "/".

### Opening a trace - the summary line

Duration 309.202ms | Services 3 | Total Spans 9 | Trace ID 6a99e3aa83f1e74ea5405f77330702eb

"Services > 1" confirms multiple microservices are genuinely connected in one trace (enabled by feign-micrometer). Always compare the FULL Trace ID across services.

### Reading the waterfall (indentation = causality, not just order)

```
order-service: http post /api/orders                 -> 309.202ms  (root span, whole request)
  order-service: http get                             -> 128.771ms (Feign call OUT to user-service)
    user-service: http get /api/users/{id}             -> 44.608ms  (user-service's own handling)
      security filterchain before                       -> 31.143ms  (your JwtAuthFilter runs here)
      authorize request                                 -> 472us    (role/permission check)
      secured request                                    -> 11.905ms (your Controller->Service->Repo)
      security filterchain after                          -> 577us
  order-service: http get                             -> 48.071ms  (Feign call OUT to product-service)
    product-service: http get /api/products/{id}        -> 26.428ms
```

- A span appearing TWICE under the same parent is normal - each represents a separate outgoing Feign call.
- Compare a span's duration to its PARENT's duration to separate network/overhead from actual work. Example: user-service call took 128.771ms as seen from order-service, but user-service itself only spent 44.608ms - the remaining ~84ms is network round-trip, not application logic.

### The automatic Spring Security spans (you never wrote this code)

Generated automatically by Spring Security's Micrometer integration (ObservationFilterChainDecorator) once tracing + actuator are on the classpath:

- "security filterchain before" -> your custom filters (e.g. JwtAuthFilter) - token extraction/validation
- "authorize request" -> Spring's AuthorizationFilter - evaluates your SecurityConfig rules
- "secured request" -> your real Controller -> Service -> Repository logic
- "security filterchain after" -> cleanup as response exits the filter chain

If "secured request" dominates total time, bottleneck is business logic/DB. If filter/authorize spans are unexpectedly large, something in auth itself is slow.

### Reading an error trace

Look at the "error" tag under the specific failing span, not just the red icon in the list. Signature seen in this project:
```
error: Access is denied
spring.security.authentication.authorities: [ROLE_ANONYMOUS]
spring.security.authorization.decision: [granted=false]
```
This means the request carried no valid token; Spring Security treated it as anonymous, hit a path requiring authentication. Common real cause: a browser tab auto-opened (e.g. Codespaces port globe icon) hitting bare root path "/" with zero headers - not an actual bug.

Always cross-check the trace's exact Start Time against your own action history to confirm which real request it was.

---

## Master Debugging Checklist (in order)

1. curl http://localhost:9411/api/v2/services - anything reporting at all?
2. Empty? Check pom.xml for the CURRENT starter name for your Boot version.
3. Check config repo property key matches your Boot version exactly.
4. Confirm the service was actually RESTARTED after any config/dependency change.
5. Services show up individually but never connected (Services:1 each, different full Trace IDs)? Add feign-micrometer to the calling service, re-verify with full Trace ID comparison.
6. Filter by serviceName using the dropdown, never by typing.
