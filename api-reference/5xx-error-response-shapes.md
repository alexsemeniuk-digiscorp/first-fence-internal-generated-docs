# Every 5xx Shape the Backends Can Return

Reference for QA: the complete set of server-error responses from the two backend APIs —
`website-api` (AdonisJS 4 REST) and `cdn-graphql-v2` (Apollo Server 4 GraphQL).

Framework behaviour below was verified against the **installed** packages in each repo, and the
GraphQL rows were additionally confirmed empirically by replaying ~35 failure classes against a
replica of the real middleware chain — not inferred from documentation.

---

## TL;DR

**The two backends fail in opposite ways, and neither is safe to assert on naively.**

| | `website-api` (REST) | `cdn-graphql-v2` (GraphQL) |
|---|---|---|
| Server errors return | **500**, in **48 different body shapes** | **HTTP 200** with an `errors[]` array |
| Standard error envelope | **none** | one envelope, but one useless code |
| Uncaught errors | fall through to the framework default — **plain text or an HTML stack page** | 200 + `INTERNAL_SERVER_ERROR` |
| Real 5xx count | 62 explicit sites + many implicit | **2 classes**, one unreachable |

Five things to take away:

1. **`website-api` has no error envelope.** 62 explicit 5xx sites produce **48 distinct body
   shapes** — `{error}`, `{message}`, `{errors:[{message}]}`, `{error:{message}}`,
   `{success,message,error}`, and 7 that serialise a raw error object. There is no shared wrapper
   to code against. → §3
2. **`website-api` has no exception handler at all.** No `app/Exceptions/Handler.js` exists, so
   uncaught errors hit the AdonisJS default, which is **content-negotiated**: without an explicit
   `Accept: application/json` you get **`text/plain`**, and when `NODE_ENV` isn't `production` you
   get a **full HTML stack-trace page**. Both `.env` files say `development` and the Dockerfile
   never sets it. → §4
3. **GraphQL almost never returns 5xx — that's the trap.** Auth failures, bad ObjectIds, Mongoose
   validation errors, S3 failures and **Mongo being completely down** all return **HTTP 200**. A
   monitor keyed on status would report the API as perfectly healthy while every request fails.
   → §8
4. **There is no 503 anywhere.** SAP outages surface as a plain `Error("SAP is temporarily
   unavailable")` → **500**, never 503. Same for MySQL and Mongo outages. → §5.2
5. **Error bodies leak internals.** Uncaught DB errors include the **full SQL with live bound
   values** (knex rewrites every error message); GraphQL attaches a **stack trace to every error**;
   and 7 REST sites serialise raw axios errors, which carry `config.headers.Authorization`. → §10

**Two deterministic, unauthenticated 500s exist right now** and should be filed as bugs, not
documented as behaviour: `GET /api/admin` and `POST /reverse-lookup/postcode`. → §11

---

## Part A — `website-api` (REST)

### 1. How a status code gets decided

```
throw / rejection
   │
   ├─ Server._handleException            node_modules/@adonisjs/framework/src/Server/index.js:241
   │     error.status = error.status || 500      ← anything without a status becomes 500
   │
   ├─ BaseHandler.handle                 .../Exception/BaseHandler.js:106-109
   │     1. error.handle()   if the exception class defines one   ← custom exceptions win
   │     2. a registered custom handler
   │     3. _defaultHandler
   │
   └─ _defaultHandler                    .../Exception/BaseHandler.js:84-91
         isJSON = request.accepts(['html','json']) === 'json'
         NODE_ENV === 'development' → Youch (HTML page, or JSON with source frames)
         otherwise                  → JSON {message,name,code,status}  OR  text "Name: message"
```

Two consequences QA must internalise:

- **A plain `throw new Error("...")` becomes a 500**, because a plain `Error` has no `.status`.
- **The body format depends on the request's `Accept` header.** With no `Accept` (or `*/*`), the
  `accepts` package returns the first candidate — `html` — so `isJSON` is false and the body is
  the **string** `"Error: message"`, typed `text/plain; charset=utf-8` by
  `node_modules/node-res/index.js:24-29`.

There is also a **last-resort catch** that fires if the handler chain itself throws
(`Server/index.js:256`):

```js
ctx.response.status(500).send(`${error.name}: ${error.message}\n${error.stack}`)
```

→ **`text/plain` 500 carrying a raw stack trace, regardless of `NODE_ENV`.**

### 2. The `NODE_ENV` question

`BaseHandler.js:87-90` branches on `NODE_ENV === 'development'`. In this repo:

- `.env:6` and `.env.example:6` both say `NODE_ENV=development`
- `Dockerfile` **never sets it** (12 lines: base image, `npm ci --omit=dev`, `HOST`, `PORT`, `CMD`)
- deploy is `pm2 restart 0` against the box's own `.env` (`.gitlab-ci.yml:36`), which isn't in the repo

**So the deployed value cannot be determined from the repo and must be checked on the server.** If
it is anything other than `production`, every uncaught 500 returns a **Youch payload**: either a
full HTML debug page, or JSON shaped

```json
{ "error": { "message": "...", "name": "...", "status": 500,
             "frames": [ { "file": "app/Controllers/...", "filePath": "/abs/path/...",
                           "method": "...", "line": 42, "column": 7,
                           "context": { "...source code..." },
                           "isApp": true } ] } }
```

(`node_modules/youch/src/Youch.js:152-172, 295-305`)

### 3. Explicit 5xx bodies — 48 distinct shapes across 62 sites

Grouped by structure. **None of these is a standard; they are 48 independent decisions.**

| Shape | Count | Example |
|---|---|---|
| `{ error: "<string>" }` | ~20 | `{"error":"Unable to fetch addresses"}` |
| `{ error: error.message \|\| "<fallback>" }` | 8 | all of `S3ManagerController` (`Upload failed`, `Download failed`, `Delete failed`, `Search failed`, `Failed to list objects`, `Failed to create folder`, `Failed to check ACL`, `Failed to set ACL`) |
| `{ message: "<string>" }` | 6 | `{"message":"Something went wrong while loading your invoices. Please try again later."}` |
| `{ errors: [ { message } ] }` | 5 | `{"errors":[{"message":"An error occurred while locking the order."}]}` |
| **`<raw error object>`** | **7** | `response.status(500).json(error)` — see §10.1 ⚠️ |
| `{ success: false, message, error: error.message }` | 6 | all of `NotificationController` |
| `{ error: { message: "Internal server error" } }` | 1 | `Portal/CustomerCreditAccountController.js:44-49` — **nested**, unlike everything else |
| `{ error, message: "Internal server error" }` | 1 | both keys at once |
| `{ error: "delete_account_error", message: "..." }` | 1 | the only machine-readable error code in the whole API |
| `{ errors: [ { message, code, error } ] }` | 1 | `Failed to get blocked collection types` / `BLOCKED_TYPES_ERROR` |
| `{ error: error.message, request: request.all() }` | 1 | `SettingController.js:109-111` — **echoes the entire request body** ⚠️ |
| `{ message: error.message }` | 1 | raw internal message |
| `response.internalServerError({ error })` | 1 | `ProductController.js:650` — the only use of Adonis's descriptive helper |
| **502** `{ message: "..." }` | 2 | `Portal/TicketController.js:49,107` — the only non-500 5xx in the codebase |

**There is no 503 and no 504 anywhere in `app/`.**

### 4. Non-controller 5xx sources

#### 4.1 Middleware

| Source | Trigger | Status | Body |
|---|---|---|---|
| `CaptchaVerify.js:43-45` | reCAPTCHA call fails (no `timeout:` set — can also hang) | 500 | `{"message":"Unable to verify captcha at the moment"}` |
| `ApiTokenInit.js:10` | **runs on every request**; DB query for the token — MySQL down → uncaught | 500 | default handler, **message contains the full SQL including the token hash** |
| `ApiTokenInit.js:19-21` | shims `auth.getUser` to throw a plain `Error("No authenticated user")` | 500 | `{"message":"No authenticated user","name":"Error","status":500}` or `text/plain` |
| `VerifyCart.js:17`, `CartProduct.js:17`, `VerifyQuoteCart.js:16,31` | Lucid query error, uncaught | 500 | default handler + SQL leak |
| `DocsAuth.js:23` | token lookup DB error | 500 | default handler + SQL leak |
| `Compression.js:13-14` | compression callback error | 500 | default handler (practically unreachable) |

Not 5xx but worth knowing: `IpBlocker.js:25-27` **swallows** DB errors and lets the request
continue; `VerifyCart.js:23` returns "Cart not found" at **HTTP 200** with no `.status()` call.

`Admin.js:16` / `Customer.js:15` call `await auth.getUser()`, which hits the throwing shim above and
would 500 on an unauthenticated request — **currently masked** because both route groups list
`auth` first (`admin.js:432`, `customer-portal.js:114`) so `ApiTokenAuth` 401s first. It's a latent
trap for any future route that uses `admin`/`customer` without `auth`.

#### 4.2 Providers — where outages actually surface

**SAP** (`providers/SAP/index.js:72,81,97,105`) throws a plain
`Error("SAP is temporarily unavailable")` — **no `.status`, so 500, never 503**:

| Endpoint | Handling | Result |
|---|---|---|
| `POST /orderCreditPayment` (and `/quote-pay/orderCreditPayment`) | **not wrapped** — the `try` starts below the SAP call | **uncaught 500**, format depends on `Accept`; Youch page in development |
| `GET /api/customer-portal/customers/credit-account` | caught | `500 {"error":{"message":"Internal server error"}}` |
| `POST .../invoices/search` | caught | `500 {"message":"Something went wrong while loading your invoices..."}` |
| `GET .../invoices/:fileName` | caught | `500 {"message":"Something went wrong while downloading your invoice..."}` |
| `GET /api/auth/user` | caught, logs and continues | no 5xx |

**MongoDB** — the provider calls `connect()` with no `.catch()`, and `config/mongodb.js:12-15`
defines options that are **never passed**. With Mongo unreachable the app boots fine, `db` stays
`undefined`, and every call becomes
`TypeError: Cannot read properties of undefined (reading 'collection')` → **500 with that
message**. Reachable from `ProductController`, `CartController`, the delivery blacklist/whitelist
controllers, `ExportController`, `app/Utils/installation.js` and `product-helpers.js`.

**MySQL** — `config/database.js:52-61` sets **no pool block and no `acquireConnectionTimeout`**, so
knex defaults apply (`{min:2, max:10}`, 60 s). Pool exhaustion stalls the request **60 seconds**
and then 500s with `Knex: Timeout acquiring a connection. The pool is probably full.` — long
enough that a proxy 504 usually lands first.

**GEO** (`providers/GEO/index.js:24`) — `JSON.parse` runs inside a `resp.on('end')` handler,
**outside** the promise executor, so a `SyntaxError` escapes as an uncaughtException. Any non-JSON
geocode response (an HTML error page, a rate-limit page, an empty body) **exits the process**. See §6.

**Grafana** is fully defensive and never produces a 5xx.

#### 4.3 Custom exceptions

Both `InvalidAccessException` and `InvalidTokenException` define their own `handle()`, so they
resolve to **401**, not 500 — `{"errors":[{"message":"User can't access without privileges"}]}` and
`{"errors":[{"message":"Invalid token"}]}` respectively. (`InvalidTokenException` is never thrown
anywhere.) Neither defines `report()`, so **neither is ever logged**.

### 5. Non-JSON 5xx paths — the list QA needs

A client that blindly `JSON.parse`es will throw rather than surface a usable error on all of these:

1. **Any uncaught 500 without `Accept: application/json`** → `text/plain`, body `"Name: message"`.
2. **`NODE_ENV=development`** → `text/html` Youch page with source frames.
3. **The last-resort catch** (`Server/index.js:256`) → `text/plain` with a raw stack trace,
   *regardless of `NODE_ENV`*.
4. **`GET /api/admin`** → `text/plain` 500, see §11.
5. **`/dev/email-preview/:template`** → 500 with `Content-Type: text/html` and `<pre>${err.stack}</pre>`
   (`start/routes/email-preview.js:499-506`). Double-guarded (skipped when `NODE_ENV === "production"`;
   localhost-only) — but `config/app.js:60` sets `trustProxy: false`, so on a directly-exposed
   dev/test box it is reachable.
6. **Invoice download** (`Portal/CustomerInvoiceController.js:126`) — `response.download()` pipes via
   the `send` package with no `'error'` listener; a mid-download failure makes `send` **clear all
   headers** and emit `text/html` `<pre>Internal Server Error</pre>`.
7. **S3 single-file download** → `500 {"error": "<raw AWS SDK message>"}` if headers aren't sent,
   otherwise a truncated `application/octet-stream` at 200.
8. **Static middleware** (`framework/src/Static/index.js:51`) rewrites the error to
   `"<message> while resolving <request.url()>"` — echoing the request URL into the body.

**And three paths fail silently at HTTP 200 rather than 5xx:**

- **CSV exports** (`OrderController.js:1175-1215`, `BasketQuoteController.js:575-618`,
  `Portal/CustomerController.js:290-336`) all write the header row *before* the data loop, so by the
  time anything can fail `headersSent` is true and the catch takes the `res.end()` branch →
  **200 + a truncated CSV with no error marker.**
- **SSE chat** (`ChatController.js:144-182`) — errors are emitted *inside* the 200 stream as
  `event: error`. Never a 5xx status.
- **`VerifyCart.js:23`** — "Cart not found" at 200.

### 6. Paths that hang or crash instead of returning a status

**Process exit** (connections reset; the proxy renders 502/504; pm2 restarts):

- `providers/GEO/index.js:24-28` — `JSON.parse` in an `'end'` handler (§4.2).
- `S3ManagerController.js:778-792` — folder-zip download pipes an archiver with **no
  `archive.on('error')` listener**.
- Generally any unhandled EventEmitter `'error'` or sync throw in a callback: **there is no
  `process.on('uncaughtException')` anywhere in the repo.**

`unhandledRejection` *is* handled — `@adonisjs/ignitor/src/Ignitor/index.js:577-586` logs and the
process survives. So the two `forEach(async …)` sites (`app/Utils/installation.js:25`,
`app/Utils/product-helpers.js:28`) don't crash anything; they just **silently drop DB writes**.

**Hang (no status ever sent):**

- `Server/index.js:441` calls `_handleException` **un-awaited and un-caught**. On a streaming
  endpoint where headers are already sent, `handle()` throws `ERR_HTTP_HEADERS_SENT`, the fallback
  at `:256` throws the same, and the socket is **never ended**.
- **The app sets no request timeout of any kind** — no `server.timeout`, `headersTimeout`,
  `requestTimeout` or `keepAliveTimeout`. On Node 16 (`Dockerfile:1`) `requestTimeout` defaults to
  **0**, so a slow upstream hangs indefinitely and the **proxy** produces the 504.
- **axios is 0.24.0 → default `timeout: 0`.** Only the OpenAI chat calls set one. Everything else
  can hang forever: reCAPTCHA, Google Geocode, Salesforce (REST *and* jsforce), Worldpay, Iwoca,
  the recommender API, PayPal, and SAP HANA (`config/hana.js` sets no CONNECTTIMEOUT /
  COMMUNICATIONTIMEOUT).

### 7. Infra

**No nginx / Apache / ALB / CloudFront config exists in the repo**, so any 502/503/504 HTML page
comes from a layer outside it. The `Dockerfile` has **no `HEALTHCHECK` and no `--init`**;
`docker-compose.yml` defines only MySQL and Mongo (a local dev stack, no app service). Deploy is
ssh → `git reset --hard` → conditional `npm ci` → `pm2 restart 0`, so there is a restart window
during which the proxy returns its own 5xx. **There is no health-check endpoint** — `GET /` is
`Route.get("/", () => "Hello World")` (`public.js:85`).

---

## Part B — `cdn-graphql-v2` (GraphQL)

### 8. The 200-with-errors trap

**Only two failure classes in the entire API produce a real 5xx, and one is unreachable.**
Everything else that is genuinely broken returns **HTTP 200**.

The mechanism is three lines in the installed `@apollo/server@4.3.2`:

```
requestPipeline.js:271-286   errors raised INSTEAD of execution → status defaults to 500
ApolloServer.js:512-534      errorResponse(): status ?? 500
express4/index.js:35         res.statusCode = httpGraphQLResponse.status || 200
```

Errors raised **during** execution are returned inside `result.errors` and never touch the first
two paths — so the status is left unset and becomes **200**.

Returns HTTP 200 despite being a server-side failure:

- all **127** `throw new GraphQLError(...)` sites in `src/`
- all **10** `throw new Error(...)` sites
- **every auth failure** — `src/middlewares/auth.js:3` throws a plain `Error("Not authenticated")`.
  **There is no 401 or 403 anywhere in this API**, and no `WWW-Authenticate` header.
- **bad ObjectId** → Mongoose `CastError` → 200, `data.<field> = null`
- Mongoose validation errors → 200
- **Mongo completely down** → 200, after a **10-second hang** (`buffering timed out after 10000ms`)
- S3 / AWS failures → 200
- variable coercion failures → 200 with `BAD_USER_INPUT`
- non-nullable-null bubbling → 200

The only genuine 500s: a **response serialization failure** (e.g. a circular object through
`scalar JSON`) and a **context-creation failure** — and the latter is unreachable because
`server.js:124-132` wraps the only throwing line in try/catch.

### 9. Status table

| Failure | Status | `extensions.code` | Content-Type |
|---|---|---|---|
| Resolver throws | **200** | `INTERNAL_SERVER_ERROR` | json |
| Parse/syntax error | 400 | `GRAPHQL_PARSE_FAILED` | json |
| Validation (unknown field) | 400 | `GRAPHQL_VALIDATION_FAILED` | json |
| Unknown `operationName` | 400 | `OPERATION_RESOLUTION_FAILURE` | json |
| Variable coercion | **200** | `BAD_USER_INPUT` | json |
| Missing/odd `Content-Type` | 400 | `BAD_REQUEST` (CSRF message) | json |
| `GET` on a mutation | 405 + `allow: POST` | `BAD_REQUEST` | json |
| **Malformed JSON body** | 400 | — | **html** (Express stack page) |
| Empty/keyless body | 400 | `BAD_REQUEST` | json |
| Batched array body | 400 | `BAD_REQUEST` | json |
| `PUT`/`DELETE`/`PATCH` | 405 + `allow: GET, POST` | `BAD_REQUEST` | json |
| `Accept: application/xml` | 406 | `BAD_REQUEST` | json |
| **Body > 100 KB** | 413 | — | **html** |
| **Malformed multipart** | 400 | — | **html** |
| Unknown route | 404 | — | **html** |
| APQ hash miss | 200 | `PERSISTED_QUERY_NOT_FOUND` | json |
| Mongo down / CastError / validation / S3 | **200** | `INTERNAL_SERVER_ERROR` | json |
| Serialization failure | **500** | `INTERNAL_SERVER_ERROR` | json |

Three practical notes:

- **`bodyParser.json()` is called with no options** (`server.js:111`), so the **100 KB default**
  applies. Any query or mutation payload above that fails with an **HTML page** a GraphQL client
  cannot parse — a real functional limit for large CMS bodies or bulk uploads.
- **`graphqlUploadExpress()` is called with no options** (`server.js:112`), so `maxFileSize` and
  `maxFiles` are **`Infinity`**. Uploads are entirely unbounded.
- **All uploads and any `GET /graphql` need `apollo-require-preflight: true`**, or CSRF prevention
  400s them with a message that looks nothing like an upload failure.

**`extensions.code` is useless for branching.** No application code sets it anywhere across all 137
throw sites, so "Invalid Product ID", "Mongo is down" and "Not authenticated" are all
`INTERNAL_SERVER_ERROR` at HTTP 200. Message-string matching is currently the only discriminator.

**Subscriptions are a separate contract.** The WebSocket transport (`server.js:71-88`) never touches
the ApolloServer instance — errors arrive as close codes (**4400** invalid message, **4401**
unauthorized, **4406** bad subprotocol, **4429** too many init requests) or in-band
`{type:"error"}` / `{type:"next", payload.errors}` frames. Nothing HTTP-shaped, no stack traces,
**and no API-key check at all**.

**Process crash on boot:** `src/db/mongodb.js:9-13` calls `mongoose.connect()` with no `.catch()`.
If Mongo is unreachable at startup the unhandled rejection **kills the process after ~30 s** — but
`httpServer.listen()` runs immediately, so **the server accepts and serves traffic for about 30
seconds and then dies**, resetting in-flight connections with no HTTP response. There is no
`uncaughtException`/`unhandledRejection` handler and **no health endpoint**.

---

## 10. What leaks in error bodies

### 10.1 ⚠️ Raw axios errors — credential exposure

Seven sites do `response.status(500).json(error)`:

- `Portal/CustomerOrderController.js:147`
- `Portal/CustomerQuoteController.js:154, 201, 248, 327`
- `IwocaController.js:96, 224`

**All three of those controllers use axios.** A plain `Error` serialises to `{}` (message and stack
are non-enumerable), but an **axios error has enumerable `config`, `request`, `response`** — so
`JSON.stringify` yields:

```json
{"config":{"url":"https://api.iwoca.co.uk/...",
           "headers":{"Authorization":"Bearer <TOKEN>"}},
 "response":{"status":401,"data":{...}},
 "isAxiosError":true}
```

Verified with a probe. The tokens at risk are the **Salesforce OAuth bearer** and the **Iwoca API
key**. This should be fixed regardless of the QA exercise.

### 10.2 Full SQL with live values

`node_modules/knex/lib/client.js:168-172` rewrites **every** DB error message:

```js
err.message = this._formatQuery(obj.sql, obj.bindings) + ' - ' + err.message;
```

`_formatQuery` substitutes the real bindings, so an uncaught DB error 500 contains the complete
statement with live values. Worst case comes from the **global** `ApiTokenInit` middleware:

```json
{"message":"select * from `api_tokens` where `token_hash` = 'a1b2c3…' limit 1 - connect ECONNREFUSED 10.0.0.5:3306",
 "name":"Error","code":"ECONNREFUSED","status":500}
```

### 10.3 Everything else

| What | Where |
|---|---|
| Raw stack traces | `Server/index.js:256` (always); Youch when `NODE_ENV=development`; `email-preview.js:504` |
| **A stack trace on every GraphQL error** | `extensions.stacktrace` — only the literal strings `production` and `test` suppress it (`ApolloServer.js:86-87`); `.env` says `development` |
| Absolute deployment filesystem paths | `GET /api/admin` `E_MISSING_VIEW` message; every GraphQL `stacktrace` |
| Mongo model, collection and schema field names | GraphQL `CastError` / validation / buffering-timeout messages |
| The entire request body echoed back | `SettingController.js:109-111` (`request: request.all()`) |
| `GOOGLE_GEOCODE_KEY` | `DeliveryController.js:189-193` — the 404 body contains the whole axios error, whose `config.url` carries the key (a 404, but the same class of leak) |
| AWS SDK internals | `S3ManagerController.js:801` |
| Request URL echo | `framework/src/Static/index.js:51` |
| `session._sessionId` | `VerifyCart.js:27`, `CartProduct.js:29,44` |
| Full GraphQL schema | `introspection: true` hardcoded at `cdn-graphql-v2/server.js:92`, unauthenticated, `Access-Control-Allow-Origin: *` |

### 10.4 ⚠️ Committed credentials — escalate

`cdn-graphql-v2/.env.examle` is **tracked in git** (the real `.env` is gitignored; the misspelled
example file is not), and its `API_KEY` value is **byte-identical to the one in the working `.env`**
(verified by hash comparison, values not printed). `MONGO_DB_PASSWORD` is committed too.

That key is the **entire auth model** for the GraphQL API — `server.js:118` compares
`x-access-token` against `process.env.API_KEY`, and that one comparison gates every
`authenticate()`-wrapped mutation. **Verify the deployed value and rotate if it matches.**

Related: 7 resolver files have no `authenticate` at all, and three of them —
`createHolidayMessage`, `updateHolidayMessage`, `deleteHolidayMessage`
(`src/resolvers/holiday-message.js:39-66`) — are **unauthenticated mutations**.

---

## 11. Deterministic 500s — file these as bugs

1. **`GET /api/admin` — unauthenticated, 500 every time.**
   `start/routes/admin.js:435` is `Route.on("/api/admin").render("home")`, registered **outside**
   the `["auth","admin"]` group, and **`resources/views/home.edge` does not exist** (`resources/views/`
   contains only `emails/`). Verified. Edge throws `E_MISSING_VIEW`, and the body leaks the absolute
   deployment path:
   `E_MISSING_VIEW: Cannot render home. Make sure the file exists at <ABSOLUTE PATH>/resources/views location.`

2. **`POST /reverse-lookup/postcode` — 500 on every successful geocode.**
   `DeliveryController.js:199` iterates `res.address_component`, but the Google Geocoding API field
   is `address_component**s**`. Verified. `for…of undefined` → `TypeError: res.address_component is
   not iterable`, thrown outside the try (which only wraps the axios call at `:184-194`).

3. **A test hook left in production.** `OrderController.js:518-520`:
   ```js
   if (person?.phoneNumber && person?.phoneNumber === "0123457890") {
     throw new Error("Something went wrong. Please try again later.");
   }
   ```
   Uncaught → 500. Verified. Because it's a `throw` rather than a `response.status(500)`, it does
   not appear in any sweep of explicit 5xx sites.

4. **Five events are fired with no registered listener** — `send::temporary-works-design`,
   `send::post-pro-site-visit`, `send::post-pro-calculations`, `send::gatedraft-order-notification`,
   `send::site-survey`, all from `WorldpayController.js:174-190`. eventemitter2 silently no-ops, so
   **those emails are never sent.** Not a 5xx, but a silent failure QA would never catch by status.

5. **Five validators return HTTP 200 on validation failure** — `StoreProduct.js:32`,
   `BankPayment.js:15`, `CardPayment.js:16`, `Login.js:23`, `StoreOrder.js:60` call
   `response.json(...)` with no `.status()`. Only `DefaultDepot`, `GetDelivery` and `Register` set 400.

---

## 12. What QA should assert

**Both APIs**

1. **Assert `Content-Type` is JSON before parsing.** Both APIs return HTML or plain text on real
   failure paths (§5, §9).
2. **Assert no response body contains `/home/`, `node_modules`, `.js:` line references, or
   `select ` / `insert ` SQL fragments.** That single check catches the stack-trace leak, the SQL
   leak and the Youch page at once.
3. **Verify the deployed `NODE_ENV` on both boxes.** It governs Youch on `website-api` and
   `extensions.stacktrace` on `cdn-graphql-v2`, and only the literal string `production` is safe.
   This is the highest-value single check in this document.

**`website-api`**

4. **Send an explicit `Accept: application/json` on every request**, or an uncaught 500 arrives as
   `text/plain` and your parser throws instead of reporting the error.
5. **Don't expect a consistent error key.** Handle `error` (string), `error` (object),
   `error.message`, `message`, and `errors[0].message` — all five exist.
6. **Don't expect 503.** SAP, MySQL and Mongo outages are all **500**.
7. **Test the CSV exports for truncation, not for status** — they fail at 200 with a short file.
8. **Boundary-test the 60-second knex pool timeout** — a proxy 504 will usually mask it.

**`cdn-graphql-v2`**

9. **Never treat HTTP 200 as success.** Assert `errors` is absent on every response. A 200 with
   `errors[]` is the normal representation of total backend failure.
10. **Never use HTTP status as a health signal.** Assert on a `data` value instead; there is no
    health endpoint to hit.
11. **Auth tests must assert on message, not status** — an unauthenticated mutation returns **200**
    with `"Not authenticated"`, never 401/403.
12. **Send `id: "abc"` to any `*(id: ID!)` query** — expect 200, `data.<field> = null`, and a
    `Cast to ObjectId failed … for model "X"` message that leaks the model name.
13. **Add `apollo-require-preflight: true`** to upload and `GET` tests, or they 400 with a CSRF
    message unrelated to what you're testing.
14. **Run the kill-Mongo test in both orders:** stop Mongo *then* start the app → it serves for
    ~30 s and exits(1); start the app *then* stop Mongo → requests hang 10 s and return 200. Two
    different failure modes, both need coverage.
15. **Test a payload over 100 KB** — it 413s as HTML.

---

## 13. Code reference index

**Framework behaviour (`website-api`)**
- `node_modules/@adonisjs/framework/src/Server/index.js:241-260` — `error.status || 500`, the
  last-resort plain-text catch, and the un-awaited `_handleException` call at `:441`
- `node_modules/@adonisjs/framework/src/Exception/BaseHandler.js:59-109` — Accept negotiation,
  Youch branch, plain-error shape, handler dispatch order
- `node_modules/youch/src/Youch.js:152-172, 295-305` — the Youch JSON/frame shape
- `node_modules/knex/lib/client.js:168-177` — SQL injected into every DB error message
- `node_modules/node-res/index.js:24-29` — why a non-`<` string becomes `text/plain`
- `node_modules/@adonisjs/ignitor/src/Ignitor/index.js:577-586` — the `unhandledRejection` handler

**`website-api` app code**
- `app/Middleware/` — `CaptchaVerify.js:43-45`, `ApiTokenInit.js:10,19-21`, `VerifyCart.js:17,23`,
  `CartProduct.js:17`, `DocsAuth.js:23`, `IpBlocker.js:25-27`
- `app/Exceptions/InvalidAccessException.js`, `InvalidTokenException.js`
- `providers/SAP/index.js:72,81,97,105` · `providers/MongoDB/` · `providers/MYSQL/index.js:52-53,99`
  · `providers/GEO/index.js:15,24-28`
- `app/Controllers/Http/Portal/CustomerOrderController.js:147`,
  `Portal/CustomerQuoteController.js:154,201,248,327`, `IwocaController.js:96,224` — raw axios errors
- `app/Controllers/Http/SettingController.js:109-111` — request echo
- `app/Controllers/Http/S3ManagerController.js:778-792,801` — archiver crash, AWS leak
- `app/Controllers/Http/Portal/CustomerInvoiceController.js:126` — HTML 500 via `send`
- `app/Controllers/Http/DeliveryController.js:189-193,199` — key leak + the `address_component` bug
- `app/Controllers/Http/OrderController.js:518-520` — the phone-number test hook
- `start/routes/admin.js:435` — `GET /api/admin`
- `start/routes/email-preview.js:499-506` — HTML stack trace
- `config/database.js:52-61` (no pool config), `config/hana.js:17-25` (no timeouts),
  `config/app.js:60` (`trustProxy: false`), `config/shield.js:134` (CSRF off)

**`cdn-graphql-v2`**
- `server.js:90-105` (no `formatError`, `introspection: true`), `:108-143` (middleware chain,
  unbounded bodyParser/upload), `:71-88` (WebSocket), `:118` (the API-key comparison)
- `src/middlewares/auth.js:1-7` — the entire auth layer
- `src/db/mongodb.js:9-13,21-23` — uncaught connect promise
- `src/resolvers/holiday-message.js:39-66` — unauthenticated mutations
- `src/utils/mongo-filter.js:83` — unescaped `$regex` construction
- `src/services/s3.js:10` — reads `AWS_REGION`, env defines `AWS_BUCKET_REGION` (silently falls back
  to `us-east-1`)
- `.env.examle` — tracked, contains the live `API_KEY`
- Installed Apollo: `@apollo/server/dist/esm/requestPipeline.js:271-286`, `ApolloServer.js:86-87,512-534`,
  `express4/index.js:35`, `errorNormalize.js:31-43`, `internalErrorClasses.js`
