# REST API Design Interview Guide — Every Hard Concept, Explained Step by Step

> **Goal:** Cover every REST API concept that actually separates a strong interview answer from a shallow one — not "what does GET mean," but the parts candidates consistently get wrong: idempotency proofs, error-response design (business outcomes vs. real errors), cursor pagination, optimistic concurrency, idempotency keys for unsafe operations, JWT pitfalls, and more — each with real, working Spring/Java code, and a sequence of escalating follow-up questions per topic, the way a real interview actually pushes back.
>
> This guide uses one running, real-world example throughout its error-design section: a real eligibility API, and the exact design tension between "is this a system error, or a business outcome" that trips up most teams in production, not just in interviews.

---

# 1. What We Are Building

A complete map of REST API design, organized the way a thorough interview actually escalates — from "what is REST, really" through the specific mechanics (status codes, error shapes, pagination, versioning, caching, concurrency, auth, idempotency, bulk/async operations) that a candidate who's only *used* REST APIs, rather than *designed* them, reliably gets wrong.

```text
Part 1: What REST Actually Is                    Part 8:  Caching and Conditional Requests
Part 2: Resource and URI Design                  Part 9:  Concurrency Control
Part 3: HTTP Methods, Safety, Idempotency         Part 10: Authentication and Authorization
Part 4: HTTP Status Codes Done Right              Part 11: Rate Limiting and Abuse Protection
Part 5: Error Response Design                     Part 12: Idempotency for Unsafe Operations
Part 6: Pagination, Filtering, Sorting            Part 13: Bulk and Long-Running Operations
Part 7: API Versioning                            Part 14: HATEOAS
                                                    Part 15: CORS
                                                    Part 16: REST vs. RPC vs. GraphQL vs. gRPC
                                                    Part 17: Security Best Practices
                                                    Part 18: Testing REST APIs
```

---

# 2. Learning Objectives

By the end of this guide you should be able to:

- State REST's actual architectural constraints precisely, and explain why "JSON over HTTP" is not, by itself, a sufficient definition.
- Prove, not just assert, which HTTP methods are safe and/or idempotent, and design `PATCH` semantics correctly.
- Design an error-response contract that cleanly separates genuine system/validation errors from valid business outcomes — the single most common real-world REST design mistake, and the one this guide's running example is built around.
- Implement cursor-based pagination and explain precisely why offset pagination breaks under concurrent writes.
- Implement optimistic concurrency control with `ETag`/`If-Match`, and idempotency-key handling for `POST` requests that must be safely retryable.
- Compare REST against RPC, GraphQL, and gRPC on their actual merits, and state precisely when REST is the wrong choice.

---

# 3. Why This Matters (Interview Motivation)

> **"Design a REST API for a real resource. I'm going to push back on every response shape, every status code, and every method choice you make, and I want to see whether your answers are principled or just habitual."**

REST is one of the most *over-assumed-understood* topics in software interviews — nearly every candidate has called a REST API, and a large fraction have never had to actually design one under real pushback:

- **The constraints are testable, not just describable** — a candidate who can recite "stateless, cacheable, uniform interface" but can't say what specifically breaks when an API violates statelessness hasn't actually internalized it.
- **Idempotency is provable, not a vibe** — "PUT is idempotent" is a claim with an actual proof (calling it N times produces the identical server state as calling it once); most candidates have memorized the conclusion without ever being asked to demonstrate it.
- **Error-response design is where real production systems actually get this wrong** — the tension in this guide's running example (is "insufficient balance" an error, or a business outcome?) is not a contrived interview scenario, it's a genuine, common design mistake with real downstream consequences for frontend consumers.

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language / Framework | Java 21, Spring Boot-style conventions | Matches this guide's `@RestController`/`@ExceptionHandler` code and the JVM ecosystem implied by a `GlobalExceptionHandler`. |
| Error format | RFC 7807 Problem Details (`application/problem+json`) | An actual IETF standard for HTTP API errors, covered in full in Part 5, rather than an ad-hoc bespoke shape. |
| Pagination | Cursor (keyset) pagination | The only pagination strategy that stays correct under concurrent writes to the underlying data, covered in Part 6. |
| Concurrency control | HTTP `ETag`/`If-Match` | Standard, protocol-level optimistic concurrency — no bespoke version-number scheme needed, covered in Part 9. |
| Auth | OAuth2/OIDC-issued JWTs | The dominant real-world pattern for stateless REST authentication, covered in Part 10. |

---

# 5. Project Structure

```text
rest-api-guide/
├── src/main/java/com/example/restapi/
│   ├── eligibility/
│   │   ├── EligibilityController.java, EligibilityResponse.java  // §21-§22
│   │   └── EligibilityReason.java
│   ├── error/
│   │   ├── GlobalExceptionHandler.java                            // §22
│   │   └── ProblemDetailBuilder.java                                // §24
│   ├── pagination/
│   │   └── CursorPage.java, CursorCodec.java                        // §28
│   ├── concurrency/
│   │   └── OptimisticConcurrencyFilter.java                          // §38
│   ├── auth/
│   │   └── JwtValidator.java                                          // §42
│   ├── idempotency/
│   │   └── IdempotencyKeyInterceptor.java                              // §49
│   └── bulk/
│       └── BulkOperationController.java                                 // §52
└── src/test/java/com/example/restapi/
    ├── ErrorContractShapeTest.java
    ├── CursorPaginationConsistencyTest.java
    └── OptimisticConcurrencyConflictTest.java
```

---

# Part 1: What REST Actually Is

# 6. The Six Constraints of REST (Most Candidates Only Know Two of Them)

REST (Representational State Transfer) is a specific, six-constraint architectural style, not a synonym for "JSON over HTTP":

1. **Client-server**: a clean separation of concerns — the client never manages server-side storage, the server never manages client-side UI state.
2. **Statelessness**: every request carries everything needed to understand it — the server holds **no client session state** between requests. (§7 pushes on this directly.)
3. **Cacheability**: every response must implicitly or explicitly declare whether it's cacheable — this is a first-class architectural constraint, not an optional optimization (Part 8).
4. **Uniform interface**: the same, small set of verbs and a consistent resource model apply everywhere — this is what makes a client able to interact with any compliant API without out-of-band knowledge.
5. **Layered system**: a client can't tell (and shouldn't need to) whether it's talking directly to the origin server or through a proxy/gateway/load balancer.
6. **Code-on-demand** (optional): a server can extend client functionality by transferring executable logic (e.g., client-side JavaScript) — the only *optional* constraint.

Most candidates can name statelessness and the use of HTTP verbs; far fewer can name all six, or explain what breaks when one is violated.

---

# 7. Follow-up — "Is a JSON-Over-HTTP API Automatically RESTful?"

> **Interviewer:** *"You've built an API that returns JSON over HTTP, uses GET/POST/PUT/DELETE, and looks exactly like every other 'REST' API you've seen. Is it RESTful?"*

Not necessarily — and the most commonly violated constraint in real-world "REST" APIs is **statelessness**: an API that requires a server-side session (a login cookie tied to in-memory server state, sticky-session load balancing) is not stateless, no matter how RESTful its URL scheme and verb usage look. A truly stateless API authenticates every request independently (a bearer token, not a session cookie tied to server memory), which is precisely why the dominant real-world pattern is a self-contained token (Part 10) rather than a server-side session store.

---

# 8. The Richardson Maturity Model: Levels 0-3

A practical way to describe *how* RESTful a given API actually is, not just whether it technically qualifies:

```text
Level 0: POX (Plain Old XML/JSON over HTTP) -- one URI, one HTTP verb (usually POST) for everything,
         RPC-style ("POST /api with {action: "getUser", id: 5}")

Level 1: Resources -- distinct URIs per resource ("/users/5"), but still often one verb for everything

Level 2: HTTP Verbs -- GET/POST/PUT/DELETE used correctly and distinctly, status codes used correctly
         (this is where the overwhelming majority of real-world "REST" APIs actually sit)

Level 3: HATEOAS -- responses include hypermedia links describing what a client can do NEXT
         (genuinely rare in production, covered honestly in Part 14)
```

---

# 9. Follow-up — "Where Does Most Real-World 'REST' Sit on This Model, and Is That a Problem?"

> **Interviewer:** *"Almost no production API you've used implements Level 3. Is that a design failure, or is Level 2 actually fine?"*

Level 2 is, in practice, the pragmatic, industry-standard target — correct resource modeling, correct verb usage, correct status codes gets nearly all of REST's real, practical benefits (cacheability, a uniform client-server contract, statelessness). Level 3's hypermedia-driven discoverability solves a specific problem (a client that never needs out-of-band documentation because every response tells it what it can do next) that most real API consumers — a known frontend team, a known set of partner integrations — simply don't have, which is exactly why Part 14 treats HATEOAS as real, legitimate, and honestly rare rather than as "the REST everyone is secretly failing to do."

---

# Part 2: Resource and URI Design

# 10. Nouns, Not Verbs: Modeling Resources Correctly

```text
WRONG:  POST /createUser                    RIGHT:  POST /users
        POST /getUserOrders?id=5                    GET  /users/5/orders
        POST /cancelOrder?id=42                      DELETE /orders/42  (or POST /orders/42/cancellations)
```

A URI names a **thing** (a resource), and the HTTP method describes the **action** on it — `POST /createUser` duplicates the verb information the method itself already carries, and worse, forces every new action into its own bespoke endpoint rather than reusing the same small, uniform verb set (Part 3) across every resource.

---

# 11. Collection vs. Singleton Resources, and Nesting Depth

```text
/users                  -- a COLLECTION (GET lists users, POST creates one)
/users/5                -- a SINGLETON member of that collection
/users/5/orders         -- the COLLECTION of orders belonging to user 5
/users/5/orders/42      -- a SINGLETON order, scoped through its parent
```

Nesting a resource under its logical parent (`/users/5/orders`) is correct exactly when the child cannot meaningfully exist without that parent context — an order genuinely belongs to a user. It stops being correct the moment a resource is independently addressable and doesn't logically require its parent in the path (`/orders/42` alone is usually sufficient once you have the order ID; needlessly requiring `/users/5/orders/42` for every subsequent operation on order 42 adds a stale, easily-wrong parent ID into every URL for no benefit).

---

# 12. Follow-up — "How Deep Should URI Nesting Go, and When Does It Become an Anti-Pattern?"

> **Interviewer:** *"`/companies/3/departments/7/employees/12/timesheets/99` — is this good REST design?"*

No — this is the classic **over-nesting** anti-pattern. Beyond one, occasionally two levels, deep nesting couples every downstream resource's URL to its entire ancestry, meaning a client has to know and carry the full parent chain just to reference a leaf resource it may already have the ID for. The standard fix: nest **one level deep at most** for genuinely dependent resources (`/departments/7/employees`), and give deeply-nested resources their own **flat, independently-addressable** URI once they have a stable ID of their own (`/timesheets/99`), optionally supporting the nested path as a *convenience* read-only view, never as the only way to reach that resource.

---

# Part 3: HTTP Methods, Safety, and Idempotency

# 13. Safe vs. Idempotent: The Precise Distinction

Two genuinely different properties, conflated constantly:

- **Safe**: the method causes **no side effects at all** — calling it changes nothing on the server. `GET`, `HEAD`, `OPTIONS` are safe.
- **Idempotent**: calling the method **N times produces the identical server state** as calling it **once** — side effects are allowed, but repeating the call must not compound them. `PUT`, `DELETE` are idempotent but **not** safe (they do change state, just not *further* on repetition).

Every safe method is automatically idempotent (doing nothing, repeatedly, is still doing nothing) — but not every idempotent method is safe.

---

# 14. Why POST Is Neither, PUT Is Idempotent, and DELETE's Idempotency Is Subtle

```text
POST /orders           -- creates a NEW order EVERY call.  N calls = N orders.  NOT idempotent.
PUT /orders/42 {...}   -- REPLACES order 42 with this exact representation.  N calls = the SAME final state.  Idempotent.
DELETE /orders/42      -- first call: order 42 is gone (204).  second call: ALREADY gone (404, or 204 if you choose
                          to treat "already deleted" as success) -- either way, the RESOURCE'S final state (does
                          not exist) is identical after 1 call or 100.  Idempotent, even though the RESPONSE
                          (204 vs 404) can legitimately differ between the first and subsequent calls.
```

This is the precise proof a strong answer gives, not an assertion: `PUT`'s idempotency comes from it being a **full replacement** — the resulting state depends only on the request body, never on how many times it's been sent before. `DELETE`'s idempotency is about the **resource's final state** (nonexistent), not about the HTTP response code staying identical across calls — a common, subtle point interviewers specifically probe for.

---

# 15. PATCH: JSON Merge Patch vs. JSON Patch

`PATCH` applies a **partial** update, and is **not**, by default, guaranteed idempotent — its idempotency depends entirely on the patch format:

- **JSON Merge Patch** (RFC 7396): a partial JSON object merged into the target — `{"status": "shipped"}` sets `status` to `"shipped"`, unconditionally, every time. Applying the identical merge patch twice produces the identical result — **idempotent**.
- **JSON Patch** (RFC 6902): a sequence of explicit operations (`add`, `remove`, `replace`, `move`, `test`) — an operation like `{"op": "add", "path": "/tags/-", "value": "urgent"}` **appends** to an array. Applying it twice **appends twice** — **not idempotent**, unless every operation in the sequence happens to be a `replace`.

---

# 16. Follow-up — "Two PATCH Requests Race. What Happens, and How Do You Prevent a Lost Update?"

> **Interviewer:** *"Two clients `PATCH` the same resource concurrently, each unaware of the other's change. One update silently overwrites the other. How do you prevent that?"*

Neither `PATCH`'s idempotency nor its partial-update semantics say anything about **concurrent** safety — that's a completely separate concern, solved by **optimistic concurrency control** (`ETag`/`If-Match`), built in full in Part 9, not by anything inherent to `PATCH` itself.

---

# Part 4: HTTP Status Codes Done Right

# 17. The Real Meaning of Each 2xx Code (200 vs. 201 vs. 202 vs. 204)

```text
200 OK              -- a successful request with a response BODY (a GET, or a PUT/PATCH returning the updated resource)
201 Created         -- a successful POST that created a new resource -- MUST include a Location header pointing to it
202 Accepted        -- the request was accepted for ASYNCHRONOUS processing, not yet complete (Part 13)
204 No Content      -- successful, but there is deliberately no response body (a DELETE, or a PUT with no return value)
```

Returning bare `200 OK` from a resource-creating `POST`, without a `Location` header and without `201`, is one of the most common small correctness gaps — a client has no standard way to discover the new resource's URI without parsing the response body and hoping it contains one.

---

# 18. 4xx Codes Candidates Consistently Misuse

```text
400 Bad Request        -- the request is STRUCTURALLY malformed (invalid JSON, wrong types) -- the server can't
                           even understand what was asked
422 Unprocessable       -- the request is well-formed JSON, but fails a VALIDATION/business rule (a required
    Entity                 field is missing, an email format is invalid) -- the server understood it, but rejects it

401 Unauthorized        -- the caller has NOT authenticated at all, or authentication failed (misleadingly named --
                           it actually means "unauthenticated")
403 Forbidden           -- the caller IS authenticated, but is not allowed to perform this specific action

404 Not Found           -- the resource does not exist, full stop
410 Gone                -- the resource USED to exist and was deliberately, permanently removed -- a stronger,
                           more informative signal than a bare 404 when you know this distinction

409 Conflict            -- the request is valid, but conflicts with the CURRENT STATE of the resource (a duplicate
                           unique key, a concurrent modification without version info, an invalid state transition)
```

`401` vs. `403` is the single most consistently confused pair in real interviews — `401` means "I don't know who you are, or you failed to prove it"; `403` means "I know exactly who you are, and the answer is still no."

---

# 19. Follow-up — "Should a Business Rule Failure Ever Return a 4xx?"

> **Interviewer:** *"An eligibility check determines a user doesn't qualify for a product due to insufficient balance. Is that a 4xx?"*

No — and this is the exact question Part 5 exists to answer in full depth. A business rule producing a **valid, expected "no"** is not a client error at all; the request was well-formed, understood, and correctly evaluated. `4xx` is reserved for genuine problems with the **request itself** (malformed, unauthorized, conflicting) — never for "the answer, correctly computed, happens to be negative."

---

# Part 5: Error Response Design

# 20. The Core Distinction: System Errors vs. Business Outcomes

Every "failure" a REST API can produce falls into exactly one of two categories, and conflating them is the single most common real-world API design mistake this guide addresses:

```text
SYSTEM / VALIDATION ERROR                        BUSINESS OUTCOME
The request could not be processed                The request WAS processed correctly,
as intended.                                       and the answer is "no," with a reason.

Malformed JSON, missing required field,           "This user is not eligible for this
invalid auth, downstream timeout.                  product because their balance is
                                                    insufficient."

-> non-2xx status (400/401/403/422/500)           -> 200 OK -- this is a CORRECT, successful
-> handled by a GlobalExceptionHandler (§22)          evaluation, not a failure of any kind
-> {errorCode, errorMessage} / RFC 7807 (§23-§24) -> a domain-specific "reason" field, in the
                                                       SAME response body as the successful result
```

Throwing an exception, and routing it through a `GlobalExceptionHandler`, for a business outcome like "insufficient balance" is a design smell — it treats an entirely normal, expected result of business logic as if it were an exceptional, unplanned failure, and it forces a non-2xx status onto a request that the server, in fact, handled completely correctly.

---

# 21. Worked Example: The Eligibility API Redesign

A real eligibility endpoint, and the exact tension a team runs into: should "insufficient balance" go through the same error pipeline as "malformed request"?

```text
GOOD (business outcome, always 200 OK):
{
  "userId": 1,
  "isEligible": false,
  "eligibleProducts": [],
  "reasonCode": "INSUFFICIENT_BALANCE",
  "reasonMessage": "User has no sufficient balance"
}

ALSO GOOD (eligible case -- no reason needed at all):
{
  "userId": 1,
  "isEligible": true,
  "eligibleProducts": ["P001", "P002", "P003"]
}

WRONG (reusing "errorCode"/"errorMessage" inside a 200 OK success response):
{
  "userId": 1,
  "isEligible": false,
  "errorCode": "ERR001",           <-- an "error" field on a SUCCESSFUL response is a contradiction
  "errorMessage": "User has no sufficient balance"
}
```

The fix that satisfies both the frontend team's real need (one flat object, everything together, easy to render) **and** correct semantics: keep the reason in the **same response body**, on the **same `200 OK`**, but name the fields for what they actually are — `reasonCode`/`reasonMessage` — reserving `errorCode`/`errorMessage` exclusively for responses that actually went through the error pipeline (§22) on a non-2xx status. A frontend that generically checks "does this response have an `errorCode`" to mean "did the call fail" keeps working correctly, everywhere, because that field genuinely never appears on a success response.

```java
public record EligibilityReason(String code, String message) { }

public record EligibilityResponse(
        long userId,
        boolean isEligible,
        List<String> eligibleProducts,
        EligibilityReason reason /* null when isEligible is true -- no reason needed for a "yes" */) {

    public static EligibilityResponse eligible(long userId, List<String> products) {
        return new EligibilityResponse(userId, true, products, null);
    }

    public static EligibilityResponse notEligible(long userId, String reasonCode, String reasonMessage) {
        return new EligibilityResponse(userId, false, List.of(), new EligibilityReason(reasonCode, reasonMessage));
    }
}
```

```java
@RestController
@RequestMapping("/users/{userId}/eligibility")
public class EligibilityController {

    private final EligibilityService eligibilityService;

    public EligibilityController(EligibilityService eligibilityService) { this.eligibilityService = eligibilityService; }

    @GetMapping
    public ResponseEntity<EligibilityResponse> checkEligibility(@PathVariable long userId) {
        // NOTE: this method NEVER throws for "insufficient balance," "age restriction," or any other
        // business-rule outcome -- those are all ordinary, successful return values, always 200 OK.
        // It DOES let a genuinely missing user throw ResourceNotFoundException, which §22's
        // GlobalExceptionHandler correctly turns into a 404 -- that IS a request-level problem
        // (the resource being asked about doesn't exist), not a business outcome about that user.
        EligibilityResponse response = eligibilityService.evaluate(userId);
        return ResponseEntity.ok(response); // ALWAYS 200 -- whether isEligible is true or false
    }
}
```

`ResponseEntity.ok(response)` is called unconditionally, for both the eligible and not-eligible cases — the *only* way this endpoint ever returns anything other than `200` is if `eligibilityService.evaluate(userId)` itself throws, and it's deliberately designed to throw only for genuine request-level problems (an unknown `userId`), never for a business rule correctly determining "not eligible."

---

# 22. Implementing a GlobalExceptionHandler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class) // Spring's own bean-validation failure exception
    public ResponseEntity<ApiError> handleValidation(MethodArgumentNotValidException ex) {
        String detail = ex.getBindingResult().getFieldErrors().stream()
                .map(f -> f.getField() + ": " + f.getDefaultMessage())
                .collect(Collectors.joining("; "));
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                .body(new ApiError("VALIDATION_FAILED", detail));
    }

    @ExceptionHandler(AuthenticationException.class)
    public ResponseEntity<ApiError> handleAuth(AuthenticationException ex) {
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
                .body(new ApiError("UNAUTHENTICATED", "Authentication is required"));
    }

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiError> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(new ApiError("NOT_FOUND", ex.getMessage()));
    }

    @ExceptionHandler(Exception.class) // the LAST-RESORT catch-all -- never leaks a raw stack trace, §25
    public ResponseEntity<ApiError> handleUnexpected(Exception ex) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(new ApiError("INTERNAL_ERROR", "An unexpected error occurred"));
    }
}

public record ApiError(String errorCode, String errorMessage) { }
```

Every one of these handlers is triggered by a genuine **exception** — a thrown, unexpected-relative-to-normal-control-flow condition — never by a business method returning `isEligible=false` as an ordinary, successful return value. This is the concrete enforcement of §20's distinction: `EligibilityController` (§21) never throws for "insufficient balance"; it returns a normal `EligibilityResponse` object, and this class never sees that code path at all.

---

# 23. RFC 7807 Problem Details: The Standard You Should Probably Be Using

Rather than inventing a bespoke `{errorCode, errorMessage}` shape, RFC 7807 defines a standard, machine-parseable error format, served as `application/problem+json`:

```json
{
  "type": "https://example.com/errors/validation-failed",
  "title": "Validation Failed",
  "status": 400,
  "detail": "field 'email': must be a valid email address",
  "instance": "/users/register"
}
```

- `type`: a URI identifying the **error category** (dereferenceable to human docs, ideally, though it doesn't have to be).
- `title`: a short, human-readable summary of the category (stable across occurrences of the same error type).
- `status`: the HTTP status code, duplicated in the body for clients that inspect the body without the transport-level status.
- `detail`: a specific, **this occurrence's** human-readable explanation.
- `instance`: a URI identifying **this specific occurrence** (useful for correlating with server-side logs).

Adopting an actual IETF standard, rather than a bespoke shape every team reinvents slightly differently, is what lets generic tooling (API gateways, client SDK generators, monitoring dashboards) parse errors uniformly across many otherwise-unrelated services.

---

# 24. Implementing a ProblemDetail Response Builder

```java
public record ProblemDetail(String type, String title, int status, String detail, String instance) {

    public static ProblemDetail of(String errorCategory, String title, HttpStatus status, String detail, String requestPath) {
        return new ProblemDetail(
                "https://api.example.com/errors/" + errorCategory,
                title,
                status.value(),
                detail,
                requestPath);
    }
}
```

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ProblemDetail> handleValidation(MethodArgumentNotValidException ex, HttpServletRequest request) {
    String detail = /* same field-error joining as §22 */ "";
    ProblemDetail problem = ProblemDetail.of("validation-failed", "Validation Failed",
            HttpStatus.BAD_REQUEST, detail, request.getRequestURI());
    return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            .contentType(MediaType.valueOf("application/problem+json"))
            .body(problem);
}
```

Setting the response's actual `Content-Type` to `application/problem+json` (not plain `application/json`) is what lets a generic HTTP client or gateway recognize "this response, regardless of which service produced it, is a structured error" without needing service-specific knowledge of the error shape.

---

# 25. Follow-up — "Should Error Responses Ever Leak Stack Traces or Internal Details?"

> **Interviewer:** *"A `500` error handler includes the exception's message and stack trace in the response body, to help debugging. Good idea?"*

Never, in a public-facing response — a stack trace or a raw exception message can leak internal class names, file paths, SQL fragments, or infrastructure details a real attacker can use for reconnaissance. The correct pattern (§22's catch-all handler already does this): return a **generic**, safe message to the client (`"An unexpected error occurred"`), while logging the **full**, detailed exception server-side, correlated by a request/trace ID the client-facing error also includes (RFC 7807's `instance` field is exactly the right place for this) — giving support/engineering a way to look up the real detail without ever exposing it over the wire.

---

# Part 6: Pagination, Filtering, and Sorting

# 26. Offset Pagination, and Why It Breaks at Scale

```text
GET /orders?offset=20&limit=10   -- "skip the first 20, give me the next 10"
```

Simple, and broken under two real, common conditions: **it degrades in performance** as `offset` grows (the database still has to scan and discard every skipped row before returning the requested page), and — the more serious issue — **it produces incorrect results under concurrent writes**: if a row is inserted or deleted ahead of the current offset between two page requests, every subsequent page shifts by one, causing a client to either see a duplicate row twice or skip one entirely, silently.

---

# 27. Cursor (Keyset) Pagination: The Fix

```text
GET /orders?limit=10                              -- first page
   response includes: nextCursor = "eyJpZCI6NDJ9"   (an opaque, encoded pointer to the LAST row returned)

GET /orders?limit=10&cursor=eyJpZCI6NDJ9           -- "give me the 10 rows AFTER the one this cursor points to"
```

Instead of "skip N rows," a cursor encodes **the actual position** (typically the last-seen row's sort key, e.g. its ID or timestamp) and the next page is a `WHERE id > :cursorId ORDER BY id LIMIT 10` query — a query whose result is correct **regardless of how many rows were inserted or deleted elsewhere in the table**, because it's anchored to a genuine data value, not a row count.

---

# 28. Implementing Cursor-Based Pagination

```java
public record CursorPage<T>(List<T> items, String nextCursor /* null when there is no next page */) { }
```

```java
public final class CursorCodec {

    // Opaque to the client -- deliberately base64-encoded so clients never construct or parse a cursor
    // themselves, which is what lets the SERVER change the underlying encoding later without breaking anyone.
    public String encode(long lastSeenId) {
        return Base64.getUrlEncoder().withoutPadding()
                .encodeToString(("id:" + lastSeenId).getBytes(StandardCharsets.UTF_8));
    }

    public long decode(String cursor) {
        String decoded = new String(Base64.getUrlDecoder().decode(cursor), StandardCharsets.UTF_8);
        return Long.parseLong(decoded.substring("id:".length()));
    }
}
```

```java
@GetMapping("/orders")
public CursorPage<Order> listOrders(@RequestParam(required = false) String cursor, @RequestParam(defaultValue = "10") int limit) {
    Long afterId = (cursor != null) ? cursorCodec.decode(cursor) : null;
    List<Order> orders = orderRepository.findAfter(afterId, limit + 1); // fetch ONE extra to detect "is there a next page"

    boolean hasMore = orders.size() > limit;
    List<Order> page = hasMore ? orders.subList(0, limit) : orders;
    String nextCursor = hasMore ? cursorCodec.encode(page.get(page.size() - 1).id()) : null;

    return new CursorPage<>(page, nextCursor);
}
```

Fetching `limit + 1` rows and checking whether the extra row actually came back is the standard, cheap trick for knowing "is there a next page" **without** a separate, expensive `COUNT(*)` query over the entire remaining table.

---

# 29. Follow-up — "How Do You Sort AND Paginate Consistently When the Underlying Data Changes Between Pages?"

> **Interviewer:** *"You're paginating orders sorted by `createdAt`. Two orders have the identical timestamp, down to the millisecond. What happens to your cursor?"*

The cursor must encode a **compound, guaranteed-unique** key — typically `(sortColumn, id)`, never the sort column alone — so that ties in the sort column are broken deterministically by a value that's always unique (the primary key). Without this, two same-timestamp rows can be ordered inconsistently between two separate queries, causing the exact same duplicate-or-skip corruption cursor pagination was supposed to eliminate, just moved to a rarer trigger condition instead of removed entirely.

---

# 30. Filtering and Sorting Conventions in Query Parameters

```text
GET /orders?status=SHIPPED&sort=-createdAt,customerName
```

- **Filtering**: plain query parameters matching a field name (`status=SHIPPED`) is the conventional, simplest approach; a more expressive filter language (`status=SHIPPED AND total>100`) is real, additional complexity, worth building only when simple equality filters genuinely aren't enough.
- **Sorting**: a comma-separated list of fields, with an optional leading `-` for descending (`sort=-createdAt,customerName` means "newest first, then alphabetically by customer name as a tiebreaker") is a widely-adopted, simple convention requiring no special query language at all.

---

# Part 7: API Versioning

# 31. Three Versioning Strategies Compared

```text
URI versioning:          GET /v2/orders/42
Header versioning:       GET /orders/42          Api-Version: 2
Media-type versioning:   GET /orders/42          Accept: application/vnd.example.v2+json
```

- **URI versioning**: simple, visible, trivially cacheable and routable (a CDN or gateway can route `/v1/*` and `/v2/*` to entirely different backends with zero content inspection) — the tradeoff is that it technically implies `/v1/orders/42` and `/v2/orders/42` are *different resources*, which is philosophically imprecise (it's the same order, described differently).
- **Header versioning**: philosophically cleaner (the URI genuinely identifies one resource, its representation just varies) — the tradeoff is it's invisible in browser address bars, harder to test by just visiting a URL, and requires every routing layer to inspect a header instead of the path.
- **Media-type versioning**: the most "correct" per REST's own content-negotiation model (`Accept` genuinely exists for exactly this) — the tradeoff is it's the least common in practice and the most unfamiliar to API consumers.

---

# 32. Follow-up — "Which One Should You Actually Pick, and Why Do Most Real APIs Use URI Versioning Despite It Being 'Less Pure'?"

> **Interviewer:** *"Purists prefer header/media-type versioning. Why does nearly every major public API (Stripe, GitHub, Twitter) use URI or a simple header, not full content-negotiation-driven versioning?"*

Because **operational simplicity wins over architectural purity** at real scale: URI versioning is trivially cacheable by intermediate proxies without any special configuration, trivially testable by anyone with a browser, and trivially routable to different backend deployments by any load balancer or gateway without content inspection. The "impurity" (two URIs "for the same resource") is a real, acknowledged cost, but it's a cost paid once, in design philosophy, in exchange for operational properties that matter every single day in production — which is exactly why the pragmatic, widely-adopted answer differs from the theoretically "purest" one.

---

# Part 8: Caching and Conditional Requests

# 33. ETag and Last-Modified: Real HTTP Caching

An `ETag` is an opaque identifier for a **specific version** of a resource's representation (commonly a hash of its content, or a version number) — the server includes it on every response; a client can send it back on a subsequent request (`If-None-Match`) to ask "has this changed since I last saw ETag X?" `Last-Modified`/`If-Modified-Since` is the older, coarser (second-resolution) equivalent based on a timestamp instead of a content hash. `ETag` is the stronger, preferred mechanism where available, because it detects a change **precisely**, even one that happens to occur within the same second `Last-Modified` couldn't distinguish.

---

# 34. Implementing Conditional GET (If-None-Match -> 304)

```java
@GetMapping("/orders/{id}")
public ResponseEntity<Order> getOrder(@PathVariable long id, @RequestHeader(value = "If-None-Match", required = false) String ifNoneMatch) {
    Order order = orderRepository.findById(id);
    String currentEtag = "\"" + order.version() + "\""; // a real ETag is quoted, per the HTTP spec

    if (currentEtag.equals(ifNoneMatch)) {
        return ResponseEntity.status(HttpStatus.NOT_MODIFIED).eTag(currentEtag).build(); // 304 -- NO body at all
    }
    return ResponseEntity.ok().eTag(currentEtag).body(order);
}
```

A `304 Not Modified` response deliberately carries **no body** — the entire point is that the client already has a valid, current copy, so re-sending the full representation would waste bandwidth for information the client can already prove it possesses.

---

# 35. Cache-Control Directives That Actually Matter

```text
Cache-Control: no-store                 -- never cache this response AT ALL, anywhere (sensitive data)
Cache-Control: private, max-age=60       -- cacheable only in the REQUESTING CLIENT's own cache, for 60s
Cache-Control: public, max-age=3600      -- cacheable by ANY intermediate cache (a CDN), for 1 hour
Cache-Control: no-cache                  -- (misleadingly named) cache it, but ALWAYS revalidate with the
                                              server (via ETag/If-None-Match) before using the cached copy
```

`no-cache` is the most commonly misunderstood directive in this list — it does **not** mean "don't cache"; it means "cache it, but never serve the cached copy without checking back with the server first," which is precisely the conditional-GET mechanism §34 builds.

---

# 36. Follow-up — "How Does This Interact With an API That's Supposed to Always Return Fresh Data?"

> **Interviewer:** *"A financial balance endpoint must never show stale data. Does that mean it can't use any of this?"*

It means using `Cache-Control: no-store` (or, at minimum, `private, no-cache`) specifically for that endpoint — caching is a **per-resource** decision, not an all-or-nothing property of the whole API. A catalog-listing endpoint and a live-balance endpoint in the *same* API can, and should, carry completely different `Cache-Control` headers, each correctly reflecting that specific resource's actual freshness requirement.

---

# Part 9: Concurrency Control

# 37. The Lost-Update Problem and Optimistic Concurrency

```text
Time  Client A                    Client B                    Server state (order.status)
t0    GET /orders/42  -> "PENDING"                             PENDING
t1                                 GET /orders/42 -> "PENDING" PENDING
t2    PUT status=SHIPPED                                       SHIPPED
t3                                 PUT status=CANCELLED                CANCELLED  <- A's update LOST, silently
```

Both clients read the same starting state, both write based on that stale read, and the second write silently overwrites the first with no error to either party — this is the **lost update problem**, and it's exactly what `ETag`/`If-Match` (already introduced in §33-34 for caching) also solves for *writes*, not just reads. The same version token that lets a client ask "has this changed since I looked?" for a `GET` lets it ask the server to **refuse the write** if the answer is yes.

---

# 38. Implementing Optimistic Concurrency with If-Match

```java
@PutMapping("/orders/{id}")
public ResponseEntity<Order> updateOrder(@PathVariable long id, @RequestHeader("If-Match") String ifMatch, @RequestBody OrderUpdateRequest request) {
    Order current = orderRepository.findById(id);
    String currentEtag = "\"" + current.version() + "\"";

    if (!currentEtag.equals(ifMatch)) {
        // the client's copy is stale -- refuse the write rather than silently overwriting a change it never saw
        return ResponseEntity.status(HttpStatus.PRECONDITION_FAILED).build(); // 412
    }

    Order updated = current.withStatus(request.status()).withVersion(current.version() + 1);
    orderRepository.save(updated);
    return ResponseEntity.ok().eTag("\"" + updated.version() + "\"").body(updated);
}
```

The client is required to send `If-Match` with the ETag it most recently read; if another write has happened in between, the version numbers no longer match, and the server rejects the write outright rather than silently applying it on top of a state the client never actually saw.

---

# 39. Follow-up — "What Status Code Should a Failed Optimistic-Concurrency Check Return, and Why Not 409?"

> **Interviewer:** *"Both 409 Conflict and 412 Precondition Failed sound plausible for a failed ETag check. Which is correct, and why?"*

**412 Precondition Failed** is correct: it specifically means "you told me a precondition (`If-Match: "7"`) that turned out to be false," which is exactly what happened. **409 Conflict** is the right code for a *substantive* business-state conflict independent of any precondition header — e.g., trying to ship an order that's already been cancelled, regardless of what version number anyone sent. The distinction matters because 412 tells the client precisely "re-fetch and retry, your copy was stale," while 409 tells the client "the operation itself conflicts with current state, retrying with a fresher copy won't necessarily help" — conflating them loses information the client actually needs to decide what to do next.

---

# Part 10: Authentication and Authorization

# 40. Authentication Mechanisms Compared

```text
Basic Auth      -- username:password, Base64-encoded, sent on EVERY request; simple but the password
                    itself travels on every call (over TLS only) and there's no session/expiry concept
API Key         -- a static, long-lived secret string sent in a header; simple, but revocation means
                    rotating the key, and a leaked key is valid until manually revoked
OAuth2 + OIDC   -- a token-issuing authorization server; the API itself never sees a password, only a
                    short-lived, independently verifiable access token; supports scopes, expiry, revocation
JWT (as the     -- the most common OAuth2 access-token FORMAT: a signed, self-contained token the API can
 token format)     verify without calling back to the authorization server for every request
```

These aren't fully alternatives at the same layer — JWT is typically the *token format* that rides inside an OAuth2/OIDC flow, not a competing mechanism to OAuth2 itself.

---

# 41. JWT Structure and the Mistakes That Actually Matter

```text
header.payload.signature
eyJhbGciOiJSUzI1NiJ9 . eyJzdWIiOiI0MiIsInJvbGUiOiJhZG1pbiJ9 . <signature bytes>
        ^ algorithm            ^ claims -- BASE64, NOT ENCRYPTED       ^ proves header+payload weren't tampered with
```

The single most common, most dangerous misunderstanding: **a JWT's payload is Base64-encoded, not encrypted** — anyone holding the token can decode and read every claim in it with zero effort (try it: paste any JWT into a Base64 decoder). The signature proves the payload *hasn't been tampered with*; it proves nothing about *confidentiality*. The second most common mistake is accepting the algorithm from the token's own `alg` header at verification time (an attacker can craft a token claiming `alg: none` or downgrade `RS256` to `HS256` using the public key as an HMAC secret) — a correct verifier always specifies the expected algorithm itself and rejects anything else, never trusting the token to declare its own algorithm.

---

# 42. Implementing JWT Validation Correctly

```java
public class JwtValidator {
    private final PublicKey verificationKey;
    private static final String EXPECTED_ALGORITHM = "RS256"; // hardcoded, never read from the token

    public Claims validate(String token) {
        Jws<Claims> parsed = Jwts.parserBuilder()
            .setSigningKey(verificationKey)
            .build()
            .parseClaimsJws(token); // throws if signature invalid, expired, or algorithm mismatched

        if (!EXPECTED_ALGORITHM.equals(parsed.getHeader().getAlgorithm())) {
            throw new JwtException("Unexpected algorithm: " + parsed.getHeader().getAlgorithm());
        }
        return parsed.getBody();
    }
}
```

Note what is deliberately absent: nothing here decrypts the payload, because there is nothing to decrypt — validation only ever *verifies the signature* and *checks expiry*; if a claim inside the token is sensitive, the fix is to not put it in the token at all, not to assume the signature hides it.

---

# 43. Follow-up — "If JWTs Aren't Encrypted, How Do You Put Sensitive Data in a Token Safely?"

> **Interviewer:** *"Your team wants to embed a user's full billing address in the JWT to save a database lookup. Good idea?"*

No — since any holder of the token (including the browser's own storage, any logging middleware that happens to log headers, any proxy in between) can trivially read every claim, a JWT should carry only **non-sensitive, low-value identity and authorization claims** (`sub`, `roles`, `exp`, maybe a tenant ID) — never PII, secrets, or anything an unintended reader shouldn't see. If a claim genuinely needs both confidentiality and the "don't call back to the server" property, the actual tool for that is a **JWE** (JSON Web Encryption, distinct from the far more common JWS/signed-only JWT) — but the much more common, much simpler answer in practice is: don't embed it, look it up.

---

# 44. RBAC vs. Resource-Based Authorization

- **RBAC (role-based access control)** answers "what *class* of actions can this user's role perform?" (e.g., `role: ADMIN` can `DELETE /orders/*`) — cheap to check (it's a claim already on the token), but it says nothing about *which specific resource instance*.
- **Resource-based (object-level) authorization** answers "can *this specific user* act on *this specific resource instance*?" (e.g., "can user 42 view order 917" requires knowing order 917's actual owner, not just user 42's role) — this check fundamentally requires a lookup, because it depends on data no token can carry (the resource's own ownership, which can change after the token was issued).

A correct API needs **both**, layered: RBAC as a coarse, cheap first gate (reject obviously-wrong-role requests immediately), then resource-based authorization as the actual, data-dependent check before returning or mutating anything.

---

# 45. Follow-up — The IDOR Question

> **Interviewer:** *"A user has a valid, unexpired token for their own account. They change the URL from `/orders/42` (their own order) to `/orders/43` (someone else's) and it works. Where did this go wrong, and where exactly should the fix live?"*

This is **IDOR (Insecure Direct Object Reference)** — the token proved *who the user is*, and RBAC may have correctly proven *they're allowed to GET orders in general*, but nothing checked *whether order 43 actually belongs to them*. The fix cannot live in authentication (the token is completely valid) or in coarse RBAC (a `CUSTOMER` role is legitimately allowed to `GET /orders/{id}` for their *own* orders) — it must live in the **service/repository layer**, as an explicit `order.customerId == authenticatedUser.id` check (or, better, by scoping the query itself: `findByIdAndCustomerId(id, authenticatedUser.id)` so a mismatched order simply doesn't exist for that query, returning a 404 rather than leaking that order 43 even exists). This is precisely why resource-based authorization from §44 cannot be skipped even when RBAC already passed.

---

# Part 11: Rate Limiting

# 46. Rate Limiting at the API Layer

Rate limiting protects the API from being overwhelmed by any single client (whether malicious or just buggy — a retry loop with no backoff is indistinguishable from an attack, from the server's point of view) by capping how many requests a given client (identified by API key, user ID, or IP) may make in a given window. This project already has a complete, dedicated guide to the algorithms and implementation (token bucket, sliding window, distributed coordination via Redis) — see *Design an API Gateway and a Rate Limiter*; the relevant part for this guide is purely the **contract** a rate-limited endpoint exposes to its callers.

---

# 47. Follow-up — "What Should the Response of a Rate-Limited Request Look Like?"

> **Interviewer:** *"A client exceeds their quota. What status code, and what headers, should come back?"*

**`429 Too Many Requests`**, always accompanied by a **`Retry-After`** header (seconds, or an HTTP date) telling the client exactly when it's safe to retry — without it, the client is left guessing, and will likely retry immediately, making the overload worse. Well-designed APIs also proactively expose `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset` on **every** response (not just 429s), so well-behaved clients can throttle themselves *before* ever hitting the limit, rather than discovering it via a wall of 429s.

---

# Part 12: Idempotency for Unsafe Operations

# 48. The Idempotency-Key Header Pattern

`PUT` is naturally idempotent (§14) because "set the full state to X" produces the same end state no matter how many times it's repeated. `POST` is not — "create an order" run twice creates two orders. But a network timeout doesn't tell the client *whether* its `POST` actually landed before the connection dropped — blindly retrying risks a duplicate order; blindly *not* retrying risks silently losing a legitimate request. The fix is the **`Idempotency-Key`** header: the client generates a unique key (a UUID) once per *logical* operation and sends the same key on every retry attempt of that same operation; the server remembers keys it has already processed and returns the *original* result instead of repeating the side effect.

---

# 49. Implementing Idempotency-Key

```java
@PostMapping("/orders")
public ResponseEntity<Order> createOrder(@RequestHeader("Idempotency-Key") String idempotencyKey, @RequestBody OrderRequest request) {
    Optional<Order> existing = idempotencyStore.findByKey(idempotencyKey);
    if (existing.isPresent()) {
        return ResponseEntity.status(HttpStatus.CREATED).body(existing.get()); // replay the ORIGINAL result, don't recreate
    }

    Order created = orderService.create(request);
    idempotencyStore.save(idempotencyKey, created); // atomically, in the SAME transaction as the order insert
    return ResponseEntity.status(HttpStatus.CREATED).body(created);
}
```

The critical implementation detail is that saving the idempotency key and creating the order must happen **in the same transaction** — if they aren't atomic, a crash between the two steps produces exactly the race condition this pattern exists to prevent.

---

# 50. Follow-up — "The Client Retries with the Same Key but a Different Body. Now What?"

> **Interviewer:** *"A client sends `Idempotency-Key: abc` with `{amount: 100}`, then retries with the SAME key but `{amount: 200}`. What should happen?"*

This should be rejected — a 422 (or a dedicated `409`) — because the key is meant to identify a single, specific *logical operation*, and a changed body means it's no longer a retry of that same operation but a different request wearing the same key, likely a client bug. The correct implementation stores a hash of the original request body alongside the key, and on replay compares the new body's hash against the stored one before returning the cached result — silently accepting the mismatched body (either running it, or worse, returning stale results for a materially different request) hides a real client-side bug instead of surfacing it.

---

# Part 13: Bulk and Long-Running Operations

# 51. Partial Success in Bulk Operations: 207 Multi-Status

```text
POST /orders/bulk
[{ "sku": "A1" }, { "sku": "INVALID" }, { "sku": "B2" }]

HTTP/1.1 207 Multi-Status
[
  { "status": 201, "sku": "A1", "orderId": 5001 },
  { "status": 422, "sku": "INVALID", "errorCode": "ERR_UNKNOWN_SKU" },
  { "status": 201, "sku": "B2", "orderId": 5002 }
]
```

A bulk endpoint where some items succeed and others fail cannot honestly be represented by any single top-level status code — `200`/`201` would hide the failures, and `400`/`422` would hide the successes. **`207 Multi-Status`** (originally a WebDAV code, now widely reused for exactly this) reports one overall envelope containing a **per-item** status, letting the client know precisely which items to retry without re-submitting the ones that already succeeded.

---

# 52. Implementing a Bulk Endpoint

```java
@PostMapping("/orders/bulk")
public ResponseEntity<List<BulkItemResult>> createBulk(@RequestBody List<OrderRequest> requests) {
    List<BulkItemResult> results = requests.stream()
        .map(this::processOneItem) // each wrapped individually -- one failure must NOT abort the others
        .toList();
    return ResponseEntity.status(207).body(results);
}

private BulkItemResult processOneItem(OrderRequest request) {
    try {
        Order created = orderService.create(request);
        return BulkItemResult.success(201, created.id());
    } catch (ValidationException e) {
        return BulkItemResult.failure(422, e.errorCode()); // caught HERE, not propagated -- isolates this item only
    }
}
```

The key design point is that each item's processing is wrapped in its **own** try/catch — a single item's exception must never propagate up and abort the items around it, since the entire premise of bulk operations is that failures are isolated per item.

---

# 53. Long-Running Operations: 202 Accepted and Polling

Some operations (generating a large export, running a video transcode, batch-processing a huge file) simply cannot complete within a normal request/response cycle. Forcing the client to hold a connection open for minutes is fragile (timeouts, proxies dropping idle connections) — the correct pattern is to **accept the request immediately**, return `202 Accepted` with a pointer to a status resource, and let the client poll (or subscribe to a webhook) for completion.

---

# 54. Implementing the 202 + Polling Pattern

```java
@PostMapping("/reports")
public ResponseEntity<Void> requestReport(@RequestBody ReportRequest request) {
    String jobId = reportJobService.enqueue(request); // returns IMMEDIATELY, work happens asynchronously
    URI statusUrl = URI.create("/reports/jobs/" + jobId);
    return ResponseEntity.accepted().location(statusUrl).build(); // 202 + Location header pointing at status
}

@GetMapping("/reports/jobs/{jobId}")
public ResponseEntity<JobStatus> getJobStatus(@PathVariable String jobId) {
    JobStatus status = reportJobService.getStatus(jobId); // PENDING | RUNNING | DONE | FAILED
    if (status.state() == JobState.DONE) {
        return ResponseEntity.ok().location(URI.create("/reports/" + status.resultId())).body(status);
    }
    return ResponseEntity.ok(status);
}
```

The `202` response's `Location` header is doing real work: it tells the client exactly where to check next, without the client needing to construct that URL itself from any convention — once the job resolves to `DONE`, the status resource itself then points at the final result's own URL.

---

# 55. Follow-up — "How Does a Client Avoid Polling Forever?"

> **Interviewer:** *"Polling every second for an operation that might take an hour is wasteful. What's the better approach?"*

Two real fixes, often combined: (1) the status endpoint should return a **`Retry-After`** header telling the client how long to wait before polling again, and the server can widen this adaptively (poll sooner right after submission, back off to longer intervals the longer the job runs) so the polling cadence matches the operation's actual expected duration rather than a fixed guess; (2) for clients that can accept it, replace polling entirely with a **webhook callback** (the client supplies a callback URL up front, the server calls it once when the job finishes) — trading a small amount of setup complexity for eliminating wasted polling traffic altogether. Production systems that care about this at scale typically offer both, letting the client choose.

---

# Part 14: HATEOAS

# 56. What HATEOAS Actually Promises

**HATEOAS** (Hypermedia As The Engine Of Application State) is the most demanding of REST's original constraints (folded into "uniform interface"): a response shouldn't just return data, it should return **links describing what the client is allowed to do next**, so the client never has to hardcode a URL structure — it just follows links, the way a human browsing a website follows `<a>` tags without knowing the site's URL scheme in advance.

```json
{
  "orderId": 42,
  "status": "PENDING",
  "_links": {
    "self":   { "href": "/orders/42" },
    "cancel": { "href": "/orders/42/cancel" },
    "pay":    { "href": "/orders/42/pay" }
  }
}
```

The genuinely powerful part: if the order's `status` were `SHIPPED` instead, the `cancel` link would simply **not appear** — the client doesn't need a hardcoded business rule "can't cancel a shipped order," it just doesn't render a cancel button, because the server never offered that link.

---

# 57. Follow-up — "If HATEOAS Is So Powerful, Why Does Almost Nobody Fully Implement It?"

> **Interviewer:** *"You just described a genuinely useful property. So why do Stripe, GitHub, and nearly every major API skip it?"*

Because it shifts real complexity onto the server (every response must compute which state-dependent links are currently valid) for a benefit that mostly matters when clients are **generic, unknown-in-advance hypermedia browsers** — but in practice, almost every real API client is a specific, purpose-built frontend or SDK written by a team that already reads the API's documentation and hardcodes the endpoints it needs anyway. The promised decoupling (client never hardcodes a URL) solves a problem most real API consumers don't actually have, so the industry has broadly decided the implementation cost isn't worth it — outside of a few structured exceptions like the `Location` header pattern already used in §54, or `next`/`prev` links in cursor pagination (§28), which are HATEOAS in miniature, applied only where it clearly pays for itself.

---

# Part 15: CORS

# 58. What CORS Solves and the Preflight Request

**CORS** (Cross-Origin Resource Sharing) exists because browsers, by default, block a page loaded from `https://app.example.com` from calling `https://api.example.com` via JavaScript (the "same-origin policy") — a necessary default, since without it any malicious page could silently make authenticated requests to any API a visitor happens to be logged into. CORS is the server's explicit, opt-in mechanism to say "requests from this other origin are allowed anyway," via response headers like `Access-Control-Allow-Origin`.

For requests beyond simple `GET`/`POST` with plain form-encoded bodies, the browser first sends a **preflight** — an automatic `OPTIONS` request, sent before the real one, asking "if I were to send this request, would you allow it?":

```text
OPTIONS /orders
Origin: https://app.example.com
Access-Control-Request-Method: DELETE
Access-Control-Request-Headers: Authorization

---response---
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, DELETE
Access-Control-Allow-Headers: Authorization
```

Only once the browser sees a permissive preflight response does it actually send the real `DELETE` request — the server's application code never even sees a rejected preflight; the browser stops it before the real request is ever made.

---

# 59. Follow-up — "Why Do Some GETs Trigger a Preflight and Others Don't?"

> **Interviewer:** *"A plain `GET /orders` never preflights, but `GET /orders` with a custom `X-Client-Version` header does. Why the difference?"*

The browser only skips preflight for a narrowly-defined set of **"simple requests"**: `GET`/`HEAD`/`POST` only, with only a small allow-list of headers (`Accept`, `Accept-Language`, `Content-Language`, and `Content-Type` restricted to a few specific values), and no custom headers at all. Adding **any** custom header — even an innocuous one like `X-Client-Version` — takes the request out of the "simple" category, because the browser has no way to know in advance whether the server recognizes or permits that header, so it must ask first. This is precisely why an `Authorization` header (used for nearly every authenticated API call) also triggers a preflight — it's not in the simple-request allow-list either.

---

# Part 16: REST vs. RPC vs. GraphQL vs. gRPC

# 60. A Real Comparison

```text
REST      -- resource-oriented (nouns + HTTP verbs), cacheable via standard HTTP, human-readable JSON,
              widely tooled, but often needs multiple round-trips for related data (§ under-fetching)
RPC       -- action-oriented ("doThing()"), maps naturally to imperative operations that aren't really
              about a resource at all (e.g. "sendPasswordResetEmail"), but gives up HTTP-native caching
GraphQL   -- client specifies EXACTLY the fields/relations it needs in one request (fixes REST's
              under-fetching/over-fetching), but standard HTTP caching no longer applies (almost every
              request is a POST to one endpoint), and the server bears query-complexity/cost risk
gRPC      -- binary (Protobuf), HTTP/2 multiplexed, very low latency, strongly-typed contracts via
              .proto files -- excellent for internal service-to-service calls, poor fit for public
              browser-facing APIs (no native browser support without a proxy layer like grpc-web)
```

None of these is strictly "better" — they optimize for different constraints: REST for cacheable, human-debuggable, broadly-interoperable public APIs; GraphQL for complex, client-driven data-shape flexibility; gRPC for high-throughput internal service mesh communication; plain RPC for simple, action-shaped operations that were never really about a resource.

---

# 61. Follow-up — "When Is REST the Wrong Choice?"

> **Interviewer:** *"You've built REST APIs your whole career. Give me a real scenario where you'd deliberately choose something else."*

Two honest scenarios: (1) an internal microservice mesh with dozens of services calling each other thousands of times per second, where **gRPC's** binary encoding and HTTP/2 multiplexing measurably reduce latency and CPU cost compared to JSON-over-HTTP/1.1, and the strongly-typed `.proto` contract catches integration mismatches at compile time instead of at runtime; (2) a mobile client on an unreliable, high-latency network fetching a deeply nested, client-varying view (a social feed mixing posts, authors, comments, and like-counts) where REST would force either many round-trips or a bespoke, brittle "everything this screen needs" endpoint — **GraphQL** lets the client ask for exactly that shape in one round trip, which matters far more on a slow mobile connection than the loss of HTTP-native caching does. The honest engineering answer is never "REST is always right" — it's "REST is the right *default*, until a specific, measurable constraint (internal latency budget, or client-driven nested data shape) says otherwise."

---

# Part 17: Security Best Practices

# 62. Mass Assignment (Over-Posting)

```java
// VULNERABLE: binding the request body directly onto the persistence entity
@PostMapping("/users/{id}")
public User updateUser(@PathVariable long id, @RequestBody User user) {
    user.setId(id);
    return userRepository.save(user); // if User has an `isAdmin` field, a client can just... POST isAdmin:true
}
```

**Mass assignment** happens when a request body is bound directly onto an internal domain/persistence object that has more fields than the endpoint intends to expose for writing — a client who knows (or guesses) the field name `isAdmin` can include it in the JSON body and silently grant themselves admin, even though no UI ever offered that control. The fix is a **dedicated request DTO** that declares only the fields this specific endpoint actually allows a client to set (`name`, `email` — never `isAdmin`, `id`, or `createdAt`), with an explicit, deliberate mapping from DTO to domain entity in the service layer — the domain entity's extra fields are then structurally unreachable from the request body, not just unreachable "by convention."

---

# 63. Input Validation and Injection Risks

Every field in a request body is attacker-controlled input until validated — this applies even to fields that "should" be safe, like a numeric ID or an enum-like string, because nothing stops a client from sending a negative number, an out-of-range enum value, or a string ten megabytes long. Beyond basic type/range/length validation (`@Valid` + Bean Validation annotations in Spring), the specific injection risk most relevant to REST APIs is passing **unvalidated, un-parameterized input into a downstream query or command** — a `sort` query parameter fed directly into a raw SQL `ORDER BY` clause, for instance, is a SQL injection vector exactly as real as a form field, even though it "looks like" harmless metadata rather than user content.

---

# 64. Follow-up — "Where Should Validation Actually Live — Controller, Service, or Both?"

> **Interviewer:** *"You've got `@Valid` on the controller's request DTO. Isn't that enough?"*

No — controller-level validation (via `@Valid`) correctly rejects a **structurally malformed** request early and cheaply (missing required field, wrong type, string too long) before any business logic runs, which is valuable and should stay. But it cannot express validation that depends on **current business state** (e.g., "this SKU exists," "this order hasn't already shipped," "this user hasn't exceeded their order limit today") — those checks are only knowable by querying live data, which is the service layer's job, not the controller's. The correct split is: controller validates *shape* (is this a well-formed request at all), service validates *business rules against current state* — conflating them either forces the controller to depend on repositories it shouldn't know about, or lets the service skip cheap, obvious rejections it shouldn't have to re-derive.

---

# Part 18: Testing REST APIs

# 65. Contract Testing (Consumer-Driven Contracts)

A unit test verifies one service in isolation; an end-to-end test verifies the whole system but is slow, flaky, and requires every service running simultaneously. **Contract testing** sits between them: the *consumer* (e.g., a frontend team) writes a contract stating precisely what it expects from an endpoint ("`GET /orders/42` returns a `status` field that is one of `PENDING`/`SHIPPED`/`CANCELLED`") — this contract is then replayed against the *provider's* actual API in the provider's own CI pipeline, failing the provider's build immediately if a change would break that consumer, **without either side needing the other's full system running**. This directly catches the class of bug that unit tests structurally cannot (a provider's internal tests all pass, yet a consumer breaks in production because the provider silently renamed a field) — tools like Pact implement exactly this pattern, and it's the practical, scalable answer to "how do dozens of independently-deployed services stay compatible with each other without one giant, brittle end-to-end suite."

---

# 66. Full Worked Example: The Eligibility API, End to End

Pulling every part of this guide together on the exact API that motivated it:

```java
public record EligibilityReason(String code, String message) { }

public record EligibilityResponse(
    long userId,
    boolean isEligible,
    List<String> eligibleProducts,
    EligibilityReason reason // null when isEligible == true
) {
    public static EligibilityResponse eligible(long userId, List<String> products) {
        return new EligibilityResponse(userId, true, products, null);
    }
    public static EligibilityResponse notEligible(long userId, String code, String message) {
        return new EligibilityResponse(userId, false, List.of(), new EligibilityReason(code, message));
    }
}

@RestController
@RequestMapping("/users/{userId}/eligibility")
public class EligibilityController {

    private final EligibilityService eligibilityService;

    public EligibilityController(EligibilityService eligibilityService) {
        this.eligibilityService = eligibilityService;
    }

    @GetMapping
    public ResponseEntity<EligibilityResponse> checkEligibility(@PathVariable long userId) {
        // a genuinely missing user IS a system-level error -- the resource itself doesn't exist
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new ResourceNotFoundException("No user with id " + userId));

        // "not eligible" is a normal, successful business OUTCOME -- always 200, never thrown
        EligibilityResult result = eligibilityService.evaluate(user);
        EligibilityResponse response = result.isEligible()
            ? EligibilityResponse.eligible(userId, result.eligibleProducts())
            : EligibilityResponse.notEligible(userId, result.reasonCode(), result.reasonMessage());

        return ResponseEntity.ok(response); // ALWAYS 200 -- isEligible:false is not a failure
    }
}

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiError> handleNotFound(ResourceNotFoundException e) {
        return ResponseEntity.status(404).body(new ApiError("ERR_USER_NOT_FOUND", e.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiError> handleUnexpected(Exception e) {
        log.error("Unhandled exception, traceId={}", MDC.get("traceId"), e); // full detail, server-side only
        return ResponseEntity.status(500).body(new ApiError("ERR_INTERNAL", "An unexpected error occurred"));
    }
}
```

This is the resolution of the tension the whole guide opened with: the frontend team's request (one flat, predictable shape) and the backend's instinct (a `GlobalExceptionHandler` for real errors) are **both satisfiable at once**, once "not eligible" is correctly classified as a business outcome, not an error — `isEligible: false` rides inside the *success* response's own `reason` field, while a `GlobalExceptionHandler` remains fully in place for what it's actually for: a missing user, a malformed request, or an unexpected server fault.

---

# 67. Design Patterns Used Throughout This Guide

- **Strategy** — pluggable versioning strategies (§31), pluggable rate-limiting algorithms (referenced §46), pluggable authentication mechanisms (§40) are each swappable behind a stable interface.
- **Chain of Responsibility** — `@RestControllerAdvice`'s ordered exception handlers (§22), and a filter chain applying `OptimisticConcurrencyFilter`/`JwtValidator`/`IdempotencyKeyInterceptor` in sequence, each handling what it recognizes and passing the rest along.
- **Builder** — `ProblemDetailBuilder` (§24) assembling an RFC 7807 body field by field.
- **Template Method** — the bulk-operation per-item processing loop (§52) applies the same try/catch-and-isolate shape to every item, varying only the specific operation performed.
- **Facade** — a single `EligibilityService.evaluate(user)` call (§66) hides the multi-step rule evaluation behind one simple method the controller depends on.

---

# 68. SOLID Principles Applied

- **Single Responsibility** — `EligibilityController` only translates HTTP concerns; `EligibilityService` only holds business rules; `GlobalExceptionHandler` only maps exceptions to responses — none of the three knows how to do the others' job.
- **Open/Closed** — adding a new authentication mechanism (§40) or a new exception type to handle (§22) means adding a new `@ExceptionHandler` method or a new `Strategy` implementation, never editing existing, already-tested branches.
- **Liskov Substitution** — any `AuthenticationStrategy` implementation (Basic, API key, OAuth2) must honestly fulfill the same contract (accept credentials, return a verified identity or fail) so callers never need to know which one is actually plugged in.
- **Interface Segregation** — a `CursorCodec` (§28) exposes only `encode`/`decode`, not the whole pagination pipeline, so a caller that only needs to decode a cursor doesn't depend on unrelated pagination logic.
- **Dependency Inversion** — `EligibilityController` depends on the `EligibilityService` *interface/abstraction*, never on a concrete rules-engine implementation, so the actual eligibility logic can be swapped or mocked without touching the controller.

---

# 69. Common Mistakes This Guide Corrects

```text
MISTAKE                                              CORRECT APPROACH (this guide's section)
Throwing for a business outcome                       Business outcomes are 200 + a field (§20-21)
Assuming JSON-over-HTTP == RESTful                    Statelessness is the constraint most often broken (§7)
Using offset pagination for a large, live dataset     Cursor/keyset pagination (§27-28)
Trusting a JWT's own declared `alg`                    Server hardcodes the expected algorithm (§41)
Embedding sensitive data in a JWT payload              JWTs are Base64, not encrypted (§41-43)
Checking only role (RBAC), not resource ownership     Resource-based authorization prevents IDOR (§44-45)
Retrying a POST with no idempotency protection         Idempotency-Key header pattern (§48-50)
One status code for a bulk operation's WHOLE result   207 Multi-Status, per-item (§51-52)
Binding a request body directly onto a domain entity  Dedicated request DTOs prevent mass assignment (§62)
```

---

# 70. Testing Strategy Summary

- **Unit tests** — pure business logic in isolation (`EligibilityService.evaluate`, `CursorCodec.encode/decode`), no HTTP or database involved at all.
- **Slice/integration tests** — `@WebMvcTest` for controller + exception-handler wiring (does a thrown `ResourceNotFoundException` actually produce a 404 with the right body?); `@DataJpaTest` for repository queries (does `findByIdAndCustomerId` really exclude another customer's order, closing the IDOR gap from §45?).
- **Contract tests** (§65) — verify this API's responses continue to satisfy every consumer's actual expectations, independent of either side's full deployment.
- **End-to-end tests** — a small number, covering only the handful of truly critical cross-service paths (e.g., the full checkout flow), kept deliberately few because they are the slowest and most brittle layer.

---

# 71. Suggested Future Enhancements

- **API deprecation headers** (`Sunset`, `Deprecation`) on an old version once a new one (§31-32) ships, giving consumers a machine-readable signal before the old version is actually removed.
- **Field-level, per-consumer response shaping** (a lightweight, REST-native alternative to full GraphQL, e.g. a `?fields=id,status` sparse-fieldset parameter) for clients on constrained connections without adopting GraphQL wholesale (§60-61).
- **Structured, machine-readable rate-limit policy discovery** (an `OPTIONS` response advertising a consumer's actual current limits) beyond the reactive `429` + `Retry-After` of §47.
- **Automatic contract-drift detection** wired directly into CI (§65), failing a provider's build the moment a schema change would violate any registered consumer contract, rather than relying on a scheduled contract-test run.

---

# 72. Progressive Interview Question Set

For an interviewer using this guide to run a structured round, in increasing difficulty:

1. What are REST's constraints, and which one is most commonly violated in APIs that call themselves "RESTful"? (§6-7)
2. Is `DELETE` idempotent? Prove it precisely, including the response-code subtlety. (§14)
3. Design the error-response contract for an endpoint that has both real failure modes and legitimate negative business outcomes. (§20-21)
4. Why does cursor pagination replace offset pagination at scale, and what breaks if you skip the tie-breaker column? (§27, §29)
5. Walk through what changes, end to end, in a `PUT` request using optimistic concurrency, and what status code a failed check returns — and why not 409. (§37-39)
6. A user makes a request with someone else's valid-looking resource ID and it works. Diagnose exactly where the fix belongs. (§45)
7. Design the retry-safety contract for a payment-creation `POST` endpoint, including the case where the retried body doesn't match. (§48-50)
8. When would you deliberately choose GraphQL or gRPC over REST for a new service — with a concrete scenario, not a generality? (§61)

---

# 73. Final Takeaway

Nearly every "hard" REST question in this guide reduces to one habit: **naming things precisely enough that the name itself prevents the bug** — calling a business outcome a business outcome instead of an "error" so it stops being tempting to throw it as an exception (§20); calling a stale-write rejection `412` instead of the vaguer `409` so the client knows exactly what to do next (§39); calling a resource-ownership check what it is instead of assuming a valid token already implies it (§45). None of these are exotic techniques — they're precise vocabulary applied consistently, which is exactly why the eligibility API's original tension resolved into something both the backend and frontend team could agree was simply, obviously correct once framed that way.

---
