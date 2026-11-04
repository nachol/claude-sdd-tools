# Defensive Programming Baseline

Shared by SDD and ODD. Single source of truth: agents and skills point here, they do not copy it.

- **Design** — `plan-agent` (Types/Signatures) and `outcome-agent` (card Contract) declare a [Defensive contract](#defensive-contract) for every exposed symbol.
- **Acceptance (ODD)** — every outcome card has at least one failure-path acceptance criterion.
- **Implementation** — `implement-agent` honors these rules as a convention.
- **Review** — `review-agent` passes DP2, DP3, DP4, DP5, DP7, DP8, DP10 to Pass 3 and DP1, DP6, DP9 to Pass 6 (see its *Defensive Programming Baseline Extension*).

An **exposed symbol** is anything callable or consumable from outside its module/package: exported functions/methods/types, HTTP/gRPC/CLI/message handlers, public config structures, persisted or wire formats.

A **trust boundary** is any point where data enters from a source the component does not control: public API callers, HTTP/CLI/message input, files, environment, configuration, and responses from external services.

---

## Defensive contract

Every exposed symbol in a plan step or outcome card declares this block, directly under its signature (as a doc comment or as a list right below the code block):

```text
<SymbolName>
  Validates:  <each input → rule (presence, type, range, length, format)>; "none — internal, inputs validated by <caller>" when applicable
  Fails with: <each failure mode → declared error (type/sentinel/exception class)>; "never fails" only if true
  Invariants: <state/properties guaranteed before and after the call>; "none" when stateless
```

Example:

```text
Reserve(ctx, tenantID string, qty int) (Reservation, error)
  Validates:  tenantID → non-empty, ≤ 64 chars, [a-z0-9-]; qty → 1..1000
  Fails with: invalid input → ErrInvalidArgument; tenant unknown → ErrNotFound;
              quota exceeded → ErrQuotaExceeded; store timeout (2s) → ErrUnavailable (wraps cause)
  Invariants: reserved total per tenant never exceeds its quota; no partial reservation on error
```

A step or card that exposes a symbol without this block is **incomplete**.

---

## Rules

### DP1 — Validate at trust boundaries

**Rule.** Every input crossing a trust boundary is validated for presence, type, range, length and format before use.
**Why.** Invalid data that enters unchecked fails far from its origin, or becomes a security hole.
**Compliant.** Validation happens once, at the edge, and produces a declared error; the inside works with validated values (ideally typed: `TenantID`, not `string`).

```text
Bad:   qty := req.Quantity; account.Credit(-qty)          // negative qty becomes a credit
Good:  if req.Quantity < 1 || req.Quantity > 1000 { return ErrInvalidArgument }
```

- **Go:** validate in handlers/constructors; parse into domain types; check `json.Decoder.DisallowUnknownFields` where the contract is strict.
- **JVM:** Bean Validation (`@Valid`, `@Size`, `@Min`) at controllers *and* explicit checks in public library APIs; Kotlin `require(...)`.
- **Node/TS:** types erase at runtime — validate external input with a schema (zod, valibot, ajv); never `as` cast a request body.
- **Python:** pydantic/attrs validators or explicit checks at entry points; type hints are not validation.

### DP2 — Fail fast and loud on programmer errors

**Rule.** A violated precondition or invariant fails immediately, at the point of violation, with a precise error. Never continue with corrupt state.
**Why.** Late failures hide the cause; silent continuation corrupts data.
**Compliant.** Constructors reject invalid configuration; impossible states raise instead of defaulting.

```text
Bad:   func NewPool(size int) *Pool { if size <= 0 { size = 1 }; ... }   // hides a caller bug
Good:  func NewPool(size int) (*Pool, error) { if size <= 0 { return nil, ErrInvalidSize } ... }
```

- **Go:** return errors from constructors; `panic` only for true programmer errors in unexported code, never across a library boundary.
- **JVM:** `Objects.requireNonNull`, `IllegalArgumentException`/`IllegalStateException`; Kotlin `require`/`check`.
- **Node/TS:** throw `TypeError`/`RangeError` (or a declared error class) at construction.
- **Python:** raise `ValueError`/`TypeError` (or a declared error) in `__init__`/`__post_init__`.

### DP3 — Explicit error model

**Rule.** Every failure mode of an exposed operation is a declared error that callers can distinguish. Dependency error types do not leak through public APIs. Original causes are preserved.
**Why.** Callers cannot handle what they cannot distinguish; leaked types couple consumers to internals.
**Compliant.** Matches the *Fails with* line of the Defensive contract; wrapping keeps the cause.

```text
Bad:   return err                                         // raw driver error escapes the API
Good:  return fmt.Errorf("reserve: %w", ErrUnavailable)   // + cause kept via errors.Join / custom type
```

- **Go:** exported sentinels/types; wrap with `%w`; callers use `errors.Is`/`errors.As`.
- **JVM:** domain exception hierarchy; `throw DomainException(msg, cause)`; no checked driver exceptions in signatures.
- **Node/TS:** error classes with `cause` (`new AppError(msg, { cause })`); discriminated union results where idiomatic.
- **Python:** exception hierarchy per package; `raise DomainError(...) from e`.

### DP4 — No swallowed errors

**Rule.** Every error is handled, propagated, or logged-and-handled with an explicit reason. No empty catch, no ignored error returns, no floating promises.
**Why.** A swallowed error turns a visible failure into silent data loss.
**Compliant.** Every ignore is deliberate and commented with why it is safe.

```text
Bad:   try { save(x) } catch (e) {}
Good:  try { save(x) } catch (e) { throw new PersistError("save failed", { cause: e }) }
```

- **Go:** no `_ =` on error returns without a comment; check `Close()` errors on writers.
- **JVM:** no empty `catch`; do not catch `Throwable`/`Exception` without rethrow or explicit handling.
- **Node/TS:** await or `.catch()` every promise; enable `no-floating-promises`.
- **Python:** no bare `except:` / `except Exception: pass`.

### DP5 — Bounded everything

**Rule.** External calls have timeouts; retries are bounded with backoff; queues, buffers, payload sizes, collection growth and recursion depth have limits.
**Why.** Unbounded resources turn one slow dependency or one large input into an outage.
**Compliant.** Every limit is a named constant or config with a default, documented in the contract.

```text
Bad:   resp, _ := http.Get(url)                           // no timeout, unbounded body
Good:  client := &http.Client{Timeout: 2*time.Second}; body := io.LimitReader(resp.Body, 1<<20)
```

- **Go:** `context.WithTimeout`; `http.Client.Timeout`; `io.LimitReader`; bounded channels/worker pools.
- **JVM:** connect/read timeouts on HTTP and JDBC clients; bounded `ThreadPoolExecutor` queues; Resilience4j retry limits.
- **Node/TS:** `AbortSignal.timeout()`; body size limits in the framework; bounded `Promise.all` concurrency (p-limit).
- **Python:** `timeout=` on every `requests`/`httpx` call; `asyncio.wait_for`; bounded `Queue(maxsize=...)`.

### DP6 — Safe defaults

**Rule.** Defaults are the safe choice (deny, closed, minimal permission, TLS verification on). Unknown enum values and unexpected states are rejected, not passed through.
**Why.** Insecure defaults ship to production because nobody overrides them.
**Compliant.** `switch`/`match` over external values has an explicit reject branch; security flags default to the strict side.

```text
Bad:   switch role { case "admin": ...; default: allow() }
Good:  switch role { case "admin": ...; case "viewer": ...; default: return ErrForbidden }
```

- **Go:** `tls.Config` without `InsecureSkipVerify`; `default:` returns an error.
- **JVM:** Kotlin `when` exhaustive over sealed types; reject unknown JSON enum values (`READ_UNKNOWN_ENUM_VALUES_AS_NULL` off).
- **Node/TS:** exhaustive `switch` with `never` check; `rejectUnauthorized` stays true.
- **Python:** `match` with an explicit `case _: raise`; `verify=True` stays.

### DP7 — Resource safety

**Rule.** Every acquired resource (connection, file, lock, goroutine/thread, subscription, timer) has a guaranteed release path on every exit, including errors.
**Why.** Leaks are invisible in tests and fatal under sustained load.
**Compliant.** Release is structural (defer / try-with-resources / `with` / `finally`), not manual on the happy path.

```text
Bad:   f := open(p); data := read(f); close(f)            // leaks if read fails
Good:  f := open(p); defer close(f); data := read(f)
```

- **Go:** `defer x.Close()` right after a successful acquire (not inside loops); goroutines have a stop signal (`ctx.Done()`).
- **JVM:** try-with-resources / Kotlin `use {}`; executors shut down; coroutine scopes cancelled.
- **Node/TS:** `try/finally`; remove listeners; clear timers/intervals; close streams on error.
- **Python:** context managers (`with`, `async with`); `contextlib.ExitStack` for several.

### DP8 — Immutability and ownership at boundaries

**Rule.** Inputs are not mutated unless the contract says so; returned internals are copies or read-only views.
**Why.** Shared mutable state across a boundary creates action-at-a-distance bugs and data races.
**Compliant.** The contract states ownership; mutating APIs are named as such.

```text
Bad:   func (c *Cache) Items() []Item { return c.items }  // caller can mutate internal state
Good:  func (c *Cache) Items() []Item { return slices.Clone(c.items) }
```

- **Go:** clone slices/maps on return and on store; do not keep caller-owned pointers without saying so.
- **JVM:** `List.copyOf`, `Collections.unmodifiableList`; Kotlin `List` (read-only) in signatures; immutable data classes.
- **Node/TS:** `readonly` types, `structuredClone`/spread on store; `Object.freeze` for config.
- **Python:** return tuples/copies; frozen dataclasses; never mutable default arguments.

### DP9 — No secrets or PII in errors and logs

**Rule.** Error messages and logs carry identifiers, not sensitive values (tokens, passwords, keys, personal data, full request bodies/headers).
**Why.** Logs and error responses are widely readable and long-lived.
**Compliant.** Sensitive fields are redacted at the logging boundary; errors returned to clients carry no internals.

```text
Bad:   log.Error("auth failed", "header", r.Header)
Good:  log.Error("auth failed", "tenant_id", tid, "request_id", rid)
```

- **Go:** typed redacting `String()`/`LogValue()` for secret types; never log whole requests.
- **JVM:** `toString()` excludes secrets (no Lombok `@Data` on credential classes); masking in logback patterns.
- **Node/TS:** logger redaction paths (pino `redact`); do not serialize `Error` with request context to clients.
- **Python:** `logging` filters for redaction; `repr=False` on secret dataclass fields; `SecretStr`.

### DP10 — Assertions are not validation

**Rule.** Debug-only assertions never replace boundary validation or error handling; they may be compiled or optimized out.
**Why.** The check disappears exactly where it is needed — in production.
**Compliant.** Assertions document internal invariants only; boundaries use real checks that return declared errors.

```text
Bad:   assert qty > 0          # stripped with python -O
Good:  if qty <= 0: raise InvalidArgument("qty must be > 0")
```

- **Go:** no assertion facility — not applicable beyond "do not use `panic` as validation".
- **JVM:** `assert` is disabled by default (`-ea`); use `require`/explicit checks.
- **Node/TS:** `console.assert` does not throw; `node:assert` in production code only for internal invariants.
- **Python:** `assert` is removed under `-O`; never validate input with it.

---

## Anti-paranoia

Defensive does not mean redundant. These are **not** required, and reviewers MUST NOT flag their absence:

- **Re-validating between internal components** that only receive already-validated values (validate once at the boundary; pass typed/validated values inward). Declare it in the contract: `Validates: none — internal, inputs validated by <caller>`.
- **Null/nil checks on values the producer guarantees** (constructor postconditions, language/runtime guarantees, non-nullable types that are actually enforced at runtime).
- **Wrapping every internal error in a new type** — wrapping is required at public API boundaries, not between private functions.
- **Timeouts on purely in-process, bounded computations.**
- **Defensive copies of values that are already immutable.**

A missing check is a defect only when you can name a concrete input or state, reachable from a trust boundary, that produces wrong behavior.
