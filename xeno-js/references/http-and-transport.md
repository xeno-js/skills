# HTTP and transport

Technical reference assembled from the website documentation. Each section
identifies its original Markdown source. Website frontmatter and the repeated
sponsorship footer are omitted; body content and fenced examples are retained.

---

## Contents

- [`docs/http/api-response.md`](#source-1)
- [`docs/http/client.md`](#source-2)
- [`docs/http/controller.md`](#source-3)
- [`docs/http/helper-http.md`](#source-4)
- [`docs/http/plugins/fastify.md`](#source-5)
- [`docs/http/plugins/node.md`](#source-6)
- [`docs/http/plugins/vercel.md`](#source-7)
- [`docs/http/routes.md`](#source-8)
- [`docs/http/server.md`](#source-9)

<a id="source-1"></a>

# Source: `docs/http/api-response.md`

## Introduction

An application operation can succeed with a value or fail with an application
error. An HTTP controller must turn either outcome into a response that a
transport can send to the client.

The controller returns a `ResponseDto<T>` instead of writing to a
server-specific response object. In a `BaseController`, use:

- `this.ok()` to construct a success response;
- `this.fail()` to construct an error response from an `AppError`.

These are protected helpers: call them inside a `BaseController` subclass. The
[HTTP Controllers](./controller) guide shows a complete controller, route,
service registration, and server bootstrap. This page focuses on the response
contract and the values each helper sets.

## What the controller returns

An `IController<TRequest, TResponse>` returns a
`Promise<ResponseDto<TResponse>>`. The DTO has four fields:

| Field     | Type                        | Purpose                                                        |
| --------- | --------------------------- | -------------------------------------------------------------- |
| `status`  | `number`                    | HTTP status selected by the controller.                        |
| `ok`      | `boolean`                   | Success or failure flag in the DTO.                            |
| `headers` | `HttpHeaders`               | Header names mapped to a string, string array, or `undefined`. |
| `data`    | `ApiResponseDto<TResponse>` | Success or error payload.                                      |

`ApiResponseDto<T>` is a union of `SuccessResponseDto<T>` and
`ErrorResponseDto`. Check `response.data.success` when narrowing the payload in
TypeScript. `TResponse` describes the successful value, not the complete HTTP
envelope or the error payload.

For example, `this.ok({ id: '42', status: 'received' })` creates this DTO:

```ts
{
  status: 200,
  ok: true,
  headers: { 'Content-Type': ['application/json'] },
  data: {
    success: true,
    data: { id: '42', status: 'received' },
    meta: {},
  },
}
```

The two `data` properties have different roles: `ResponseDto.data` holds the API
payload, and the success payload's `data` holds the returned value.
`ResponseDto.ok` is not part of that inner success payload.

## From controller to HTTP response

When a route returns `ctx.handle(token, input)`, Xeno.JS dispatches to the
registered controller and returns its DTO through the route. Configured
middleware can also modify or produce a response. The server integration maps
the resulting status, headers, and body to the transport.

The body mapping differs in the current integrations:

| Integration | Body serialized for a returned `ResponseDto` |
| ----------- | -------------------------------------------- |
| Node.js     | `response.data`                              |
| Fastify     | `response.data`                              |
| Vercel      | `response.data`                              |

<!-- | Hono | The entire `response`, including `status`, `ok`, `headers`, and `data` | -->

For Node.js, Fastify, and Vercel, the example above produces this JSON body with
HTTP status `200`:

```json
{
  "success": true,
  "data": { "id": "42", "status": "received" },
  "meta": {}
}
```

<!-- 
Hono currently serializes the outer DTO as well, so clients receive a
different body shape.  -->Test the actual HTTP response for the integration you

deploy, especially if clients switch transports. Transport-level failures and
unmatched routes can also use different response shapes; do not assume every
response comes from `this.ok()` or `this.fail()`.

## Return a success with `this.ok()`

Inside a `BaseController` subclass:

```ts
return this.ok(order)
```

The helper accepts these arguments:

| Argument  | Meaning                                               | Default  |
| --------- | ----------------------------------------------------- | -------- |
| `data`    | Successful value to return.                           | Required |
| `status`  | HTTP status for the DTO.                              | `200`    |
| `meta`    | Additional metadata in the success payload.           | `{}`     |
| `headers` | Additional headers, as a dictionary of string arrays. | `{}`     |

`this.ok()` delegates to `HttpHelper.success()`. It sets `ok: true` and wraps
the value in `{ success: true, data, meta }`. It also sets `Content-Type` to
`['application/json']`.

### Set status, metadata, and headers

The following excerpt belongs inside a controller whose response type is
`OrderSummary`:

```ts
interface OrderSummary {
  id: string
  status: string
}

const order: OrderSummary = { id: '42', status: 'received' }

return this.ok(
  order,
  201,
  { source: 'checkout' },
  { Location: ['/api/orders/42'] },
)
```

The DTO has status `201`, includes the `Location` header, and carries this
success payload:

```json
{
  "success": true,
  "data": { "id": "42", "status": "received" },
  "meta": { "source": "checkout" }
}
```

Choose a status that reflects the operation; `201` is appropriate when the
request created a resource. The helper does not derive `ok` from the status or
validate that the status is a success code. Pass consistent values.

`HttpHelper.success()` merges custom headers first, then writes `Content-Type`.
A custom header with the exact key `Content-Type` therefore does not replace its
JSON value. Use the `string[]` header values accepted by `BaseController.ok()`.
The broader `HttpHeaders` type also permits strings; the helper's controller
signature is narrower.

### Paginated values

`SuccessResponseDto<T>` can hold a value of `T` or an `IPaginatedResult<T>`. A
paginated result contains `items`, `total`, `page`, `pageSize`, `totalPages`,
`hasNextPage`, and `hasPreviousPage`. Pass the result as the success value; the
helper does not compute or move those fields into `meta`.

For example, a controller with response type `IPaginatedResult<OrderSummary>`
can return `this.ok(page)` after obtaining and validating `page`. The success
payload contains the page object under `data`; `meta` remains `{}` unless you
supply it separately.

### Responses without a body

`this.ok()` always constructs a success payload, even when passed status `204`
or an `undefined` value. HTTP `204` responses must not contain a body, and
serialization or transport behavior can omit values. Do not rely on
`this.ok(undefined, 204)` to deliver the success envelope. Choose the endpoint's
status and body contract deliberately and verify it against the selected server
integration.

## Return an error with `this.fail()`

Pass an `AppError` to `this.fail()`:

```ts
import { AppError } from '@xeno-js/core'

// Inside a BaseController subclass.
return this.fail(
  AppError.notFound('GetOrderController', 'Order does not exist'),
)
```

The helper accepts:

| Argument  | Meaning                                               | Default         |
| --------- | ----------------------------------------------------- | --------------- |
| `error`   | `AppError` containing the code, message, and status.  | Required        |
| `details` | Client-visible error details.                         | `error.message` |
| `headers` | Additional headers, as a dictionary of string arrays. | `{}`            |

`this.fail()` sets `ok: false`, uses the error's status, and produces an
`ErrorResponseDto` with `success: false`. For the example above, a Node.js,
Fastify, or Vercel response has status `404` and a body shaped like:

```json
{
  "success": false,
  "error": {
    "code": "NOT_FOUND",
    "message": "errors.not_found",
    "details": "errors.not_found",
    "path": "/api/orders/42"
  },
  "correlationId": "11111111-1111-4111-8111-111111111111",
  "requestId": "22222222-2222-4222-8222-222222222222",
  "spanId": "33333333-3333-4333-8333-333333333333",
  "timestamp": "2026-11-07T12:00:00.000Z"
}
```

The identifiers and timestamp above are illustrative. The helper uses the active
request context for the path and identifiers when available, and generates
missing identifiers otherwise. The path can be `undefined` and therefore absent
from serialized JSON when no context is available. `HttpHelper.error()` assigns
the output timestamp when it builds the response.

`AppError.notFound()` sets the code, status, and standard message. Its second
argument becomes the cause's message; `this.fail()` does **not** copy that cause
into the response. To supply client-safe details explicitly:

```ts
import { AppError } from '@xeno-js/core'

// Inside a BaseController subclass.
return this.fail(
  AppError.notFound('GetOrderController', 'Order does not exist'),
  'The requested order was not found.',
  { 'X-Error-Category': ['order'] },
)
```

Here `error.message` remains `errors.not_found`, while `error.details` is
`The requested order was not found.` Only expose details appropriate for
clients. Do not return stack traces, credentials, internal service responses, or
other sensitive information.

### Error headers

`HttpHelper.error()` supplies JSON content type, correlation, request, and span
headers, plus `Cache-Control`, `Pragma`, and `Expires` headers that discourage
caching. `this.fail()` passes a context-derived content type, then merges any
headers from `AppError.header`, followed by the headers supplied to `fail()`.

For identical dictionary keys, later values override earlier ones, including the
default error headers. HTTP header names are case-insensitive, so avoid
providing the same header under different casing. Custom headers can differ from
the identifiers in the response body if you override them; keep them consistent
for tracing.

## Map a CQRS result

The mediator returns `ResultType<T>`, not an HTTP response. A controller checks
the outcome and chooses the corresponding helper:

```ts
// Inside a BaseController whose response type matches the query result.
const result = await this._query(query, signal)

if (!result.isOk()) {
  return this.fail(result.getErrorOrThrow())
}

const value = result.getValueOrThrow()
if (value === undefined) {
  return this.fail(
    AppError.internalServerError(
      'GetOrderController',
      new Error('Missing result'),
    ),
  )
}

return this.ok(value)
```

This excerpt uses an application-defined `query`, the controller's `signal`, and
`AppError` imported from `@xeno-js/core`. A successful result can still contain
`undefined`; decide whether that is valid for your operation instead of assuming
a value exists. For a `void` result, use an appropriate endpoint response rather
than treating an absent value as an error.

`this.fail()` formats an `AppError`; it does not catch arbitrary exceptions. The
default CQRS exception pipeline converts exceptions inside the mediator pipeline
to failed results. Exceptions elsewhere follow the transport's error handling
and may not have the same envelope.

## Use the helpers without BaseController

`this.ok()` and `this.fail()` are protected methods on `BaseController`. When
implementing `IController` directly, use the public `HttpHelper.success()` and
`HttpHelper.error()` functions or construct a valid `ResponseDto`.
`HttpHelper.error()` requires an `ErrorResponseDto` with its identifiers and
error fields; it does not accept an `AppError` directly or obtain the active
request context for you.

For route parameters, policies, and other HTTP utilities, see
[HTTP Helpers](./helper-http).

---

---

<a id="source-2"></a>

# Source: `docs/http/client.md`

## Outbound HTTP clients in Xeno.JS

An HTTP client sends requests to another service, such as a backend-for-frontend
(BFF) or a third-party API. Xeno.JS registers clients by name and exposes them
through `httpClientFactory`. Each registered client is wrapped in a remote data
source and an optional resilience service.

Use `withAxiosHttp()` when Axios is the transport. Use `withCustomHttp()` when
the application supplies its own `IHttpClient` implementation. Both paths use
the same factory and return an `IRemoteDataSource` to application code.

```text
Application service or handler
  → httpClientFactory.get(name)
  → IRemoteDataSource
  → resilience service
  → IHttpClient transport
  → remote API
```

Outbound clients are independent of the inbound
[HTTP server](/docs/http/server). Adding a client does not register routes or
start a server.

## Install the Axios integration

Install the core framework and Axios plugin:

```bash
npm install @xeno-js/core @xeno-js/axios
```

The Axios package depends on Axios and Cockatiel. Install those packages
directly only if your application imports them itself.

## Register an Axios client

Pass `withAxiosHttp()` to `addHttpClient()`. Its setup callback receives an
array of client definitions and the Xeno.JS configuration service. The following
example uses a configuration key named `API_BASE_URL`; make that key available
to the configuration service before building the application.

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withAxiosHttp } from '@xeno-js/axios'

const builder = new XenoAppBuilder()

builder.addHttpClient(
  withAxiosHttp((clients, config) => {
    clients.push({
      name: 'bffClient',
      axios: {
        baseURL: config.getOrThrow('API_BASE_URL'),
        timeout: 30_000,
        withCredentials: false,
      },
    })
  }),
)

const application = await builder.build()
```

The `axios` object accepts Axios creation options. The plugin defaults include a
10-second timeout, JSON `Content-Type` and `Accept` headers, and
`http://localhost:3000` as the base URL. Set the base URL and timeout explicitly
for your environment. The transport's per-request configuration sets its own
status validation; do not rely on an Axios `validateStatus` default to change
how HTTP errors are handled.

### Register multiple named clients

Configure all clients in one `addHttpClient()` call:

```ts
builder.addHttpClient(
  withAxiosHttp((clients, config) => {
    const baseURL = config.getOrThrow('API_BASE_URL')

    clients.push({
      name: 'publicApi',
      axios: { baseURL, timeout: 10_000 },
    })

    clients.push({
      name: 'sessionApi',
      axios: {
        baseURL,
        timeout: 30_000,
        withCredentials: true,
        xsrfCookieName: 'XSRF-TOKEN',
        xsrfHeaderName: 'x-xsrf-token',
      },
    })
  }),
)
```

Names must be unique. If a name is omitted, the registration uses `default`;
resolve it with `get('default')`. A second `addHttpClient()` call on the same
builder is ignored, so put every client for that builder in the first call.

The XSRF example reads the named cookie from `document.cookie` and copies its
value to the named header when a browser document is available. The cookie must
be readable by JavaScript. `withCredentials` and this header alone do not
configure server CORS or CSRF protection. Verify the server's cookie, origin and
CSRF policies separately.

## Resolve and use a named client

`httpClientFactory.get(name)` returns `IRemoteDataSource`, not an Axios
instance. Resolve it in an application service factory, as in the following
framework-independent example:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import type { IRemoteDataSource, XenoRegistry } from '@xeno-js/core'
import { withAxiosHttp } from '@xeno-js/axios'

interface Product {
  id: string
  name: string
}

class ProductsService {
  constructor(private readonly api: IRemoteDataSource) {}

  list(signal?: AbortSignal) {
    return this.api.get<Product[]>('/products', {
      query: { page: 1 },
      signal,
    })
  }
}

type AppRegistry = XenoRegistry<
  unknown,
  unknown,
  {
    productsService: ProductsService
  }
>

const application = await new XenoAppBuilder<AppRegistry>()
  .addHttpClient(
    withAxiosHttp((clients, config) => {
      clients.push({
        name: 'bffClient',
        axios: {
          baseURL: config.getOrThrow('API_BASE_URL'),
          timeout: 30_000,
        },
      })
    }),
  )
  .addServices((services) => {
    services.addScoped('productsService', (scope) => {
      return new ProductsService(
        scope.resolve('httpClientFactory').get('bffClient'),
      )
    })
  })
  .build()

await application.runInScope(async (services) => {
  const result = await services.productsService.list()
  console.log(result.getValueOrThrow())
})
```

This registration pattern also works in Vue: build the Xeno application during
bootstrap, provide it to Vue, and resolve services through the integration. The
client registration and factory are core APIs; they do not depend on Vue.

For code that does not need its own scoped service, resolve the singleton
factory from the built application:

```ts
const api = application.getServices().httpClientFactory.get('bffClient')
const result = await api.get<Product[]>('/products', { signal: undefined })
```

Use `get`, `post`, `put`, `patch`, or `delete`. The public `IRemoteDataSource`
contract requires a request-options argument, including a `signal` property,
even when its value is `undefined`:

```ts
const result = await api.post<Product, { name: string }>(
  '/products',
  { name: 'Keyboard' },
  { signal: undefined, timeoutMs: 5_000 },
)
```

Options include `query`, `headers`, `timeoutMs`, `responseType`, and `signal`.
Pass an `AbortSignal` from the caller when cancellation is needed. The current
Axios adapter forwards headers, query, signal and timeout, but does not forward
the per-request `responseType`; do not rely on that option with this adapter.

On success, the remote data source wraps the transport's response body in
`Result.ok()`. It does not expose the HTTP status or headers through that
result. `getValueOrThrow()` can return `undefined`, so validate the returned
payload before using it. Type parameters describe the expected payload but do
not validate it at runtime.

Transport and resilience errors reject the promise; the remote data source does
not convert them into `Result.fail()`. Handle or propagate errors at the
application boundary:

```ts
try {
  const result = await api.get<Product[]>('/products', { signal: undefined })
  const products = result.getValueOrThrow()
  if (products === undefined) throw new Error('Products response is missing')
  console.log(products)
} catch (error) {
  // Report the failure at the application boundary.
  throw error
}
```

The Axios adapter throws an `AppError` for responses with status `400` or
greater. For responses below `400`, it sets `ok` according to the 2xx range; the
remote data source currently returns the body without checking that flag. In
particular, do not assume an unhandled 3xx response becomes a failed result.

## Configure Axios resilience

Add a `resilience` object to a client definition to enable its Cockatiel
policies:

```ts
builder.addHttpClient(
  withAxiosHttp((clients, config) => {
    clients.push({
      name: 'catalogApi',
      axios: {
        baseURL: config.getOrThrow('CATALOG_API_URL'),
        timeout: 5_000,
      },
      resilience: {
        retry: {
          maxAttempts: 3,
          backoff: { initialDelay: 100, maxDelay: 1_000 },
        },
        circuitBreaker: {
          threshold: 5,
          halfOpenAfter: 30_000,
        },
        bulkhead: {
          limit: 10,
          queue: 10,
        },
      },
    })
  }),
)
```

The plugin creates a separate policy for each client entry. It wraps the remote
call with a bulkhead, circuit breaker and retry policy:

- The bulkhead limits concurrent operations and queues additional operations up
  to the configured capacity.
- The circuit breaker tracks consecutive classified transient failures. After
  the threshold, it opens and later allows a half-open attempt.
- Retry uses exponential backoff for classified transient errors when the
  original request method is eligible.

When `resilience` is present, omitted values default to `retry.maxAttempts: 3`,
backoff delays of `100` and `1000` ms, a circuit breaker threshold of `5` with
`halfOpenAfter: 30000` ms, and a bulkhead limit and queue of `10` each. In the
bundled Cockatiel policy, `maxAttempts: 3` permits up to three retries after the
initial call, for four calls total. Account for retry delays and request
timeouts in your total latency budget. Without `resilience`, the core module
uses a pass-through service.

Axios `timeout` limits an individual request. The resilience configuration shown
above does not add a separate overall timeout for all attempts.

### Which failures are retried?

The current factory classifies status `408`, statuses greater than `500`, and
certain network error codes as transient. It retries only when it can also find
an eligible method in the error: `GET`, `PUT`, `DELETE`, `HEAD`, or `OPTIONS`.
The remote data source exposes the first three of those methods; it does not
expose `HEAD` or `OPTIONS`.

The Axios adapter wraps failures in `AppError` before the policy sees them. For
HTTP responses such as `503`, the status and original Axios request
configuration are available to the policy. HTTP `500` and `429` do not satisfy
the current transient-status check. Network errors without a response, including
typical Axios timeouts, are wrapped with status `500` and a framework error
code, so their original network code alone does not make them eligible for retry
or circuit-breaker counting.

`POST` and `PATCH` are not retried by this policy. Even for a method considered
idempotent, verify the remote endpoint's behavior and use a stable idempotency
key where needed. Test your actual failure cases against the installed package
before depending on automatic recovery.

## Register a custom HTTP client with the core plugin

The core plugin is transport-agnostic. It does not create a Fetch, Axios, or
Node client for you. Implement `IHttpClient` in your application, then provide a
factory:

```ts
import { withCustomHttp, XenoAppBuilder } from '@xeno-js/core'
import { MyHttpClient } from './my-http-client.js'

const application = await new XenoAppBuilder()
  .addHttpClient(
    withCustomHttp({
      clients: [
        {
          name: 'inventoryApi',
          client: () => new MyHttpClient('https://api.example.com'),
        },
      ],
    }),
  )
  .build()

const inventory = application
  .getServices()
  .httpClientFactory.get('inventoryApi')
```

`MyHttpClient` is an application-defined class implementing `IHttpClient`.
Implement `get`, `post`, `put`, `patch`, and `delete`; each returns a promise of
`HttpResponse<T>`, containing `status`, `ok`, normalized `headers`, and `data`.
Honor the request options supported by your adapter, including the abort signal.
Decide how non-success HTTP responses become errors: the remote data source
forwards the body without inspecting `ok`, and resilience policies observe
thrown errors, not an `ok: false` response.

`clients` may be a single entry or an array. A client factory receives the
service container, allowing it to resolve registered dependencies. The module
invokes client and resilience factories while configuring the singleton HTTP
factory; avoid capturing request-scoped services in these long-lived objects.

### Add custom resilience

Provide a factory returning an application implementation of
`IServiceResilience`:

```ts
import { withCustomHttp, XenoAppBuilder } from '@xeno-js/core'
import { MyHttpClient } from './my-http-client.js'
import { MyResilienceService } from './my-resilience-service.js'

const application = await new XenoAppBuilder()
  .addHttpClient(
    withCustomHttp({
      clients: {
        name: 'inventoryApi',
        client: () => new MyHttpClient('https://api.example.com'),
        resilience: () => new MyResilienceService(),
      },
    }),
  )
  .build()
```

`MyResilienceService` must implement
`execute<T>(operation: () => Promise<T>, signal: AbortSignal | undefined): Promise<T>`.
It owns its retry, timeout and cancellation behavior. Propagate the final error
rather than returning a success-shaped fallback. A policy applies to the
operation supplied by the remote data source, so design retries with the
endpoint's idempotency and error shapes in mind.

Omit `resilience` for the core pass-through implementation. It checks for an
already-aborted signal before calling the operation but adds no retry, circuit
breaker, bulkhead, or timeout.

### Use the core setup callback

`addHttpClient()` also accepts a setup callback instead of an explicit
`withCustomHttp()` plugin:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { MyHttpClient } from './my-http-client.js'

const application = await new XenoAppBuilder()
  .addHttpClient((options, config) => {
    const baseURL = config.getOrThrow('INVENTORY_API_URL')
    options.clients = {
      name: 'inventoryApi',
      client: () => new MyHttpClient(baseURL),
    }
  })
  .build()
```

This callback delegates to the same core plugin. Choose it or `withCustomHttp()`
for the builder's single HTTP-client registration.

## Context and interceptors

The Axios plugin attempts to read the current request context. When values are
available, its request interceptor adds tracing and request headers such as
`x-correlation-id`, `x-span-id`, and `x-request-id`. It can also add
`x-forwarded-for` and messaging headers from that context. These headers are not
guaranteed when no context is available; review which context information may be
sent to each destination.

Axios request and response interceptors can be configured through
`axios.interceptors`. They run alongside the plugin's context and XSRF
interceptor. Avoid logging credentials or sensitive response bodies.

## Common problems

### The named client cannot be resolved

Check that `addHttpClient()` ran before `build()`, and that the name passed to
`get()` matches the registered name. An unknown name throws an error. When no
name is configured, use `get('default')`.

### A second client is missing

The builder only processes the first `addHttpClient()` call. Register all
entries in its client array instead of chaining multiple calls. Duplicate names
in the same factory cause registration to fail.

### A failed request did not return `Result.fail()`

The remote data source wraps successful response data in a result, but rejected
transport and resilience operations propagate as exceptions. Catch them at an
appropriate boundary or let the caller handle them.

### A timeout or HTTP 500 was not retried

The Axios adapter and policy classify errors as described above. A timeout
without an HTTP response is normally wrapped as an `AppError` with status `500`,
which the current retry predicate does not classify as transient. HTTP `500` and
`429` are also excluded. Use a tested custom transport and resilience
implementation when your application requires a different policy.

### A browser request is missing a CSRF header

Check the cookie and header names, whether the cookie is readable by JavaScript,
and whether the request runs in a browser. Configure the server's CORS and CSRF
rules independently.

### Request-scoped context is missing

The Axios interceptor adds context headers only when the context accessor
returns values. Run the operation within an appropriate application or request
context when tracing is required.

## Related documentation

- [App Builder](/docs/fundamentals/app-builder)
- [Context Scopes](/docs/fundamentals/context-scopes)
- [Service Resolution](/docs/dependency-injection/resolution)
- [HTTP Server](/docs/http/server)
- [Data Sources](/docs/data/datasources/datasource)

---

---

<a id="source-3"></a>

# Source: `docs/http/controller.md`

## Introduction

Routes map HTTP methods and paths to operations. Controllers provide the
presentation boundary for those operations: they receive input, call application
code, and turn the outcome into an HTTP response.

Keep the responsibilities separate:

```text
HTTP request
    ↓
Route: select the operation and prepare input
    ↓
Controller: invoke application code and select a response
    ↓
Application handler: execute the use case
    ↓
Domain and infrastructure services
```

In a layered application, domain code has no dependency on the HTTP transport.
Application handlers orchestrate use cases; infrastructure implements external
dependencies. A controller belongs at the presentation boundary rather than in
the domain.

Xeno.JS offers two ways to write a controller:

- Implement `IController<TRequest, TResponse>` when you need only the controller
  contract.
- Extend `BaseController<TRequest, TResponse>` when you want its mediator,
  response, and request-context helpers.

Use [HTTP Routes](./routes) to define paths, extract input, and dispatch
requests. Use [HTTP Server](./server) to select and start a transport.

## Create and register a controller

The following example connects a route to a `BaseController`, a query, and a
handler. The handler returns a fixed order so that the example runs without a
database. In an application, the handler would use a repository or another
application dependency.

Install the core package and the Node.js server integration:

```bash
npm install @xeno-js/core @xeno-js/node
```

```ts
import {
  AppError,
  BaseController,
  HttpRouteParamHelper,
  Query,
  Result,
  TOKENS,
  XenoAppBuilder,
} from '@xeno-js/core'
import type {
  IContextAccessor,
  IController,
  IHandler,
  IMediator,
  NativeRouteDefinition,
  RequestContext,
  ResponseDto,
  ResultType,
  XenoRegistry,
} from '@xeno-js/core'
import { withNodeHttpServer } from '@xeno-js/node'

interface GetOrderInput {
  id: string
}

interface OrderSummary {
  id: string
  status: string
}

class GetOrderQuery extends Query<OrderSummary> {
  constructor(readonly id: string) {
    super('GET_ORDER_HANDLER', {
      cacheKey: `order:${id}`,
      ttl: undefined,
      bypassCache: true,
      consistentRead: false,
      isUserScoped: false,
    })
  }
}

class GetOrderHandler implements IHandler<GetOrderQuery, OrderSummary> {
  async handle(query: GetOrderQuery): Promise<ResultType<OrderSummary>> {
    return Result.ok({ id: query.id, status: 'received' })
  }
}

class GetOrderController extends BaseController<GetOrderInput, OrderSummary> {
  constructor(context: IContextAccessor<RequestContext>, mediator: IMediator) {
    super(context, mediator)
  }

  async handle(
    input: GetOrderInput,
    signal?: AbortSignal,
  ): Promise<ResponseDto<OrderSummary>> {
    if (typeof input?.id !== 'string' || input.id.trim() === '') {
      return this.fail(
        AppError.badRequest('GetOrderController', 'Invalid order id'),
      )
    }

    const result = await this._query(new GetOrderQuery(input.id), signal)

    if (!result.isOk()) {
      return this.fail(result.getErrorOrThrow())
    }

    const order = result.getValueOrThrow()
    if (order === undefined) {
      return this.fail(
        AppError.internalServerError(
          'GetOrderController',
          new Error('The order handler returned no value'),
        ),
      )
    }

    return this.ok(order)
  }
}

interface OrdersRegistry extends XenoRegistry {
  GET_ORDER_CONTROLLER: IController<GetOrderInput, OrderSummary>
  GET_ORDER_HANDLER: IHandler<GetOrderQuery, OrderSummary>
}

const routes: NativeRouteDefinition<OrdersRegistry>[] = [
  {
    method: 'GET',
    path: '/api/orders/:id',
    handler: async (ctx) => {
      const input = await HttpRouteParamHelper.extractInput<GetOrderInput>(
        ctx.req,
        '/api/orders/:id',
      )
      return ctx.handle('GET_ORDER_CONTROLLER', input)
    },
  },
]

async function main() {
  const app = await new XenoAppBuilder<OrdersRegistry>()
    .addRoutes(routes)
    .addPipeline()
    .addServices((services) => {
      services.addScoped('GET_ORDER_CONTROLLER', (scope) => {
        return new GetOrderController(
          scope.resolve(TOKENS.CONTEXT_ACCESSOR),
          scope.resolve(TOKENS.MEDIATOR),
        )
      })
      services.addScoped('GET_ORDER_HANDLER', () => new GetOrderHandler())
    })
    .addHttpServer(withNodeHttpServer<OrdersRegistry>())
    .build()

  await app.listen({ host: '127.0.0.1', port: 3000 })
}

main().catch((error: unknown) => {
  console.error('Failed to start the application', error)
  process.exitCode = 1
})
```

`addPipeline()` registers the mediator and the default CQRS pipeline. The
query's intent, `GET_ORDER_HANDLER`, must match the handler's container token.
The controller and handler are registered separately from their registry types.
Query cache options do not activate caching; this example does not register a
query caching plugin.

For `GET /api/orders/42`, the route passes the extracted ID to the controller.
The controller sends the query through the mediator and returns a `200` response
when the handler succeeds. A failed result becomes an error response. The input
check matters because the generic argument to `extractInput()` is a TypeScript
type, not runtime validation.

This example does not configure authentication, authorization, CORS, or
rate-limiting middleware. Configure the protections required by your application
before exposing a real endpoint.

## The controller contract

`IController<TRequest, TResponse>` declares an asynchronous `handle()` method:

```ts
handle(
  request: TRequest,
  signal?: AbortSignal,
): Promise<ResponseDto<TResponse>>
```

The route calls `ctx.handle(token, input)`. The dispatcher runs configured HTTP
middleware, resolves the controller from the active scope, and calls its
`handle()` method. The method returns a `ResponseDto`; the selected transport
maps its status, headers, and body to the HTTP response.

The registry constrains the token passed to `ctx.handle()`, but the method's
generic signature does not infer the controller's input and output types from
that token. Keep the route input and controller contract aligned and validate
untrusted input at runtime. A missing registration or a resolved service without
a `handle` method fails during dispatch.

A controller may implement `IController` directly. It does not need a mediator
or request-context accessor unless the implementation uses them. See the
standalone controller example in
[HTTP Routes](./routes#define-and-register-a-route).

## Use BaseController

`BaseController` implements `IController` and requires two constructor
dependencies:

| Dependency                         | Purpose                                                                   |
| ---------------------------------- | ------------------------------------------------------------------------- |
| `IContextAccessor<RequestContext>` | Reads the active request context for controller code and error responses. |
| `IMediator`                        | Dispatches commands and queries through the CQRS pipeline.                |

Implement `handle()` in the subclass. The following helpers are protected and
available to that subclass:

| Helper                                    | Purpose                                                                    |
| ----------------------------------------- | -------------------------------------------------------------------------- |
| `this._query(query, signal)`              | Sends an `IQuery<TResponse>` through the mediator.                         |
| `this._send(command, signal)`             | Sends an `ICommand<TResponse>` through the mediator.                       |
| `this.ok(data, status?, meta?, headers?)` | Creates a successful `ResponseDto`.                                        |
| `this.fail(error, details?, headers?)`    | Creates an error `ResponseDto` from an `AppError`.                         |
| `this.getContext()`                       | Returns the current `RequestContext`, or `undefined` if none is available. |

`_query()` and `_send()` return a `ResultType<TResponse>`. Check `isOk()` before
reading the error or value. `getValueOrThrow()` can return `undefined` even for
a successful result, so handle that case when your response requires a value.

To send a command instead of a query, use the same result handling pattern:

```ts
// Inside a BaseController whose response type requires a command result value.
const result = await this._send(command, signal)

if (!result.isOk()) {
  return this.fail(result.getErrorOrThrow())
}

const value = result.getValueOrThrow()
if (value === undefined) {
  return this.fail(
    AppError.internalServerError(
      'OrderController',
      new Error('Missing command result'),
    ),
  )
}

return this.ok(value)
```

Here `command` is an application-defined `ICommand` whose response type matches
the controller's response type. Register its handler using the command's intent,
just as for the query example. A command that intentionally returns `void` needs
a response appropriate to that contract instead of this missing-value check.

## Build responses

`this.ok()` defaults to status `200` and wraps the value as:

```json
{
  "success": true,
  "data": { "id": "42", "status": "received" },
  "meta": {}
}
```

Pass a different status, metadata, or headers when the endpoint needs them. For
example, `this.ok(order, 201, {}, { Location: ['/api/orders/42'] })` returns a
success response with status `201` and a `Location` header. The helper sets the
response content type to JSON.

`this.fail()` uses the `AppError` code, message, and status. It also includes
the request path, correlation ID, request ID, span ID, and timestamp when
available. Without an active context, it generates missing identifiers. The
optional `details` argument overrides the default error message used for
details; headers passed to `fail()` are added after any headers on the error.
Only include details and headers that are safe to expose to clients.

For the complete response contract, helper options, and transport-specific body
mapping, see [API Responses](./api-response).

Neither helper automatically converts every exception thrown by controller code.
Handle expected failures explicitly. The default CQRS exception pipeline
converts exceptions thrown within the mediator pipeline to failed results;
exceptions outside that pipeline follow the selected transport's error-handling
behavior.

## Lifetimes and request context

Register controllers and their dependencies with lifetimes suited to their
state:

- **Scoped:** one instance within a service scope; useful when dependencies are
  request-scoped.
- **Transient:** a new instance on each resolution.
- **Singleton:** one shared instance for the container.

Do not inject scoped dependencies into a singleton: the container rejects this
captive dependency. `ctx.handle()` resolves controllers within the active
request scope. With the default Node context, Xeno.JS uses `AsyncLocalStorage`
to make `RequestContext` available during that execution.

Within a `BaseController`, `this.getContext()` can provide identity, network,
and tracing data when a context is active. Without configured HTTP middleware,
route dispatch uses a minimal synthetic guest context; do not treat it as an
authenticated identity. Configure authentication and authorization explicitly
and check the relevant policy before exposing protected operations. See
[Context and Scopes](../fundamentals/context-scopes) for context access.

The dispatcher passes the adapted request's `AbortSignal` to the controller.
Forward it to `_query()` or `_send()` and to application dependencies that
support cancellation. The current HTTP adapter creates a new `Request` without
forwarding the incoming transport signal, so do not assume client disconnection
will cancel controller work across server integrations.

---

---

<a id="source-4"></a>

# Source: `docs/http/helper-http.md`

## Introduction

Routes and controllers frequently need to extract request input, apply route
policies, and build responses. Xeno.JS provides helpers for these tasks so that
application code can use the same contracts across HTTP integrations.

| Helper                                      | Use                                                                    |
| ------------------------------------------- | ---------------------------------------------------------------------- |
| `HttpRouteParamHelper`                      | Extract path parameters, query parameters, and combined request input. |
| `withCors()`, `withCsrf()`, `withoutCsrf()` | Add security settings to a route definition.                           |
| `RoutePatternHelper`                        | Normalize and inspect route patterns.                                  |
| `HttpHelper`                                | Build response DTOs and work with headers, cookies, URLs, and bodies.  |
| `AppError`                                  | Create application errors that a controller can map to HTTP responses. |

These APIs are available from `@xeno-js/core`. The route and HTTP utilities and
`AppError` are also exported by `@xeno-js/shared`. Helpers prepare values; they
do not replace runtime validation, authentication, authorization, or middleware
configuration. For the full utility references, see
[HttpHelper](/docs/utils/http) and [Route Helpers](/docs/utils/route).

## Use helpers in an application

This example registers a route and a controller, applies a route CORS policy,
and extracts a path parameter. It returns a fixed order so that no database
setup is needed.

```bash
npm install @xeno-js/core @xeno-js/node
```

```ts
import {
  HttpHelper,
  HttpRouteParamHelper,
  withCors,
  XenoAppBuilder,
} from '@xeno-js/core'
import type {
  IController,
  NativeRouteDefinition,
  ResponseDto,
  XenoRegistry,
} from '@xeno-js/core'
import { withNodeHttpServer } from '@xeno-js/node'

interface GetOrderInput {
  id: string
}

interface OrderSummary {
  id: string
  status: string
}

interface OrdersRegistry extends XenoRegistry {
  GET_ORDER_CONTROLLER: IController<GetOrderInput, OrderSummary>
}

class GetOrderController implements IController<GetOrderInput, OrderSummary> {
  async handle(input: GetOrderInput): Promise<ResponseDto<OrderSummary>> {
    return HttpHelper.success({ id: input.id, status: 'received' })
  }
}

const orderRoute: NativeRouteDefinition<OrdersRegistry> = {
  method: 'GET',
  path: '/api/orders/:id',
  handler: async (ctx) => {
    const input = await HttpRouteParamHelper.extractInput<GetOrderInput>(
      ctx.req,
      '/api/orders/:id',
    )
    return ctx.handle('GET_ORDER_CONTROLLER', input)
  },
}

async function main() {
  const app = await new XenoAppBuilder<OrdersRegistry>()
    .addRoutes([withCors(orderRoute, ['https://shop.example.com'], false)])
    .addMiddlewares((options) => {
      options.cors = {
        enabled: true,
        enablePreflight: true,
        allowOrigins: ['https://shop.example.com'],
        withCredentials: false,
      }
    })
    .addServices((services) => {
      services.addScoped('GET_ORDER_CONTROLLER', () => new GetOrderController())
    })
    .addHttpServer(withNodeHttpServer<OrdersRegistry>())
    .build()

  await app.listen({ host: '127.0.0.1', port: 3000 })
}

main().catch((error: unknown) => {
  console.error('Failed to start the application', error)
  process.exitCode = 1
})
```

The builder registers the route, middleware, controller, and server before
`build()`. For a matching request, the route extracts the ID and delegates
through `ctx.handle()`; the controller returns a success DTO. The middleware
configuration and route policy are separate: `withCors()` alone does not enable
CORS middleware. With this explicit route origin list, requests without an
`Origin` header are rejected by the current origin middleware; test with
`Origin: https://shop.example.com` or choose a policy appropriate to your
clients. Configure authentication, authorization, validation, and other
protections for a real endpoint. For controller registration and service
lifetimes, see [HTTP Controllers](./controller).

## Extract request input

### `HttpRouteParamHelper.extractInput()`

Use `extractInput<TInput>(req, routePattern)` inside a route handler to combine
values from the body, query, and path:

```ts
import { HttpRouteParamHelper } from '@xeno-js/core'

// Inside the handler for POST /api/orders/:id.
const input = await HttpRouteParamHelper.extractInput<{
  id: string
  note?: string
}>(ctx.req, '/api/orders/:id')

return ctx.handle('UPDATE_ORDER_CONTROLLER', input)
```

Here `ctx` is the route handler context and `UPDATE_ORDER_CONTROLLER` must be
declared in the application's registry and registered in the builder. Pass the
route's actual pattern to extract the intended path keys.

The helper merges values in this order, with later sources taking precedence:

1. Body fields.
2. `req.query`.
3. Named path parameters.

For example, if the body contains `id: "body"`, the query contains `id=query`,
and the path is `/api/orders/42`, the resulting `id` is `"42"`.

The helper reads a body only for `POST`, `PUT`, and `PATCH`:

| Content type                                 | Body handling                      |
| -------------------------------------------- | ---------------------------------- |
| Contains `application/json`                  | Calls `req.json()`.                |
| Contains `application/x-www-form-urlencoded` | Reads text and parses form fields. |
| Other or absent                              | Reads text.                        |

Object body fields are merged into the result; a non-object body is placed under
`body`. If no body is available, the result can contain `body: undefined`. For
`GET`, `DELETE`, `HEAD`, and `OPTIONS`, this helper does not read the body.

The helper strips control characters from decoded path values and recursively
sanitizes object body strings. It removes the keys `__proto__`, `constructor`,
and `prototype` from object bodies. It uses `req.query` directly; the selected
server integration can influence how query values arrive.

**Validate the result.** The generic type does not check or convert data: query
parameters are not automatically numbers, required properties may be missing,
and input is still untrusted. Body read and parse errors are caught and the
helper proceeds without the parsed body. If the endpoint must reject malformed
JSON specifically, parse and handle the body separately before it is consumed;
do not rely on `extractInput()` to report that failure.

### `HttpRouteParamHelper.extractPathParams()`

Use `extractPathParams(routePattern, requestPath)` to extract named segments:

```ts
import { HttpRouteParamHelper } from '@xeno-js/core'

const params = HttpRouteParamHelper.extractPathParams(
  '/api/orders/:id/items/{itemId}',
  '/api/orders/42/items/item%20one',
)
// { id: '42', itemId: 'item one' }
```

Both `:name` and `{name}` identify a segment. Values are URI-decoded, stripped
of control characters, trimmed, and limited by the sanitization helper. Missing
or empty values can be omitted. Invalid percent encoding can throw a `URIError`.
This method extracts positional segments; it does not verify that the entire
path matches the route pattern.

### `HttpRouteParamHelper.extractQueryParams()`

Use `extractQueryParams(requestUrl)` when you need query values from a URL:

```ts
import { HttpRouteParamHelper } from '@xeno-js/core'

const query = HttpRouteParamHelper.extractQueryParams(
  '/api/orders?status=open&page=2',
)
// { status: 'open', page: '2' }
```

The helper strips control characters, trims values, and limits keys to 64 and
values to 512 characters. It skips entries that sanitize to an empty key or
value. For repeated keys, the last accepted value wins. Values remain strings;
parse and validate them before use.

## Configure route policies

The route helpers return a new definition with updated `security` fields; they
preserve other existing route settings. They do not mutate the input route or
register middleware.

### `withCors()`

```ts
import { withCors } from '@xeno-js/core'

// orderRoute is a NativeRouteDefinition registered with the builder.
const shopOrderRoute = withCors(orderRoute, ['https://shop.example.com'], false)
```

Arguments are the route, allowed origins, and optional `withCredentials` flag,
which defaults to `true`. These settings override the corresponding global CORS
settings for that route. Configure CORS in `addMiddlewares()` and register the
returned route rather than the original. For a wildcard origin, avoid enabling
credentials without a deliberate, restrictive policy. CORS is not authentication
or authorization.

### `withCsrf()` and `withoutCsrf()`

```ts
import { withCsrf, withoutCsrf } from '@xeno-js/core'

// Both routes are application-defined NativeRouteDefinition values.
const protectedOrderRoute = withCsrf(orderRoute)
const webhookWithoutCsrf = withoutCsrf(webhookRoute)
```

`withCsrf()` sets `security.withCsrf` to `true`; `withoutCsrf()` sets it to
`false`. Configure CSRF middleware and its secret separately. In the current
implementation, CSRF validation applies to `POST`, `PUT`, `PATCH`, and `DELETE`
by default when configured; `withCsrf()` does not extend validation to `GET` or
other methods. Only disable CSRF when another appropriate security boundary
protects the route, such as verified webhook signatures.

For route policies and preflight behavior, see [HTTP Routes](./routes).

## Inspect route patterns

`RoutePatternHelper` supports route-pattern processing. Most applications can
register routes without calling it directly.

```ts
import { RoutePatternHelper } from '@xeno-js/core'

const normalized = RoutePatternHelper.normalizePath(' orders/:id/ ')
// '/orders/:id'

const dynamic = RoutePatternHelper.isDynamicPath(normalized)
// true

const pattern = RoutePatternHelper.createRoutePattern(normalized)
const matches = pattern.test('/orders/42')
// true

const parsed = RoutePatternHelper.parse('/orders/{id}/')
// { pattern: RegExp, isDynamic: true }
```

| Method                     | Behavior                                                                                                      |
| -------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `normalizePath(path)`      | Trims surrounding whitespace, adds a leading slash when missing, and removes trailing slashes except for `/`. |
| `isDynamicPath(path)`      | Detects `:` or a pair of braces in the path.                                                                  |
| `createRoutePattern(path)` | Builds a regular expression; named segments match a single non-empty path segment.                            |
| `parse(path)`              | Normalizes a path and returns `{ pattern, isDynamic }`.                                                       |

`normalizePath()` does not collapse repeated internal slashes.
`createRoutePattern()` escapes static segments, while `parse()` uses a different
regular-expression construction for a fully static path. Do not treat these
functions as general-purpose security validation or assume that arbitrary static
metacharacters are literal in every `parse()` result. See
[HTTP Routes](./routes) for matching order and route registration.

## Build response DTOs

### `HttpHelper.success()`

Use `HttpHelper.success(data, status?, meta?, customHeaders?)` outside
`BaseController` to create a success `ResponseDto`:

```ts
import { HttpHelper } from '@xeno-js/core'

const response = HttpHelper.success(
  { id: '42', status: 'received' },
  201,
  { source: 'checkout' },
  { Location: '/api/orders/42' },
)
```

The default status is `200`. The result has `ok: true` and a `data` payload of
`{ success: true, data, meta }`. The helper sets `Content-Type` to
`application/json` after merging custom headers, so an exact `Content-Type` key
in `customHeaders` does not override it.

### `HttpHelper.error()`

`HttpHelper.error(dto, status?, customHeaders?)` accepts an `ErrorResponseDto`,
not an `AppError`. Supply the error fields and tracing identifiers explicitly:

```ts
import { GuidHelper, HttpHelper, STATUS_CODES } from '@xeno-js/core'
import type { ErrorResponseDto } from '@xeno-js/core'

const error: ErrorResponseDto = {
  success: false,
  error: {
    code: 'ORDER_NOT_FOUND',
    message: 'Order not found',
    details: undefined,
    path: '/api/orders/42',
  },
  correlationId: GuidHelper.generate(),
  requestId: GuidHelper.generate(),
  spanId: GuidHelper.generate(),
  timestamp: new Date().toISOString(),
}

const response = HttpHelper.error(error, STATUS_CODES.NOT_FOUND, {
  'X-Error-Category': 'order',
})
```

The helper returns `ok: false`, uses status `500` when omitted, adds tracing and
no-cache headers, and creates a timestamp when constructing the output payload.
Custom headers override defaults with the same exact key. When an active request
context exists, use its tracing identifiers instead of generating unrelated ones
so logs and responses can be correlated.

In a `BaseController`, prefer `this.ok()` and `this.fail()`: `this.fail()`
accepts `AppError` and retrieves context data for you. See
[API Responses](./api-response) for the full DTO, header precedence,
transport-specific body mapping, and error response examples.

## Create and map application errors

`AppError` carries an error code, HTTP status, message, and cause. `AppError` is
an application error contract; creating one does not itself send a response.

```ts
import { AppError } from '@xeno-js/core'

// Inside a BaseController subclass.
return this.fail(
  AppError.notFound('GetOrderController', 'Order does not exist'),
  'The requested order was not found.',
)
```

Common factories include `badRequest()`, `validationError()`, `unauthorized()`,
`authFailed()`, `forbidden()`, `notFound()`, `conflict()`, `notSupported()`,
`internalServerError()`, and `aborted()`. Use `AppError.create()` when you need
an explicit code, status, message, name, and cause; `AppError.throw()`
constructs and throws an error. `AppError.throwIfAborted(signal, name)` throws
an aborted error when the provided signal is already aborted.

For the message-based factories, the supplied message is stored in the cause;
the exposed `error.message` is the framework's standard message key.
`this.fail()` does not automatically expose the cause. Its optional second
argument sets client-visible details: never put sensitive internal errors,
credentials, or stack traces there.

## Work with headers, cookies, URLs, and bodies

The remaining `HttpHelper` methods are useful at transport or integration
boundaries. They do not authenticate callers or validate all untrusted data.

### `normalizeHeaders()`

```ts
import { HttpHelper } from '@xeno-js/core'

const headers = HttpHelper.normalizeHeaders({
  'accept': ['application/json', 'text/plain'],
  'x-retry': 2,
  'unused': undefined,
})
// { accept: 'application/json,text/plain', 'x-retry': '2' }
```

The helper converts values to strings, joins arrays with commas, and skips
`null` and `undefined`. It preserves input key casing; it does not normalize
header names or validate their safety. Invalid non-object input yields `{}`. Use
it with plain header objects, not as a substitute for iterating a `Headers`
instance.

### `parseCookies()`

```ts
import { HttpHelper } from '@xeno-js/core'

const cookies = HttpHelper.parseCookies('theme=dark; language=en%2DGB')
// { theme: 'dark', language: 'en-GB' }
```

The helper splits a `Cookie` header into a dictionary, decodes values, and uses
the last value for duplicate names. A value with invalid percent encoding
remains undecoded; a name without `=` gets an empty value. This parses cookies
but does not verify signatures, decrypt values, or authenticate them.

### `parseSafeUrl()` and `sanitizeOriginUrl()`

```ts
import { HttpHelper } from '@xeno-js/core'

const url = HttpHelper.parseSafeUrl(
  '/api/orders?page=2',
  'https://api.example.com',
)
// URL for https://api.example.com/api/orders?page=2

const origin = HttpHelper.sanitizeOriginUrl('https://shop.example.com/cart')
// 'https://shop.example.com'
```

`parseSafeUrl()` first tries the input as an absolute URL, then resolves it
against the supplied base, which defaults to `http://localhost`. Invalid input
or base can still throw. Despite its name, it does not enforce an allowed
scheme, hostname, or destination and is not an SSRF defense.

`sanitizeOriginUrl()` returns the parsed URL's `origin`, or `undefined` if
parsing fails. Some URL schemes have the string origin `'null'`; the helper does
not reject that value or check an allowlist. Validate schemes and origins
against the deployment's policy before trusting them.

### `parseBody()`

```ts
import { HttpHelper } from '@xeno-js/core'

const body = HttpHelper.parseBody('POST', { orderId: '42' })
// JSON string: '{"orderId":"42"}'
```

`parseBody(method, rawBody)` prepares a `BodyInit` for `POST`, `PUT`, `PATCH`,
or `DELETE`; it returns `undefined` for other methods or a missing body. It
passes supported body values such as strings, `Blob`, `FormData`, and
`URLSearchParams` through, stringifies objects, and converts other values to
strings. If object JSON serialization fails, the stringification helper falls
back to `String(rawBody)`. It does not choose a content type or validate a
request payload. This transport-level method is distinct from
`HttpRouteParamHelper.extractInput()`, which does not read `DELETE` bodies.

### `maskIp()`

```ts
import { HttpHelper } from '@xeno-js/core'

const masked = HttpHelper.maskIp('192.0.2.14')
// '192.0.2.x'
```

For a dotted four-part value, the helper replaces the last part with `x`; for a
value containing `:`, it replaces the last colon-separated part. It returns
other values unchanged and `undefined` for an absent input. It does not validate
addresses and is not a guarantee of anonymization; choose a logging policy
appropriate to the data and deployment.

## Next steps

- Define and register endpoints in [HTTP Routes](./routes).
- Implement controller dispatch in [HTTP Controllers](./controller).
- Review response payloads and transport differences in
  [API Responses](./api-response).
- Configure a server in [HTTP Server](./server).

---

---

<a id="source-5"></a>

# Source: `docs/http/plugins/fastify.md`

## Introduction

`@xeno-js/fastify` connects a Xeno.JS application to
[Fastify](https://fastify.dev/) as its HTTP transport. Use it when your
application runs as a Node.js process and you want to configure Fastify's server
options while keeping routes and application services in Xeno.JS.

The plugin selects and configures the transport. Your application still defines
routes, registers controllers and services, and configures the middleware and
security it needs.

The setup is:

1. Install the plugin and its core peer dependency.
2. Register routes, services, and `withFastify()` in `XenoAppBuilder`.
3. Build the application and start the listener.

## Installation

```bash
npm install @xeno-js/core @xeno-js/fastify
```

The package declares Node.js `>=20` and `@xeno-js/core` as a peer dependency.
Fastify is a dependency of `@xeno-js/fastify`; install it directly as well if
your application imports Fastify APIs. Select compatible package versions for
your application.

## Start a server

This example registers a controller and a route, then listens on the local
interface. The controller returns a fixed order; replace it with application
services for a real endpoint.

```ts
import { HttpHelper, XenoAppBuilder } from '@xeno-js/core'
import type {
  IController,
  NativeRouteDefinition,
  ResponseDto,
  XenoRegistry,
} from '@xeno-js/core'
import { withFastify } from '@xeno-js/fastify'

interface OrderSummary {
  id: string
  status: string
}

interface OrdersRegistry extends XenoRegistry {
  ORDER_STATUS_CONTROLLER: IController<void, OrderSummary>
}

class OrderStatusController implements IController<void, OrderSummary> {
  async handle(): Promise<ResponseDto<OrderSummary>> {
    return HttpHelper.success({ id: '42', status: 'received' })
  }
}

const routes: NativeRouteDefinition<OrdersRegistry>[] = [
  {
    method: 'GET',
    path: '/api/orders/current',
    handler: async (ctx) => ctx.handle('ORDER_STATUS_CONTROLLER'),
  },
]

async function main() {
  const app = await new XenoAppBuilder<OrdersRegistry>()
    .addRoutes(routes)
    .addServices((services) => {
      services.addScoped(
        'ORDER_STATUS_CONTROLLER',
        () => new OrderStatusController(),
      )
    })
    .addHttpServer(
      withFastify<OrdersRegistry>((options) => {
        options.trustProxy = false
      }),
    )
    .build()

  await app.listen({
    host: '127.0.0.1',
    port: 3000,
    onListening: ({ host, port }) => {
      console.info(`Server listening on ${host}:${port}`)
    },
  })
}

main().catch((error: unknown) => {
  console.error('Application startup failed', error)
  process.exitCode = 1
})
```

After startup, request the endpoint:

```bash
curl http://127.0.0.1:3000/api/orders/current
```

The server returns status `200` and the controller's success payload:

```json
{
  "success": true,
  "data": { "id": "42", "status": "received" },
  "meta": {}
}
```

`build()` initializes the queued modules. `listen()` creates and starts the
Fastify server; `onListening` runs after Fastify starts listening. The example
uses `ctx.handle()`, so controller dispatch uses a request scope. It does not
configure the full HTTP middleware or authentication; configure these before
exposing endpoints that require them.

Await `listen()` to handle a rejected Fastify startup, such as a failure to bind
the requested address. The entry point reports rejected build or startup
operations.

## Configure Fastify

`withFastify()` accepts an optional setup callback. Its first argument is
Fastify's `FastifyServerOptions`; its second is the Xeno.JS configuration
service:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withFastify } from '@xeno-js/fastify'

const builder = new XenoAppBuilder().addRoutes([]).addHttpServer(
  withFastify((options, config) => {
    options.logger = false
    options.trustProxy = false
    options.bodyLimit = config.getNumber('HTTP_BODY_LIMIT', 1048576)
  }),
)

const app = await builder.build()
await app.listen({ host: '127.0.0.1', port: 3000 })
```

The callback sets options on the Fastify instance; `HTTP_BODY_LIMIT` is an
application-chosen configuration key, not a built-in Xeno.JS environment
variable. `bodyLimit` uses bytes. Choose a limit for the requests your
application accepts and account for any proxy limits. With the default Node
configuration service, an invalid numeric value raises a configuration error.
See the
[Fastify server options documentation](https://fastify.dev/docs/latest/Reference/Server/)
for supported upstream settings.

The plugin initializes these Fastify options before invoking the callback:

| Option       | Plugin default | Consideration                                                                        |
| ------------ | -------------- | ------------------------------------------------------------------------------------ |
| `logger`     | `false`        | Fastify's logger is separate from the Xeno.JS logger.                                |
| `trustProxy` | `true`         | Review this default; set `false` when the deployment should not trust proxy headers. |

Set `trustProxy` to an appropriate value for the actual proxy topology.
Fastify's `trustProxy` and Xeno.JS middleware's `trustedProxies` and
`trustedIpHeader` are separate settings. Configuring one does not configure the
other; trust only proxies you control.

`withFastify()` configures Fastify construction options, not Xeno.JS middleware,
routes, or controller registrations. Those remain builder configuration.

## Configure the listener

Pass listener options to `app.listen()`:

| Option        | Type                       | Use                                              |
| ------------- | -------------------------- | ------------------------------------------------ |
| `host`        | `string`                   | Network interface or host on which to listen.    |
| `port`        | `number`                   | TCP port on which to listen.                     |
| `onListening` | `({ host, port }) => void` | Callback invoked after Fastify starts listening. |

You can omit `host`, `port`, or the whole options object:

```ts
// After registering routes and Fastify and building the application.
await app.listen()
```

Xeno.JS selects each value in this order:

1. The value passed to `listen()`.
2. `HOST` or `PORT` from the configuration service.
3. `0.0.0.0` for the host and `3000` for the port.

The default Node configuration service reads environment variables; it does not
load a `.env` file by itself. `0.0.0.0` listens on all IPv4 interfaces. Use
`127.0.0.1` for local-only access, or set the host required by the deployment.
The options factory reads `PORT` even if an explicit port is provided, so a
non-numeric `PORT` causes a configuration error.

`onListening` receives the configured host and port, not the resolved server
address. In particular, if you specify port `0`, its `port` is still `0`, not
the operating system's selected port.

The current Fastify implementation returns its Fastify instance from `listen()`,
but the public `XenoApplication.listen()` contract returns `Promise<unknown>`.
Do not assume the public API provides a typed Fastify instance or a pre-listen
plugin-registration hook.

## Register routes and middleware

Use `addRoutes()` before `build()`. The route module provides the dispatch
services needed by the Fastify integration. To start a listener without
endpoints, register an empty array with `addRoutes([])`; requests then receive
`404`. Registering the server plugin alone does not provide all the dispatcher
dependencies needed for `listen()`.

For application routes, register the controller token and its implementation as
in the example above. See [HTTP Routes](../routes) for route definitions,
parameters, matching, and policies.

Use `addMiddlewares()` to configure HTTP middleware such as CORS, rate limiting,
and CSRF. Authentication and authorization need their own configuration. Route
handlers that return a response directly do not run the `ctx.handle()`
middleware dispatch path. See [HTTP Server](../server) for a middleware example.

## Requests and responses

The integration registers a Fastify wildcard route for `GET`, `POST`, `PUT`,
`PATCH`, `DELETE`, `OPTIONS`, and `HEAD`. It maps matching requests to Xeno.JS
routes and writes the returned `ResponseDto` status, headers, and `data`
payload. See [API Responses](../api-response) for success and error payloads.

Some transport-specific behavior matters when configuring an application:

- Fastify parses request bodies before the Xeno.JS route handler receives them.
  Parser errors, content-type handling, and body-size limits can produce Fastify
  responses before controller dispatch; consult Fastify's documentation and test
  the content types your application accepts.
- Query data comes from Fastify's parsed query object. Validate query and body
  values before using them; `HttpRouteParamHelper.extractInput()` does not
  validate its generic type and does not read `DELETE` bodies.
- The mapper creates a signal for an aborted Fastify request, but the current
  Xeno.JS adapter does not forward the incoming signal into the `Request` used
  for controller dispatch. Do not rely on client disconnects to cancel
  controller work.
- If Xeno.JS cannot match a route, the adapter returns status `404` and a body
  such as `{ "error": "Route GET /missing not found" }`. This is not the
  standard controller error envelope.
- When a route or dispatch throws outside the controller's formatted response
  path, the adapter returns `500` with the thrown error's message in an `error`
  field. Avoid exposing sensitive exception messages; handle expected failures
  explicitly and protect the deployment boundary.
- The mapper removes the exact lowercase `content-length` key from response
  headers before handing them to Fastify. Do not depend on manually setting
  content length through the controller response.

## Stop the server

After startup, the registered HTTP server exposes `dispose()`:

```ts
// app is the application returned by build().
await app.getServices().httpServer.dispose()
```

The integration calls Fastify's `close()` and clears its server reference. This
call closes the HTTP server only; it is not an application-wide shutdown API and
does not by itself dispose all other registered services. Coordinate shutdown of
the rest of the application and external resources according to your
deployment's lifecycle. Avoid calling `listen()` repeatedly on the same server
instance without closing it.

## Next steps

- Configure host, middleware, and other transports in [HTTP Server](../server).
- Define endpoints in [HTTP Routes](../routes).
- Implement controller responses in [HTTP Controllers](../controller) and
  [API Responses](../api-response).
- Review [Context and Scopes](../../fundamentals/context-scopes) for request
  context and scoped services.

---

---

<a id="source-6"></a>

# Source: `docs/http/plugins/node.md`

## Introduction

`@xeno-js/node` connects a Xeno.JS application to Node.js's built-in HTTP
server. Use it when you want to run the application as a Node.js process without
selecting a separate web server framework.

The plugin configures the transport. Your application still defines routes,
registers controllers and services, and configures any required middleware. This
keeps the application boundary separate from the choice of server.

The setup is:

1. Install the plugin and its core peer dependency.
2. Register routes, services, and `withNodeHttpServer()` in `XenoAppBuilder`.
3. Build the application and start the listener.

## Installation

```bash
npm install @xeno-js/core @xeno-js/node
```

The package declares Node.js `>=20` and `@xeno-js/core` as a peer dependency.
Select compatible package versions for your application.

## Start a server

This example registers a controller and a route, then listens on the local
interface. The controller returns a fixed order; replace it with application
services for a real endpoint.

```ts
import { HttpHelper, XenoAppBuilder } from '@xeno-js/core'
import type {
  IController,
  NativeRouteDefinition,
  ResponseDto,
  XenoRegistry,
} from '@xeno-js/core'
import { withNodeHttpServer } from '@xeno-js/node'

interface OrderSummary {
  id: string
  status: string
}

interface OrdersRegistry extends XenoRegistry {
  ORDER_STATUS_CONTROLLER: IController<void, OrderSummary>
}

class OrderStatusController implements IController<void, OrderSummary> {
  async handle(): Promise<ResponseDto<OrderSummary>> {
    return HttpHelper.success({ id: '42', status: 'received' })
  }
}

const routes: NativeRouteDefinition<OrdersRegistry>[] = [
  {
    method: 'GET',
    path: '/api/orders/current',
    handler: async (ctx) => ctx.handle('ORDER_STATUS_CONTROLLER'),
  },
]

async function main() {
  const app = await new XenoAppBuilder<OrdersRegistry>()
    .addRoutes(routes)
    .addServices((services) => {
      services.addScoped(
        'ORDER_STATUS_CONTROLLER',
        () => new OrderStatusController(),
      )
    })
    .addHttpServer(withNodeHttpServer<OrdersRegistry>())
    .build()

  await app.listen({
    host: '127.0.0.1',
    port: 3000,
    onListening: ({ host, port }) => {
      console.info(`Server listening on ${host}:${port}`)
    },
  })
}

main().catch((error: unknown) => {
  console.error('Application startup failed', error)
  process.exitCode = 1
})
```

After startup, request the endpoint:

```bash
curl http://127.0.0.1:3000/api/orders/current
```

The server returns status `200` and the controller's success payload:

```json
{
  "success": true,
  "data": { "id": "42", "status": "received" },
  "meta": {}
}
```

`build()` initializes the queued modules. `listen()` resolves and starts the
Node server; `onListening` runs after the listener starts. The example uses
`ctx.handle()`, so controller dispatch uses a request scope. It does not
configure the full HTTP middleware or authentication; configure these before
exposing endpoints that require them.

The `catch` above reports rejected bootstrap and startup operations, but the
current Node server implementation does **not** attach a listener that rejects
the promise on Node's asynchronous server `error` event. For example, an
occupied port can emit an unhandled error instead of reaching that `catch`. Do
not treat `await app.listen()` as complete coverage of listener errors in this
integration; account for this limitation in deployment monitoring and testing.

## Configure the listener

`withNodeHttpServer()` takes no arguments. Configure the listener with
`app.listen()`:

| Option        | Type                       | Use                                           |
| ------------- | -------------------------- | --------------------------------------------- |
| `host`        | `string`                   | Network interface or host on which to listen. |
| `port`        | `number`                   | TCP port on which to listen.                  |
| `onListening` | `({ host, port }) => void` | Callback invoked when the listener starts.    |

You can omit `host`, `port`, or the whole options object:

```ts
// After registering routes and the Node server and building the application.
await app.listen()
```

Xeno.JS selects each value in this order:

1. The value passed to `listen()`.
2. `HOST` or `PORT` from the configuration service.
3. `0.0.0.0` for the host and `3000` for the port.

The default Node configuration service reads environment variables; it does not
load a `.env` file by itself. `0.0.0.0` listens on all IPv4 interfaces. Use
`127.0.0.1` for local-only access, or set the host required by the deployment.
The options factory reads `PORT` even if an explicit port is provided, so a
non-numeric `PORT` causes a configuration error.

`onListening` receives the configured host and port, not the resolved server
address. In particular, if you specify port `0`, its `port` is still `0`, not
the operating system's selected port. The public plugin does not expose the
native `http.Server` instance through `app.listen()`.

The plugin uses `node:http`, not a TLS listener. If clients require HTTPS,
terminate TLS through a properly configured reverse proxy or hosting platform.
The plugin does not expose native server construction options through
`withNodeHttpServer()`.

## Register routes and middleware

Use `addRoutes()` before `build()`. The route module provides the dispatch
services needed by the Node integration. To start a listener without endpoints,
register an empty array with `addRoutes([])`; requests then receive `404`.
Registering the server plugin alone does not provide all the dispatcher
dependencies needed for `listen()`.

For application routes, register the controller token and its implementation as
in the example above. See [HTTP Routes](../routes) for route definitions,
parameters, matching, and policies.

Use `addMiddlewares()` to configure HTTP middleware such as CORS, rate limiting,
and CSRF. Authentication and authorization need their own configuration. Route
handlers that return a response directly do not run the `ctx.handle()`
middleware dispatch path. If your application is behind a reverse proxy,
configure forwarded headers and trusted proxies only for proxies you control.
See [HTTP Server](../server) for a middleware example.

## Requests and responses

The Node integration maps native requests to Xeno.JS request objects and passes
matching requests to the registered route. For controller responses, it writes
the `ResponseDto` status and headers and serializes `ResponseDto.data` as the
body. See [API Responses](../api-response) for success and error payloads.

Some transport-specific behavior matters when configuring an application:

- The Node integration buffers the body of a matching request in memory before
  dispatch. It does not expose a request-body size limit through this plugin;
  enforce an appropriate limit at a trusted proxy or another boundary.
- `GET`, `HEAD`, and `OPTIONS` bodies are not read by the Node mapper.
  `HttpRouteParamHelper.extractInput()` also does not read `DELETE` bodies;
  consult [HTTP Helpers](../helper-http) before designing body handling.
- The adapter does not forward the incoming transport signal into the `Request`
  used for controller dispatch. Do not rely on client disconnects to cancel
  controller work.
- An unmatched route returns status `404` and a body such as
  `{ "error": "Route GET /missing not found" }`. This is not the standard
  controller error envelope.
- When a route or dispatch throws outside the controller's formatted response
  path, the Node integration returns `500` with the thrown error's message in an
  `error` field. Avoid exposing sensitive exception messages; handle expected
  failures explicitly and protect the deployment boundary.

## Stop the server

After startup, the registered HTTP server exposes `dispose()`:

```ts
// app is the application returned by build().
await app.getServices().httpServer.dispose()
```

The Node integration closes its listener and clears its server reference. This
call closes the HTTP server only; it is not an application-wide shutdown API and
does not by itself dispose all other registered services. Coordinate shutdown of
the rest of the application and external resources according to your
deployment's lifecycle. Avoid calling `listen()` repeatedly on the same server
instance without closing it.

## Next steps

- Configure host, middleware, and other transports in [HTTP Server](../server).
- Define endpoints in [HTTP Routes](../routes).
- Implement controller responses in [HTTP Controllers](../controller) and
  [API Responses](../api-response).
- Review [Context and Scopes](../../fundamentals/context-scopes) for request
  context and scoped services.

---

---

<a id="source-7"></a>

# Source: `docs/http/plugins/vercel.md`

## Introduction

`@xeno-js/vercel` connects a Xeno.JS application to a Vercel Function. Vercel
receives the HTTP request and invokes your function; Xeno.JS matches the route
and dispatches to the configured application controller.

Unlike the [Node.js](./node) and [Fastify](./fastify) integrations,
`@xeno-js/vercel` does not provide an `addHttpServer()` plugin. Use
`XenoVercel.handle()` as the function entry point. Do not start a listener with
`app.listen()` in that function.

The setup is:

1. Install the integration and its core peer dependency.
2. Register routes and services with `XenoAppBuilder`.
3. Export a Vercel Node.js handler that passes the request, response, and an
   asynchronous application getter to `XenoVercel.handle()`.

## Installation

```bash
npm install @xeno-js/core @xeno-js/vercel @vercel/node
```

The package declares Node.js `>=20` and `@xeno-js/core` as a peer dependency. It
also depends on `@vercel/node`; the example installs it directly because the
application imports its request and response types. Choose compatible package
versions and a Node.js version supported by Vercel.

## Create a function

The following example defines one function and one matching Xeno.JS route. The
controller returns a fixed order; replace it with application services for a
real endpoint.

Create `api/order-status.ts`:

```ts
import { HttpHelper, XenoAppBuilder } from '@xeno-js/core'
import type {
  IController,
  NativeRouteDefinition,
  ResponseDto,
  XenoRegistry,
} from '@xeno-js/core'
import { XenoVercel } from '@xeno-js/vercel'
import type { VercelRequest, VercelResponse } from '@vercel/node'

interface OrderSummary {
  id: string
  status: string
}

interface OrdersRegistry extends XenoRegistry {
  ORDER_STATUS_CONTROLLER: IController<void, OrderSummary>
}

class OrderStatusController implements IController<void, OrderSummary> {
  async handle(): Promise<ResponseDto<OrderSummary>> {
    return HttpHelper.success({ id: '42', status: 'received' })
  }
}

const routes: NativeRouteDefinition<OrdersRegistry>[] = [
  {
    method: 'GET',
    path: '/api/order-status',
    handler: async (ctx) => ctx.handle('ORDER_STATUS_CONTROLLER'),
  },
]

const builder = new XenoAppBuilder<OrdersRegistry>()
  .addRoutes(routes)
  .addServices((services) => {
    services.addScoped(
      'ORDER_STATUS_CONTROLLER',
      () => new OrderStatusController(),
    )
  })

function getApp() {
  return builder.build()
}

export default async function handler(
  req: VercelRequest,
  res: VercelResponse,
): Promise<VercelResponse> {
  return XenoVercel.handle(req, res, getApp)
}
```

Deploy this as a Vercel Node.js Function. A request to `GET /api/order-status`
reaches the function and matches the Xeno.JS route. On success, the response has
status `200` and this JSON body:

```json
{
  "success": true,
  "data": { "id": "42", "status": "received" },
  "meta": {}
}
```

The builder queues the route and controller registrations. `getApp()` builds the
application when the function handles a request, and the adapter invokes the
route and controller. The example uses `ctx.handle()` and the default request
scope; it does not configure the full HTTP middleware or authentication.
Configure the protections required by your application before exposing real
endpoints.

See Vercel's
[Node.js runtime documentation](https://vercel.com/docs/functions/runtimes/node-js)
for function discovery, runtime versions, and deployment configuration.

## Reuse the application

`XenoVercel.handle()` calls the supplied `getApp()` for each invocation. It does
not cache the application itself. In the example, the builder lives at module
scope and `getApp()` returns `builder.build()`. The builder reuses its build
promise, so concurrent calls in the same function instance share initialization
and later calls can reuse the resulting application.

This reuse is local to that function instance. Vercel can start multiple
instances or replace them, so do not depend on module state for cross-instance
coordination or durability. In-memory caches, counters, and rate limits do not
provide a shared global store across instances; use an appropriate shared
service when the application requires one.

If bootstrap fails, the adapter logs the error and returns a `500` response; the
builder attempts to reset its failed build state. Do not convert a failed
initialization into a successful-looking application.

## Route requests to the function

For a simple endpoint, use a function file and Xeno.JS route whose path matches
the URL received by the function, as in the example above. `addRoutes()` is
required to register the route matcher and dispatch services. An empty route
list has no application endpoints.

To serve several paths through one function, configure Vercel routing or
rewrites for the deployment. The adapter matches using `req.url` and
`req.method` as received by the function, not the URL displayed in the browser.
Verify that each requested path reaches the function with the path your Xeno.JS
route expects. A rewrite's exact behavior depends on your deployment
configuration; test it against the deployed or locally emulated function. See
[Vercel rewrites](https://vercel.com/docs/routing/rewrites) and
[HTTP Routes](../routes).

Use the Node.js `VercelRequest` and `VercelResponse` handler contract shown
above. The package also mentions edge deployments, but the provided handler
requires these Node-style objects; it is not a Web `Request`/`Response` handler.
Edge runtime compatibility for this integration cannot be verified from the
current implementation.

## Configure middleware and services

Register middleware and application services on the builder before the first
call to `build()`:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { routes } from './routes.js'

const builder = new XenoAppBuilder()
  .addRoutes(routes)
  .addMiddlewares((options) => {
    options.cors = {
      enabled: true,
      enablePreflight: true,
      allowOrigins: ['https://shop.example.com'],
      withCredentials: false,
    }
    options.rateLimit = {
      enabled: true,
      maxRequests: 100,
      windowSeconds: 60,
    }
  })
// Register controllers, authentication, and other application services.

const app = await builder.build()
```

This is a composition excerpt; use the application registry type and register
the controllers required by `routes`. Choose origins and limits for your
deployment. Route handlers that return directly do not run the `ctx.handle()`
middleware dispatch path. CORS is not authentication or authorization. Without
`addMiddlewares()`, route dispatch uses a minimal synthetic guest context
instead of the configured HTTP middleware.

When using forwarded client IPs, configure `trustedIpHeader` and
`trustedProxies` for the actual proxy topology rather than trusting arbitrary
headers. The current resolver compares socket addresses against the configured
entries exactly; `'*'` is not a wildcard for that check. When no socket address
is available, the resolver can use a forwarded header, so ensure the deployment
boundary supplies trustworthy values. Verify client IP and rate-limit behavior
in the deployed environment.

## Requests and responses

The adapter passes Vercel's request headers, body, query, URL, method, and
socket information to Xeno.JS. Body and query parsing depend on the Vercel
Node.js runtime; validate input in your application. The adapter forwards
`req.query` as supplied by Vercel, which can include arrays or routing
parameters. Do not assume every query value is a string.
`HttpRouteParamHelper.extractInput()` does not read `DELETE` bodies and its
generic type does not validate values. See [HTTP Helpers](../helper-http).

For a controller response, the adapter copies response headers, uses the
`ResponseDto` status, and sends `ResponseDto.data` as JSON. It converts the
headers through the Web `Headers` API before writing them to Vercel; verify
multi-value and cookie headers in your deployment rather than assuming they
preserve the original array representation. See [API Responses](../api-response)
for the response DTO.

The adapter also handles these cases:

- If no route matches, it returns `404` with `success: false` and an `error`
  containing `NOT_FOUND`, a message, and the requested path. This body does not
  include all fields of a controller error response.
- If a route returns no result at runtime, it responds with `204` and ends the
  response. The public route contract normally requires a `ResponseDto`; do not
  rely on this fallback in typed route implementations.
- If initialization or dispatch throws, it logs the error and returns a `500`
  system-error payload with generated correlation and request IDs. It includes
  the error message in `details` only when `NODE_ENV === 'development'`.
  Controller `this.fail()` responses follow their own configured error-details
  behavior.

The adapter passes the incoming request signal to Xeno.JS, but the current HTTP
adapter creates a new `Request` without forwarding that signal. Do not assume a
client disconnect cancels controller work.

## Context and lifetimes

With the default Node context, Xeno.JS uses `AsyncLocalStorage` to make the
request context available during controller dispatch. Scoped services resolve
within that execution; transient services are created per resolution and
singletons are shared by the application container in the function instance. Do
not inject scoped dependencies into a singleton. Avoid putting request-specific
mutable state in singletons, especially when an instance handles concurrent
requests.

Vercel manages function and listener lifecycles. The integration does not expose
a server to close with `httpServer.dispose()`; design external connections and
resources for the platform lifecycle instead of assuming that a process will
remain alive or shut down on a fixed schedule.

## Next steps

- Define paths and route policies in [HTTP Routes](../routes).
- Implement controller responses in [HTTP Controllers](../controller) and
  [API Responses](../api-response).
- Review [Context and Scopes](../../fundamentals/context-scopes) and
  [Dependency Injection](../../fundamentals/dependency-injection).
- Compare server integrations in [HTTP Server](../server).

---

---

<a id="source-8"></a>

# Source: `docs/http/routes.md`

## Introduction

An HTTP application needs to map incoming methods and paths to application
operations. Xeno.JS expresses these mappings as route definitions, separately
from the server integration and the controller implementation.

Each route specifies:

- an HTTP method;
- a path, optionally containing named parameters;
- a handler that receives the request and can dispatch to a controller;
- optional security settings for the route.

Register the definitions with `addRoutes()` during application composition. The
server integration receives requests; routes select the application operation to
run. For server setup, see [HTTP Server](./server).

## Define and register a route

The following example defines a route and a controller, registers both, and
starts a Node.js server. It uses a fixed response to keep the example focused on
the route; a real controller would delegate to application services.

```bash
npm install @xeno-js/core @xeno-js/node
```

```ts
import { HttpHelper, HttpRouteParamHelper, XenoAppBuilder } from '@xeno-js/core'
import type {
  IController,
  NativeRouteDefinition,
  ResponseDto,
  XenoRegistry,
} from '@xeno-js/core'
import { withNodeHttpServer } from '@xeno-js/node'

interface GetOrderInput {
  id: string
}

interface OrderSummary {
  id: string
  status: string
}

interface OrdersRegistry extends XenoRegistry {
  GET_ORDER_CONTROLLER: IController<GetOrderInput, OrderSummary>
}

class GetOrderController implements IController<GetOrderInput, OrderSummary> {
  async handle(input: GetOrderInput): Promise<ResponseDto<OrderSummary>> {
    return HttpHelper.success({ id: input.id, status: 'received' })
  }
}

const orderRoute: NativeRouteDefinition<OrdersRegistry> = {
  method: 'GET',
  path: '/api/orders/:id',
  handler: async (ctx) => {
    const input = await HttpRouteParamHelper.extractInput<GetOrderInput>(
      ctx.req,
      '/api/orders/:id',
    )
    return ctx.handle('GET_ORDER_CONTROLLER', input)
  },
}

async function main() {
  const app = await new XenoAppBuilder<OrdersRegistry>()
    .addRoutes([orderRoute])
    .addServices((services) => {
      services.addScoped('GET_ORDER_CONTROLLER', () => new GetOrderController())
    })
    .addHttpServer(withNodeHttpServer<OrdersRegistry>())
    .build()

  await app.listen({ host: '127.0.0.1', port: 3000 })
}

main().catch((error: unknown) => {
  console.error('Failed to start the application', error)
  process.exitCode = 1
})
```

A `GET` request to `/api/orders/42` matches the route. The handler extracts an
input containing `id: '42'` and passes it to the registered controller.
`HttpHelper.success` creates a `ResponseDto` with status `200` and a JSON
response body:

```json
{
  "success": true,
  "data": { "id": "42", "status": "received" },
  "meta": {}
}
```

The registry describes valid service keys at compile time. `addServices()`
registers the controller at runtime; defining a key in the registry alone does
not register it. The controller resolves within the dispatch scope.
`addScoped()` reuses an instance within that scope; `addSingleton()` shares an
instance across the container, while `addTransient()` creates one per
resolution. Do not inject a scoped dependency into a singleton. See
[Dependency Injection](../fundamentals/dependency-injection) for lifetimes.

This example does not call `addMiddlewares()`. Xeno.JS supplies a minimal
dispatch context for the controller, but it does not enable the configured CORS,
authentication, rate limiting, or CSRF middleware. Configure those explicitly
when the application needs them.

## Route definition

`NativeRouteDefinition<TRegistry>` has the following properties:

| Property   | Purpose                                                                           |
| ---------- | --------------------------------------------------------------------------------- |
| `method`   | HTTP method: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`, or `OPTIONS`.       |
| `path`     | Route path, with optional named segments such as `:id` or `{id}`.                 |
| `handler`  | Async function that receives a `RouteHandlerContext` and returns a `ResponseDto`. |
| `security` | Optional per-route settings consumed by the relevant middleware.                  |

Use the same registry type for the builder and your route definitions when
dispatching to application-specific controller tokens.

### Match paths and methods

The matcher compares both method and path. It ignores the query string when
matching, normalizes trailing slashes, and matches methods without case
sensitivity. For example, `GET /api/orders/42?include=items` matches
`GET /api/orders/:id`.

Both `:id` and `{id}` match one path segment. Static path matching is
case-sensitive. The first matching definition wins, including when a dynamic
route overlaps a static route. Register more specific routes first, for example
`/api/orders/recent` before `/api/orders/:id`, when both use the same method.

If no route matches, the server returns `404`. The matcher has a special
fallback for `OPTIONS`: if no explicit `OPTIONS` route matches but another route
matches the path, it supplies a preflight route. This fallback does not replace
middleware configuration; see [Preflight requests](#preflight-requests).

### Organize route lists

Group routes by feature and add each list before `build()`:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import type { OrdersRegistry } from './registry'
import { orderRoutes } from './orders.routes'
import { productRoutes } from './products.routes'

const builder = new XenoAppBuilder<OrdersRegistry>()
  .addRoutes(orderRoutes)
  .addRoutes(productRoutes)

// Register controllers, middleware, and a server integration as needed.
const app = await builder.build()
```

Each call appends definitions; it does not replace the earlier list.
Registration order therefore matters across lists as well as within a list. Use
route arrays typed with the application's registry, and make sure that the
registry includes all controller tokens used by those arrays.

## Use the route handler

The handler receives a context with two members:

| Member                      | Purpose                                                                                                           |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `ctx.req`                   | A `XenoRequest`, extending the standard `Request` with `path`, optional `query`, and optional socket information. |
| `ctx.handle(token, input?)` | Dispatches to a registered controller and returns its `ResponseDto`.                                              |

Call `ctx.handle()` with a token from the registry. The dispatcher runs the
configured middleware, resolves the controller in the active request scope, and
calls its `handle()` method. The token must resolve to a service with a `handle`
method at runtime. The generic dispatch signature checks the token against the
registry, but does not infer and validate the input and output types from that
controller's contract. Validate input and keep those types aligned explicitly.

The handler may instead return a `ResponseDto` directly. In that case it
bypasses `ctx.handle()` and its middleware and controller dispatch path. Do not
use a direct response as a substitute for configured authentication,
authorization, rate limiting, or other middleware checks.

When the Node context is used, request-scoped execution uses `AsyncLocalStorage`
to make `RequestContext` available to dispatched work. See
[Context and Scopes](../fundamentals/context-scopes) for access to identity,
tracing, and scoped services.

## Extract request input

`HttpRouteParamHelper.extractInput<TInput>(ctx.req, routePattern)` combines
available body, query, and path values into one object. Pass the same pattern
used by the route so named path segments map to the intended keys.

For example:

```ts
const input = await HttpRouteParamHelper.extractInput<{
  id: string
  include?: string
}>(ctx.req, '/api/orders/:id')
```

For `/api/orders/42?include=items`, the extracted values include
`{ id: '42', include: 'items' }`. This excerpt belongs inside a route handler
where `ctx` is available.

The helper applies these rules:

1. For `POST`, `PUT`, and `PATCH`, it reads JSON or URL-encoded bodies according
   to `Content-Type`; other content types are read as text.
2. Object body fields are merged into the input. Non-object values are placed
   under `body`.
3. Query values override body fields with the same name.
4. Path parameters override both query and body fields with the same name.

The helper does not read a body for `GET`, `DELETE`, `HEAD`, or `OPTIONS`. When
no body value is available, the result can include `body: undefined`. It decodes
path parameters, strips control characters from extracted path values and object
body strings, and removes `__proto__`, `constructor`, and `prototype` keys from
object bodies. Query values come from `ctx.req.query`; their parsing can depend
on the server integration.

The type argument describes the expected shape; it does **not** validate,
convert, or guarantee the values. In particular, do not assume a query value is
a number without parsing and validating it. The helper also catches body read or
parsing errors and proceeds without the parsed body. Use explicit validation
before performing application operations; if a malformed body must produce a
specific error, handle body parsing separately rather than relying on this
helper to report the failure.

For separate extraction, `extractPathParams(routePattern, requestPath)` and
`extractQueryParams(requestUrl)` return string dictionaries. Invalid
percent-encoding in a path parameter can throw during decoding.

## Configure route security

Route security settings customize middleware behavior; they do not activate
middleware by themselves. Configure `addMiddlewares()` during bootstrap, then
apply policies to the relevant route definitions. Configure authentication and
authorization separately: CORS is not an authorization mechanism.

### CORS

`withCors()` returns a route definition with allowed origins and a credentials
setting. It defaults `withCredentials` to `true`, so pass it explicitly when
appropriate:

```ts
import { withCors } from '@xeno-js/core'

// Reuse orderRoute and OrdersRegistry from the route definition.
const publicOrderRoute = withCors<OrdersRegistry>(
  orderRoute,
  ['https://shop.example.com'],
  false,
)
```

Add `publicOrderRoute` to the builder instead of the original definition and
enable CORS through `addMiddlewares()`. A route's `allowOrigins` and
`withCredentials` override the corresponding global values for that route. For
wildcard origins, use `false` for credentials unless you deliberately configure
a more restrictive credentialed policy. Review the allowed origins for the
deployment.

### CSRF

When you configure CSRF middleware, state-changing `POST`, `PUT`, `PATCH`, and
`DELETE` requests require its checks by default. Use `withoutCsrf()` only for a
route where disabling that protection is justified and another suitable security
boundary exists, such as a verified webhook:

```ts
import { withoutCsrf } from '@xeno-js/core'

// Reuse an application-defined, authenticated webhook route.
const webhookRouteWithoutCsrf = withoutCsrf(webhookRoute)
```

`withCsrf()` sets the route's `withCsrf` policy to `true`; it does not enable
the middleware. In the current implementation, CSRF validation runs for the
state-changing methods listed above, not for `GET` even when its policy is
`true`. Configure the CSRF middleware and its secret before relying on these
policies.

### Preflight requests

For a matching path without an explicit `OPTIONS` route, the matcher provides a
fallback route that dispatches the preflight request. Configure CORS and
`enablePreflight` in `addMiddlewares()` to use the preflight middleware. It uses
`Access-Control-Request-Method` to inspect the target route's policy and returns
allowed methods and headers.

The origin check also runs for `OPTIONS` before the preflight middleware. When
using the fallback route, make sure the global allowed origins permit the
preflight origin as well as configuring the target route's policy. Test the
actual browser preflight for each deployed transport and route. An unmatched
path does not receive the fallback route.

For controller implementation, registration, and response helpers, see
[HTTP Controllers](./controller). For a reference to request, route-policy, and
HTTP utilities, see [HTTP Helpers](./helper-http).

---

---

<a id="source-9"></a>

# Source: `docs/http/server.md`

## Introduction

An application needs an HTTP transport to receive requests, but its routes and
application services should not depend on a particular server implementation.
Xeno.JS lets you select a server integration when you compose the application.

For a server that listens on a network port:

1. Install a server integration.
2. Register its plugin and your routes with `XenoAppBuilder`.
3. Build the application.
4. Call and await `listen()`.

The server integration and the routes are separate configuration choices.
Changing the transport does not require you to redefine your Xeno.JS routes. For
serverless deployments, the platform may own the listener instead; see
[Vercel](#vercel) below.

## Start a Node.js server

Install the core package and the Node.js server integration:

```bash
npm install @xeno-js/core @xeno-js/node
```

Create and start the application:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withNodeHttpServer } from '@xeno-js/node'

async function main() {
  const app = await new XenoAppBuilder()
    .addRoutes([])
    .addHttpServer(withNodeHttpServer())
    .build()

  await app.listen({
    port: 3000,
    host: '127.0.0.1',
    onListening: ({ host, port }) => {
      console.info(`Server listening on ${host}:${port}`)
    },
  })
}

main().catch((error: unknown) => {
  console.error('Failed to start the application', error)
  process.exitCode = 1
})
```

This example registers an empty route list so that the HTTP services needed by
the server are available. It starts a listener but exposes no endpoints:
requests return `404`. Replace the empty array with your route definitions to
serve application requests. Binding to `127.0.0.1` restricts access to the local
machine; choose a different host if the deployment must accept external
connections.

`build()` initializes the configured modules and returns the application;
`listen()` starts the server. `onListening` runs after the listener starts.
Await startup so that rejected initialization or listening operations can be
handled by your entry point. Error reporting during listening depends on the
selected server integration.

## Configure the listener

Pass listener options to `app.listen()`:

| Option        | Type                       | Purpose                                                |
| ------------- | -------------------------- | ------------------------------------------------------ |
| `port`        | `number`                   | Port on which the server listens.                      |
| `host`        | `string`                   | Host or network interface on which the server listens. |
| `onListening` | `({ host, port }) => void` | Callback invoked after the listener starts.            |

You can omit either option or call `listen()` without arguments. Xeno.JS
resolves the host and port in this order:

1. The value passed to `listen()`.
2. `HOST` or `PORT` from the configuration service.
3. The defaults: `0.0.0.0` and `3000`.

The options factory reads the configuration values even when `listen()` supplies
explicit values. An invalid numeric `PORT` therefore raises a configuration
error instead of being ignored.

For example, with the default Node configuration service, set environment
variables before starting the process:

```text
HOST=127.0.0.1
PORT=8080
```

Then call:

```ts
await app.listen()
```

The default host `0.0.0.0` listens on all IPv4 interfaces. Set `HOST` or pass
`host` explicitly when that is not appropriate for your environment. The
configuration service reads environment values; loading a `.env` file is a
separate application setup step. For port `0`, Node.js and Fastify pass the
configured port to `onListening`, not the dynamically assigned port.

## Register routes

Pass an array of route definitions to `addRoutes()` before `build()`:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withNodeHttpServer } from '@xeno-js/node'
import { routes } from './routes'

const app = await new XenoAppBuilder()
  .addRoutes(routes)
  .addHttpServer(withNodeHttpServer())
  // Register the controllers and other services required by the routes.
  .build()

await app.listen()
```

Here `routes` is an application-defined array of `NativeRouteDefinition`
objects. Each route specifies a method, a path, and a handler. Register its
controllers and dependencies in the same builder; use the application's registry
type with `XenoAppBuilder` and its route definitions when those controllers use
custom tokens.

Calling `addRoutes()` more than once appends definitions. Requests that do not
match a registered route receive a `404` response, except for the `OPTIONS`
fallback on a matching path. See [HTTP Routes](./routes) for route definitions,
input handling, and route policies.

## Configure request middleware

Use `addMiddlewares()` when controller dispatch needs the configured Xeno.JS
middleware, such as CORS or rate limiting:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withNodeHttpServer } from '@xeno-js/node'
import { routes } from './routes'

const app = await new XenoAppBuilder()
  .addRoutes(routes)
  .addMiddlewares((options) => {
    options.cors = {
      enabled: true,
      enablePreflight: true,
      allowOrigins: ['https://app.example.com'],
      withCredentials: true,
    }
    options.rateLimit = {
      enabled: true,
      maxRequests: 100,
      windowSeconds: 60,
    }
  })
  .addHttpServer(withNodeHttpServer())
  // Register the controllers and other services required by the routes.
  .build()

await app.listen()
```

Choose allowed origins and rate limits for your deployment rather than copying
these values unchanged. The middleware executes when a route delegates to a
controller through `ctx.handle()`; a route handler that returns directly does
not go through that dispatch path. For a path that matches another route, the
matcher supplies an `OPTIONS` fallback when no explicit `OPTIONS` route matches;
origin checks and preflight configuration still apply. Configure proxy trust
only for proxies you control, especially when using forwarded client IP headers
for rate limiting.

Without `addMiddlewares()`, route registration provides a minimal dispatch
context; it does not enable the configured CORS, authentication, or rate-limit
middleware. Middleware and route-specific security configuration are separate
from the choice of server plugin.

## Choose a server integration

Register one server plugin with `addHttpServer()`. The builder accepts the first
plugin and ignores later `addHttpServer()` calls; choose the integration before
building rather than attempting to replace it afterwards.

| Integration | Package            | Registration           | Use                                           |
| ----------- | ------------------ | ---------------------- | --------------------------------------------- |
| Node.js     | `@xeno-js/node`    | `withNodeHttpServer()` | [Node.js native HTTP server](./plugins/node). |
| Fastify     | `@xeno-js/fastify` | `withFastify()`        | [Fastify HTTP server](./plugins/fastify).     |

<!-- | Hono | `@xeno-js/hono` | `withHono()` | Hono server with local listening and an edge-function mode. | -->

Install the package for the selected integration. For example, to use Fastify:

```bash
npm install @xeno-js/core @xeno-js/fastify
```

Replace the Node plugin in the bootstrap with:

```ts
import { withFastify } from '@xeno-js/fastify'

builder.addHttpServer(
  withFastify((options) => {
    options.logger = false
    options.trustProxy = false
  }),
)
```

In this excerpt, `builder` is the application's `XenoAppBuilder`. The callback
configures upstream `FastifyServerOptions`; `host` and `port` remain Xeno.JS
`listen()` options. The integration defaults to `logger: false` and
`trustProxy: true`. Review proxy trust for your deployment rather than relying
on that default. Fastify's proxy setting and Xeno.JS middleware's
`trustedProxies` and `trustedIpHeader` are separate settings.

For Hono, `withHono()` uses local listening by default.
`withHono({ edgeFunction: true })` instead makes `listen()` return the Hono
application without opening a TCP listener. The hosting platform must use that
application; do not treat this mode as a conventional listening server. The
individual server integrations will be covered in the HTTP plugins guides.

### Vercel

`@xeno-js/vercel` provides `XenoVercel.handle(req, res, getApp)` rather than an
`addHttpServer()` plugin. In this deployment, Vercel invokes your function and
owns the listener. Build an application with `addRoutes(routes)` and any
required middleware and services, then pass the incoming Vercel request,
response, and asynchronous application getter to `XenoVercel.handle()`. Do not
call `app.listen()` for this integration. See the
[Vercel Functions guide](./plugins/vercel) for setup and deployment
considerations.

---
