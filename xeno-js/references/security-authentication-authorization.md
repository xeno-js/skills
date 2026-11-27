# Security, authentication and authorization

Technical reference assembled from the website documentation. Each section
identifies its original Markdown source. Website frontmatter and the repeated
sponsorship footer are omitted; body content and fenced examples are retained.

---

## Contents

- [`docs/security/authentication/custom.md`](#source-1)
- [`docs/security/authentication/overview.md`](#source-2)
- [`docs/security/authentication/supabase.md`](#source-3)
- [`docs/security/authorization/overview.md`](#source-4)
- [`docs/security/middlewares/allow-method.md`](#source-5)
- [`docs/security/middlewares/allow-origin.md`](#source-6)
- [`docs/security/middlewares/authentication-middleware.md`](#source-7)
- [`docs/security/middlewares/cookie-middleware.md`](#source-8)
- [`docs/security/middlewares/cors-middleware.md`](#source-9)
- [`docs/security/middlewares/csrf-middleware.md`](#source-10)
- [`docs/security/middlewares/options-middleware.md`](#source-11)
- [`docs/security/middlewares/overview.md`](#source-12)
- [`docs/security/middlewares/rate-limit.md`](#source-13)
- [`docs/security/middlewares/request-context-midleware.md`](#source-14)
- [`docs/security/overview.md`](#source-15)

<a id="source-1"></a>

# Source: `docs/security/authentication/custom.md`

## How do I replace Supabase with my own authentication provider?

Xeno.JS does not select Supabase by default. `addAuth()` configures the
authentication service explicitly; use the Supabase plugin for Supabase or
provide your own service through `AuthModuleOptions`.

A custom authentication integration has two separate responsibilities:

1. **Token extraction** — retrieve the credential from the incoming HTTP
   request.
2. **Credential validation** — validate that credential and return the
   authenticated user's claims.

Configure the authentication service and optionally provide a token extractor
through `addAuth()`:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
// Import the application-specific MyAuthService and MyTokenExtractor implementations.

const app = new XenoAppBuilder()

app.addAuth((options) => {
  options.authService = () => new MyAuthService()
  options.tokenExtractor = () => new MyTokenExtractor()
})

app.addMiddlewares()

await app.build()
```

You do not need to configure Supabase credentials when using a custom
authentication service.

---

## Authentication flow

With a custom provider, the request authentication flow is:

```text
HTTP request
      │
      ▼
Configured token extractor
      │
      ▼
AuthenticationMiddleware
      │
      ▼
Custom authentication service
      │
      ▼
AuthClaims
      │
      ▼
ClaimsIdentityMapper
      │
      ▼
Request Identity
```

The extractor answers:

> Where do I get the credential?

The authentication service answers:

> Is this credential valid, and which identity does it represent?

Keeping these responsibilities separate allows the authentication provider and
the HTTP credential format to change independently.

The middleware writes the resulting identity to the request context. In the
default Node context, `AsyncLocalStorage` carries that request context through
asynchronous work.

---

## Configure custom authentication

`addAuth()` accepts a setup action that receives the authentication module
options and configuration service. The relevant options are:

```ts
import type {
  IExtendendAuthService,
  IServiceExtractor,
  IServiceScope,
  Optional,
} from '@xeno-js/shared'

interface AuthModuleOptions<TRegistry> {
  authService: (scope: IServiceScope<TRegistry>) => IExtendendAuthService
  tokenExtractor?: (
    scope: IServiceScope<TRegistry>,
  ) => IServiceExtractor<Request['headers'], string | undefined>
  defaultCookieName?: Optional<string>
}
```

`authService` is required; `tokenExtractor` is optional.

| Property            | Responsibility                                                                                       |
| ------------------- | ---------------------------------------------------------------------------------------------------- |
| `authService`       | Creates the scoped authentication service that validates credentials and returns authentication data |
| `tokenExtractor`    | Creates the extractor used to retrieve a credential from request headers; optional                   |
| `defaultCookieName` | Cookie name checked by the default bearer extractor; defaults to `sb-access-token`                   |

Example:

```ts
app.addAuth((options) => {
  options.authService = () => new MyAuthService()
  options.tokenExtractor = () => new MyTokenExtractor()
})
```

`authService` is required. The authentication module registers its result as a
scoped service and passes the active service scope to its factory. Resolve any
scoped dependencies from that scope. A custom token extractor is registered as a
singleton; keep it stateless and do not capture request-scoped services in its
factory.

---

## Create a custom token extractor

The extractor implements `IServiceExtractor`.

Its contract is:

```ts
interface IServiceExtractor<TRequest, TResponse = unknown> {
  extract(request: TRequest): TResponse
}
```

For authentication, Xeno.JS expects an extractor with this shape:

```ts
IServiceExtractor<Request['headers'], string | undefined>
```

The extractor receives the request headers and returns either:

- the authentication credential;
- `undefined` when no credential is present.

### Example: custom `Authorization` header

Suppose your provider uses:

```http
Authorization: Token <credential>
```

instead of the standard bearer format.

You can implement:

```ts
import type { IServiceExtractor } from '@xeno-js/shared'

export class MyTokenExtractor implements IServiceExtractor<
  Request['headers'],
  string | undefined
> {
  extract(headers: Request['headers']): string | undefined {
    const authorization = headers.get('authorization')

    if (!authorization) {
      return undefined
    }

    if (!authorization.startsWith('Token ')) {
      return undefined
    }

    const token = authorization.slice('Token '.length).trim()
    return token || undefined
  }
}
```

Register it through `AuthModuleOptions.tokenExtractor`:

```ts
app.addAuth((options) => {
  options.authService = () => new MyAuthService()
  options.tokenExtractor = () => new MyTokenExtractor()
})
```

The extractor should only be responsible for retrieving the credential. It
should not validate the token or construct the application identity.

---

## Create a custom authentication service

The custom authentication service must implement `IExtendendAuthService`.

`IExtendendAuthService` combines the base authentication contract with
session/provider operations:

```ts
interface IExtendendAuthService extends IBaseAuthService, IAuthService {}
```

### Base authentication contract

The base service contract exposes these methods:

```ts
interface IBaseAuthService {
  isAuthenticated(): Promise<boolean>

  getUser(): Promise<ResultType<Maybe<AuthClaims>>>

  authenticate(token: string): Promise<ResultType<AuthClaims>>
}
```

The important method for request authentication is:

```ts
authenticate(token: string): Promise<ResultType<AuthClaims>>
```

It receives the credential returned by the token extractor and must return the
authenticated user's claims. When no token is extracted, the gatekeeper calls
`getUser()` instead. `isAuthenticated()` remains part of the service contract
but the HTTP authentication middleware does not call it in this flow.

The extended interface also requires `signInWithProvider()`, `getSession()`,
`signOut()`, and `exchangeCodeForSession()`. Implement these methods even if the
HTTP middleware does not use them; application features may resolve the extended
authentication service for those operations.

---

## Implement the authentication service

A provider-specific service must implement the full contract. The following is a
skeleton, not a production implementation:

```ts
import type {
  IExtendendAuthService,
  ResultType,
  AuthClaims,
  Maybe,
  Optional,
  Provider,
  Session,
} from '@xeno-js/shared'

export class MyAuthService implements IExtendendAuthService {
  async isAuthenticated(): Promise<boolean> {
    // Check whether the current authentication state is valid.
    return false
  }

  async getUser(): Promise<ResultType<Maybe<AuthClaims>>> {
    // Resolve the current authenticated user when no explicit token
    // is supplied to the authentication flow.
    //
    // Return the provider's AuthClaims through ResultType.
    throw new Error('Implement getUser()')
  }

  async authenticate(token: string): Promise<ResultType<AuthClaims>> {
    // Validate the token with your authentication provider.
    //
    // Return AuthClaims through ResultType.
    throw new Error('Implement authenticate()')
  }

  async signInWithProvider(
    provider: Provider,
  ): Promise<ResultType<{ url: string }>> {
    // Implement provider-specific sign-in if your application uses it.
    throw new Error('Implement signInWithProvider()')
  }

  async getSession(): Promise<ResultType<Maybe<Session>>> {
    // Return the current provider session.
    throw new Error('Implement getSession()')
  }

  async signOut(): Promise<ResultType<void>> {
    // Invalidate the current provider session.
    throw new Error('Implement signOut()')
  }

  async exchangeCodeForSession(
    code: string,
  ): Promise<ResultType<Optional<Session>>> {
    // Exchange an authentication callback code for a session.
    throw new Error('Implement exchangeCodeForSession()')
  }
}
```

Replace every placeholder with provider-specific behavior before registering the
service. In particular, do not ship methods that always return `false` or throw
placeholder errors.

The important requirement for HTTP authentication is that `authenticate()`
returns valid `AuthClaims`.

---

## Return `AuthClaims`

Xeno.JS defines the claims shape as:

```ts
interface AuthClaims {
  readonly sub: string
  readonly email: Optional<string>
  readonly name: Optional<string>
  readonly tenantId: Optional<string>
  readonly roles: Optional<string[]>
  readonly permissions: Optional<string[]>
}
```

`sub` is the user identifier. It is required and must be a GUID string because
the identity mapper converts it to the request identity's `userId`.

The remaining fields provide identity and authorization information that can be
propagated into the application request context. Treat provider claims as
untrusted until the provider has validated them. For JWTs, verify the signature
and the expected issuer, audience, and expiry before using identity or
authorization claims.

For example:

```ts
{
  sub: user.id,
  email: user.email,
  name: user.name,
  tenantId: user.tenantId,
  roles: user.roles,
  permissions: user.permissions,
}
```

Do not return an arbitrary application user object from `authenticate()`.

Return the authentication claims expected by Xeno.JS.

---

## How claims become the request identity

After successful authentication, Xeno.JS maps `AuthClaims` to the request
`Identity`.

The mapping is:

```text
AuthClaims
    │
    ├── sub         → userId
    ├── email       → email
    ├── name        → name
    ├── tenantId    → tenantId
    ├── roles       → roles
    └── permissions → permissions
```

The `sub` and `tenantId` values are parsed as GUID values during this mapping.
An absent or invalid value maps to an undefined identifier; return valid GUID
strings for identifiers you expect authorization to consume.

This means your custom authentication provider should return identifiers in the
format expected by the application's Xeno.JS identity model.

---

## Build the application

A complete composition root can look like this:

```ts
import { XenoAppBuilder } from '@xeno-js/core'

const app = new XenoAppBuilder()

app.addAuth((options) => {
  options.authService = () => new MyAuthService()
  options.tokenExtractor = () => new MyTokenExtractor()
})

app.addMiddlewares()

const application = await app.build()
```

Authentication middleware must be enabled with `addMiddlewares()`.

Without the middleware module, the custom extractor is not used to authenticate
incoming HTTP requests. If middleware runs without an authentication service,
Xeno.JS assigns the guest identity; it does not enforce authentication by
default.

---

## Custom authentication with a different credential format

The custom extractor can support authentication formats other than:

```http
Authorization: Bearer <token>
```

For example:

```http
X-API-Token: <token>
```

The extractor can read that header:

```ts
export class ApiTokenExtractor implements IServiceExtractor<
  Request['headers'],
  string | undefined
> {
  extract(headers: Request['headers']): string | undefined {
    return headers.get('x-api-token') ?? undefined
  }
}
```

Register it:

```ts
app.addAuth((options) => {
  options.authService = () => new MyAuthService()
  options.tokenExtractor = () => new ApiTokenExtractor()
})
```

The authentication service then receives the value returned by `extract()`:

```text
X-API-Token
     │
     ▼
ApiTokenExtractor
     │
     ▼
credential
     │
     ▼
MyAuthService.authenticate()
     │
     ▼
AuthClaims
```

---

## Custom authentication with JWTs

If your provider issues JWTs, the custom authentication service can validate the
JWT and map its trusted claims to `AuthClaims`.

For example, conceptually:

```ts
import { Result } from '@xeno-js/shared'
import type { AuthClaims, ResultType } from '@xeno-js/shared'

async authenticate(token: string): Promise<ResultType<AuthClaims>> {
  const claims = await this.jwtVerifier.verify(token)

  return Result.ok({
    sub: claims.sub,
    email: claims.email,
    name: claims.name,
    tenantId: claims.tenantId,
    roles: claims.roles,
    permissions: claims.permissions,
  })
}
```

The provider-specific verification library is your responsibility. Verify the
signature and the expected issuer, audience, and expiry; decoding a JWT payload
alone does not authenticate it.

Xeno.JS consumes the resulting `AuthClaims`; it does not require the
authentication provider to use a particular token format.

---

## Custom authentication with an external identity service

The authentication service can also delegate validation to an external identity
provider.

For example:

```ts
import { Result } from '@xeno-js/shared'
import type { AuthClaims, ResultType } from '@xeno-js/shared'

async authenticate(token: string): Promise<ResultType<AuthClaims>> {
  const user = await this.identityProvider.validate(token)

  return Result.ok({
    sub: user.id,
    email: user.email,
    name: user.name,
    tenantId: user.tenantId,
    roles: user.roles,
    permissions: user.permissions,
  })
}
```

The important boundary remains:

```text
credential
    ↓
provider-specific validation
    ↓
AuthClaims
    ↓
Xeno.JS Identity
```

The provider can be a JWT issuer, API gateway, OAuth/OIDC service, internal
identity service, or another authentication system.

---

## Choose one authentication provider

Xeno.JS does not create a Supabase authentication service unless the Supabase
plugin is explicitly configured. The builder queues only one authentication
module: after one `addAuth()` call queues it, later calls to `addAuth()` are
ignored.

Configure one provider per application. Do not expect multiple authentication
services to participate in the same request.

For example, choose either the Supabase plugin:

```text
withSupabase(...)
    ↓
Supabase authentication service
```

or the custom configuration:

```text
addAuth(...)
    ↓
IExtendendAuthService implementation
```

The selected module provides the authentication service used by the
authentication flow.

---

## Custom token extraction takes precedence over built-in extractors

The authentication module registers a token extractor. If `tokenExtractor` is
omitted, Xeno.JS uses its default `BearerTokenExtractor`.

Default:

```text
BearerTokenExtractor
    → Authorization: Bearer <token>
    → cookie named by defaultCookieName (default: sb-access-token)
```

When `tokenExtractor` is configured:

```text
AuthModuleOptions.tokenExtractor
        ↓
custom token extractor
```

The configured extractor replaces the default. Supabase's SSR plugin supplies
its own SSR extractor when SSR is enabled; Xeno.JS core does not select an SSR
extractor based on an application mode.

This is important when using a custom provider with a non-standard header or
credential format.

---

## Do not put authentication validation in the extractor

Avoid this design:

```ts
class MyTokenExtractor {
  extract(headers: Request['headers']) {
    // Parse token
    // Verify token
    // Load user
    // Build identity
    // ...
  }
}
```

The extractor should retrieve the credential.

Prefer:

```text
Extractor
    ↓
credential
    ↓
Authentication service
    ↓
validated AuthClaims
```

This keeps transport-specific logic separate from provider-specific
authentication logic.

---

## Authentication errors

If the custom authentication service returns a failed `ResultType` from
`authenticate()` for an extracted token, Xeno.JS treats the request
authentication as failed.

The authentication middleware returns an error response using the error status,
or `401 Unauthorized` when the error has no status, instead of continuing to the
next middleware.

Therefore, your authentication service should return an appropriate failed
result when:

- the token is invalid;
- the token is expired;
- the token cannot be verified;
- the credential is revoked;
- the authentication provider rejects the credential.

Do not convert invalid credentials into successful claims.

---

## Missing credentials

A request may reach the authentication service without an explicit token.

The authentication flow can then use:

```ts
getUser()
```

to resolve the current authenticated user.

If `getUser()` succeeds with no user, or returns a failed result, the gatekeeper
returns the guest identity. The authentication middleware does not reject guest
identities by itself; configure authorization or route protection where
authenticated access is required.

This makes `getUser()` relevant for authentication mechanisms where the current
user can be resolved without passing an explicit token to `authenticate()`.

---

## Security responsibilities

A custom authentication implementation should keep these responsibilities
separate:

| Component              | Responsibility                                                 |
| ---------------------- | -------------------------------------------------------------- |
| Token extractor        | Retrieve the credential from the HTTP request                  |
| Authentication service | Validate the credential                                        |
| `AuthClaims`           | Represent the authenticated identity                           |
| Claims mapper          | Convert claims into Xeno.JS `Identity`                         |
| Authorization          | Decide whether the identity may execute an application request |

Authentication does not replace authorization.

For example, returning:

```ts
{
  sub: user.id,
  roles: ['admin'],
}
```

establishes identity and claims.

Whether that identity may execute a specific command is handled by the
application's authorization configuration.

See [Authorization Overview](../authorization/overview) and
[Authorization Policies](../authorization/policies).

---

## Troubleshooting

### The custom authentication service is not used

Check that `authService` is configured in the `addAuth()` setup action:

```ts
app.addAuth((options) => {
  options.authService = () => new MyAuthService()
})
```

The auth module requires `authService`; registering a class elsewhere does not
make the builder discover it automatically. Also check that another `addAuth()`
call did not queue the authentication module first, because subsequent calls are
ignored.

---

### The custom extractor is not called

Make sure:

```ts
app.addMiddlewares()
```

is configured.

Also verify that the custom extractor is returned by the factory:

```ts
tokenExtractor: () => new MyTokenExtractor()
```

The property expects a factory that receives the service scope and creates an
`IServiceExtractor`; it is not the extractor instance itself. The extractor
instance is singleton-scoped, so do not capture request-scoped services in its
factory.

---

### `authenticate()` is never called

Check the credential extraction first.

The flow is:

```text
HTTP headers
    ↓
extract()
    ↓
credential
    ↓
authenticate(token)
```

If the extractor returns `undefined`, the gatekeeper calls `getUser()` instead
of `authenticate()`. A failed `getUser()` result becomes a guest identity, not
an authentication error response.

---

### Authentication succeeds but authorization fails

Check the claims returned by your authentication service.

The following claims are propagated into the Xeno.JS identity:

```ts
{
  sub,
  email,
  name,
  tenantId,
  roles,
  permissions,
}
```

For example, if an authorization policy requires:

```ts
roles: ['admin']
```

your authenticated claims must contain the appropriate role.

Authentication establishes the identity; authorization evaluates whether that
identity satisfies the configured policy.

---

### The user ID or tenant ID is not available to authorization

Check `sub` and `tenantId` in the returned `AuthClaims`.

Xeno.JS maps:

```ts
sub      → userId
tenantId → tenantId
```

Both values are parsed as GUIDs during identity mapping.

If the values are not valid GUIDs, they will not produce the expected `Identity`
values.

---

## Related documentation

- [Authentication Overview](./overview)
- [Supabase Authentication](./supabase)
- [Authentication Middleware](../middleware/authentication)
- [Authentication Extractors](../middleware/extractors)
- [Authorization Overview](../authorization/overview)
- [Authorization Policies](../authorization/policies)

---

---

<a id="source-2"></a>

# Source: `docs/security/authentication/overview.md`

## Introduction

Authentication answers a fundamental security question:

> **Who is making this request?**

In Xeno.JS, the HTTP authentication middleware extracts a credential from the
incoming request, asks the configured authentication service to validate it, and
places the resulting identity in the request context. If there is no credential,
the gatekeeper can resolve the current user through `getUser()` or return a
guest identity.

The authentication flow is:

```text
HTTP credential
      │
      ▼
Token extractor
      │
      ▼
Authentication middleware
      │
      ▼
Authentication service
      │
      ▼
Auth claims
      │
      ▼
Request identity
      │
      ▼
Application execution
```

Authentication establishes **identity**. It does not decide whether that
identity is allowed to perform a particular application operation.

Authorization is responsible for that decision.

## Add Authentication

Authentication is configured through `XenoAppBuilder`.

A basic application configuration looks like this:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withSupabase } from '@xeno-js/supabase'

const app = new XenoAppBuilder()

app.addAuth(
  withSupabase((options, config) => {
    options.url = config.get('SUPABASE_URL')!
    options.key = config.get('SUPABASE_KEY')!
  }),
)

app.addMiddlewares()

const application = await app.build()
```

There are two important pieces here:

- `withSupabase()` configures the Supabase authentication plugin, and
  `addAuth()` queues it with the builder.
- `addMiddlewares()` enables the HTTP middleware that extracts the request
  credential and authenticates the request.

Without the authentication middleware, configuring an authentication service
does not by itself authenticate incoming HTTP requests.

## How Authentication Works

Xeno.JS separates two responsibilities that are often combined in application
code.

### 1. Credential extraction

The **token extractor** answers:

> Where does the credential come from?

For example, an HTTP authentication credential may come from:

```http
Authorization: Bearer <access-token>
```

or from a cookie, depending on the selected authentication integration.

The extractor is responsible for retrieving the credential from the transport.

It does not decide whether the credential is valid.

### 2. Credential validation

The **authentication service** answers:

> Is this credential valid, and which identity does it represent?

The authentication service validates the credential and returns authentication
claims.

Conceptually:

```text
Request
   │
   ├── extractor ──────► credential
   │
   └── auth service ◄─── credential
             │
             ▼
         AuthClaims
```

Keeping these responsibilities separate allows the same authentication model to
work with different credential formats and authentication providers.

## Authentication Claims

A successful authentication produces claims describing the authenticated
identity.

The `AuthClaims` contract requires `sub` and defines the following fields:

```ts
import type { AuthClaims } from '@xeno-js/shared'

const claims: AuthClaims = {
  sub: '550e8400-e29b-41d4-a716-446655440000',
  email: 'user@example.com',
  name: 'Jane Doe',
  tenantId: '550e8400-e29b-41d4-a716-446655440001',
  roles: ['admin'],
  permissions: ['orders:read'],
}
```

`email`, `name`, `tenantId`, `roles`, and `permissions` can each have an
`undefined` value. Xeno.JS maps `sub` to `Identity.userId` and `tenantId` to
`Identity.tenantId`; both values are parsed as GUIDs, so provide valid GUID
strings when the application needs those identifiers.

Xeno.JS then maps the authentication claims to the request identity used by the
application.

This identity is what authorization strategies consume later in the request
lifecycle.

```text
Authentication
      │
      ▼
Authentication claims
      │
      ▼
Request identity
      │
      ▼
Configured authorization (if any)
```

## Configure Authentication with Supabase

The `@xeno-js/supabase` package provides a Supabase authentication integration.

The minimal configuration is:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withSupabase } from '@xeno-js/supabase'

const app = new XenoAppBuilder()

app.addAuth(
  withSupabase((options, config) => {
    options.url = config.get('SUPABASE_URL')!
    options.key = config.get('SUPABASE_KEY')!
  }),
)

app.addMiddlewares()

const application = await app.build()
```

The Supabase integration reads `SUPABASE_URL` and `SUPABASE_KEY` by default. It
requires both values and selects the standard client unless you define its `ssr`
options. `redirectTo`, when set, is passed to Supabase's OAuth sign-in
operation.

For Supabase-specific configuration, including client options, SSR cookies, and
token extraction, see:

**[Supabase Authentication](./supabase)**

## Use a Custom Authentication Service

You do not have to use the Supabase integration.

Xeno.JS supports a custom authentication service when your application uses
another identity provider or has its own authentication mechanism.

Configure `authService` and, optionally, `tokenExtractor` through `addAuth()`:

```ts
import { XenoAppBuilder } from '@xeno-js/core'

const app = new XenoAppBuilder()

app.addAuth((options) => {
  options.authService = () => new MyAuthService()
  options.tokenExtractor = () => new MyTokenExtractor()
})

app.addMiddlewares()

const application = await app.build()
```

The two components have different responsibilities:

```text
MyTokenExtractor
      │
      └── extracts the credential

MyAuthService
      │
      └── validates the credential
          and returns authentication claims
```

The custom authentication service implements the Xeno.JS authentication
contract.

At its base, the authentication service provides operations such as:

```ts
import type { AuthClaims, Maybe, ResultType } from '@xeno-js/shared'

interface IBaseAuthService {
  isAuthenticated(): Promise<boolean>

  getUser(): Promise<ResultType<Maybe<AuthClaims>>>

  authenticate(token: string): Promise<ResultType<AuthClaims>>
}
```

The configured service must implement `IExtendendAuthService`, which also
includes `signInWithProvider()`, `getSession()`, `signOut()`, and
`exchangeCodeForSession()`. When the extractor returns no credential, Xeno.JS
calls `getUser()`; `isAuthenticated()` is part of the contract but is not called
by the HTTP authentication middleware.

The custom service factory receives the active service scope, and the service is
scoped to the request. The optional token extractor is registered as a singleton
and should not capture request-scoped services.

For the complete custom provider implementation, see:

**[Custom Authentication](./custom)**

## Authentication Middleware

Authentication is applied at the HTTP boundary by Xeno.JS middleware.

The resulting flow is:

```text
HTTP request
      │
      ▼
Credential extraction
      │
      ▼
Authentication
      │
      ▼
Authenticated identity
      │
      ▼
Request context
      │
      ▼
Application execution
```

When the configured service authenticates a supplied credential successfully,
Xeno.JS maps the returned claims to an identity and makes it available through
the request context.

When the service returns a failed result for a supplied credential, the
middleware returns an error response and does not continue to the next
middleware. It uses the error's status when provided, or `401 Unauthorized`
otherwise.

When no credential is extracted, Xeno.JS calls `getUser()`. If that call fails
or returns no user, the gatekeeper sets the guest identity and the middleware
continues. The middleware therefore does not, by itself, require every request
to be authenticated; use route protection or authorization rules where
authentication is mandatory.

This makes authentication available to the application without requiring
individual commands or services to manually parse HTTP headers or authentication
cookies.

## Authentication and Authorization

Authentication and authorization solve different problems.

### Authentication

Authentication answers:

> **Who are you?**

```text
credential
    ↓
authenticated identity
```

### Authorization

Authorization answers:

> **Are you allowed to perform this operation?**

```text
authenticated identity
    ↓
authorization policy
    ↓
allowed / denied
```

For example, authentication may establish:

```ts
{
  sub: '550e8400-e29b-41d4-a716-446655440000',
  tenantId: '550e8400-e29b-41d4-a716-446655440001',
  roles: ['operator']
}
```

Authorization can then decide whether that identity may execute a specific
command.

Do not put application authorization rules into the authentication service.

Authentication establishes identity. Authorization evaluates access.

See **[Authorization](../authorization/overview)** for application-level access
control.

## Authentication and the Request Context

Once the gatekeeper resolves an identity, Xeno.JS associates it with the current
request context. This includes the guest identity when no authenticated user is
found.

This allows application code to work with the authenticated identity without
depending directly on the HTTP transport.

In the default Node context, `AsyncLocalStorage` carries the `RequestContext`
across asynchronous work. Authentication services are scoped to the active
service scope; avoid retaining request-scoped services in singletons.

The architectural flow is therefore:

```text
HTTP
 │
 │ authentication
 ▼
Request Context
 │
 │ identity
 ▼
Application
 │
 │ authorization
 ▼
Use Case
```

This is particularly useful for CQRS handlers and application services because
they can consume identity information without directly depending on `Request`,
`Response`, or HTTP headers.

## When Should You Use Custom Authentication?

Use the `@xeno-js/supabase` integration when Supabase is responsible for
authenticating your application users.

Use a custom authentication service when your application needs to integrate
with another authentication provider or an existing authentication system.

Typical examples include:

- an enterprise identity provider;
- an internal authentication service;
- a custom JWT issuer;
- an API gateway that provides application credentials;
- an existing authentication system that cannot be replaced.

The important requirement is that the custom implementation can turn the
incoming credential into the authentication claims required by the application.

## Authentication Is Not Authorization

Do not assume that an authenticated request is automatically authorized to
perform every operation.

For example, an authenticated identity might contain:

```ts
{
  sub: '550e8400-e29b-41d4-a716-446655440000',
  tenantId: '550e8400-e29b-41d4-a716-446655440001',
  roles: ['user']
}
```

This only establishes who the caller is.

An application can still require:

```text
role = admin
```

or:

```text
permission = orders:create
```

or a more specific business rule such as:

```text
the authenticated user owns this order
```

Those checks belong to authorization.

Xeno.JS provides built-in authorization strategies for common identity and
access checks and allows custom authorization strategies for
application-specific rules.

## Recommended Authentication Configuration

For a typical HTTP application, configure authentication together with the
middleware layer:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withSupabase } from '@xeno-js/supabase'

const app = new XenoAppBuilder()

app.addAuth(
  withSupabase((options, config) => {
    options.url = config.get('SUPABASE_URL')!
    options.key = config.get('SUPABASE_KEY')!
  }),
)

app.addMiddlewares()

app.addPipeline()

const application = await app.build()
```

This gives the application the following security flow:

```text
HTTP request
     │
     ▼
Authentication middleware
     │
     ▼
Authenticated identity
     │
     ▼
Application pipeline
     │
     ▼
Authorization
     │
     ▼
Command / Query handler
```

The exact middleware configuration depends on the application's transport and
security requirements. Authentication can be combined with CSRF protection, CORS
controls, allowed HTTP methods, and rate limiting.

See the **[HTTP Security Middleware](../middleware/overview)** documentation for
those controls.

## Common Configuration Mistakes

### Authentication is configured but requests are not authenticated

Make sure the middleware layer is enabled:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withSupabase } from '@xeno-js/supabase'

const app = new XenoAppBuilder()

app.addAuth(
  withSupabase((options, config) => {
    options.url = config.get('SUPABASE_URL')!
    options.key = config.get('SUPABASE_KEY')!
  }),
)
app.addMiddlewares()
await app.build()
```

`addAuth()` configures the authentication service; `addMiddlewares()` enables
the HTTP middleware that uses it. A missing credential may result in the guest
identity rather than an error response, so ensure the relevant route or
authorization policy requires an authenticated identity.

### The application parses the Authorization header inside handlers

Do not make application handlers responsible for extracting HTTP credentials.

Keep transport-specific credential extraction in the authentication middleware
and expose the resulting identity through the request context.

This keeps the application layer independent from HTTP.

### Authentication logic contains business authorization rules

Avoid implementing rules such as:

```text
is this user an administrator?
does this user own this resource?
does this user belong to this tenant?
```

inside the authentication service.

The authentication service should establish the identity.

Use authorization policies and custom authorization strategies for access
decisions.

### A custom provider only implements token extraction

A token extractor only retrieves the credential.

It does not validate the credential or establish the authenticated identity.

A custom authentication integration normally requires both:

```text
credential extractor
        +
authentication service
```

## Next Steps

Once authentication is configured, continue with the provider-specific or
application-specific integration:

- **[Supabase Authentication](./supabase)** — configure the Supabase
  integration, cookies, SSR, and token extraction.
- **[Custom Authentication](./custom)** — implement your own authentication
  service and credential extractor.
- **[Authorization](../authorization/overview)** — use the authenticated
  identity to control access to application operations.
- **[HTTP Security Middleware](../middleware/overview)** — configure
  authentication together with CSRF, CORS, method restrictions, and rate
  limiting.

---

---

<a id="source-3"></a>

# Source: `docs/security/authentication/supabase.md`

## How do I configure Supabase authentication with Xeno.JS?

Xeno.JS provides a built-in Supabase authentication integration.

The integration configures the Supabase authentication service and connects it
to Xeno.JS request authentication. HTTP middleware extracts the credential from
the request, the authentication service validates it, and the resulting claims
are mapped to the request identity.

The basic setup is:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withSupabase } from '@xeno-js/supabase'

const app = new XenoAppBuilder()

app.addAuth(
  withSupabase((options, config) => {
    options.url = config.get('SUPABASE_URL')!
    options.key = config.get('SUPABASE_KEY')!
  }),
)

app.addMiddlewares()

await app.build()
```

`withSupabase()` creates the Supabase authentication plugin. `addAuth()` queues
that plugin with the builder; the selected CSR or SSR module registers the
scoped authentication service when `build()` resolves the queued modules.

`addMiddlewares()` enables the HTTP middleware that extracts the credential and
establishes the authenticated request identity. It is required for
authentication of incoming HTTP requests.

For a general explanation of the authentication model, see
[Authentication Overview](./overview).

---

## Prerequisites

You need:

- `@xeno-js/core` and `@xeno-js/supabase`;
- `@supabase/ssr` when using SSR mode;
- a Supabase project;
- the Supabase project URL;
- a Supabase key appropriate to the application runtime;
- an application configuration source for these values;
- Xeno.JS authentication and middleware enabled.

For SSR applications, you also need to configure the SSR-specific authentication
options described below.

Install `@xeno-js/core`, `@xeno-js/supabase`, and `@supabase/supabase-js`. Add
`@supabase/ssr` when using SSR mode. Keep keys in application configuration or
environment variables rather than hard-coding them. Do not expose a privileged
service-role key to an untrusted client.

---

## Configure the Supabase project

At minimum, configure:

- `SUPABASE_URL` — the Supabase project URL;
- `SUPABASE_KEY` — the key used by the Xeno.JS Supabase integration.

Example:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withSupabase } from '@xeno-js/supabase'

const app = new XenoAppBuilder()

app.addAuth(
  withSupabase((options, config) => {
    options.url = config.get('SUPABASE_URL')!
    options.key = config.get('SUPABASE_KEY')!
  }),
)

app.addMiddlewares()

await app.build()
```

`withSupabase()` reads `SUPABASE_URL` and `SUPABASE_KEY` by default, and throws
during builder configuration if either value is missing. The values create the
Supabase client; `ssr` selects the server-side client while omitting it selects
the standard client.

---

## Configure the application URL

If Supabase OAuth sign-in needs a redirect URL, configure `redirectTo`.

For example:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withSupabase } from '@xeno-js/supabase'

const app = new XenoAppBuilder()

app.addAuth(
  withSupabase((options, config) => {
    options.url = config.get('SUPABASE_URL')!
    options.key = config.get('SUPABASE_KEY')!
    options.redirectTo = config.get('SUPABASE_REDIRECT_TO')
  }),
)
```

A typical configuration uses:

```text
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_KEY=your-supabase-key
SUPABASE_REDIRECT_TO=https://app.example.com/auth/callback
```

`redirectTo` defaults to `SUPABASE_REDIRECT_TO` and is passed to Supabase when
`signInWithProvider()` starts an OAuth sign-in. It is not a general application
URL setting.

---

## Configure authentication cookies

Xeno.JS supports cookie-based authentication in addition to bearer-token
authentication.

Cookie behavior has separate settings for the standard client and SSR mode.

For example:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withSupabase } from '@xeno-js/supabase'

const app = new XenoAppBuilder()

app.addAuth(
  withSupabase((options, config) => {
    options.url = config.get('SUPABASE_URL')!
    options.key = config.get('SUPABASE_KEY')!

    options.ssr = {
      cookieOptions: {
        path: '/',
        httpOnly: true,
        sameSite: 'lax',
        secure: true,
      },
    }
  }),
)
```

In SSR mode, `ssr.cookieOptions` configures cookie attributes. Xeno.JS defaults
to `path: '/'`, `httpOnly: true`, `sameSite: 'lax'`, and `secure: true`. In CSR
mode, use `csr.cookieName` to configure the cookie name that the request
extractor checks; it does not configure Supabase client storage.

Use secure cookie settings in production. In particular, `secure: true` marks
cookies for HTTPS-only transport. For local HTTP development, override it with
`false`; do not disable it in production.

---

## How Xeno.JS extracts the authentication credential

Supabase authentication can reach Xeno.JS through a Bearer authorization header
or, depending on the selected mode, a cookie.

### Bearer token

The standard authentication extractor looks for:

```http
Authorization: Bearer <access-token>
```

For example:

```http
GET /api/orders HTTP/1.1
Host: api.example.com
Authorization: Bearer <access-token>
```

The extracted token is passed to the configured authentication service.

The authentication service is responsible for validating the credential and
producing authentication claims.

---

## Cookie-based authentication

In CSR mode, the standard extractor can also fall back to a cookie named by
`csr.cookieName`.

The default cookie name is:

```text
sb-access-token
```

Therefore, an authenticated request may carry its credential through a cookie
rather than the `Authorization` header.

The cookie value in CSR mode is treated as the access token. This setting
configures the request extractor; it does not change the Supabase client's
storage settings.

The important distinction is:

```text
HTTP request
    │
    ├── Authorization: Bearer <access-token>
    │
    └── authentication cookie
             │
             ▼
       token extraction
             │
             ▼
       authentication service
             │
             ▼
          AuthClaims
             │
             ▼
        request Identity
```

The extractor determines **where the credential comes from**. In SSR mode,
`ssr.cookiePrefix` configures the cookie name prefix used by the SSR extractor;
it defaults to `sb-access-token`, and numbered chunks use names such as
`sb-access-token.0`.

The authentication service determines **whether the credential is valid and
which claims it represents**.

---

## Supabase SSR authentication

For server-side rendering, Xeno.JS provides a dedicated Supabase SSR
integration.

The server-side integration requires `@supabase/ssr` and creates the Supabase
server client with cookie access through the active Xeno.JS service scope.

The SSR extractor can read:

- an `Authorization` bearer token;
- the cookie name configured by `ssr.cookiePrefix` (default: `sb-access-token`);
- chunked Supabase authentication cookies.

This matters because Supabase SSR sessions may be represented by cookies whose
data is split across multiple cookie values.

The SSR extractor checks the bearer header first, then reads the matching
cookie. It joins numbered chunks and extracts `access_token` from the cookie
payload. Set `ssr.cookiePrefix` to the cookie name used by your Supabase SSR
client; the default is `sb-access-token`.

Do not treat the standard bearer-token extractor and the SSR extractor as
interchangeable implementation details. Choose the authentication configuration
according to how your application receives the Supabase credential.

---

## Configure Supabase SSR

A complete SSR-oriented configuration can include:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withSupabase } from '@xeno-js/supabase'

const app = new XenoAppBuilder()

app.addAuth(
  withSupabase((options, config) => {
    options.url = config.get('SUPABASE_URL')!
    options.key = config.get('SUPABASE_KEY')!
    options.redirectTo = config.get('SUPABASE_REDIRECT_TO')
    options.ssr = {
      cookiePrefix: 'sb-access-token',
      cookieEncoding: 'base64url',
      cookieOptions: {
        path: '/',
        httpOnly: true,
        sameSite: 'lax',
        secure: true,
      },
    }
  }),
)

app.addMiddlewares()

await app.build()
```

Defining `options.ssr` selects the SSR module. The Supabase integration also
exposes these mode-specific settings:

- `ssr.options` — options forwarded to the Supabase SSR client;
- `ssr.cookieEncoding` — cookie encoding (`raw` or `base64url`);
- `ssr.cookieHandler` — a custom request cookie handler; if omitted, Xeno.JS
  resolves its default handler from the active service scope;
- `ssr.cookieOptions` — cookie attributes used by the SSR integration;
- `ssr.cookiePrefix` — cookie name or prefix read by the SSR token extractor;
- `csr.options` — options forwarded to the standard Supabase client;
- `csr.cookieName` — cookie name checked by the standard token extractor.

Authentication services and the default cookie handler are scoped to the active
service scope. Do not retain these request-specific instances in a singleton. In
the default Node context, Xeno.JS propagates request context through
`AsyncLocalStorage`; SSR cookie access therefore belongs to the current request,
not a shared singleton.

---

## How SSR cookie handling works

The Xeno.JS Supabase server integration creates the Supabase server client
through `@supabase/ssr`.

The important application-level behavior is:

```text
HTTP request
    │
    ├── Authorization header
    │
    └── Supabase SSR cookies
             │
             ▼
       SSR token extraction
             │
             ▼
       Supabase authentication
             │
             ▼
          AuthClaims
             │
             ▼
        request Identity
```

The default handler reads cookies from the current network context and buffers
cookie updates for the HTTP response path. A custom `ssr.cookieHandler` can
replace it when the application supplies its own request/response cookie
integration.

This allows server-side authentication state to participate in the same Xeno.JS
authentication flow as bearer-token authentication.

For the underlying Supabase server-side authentication model, consult the
current
[Supabase SSR documentation](https://supabase.com/docs/guides/auth/server-side/creating-a-client).

---

## Enable authentication middleware

Configuring `addAuth()` does not by itself make an HTTP request authenticated.

The HTTP middleware must also be enabled:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withSupabase } from '@xeno-js/supabase'

const app = new XenoAppBuilder()

app.addAuth(
  withSupabase((options, config) => {
    options.url = config.get('SUPABASE_URL')!
    options.key = config.get('SUPABASE_KEY')!
  }),
)

app.addMiddlewares()

await app.build()
```

The authentication middleware connects the incoming HTTP credential to the
configured authentication service.

The resulting identity is stored in the request context and can then be consumed
by application-level authorization. In the default Node runtime,
`AsyncLocalStorage` carries the `RequestContext` across asynchronous work.

The overall flow is:

```text
HTTP credential
      │
      ▼
Token extractor
      │
      ▼
Authentication middleware
      │
      ▼
Supabase authentication service
      │
      ▼
AuthClaims
      │
      ▼
ClaimsIdentityMapper
      │
      ▼
Request Identity
```

Authentication establishes **who the caller is**.

Authorization determines **whether that identity is allowed to execute a
specific application request**.

---

## Complete configuration example

A typical SSR application composition root can look like this:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withSupabase } from '@xeno-js/supabase'

const app = new XenoAppBuilder()

app.addAuth(
  withSupabase((options, config) => {
    options.url = config.get('SUPABASE_URL')!
    options.key = config.get('SUPABASE_KEY')!
    options.redirectTo = config.get('SUPABASE_REDIRECT_TO')
    options.ssr = {
      cookiePrefix: 'sb-access-token',
      cookieEncoding: 'base64url',
      cookieOptions: {
        path: '/',
        httpOnly: true,
        sameSite: 'lax',
        secure: true,
      },
    }
  }),
)

app.addMiddlewares()

await app.build()
```

The responsibilities are intentionally separated:

| Configuration               | Responsibility                                      |
| --------------------------- | --------------------------------------------------- |
| `options.url`               | Supabase project URL                                |
| `options.key`               | Supabase authentication key                         |
| `options.redirectTo`        | OAuth redirect destination                          |
| `options.ssr.cookieOptions` | SSR cookie attributes                               |
| `options.ssr.cookiePrefix`  | SSR cookie name or prefix for extraction            |
| `options.csr.cookieName`    | CSR fallback cookie name                            |
| `app.addMiddlewares()`      | Enables HTTP authentication processing              |
| `app.build()`               | Resolves queued modules and returns the application |

---

## Do not trust client-side session state as identity

A session object stored or supplied by the client should not be treated as proof
of authentication by application code.

The server-side authentication flow must establish the identity:

```text
credential
    │
    ▼
authentication service
    │
    ▼
validated AuthClaims
    │
    ▼
request Identity
```

Application authorization should consume the resulting request identity rather
than trusting arbitrary identity data supplied in the request payload or
client-side state.

This distinction is especially important when protecting commands, queries, or
resources belonging to a specific user or tenant.

---

## When should I use bearer authentication?

Bearer authentication is appropriate when the client explicitly sends an access
token:

```http
Authorization: Bearer <access-token>
```

This is common for:

- APIs;
- service-to-service HTTP clients;
- clients that already manage access tokens;
- non-browser clients.

In this case, Xeno.JS extracts the bearer token and passes it to the
authentication service.

---

## When should I use SSR cookies?

Use the SSR integration when the application relies on Supabase's server-side
cookie-based session model.

This is particularly relevant for applications where:

- authentication state is stored in cookies;
- server-side rendering needs access to the authenticated session;
- the request and response participate in cookie management;
- Supabase SSR cookies may be chunked.

The SSR integration handles these cookies through the server-side Supabase
client.

---

## Troubleshooting

### Authentication is not running

Make sure both authentication and middleware are configured:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withSupabase } from '@xeno-js/supabase'

const app = new XenoAppBuilder()

app.addAuth(
  withSupabase((options, config) => {
    options.url = config.get('SUPABASE_URL')!
    options.key = config.get('SUPABASE_KEY')!
  }),
)

app.addMiddlewares()

await app.build()
```

Configuring `addAuth()` without enabling the middleware does not establish
authentication for incoming HTTP requests.

---

### The bearer token is not detected

Verify that the request contains the expected header:

```http
Authorization: Bearer <access-token>
```

Also verify that the token is sent to the Xeno.JS HTTP execution path so that
the authentication middleware can process the request.

---

### Cookie authentication is not detected

Check:

1. the authentication cookie is present in the request;
2. in CSR mode, the cookie name matches `csr.cookieName`; in SSR mode, it
   matches `ssr.cookiePrefix`;
3. the cookie is available to the request path and domain;
4. HTTPS is used when secure cookies are enabled;
5. SSR applications use the SSR authentication configuration when the session is
   managed through Supabase SSR cookies.

The default CSR cookie name and SSR cookie prefix are:

```text
sb-access-token
```

---

### SSR authentication does not see the session

For SSR applications, verify that:

- the request contains the Supabase authentication cookies;
- the server-side Supabase integration is configured;
- the application is using the SSR extractor when required;
- the request cookie names match `ssr.cookiePrefix`;
- the SSR cookie handler is resolved in the active request scope.

If the Supabase session uses chunked cookies, make sure the complete cookie set
reaches the server.

---

## Related documentation

- [Authentication Overview](./overview)
- [Custom Authentication](./custom)
- [Authentication Middleware](../middleware/authentication)
- [Authentication Extractors](../middleware/extractors)
- [Authorization Overview](../authorization/overview)
- [Authorization Policies](../authorization/policies)
- [Supabase SSR documentation](https://supabase.com/docs/guides/auth/server-side/creating-a-client)

---

---

<a id="source-4"></a>

# Source: `docs/security/authorization/overview.md`

## How do I protect a command or query with authorization?

Xeno.JS provides **intent-based authorization** for CQRS requests.

You define an authorization policy for a request intent and Xeno.JS evaluates
the policy against the authenticated request identity before the request reaches
the next pipeline stage.

The built-in policy dimensions are:

- `userId`
- `tenantId`
- `roles`
- `permissions`

You can also add custom authorization strategies for application-specific rules.

The basic flow is:

```text
Command or Query
      │
      ▼
AuthorizationPipeline
      │
      ├── user identity check
      ├── tenant identity check
      ├── role check
      ├── permission check
      └── custom strategies
      │
      ▼
Next pipeline stage
      │
      ▼
Handler
```

Authorization is an **application-layer** concern. HTTP authentication
establishes the request identity; authorization decides whether that identity
may execute the application request.

---

## Enable authorization

Authorization is opt-in.

Configure it by passing the `withAuthZ()` plugin to `AppBuilder.addPipeline()`:

```ts
const app = new XenoAppBuilder()

app.addPipeline(
  withAuthZ((opts) => {
    opts.policies = {
      CreateInvoice: {
        userId: true,
        tenantId: true,
        roles: ['admin', 'billing'],
        permissions: ['invoice:create'],
      },
    }
  }),
)

const container = await app.build()
```

When authorization policies or custom authorization strategies are configured,
Xeno.JS adds an `AuthorizationPipeline` to the CQRS pipelines.

The same authorization pipeline is used for:

- commands;
- queries.

You do not configure separate authorization policies for the command bus and
query bus.

---

## Configure a policy

A policy is associated with a request intent:

```ts
app.addPipeline(
  withAuthZ((opts) => {
    opts.policies = {
      CreateInvoice: {
        userId: true,
        tenantId: true,
        roles: ['admin', 'billing'],
        permissions: ['invoice:create'],
      },
    }
  }),
)
```

The policy describes which identity requirements must be satisfied when the
`CreateInvoice` intent is executed.

Conceptually:

```text
CreateInvoice
    │
    ├── userId
    ├── tenantId
    ├── roles
    └── permissions
```

The policy registry normalizes intent names to lowercase.

Therefore these intent names resolve to the same policy:

```text
CreateInvoice
createinvoice
CREATEINVOICE
```

The request intent must still identify the same application operation.

---

## Policy dimensions

Xeno.JS supports four built-in policy dimensions.

| Property      | Checks                                             |
| ------------- | -------------------------------------------------- |
| `userId`      | Authenticated identity contains a valid user ID    |
| `tenantId`    | Authenticated identity contains a valid tenant ID  |
| `roles`       | Identity contains at least one required role       |
| `permissions` | Identity contains at least one required permission |

The checks are independent strategies.

If a policy contains multiple dimensions, every applicable strategy must
succeed.

For example:

```ts
CreateInvoice: {
  userId: true,
  tenantId: true,
  roles: ['billing'],
  permissions: ['invoice:create'],
}
```

means:

```text
valid user ID
    AND
valid tenant ID
    AND
billing role
    AND
invoice:create permission
```

If any strategy fails, the authorization pipeline stops and the request does not
continue.

---

## User authorization

### Require an authenticated user identity

Use:

```ts
CreateProfile: {
  userId: true,
}
```

This requires the authenticated request identity to contain a valid GUID user
ID.

The check is performed against:

```ts
context.identity.userId
```

It does **not** compare that value with an ID contained in the command or query.

For example:

```ts
CreateProfile: {
  userId: true,
}
```

means:

> The authenticated identity must contain a valid user ID.

It does **not** mean:

> The authenticated user must match `request.userId`.

If the application needs ownership authorization, such as:

```text
currentUser.id === profile.ownerId
```

that is an application-specific rule and should be implemented with a custom
authorization strategy or application logic.

---

## Tenant authorization

### Require an authenticated tenant identity

Use:

```ts
CreateInvoice: {
  tenantId: true,
}
```

This requires the authenticated request identity to contain a valid GUID tenant
ID.

The check is performed against:

```ts
context.identity.tenantId
```

It does **not** automatically compare the identity tenant ID with a tenant ID
contained in:

- the command;
- the query;
- a route parameter;
- a database entity.

For example:

```ts
CreateInvoice: {
  tenantId: true,
}
```

means:

> The authenticated identity must contain a valid tenant ID.

It does not by itself enforce:

```text
identity.tenantId === invoice.tenantId
```

Resource-level tenant isolation remains an application-specific rule.

This distinction is important for multi-tenant applications.

---

## Role authorization

### Require one of several roles

Configure:

```ts
DeleteUser: {
  roles: ['admin', 'support'],
}
```

The role strategy succeeds when the authenticated identity contains **at least
one** of the configured roles.

Therefore:

```text
roles: ['admin', 'support']
```

means:

```text
admin OR support
```

not:

```text
admin AND support
```

For example, an identity containing:

```ts
{
  roles: ['support'],
}
```

satisfies:

```ts
roles: ['admin', 'support']
```

An identity containing:

```ts
{
  roles: ['viewer'],
}
```

does not.

If the required role is missing, authorization fails with a forbidden result.

---

## Permission authorization

### Require one of several permissions

Configure:

```ts
CreateInvoice: {
  permissions: ['invoice:create', 'invoice:admin'],
}
```

The permission strategy succeeds when the authenticated identity contains **at
least one** configured permission.

Therefore:

```text
permissions: ['invoice:create', 'invoice:admin']
```

means:

```text
invoice:create OR invoice:admin
```

For example:

```ts
{
  permissions: ['invoice:create'],
}
```

satisfies the policy.

An identity containing only:

```ts
{
  permissions: ['invoice:read'],
}
```

does not.

Missing required permissions produce a forbidden authorization result.

---

## Combine authorization requirements

You can combine multiple policy dimensions:

```ts
app.addPipeline(
  withAuthZ((opts) => {
    opts.policies = {
      CreateInvoice: {
        userId: true,
        tenantId: true,
        roles: ['admin', 'billing'],
        permissions: ['invoice:create'],
      },

      ReadInvoice: {
        tenantId: true,
        permissions: ['invoice:read', 'invoice:admin'],
      },
    }
  }),
)
```

The semantics are:

```text
CreateInvoice

valid user ID
    AND
valid tenant ID
    AND
(admin OR billing)
    AND
(invoice:create)
```

and:

```text
ReadInvoice

valid tenant ID
    AND
(invoice:read OR invoice:admin)
```

This gives you a simple model:

- **different policy dimensions** are combined with AND semantics;
- **multiple roles** use OR semantics;
- **multiple permissions** use OR semantics.

---

## What happens when a policy is not defined?

The authorization strategies are created from the policy dimensions found in the
configured policy registry.

At execution time, each built-in strategy looks up the policy for the current
request intent.

If that policy does not define the dimension handled by the strategy, that
strategy succeeds without applying a restriction.

An intent with no matching policy requirements therefore continues without a
built-in policy restriction (provided a request context is available).
Authorization policies do not deny requests by default.

For example:

```ts
app.addPipeline(
  withAuthZ((opts) => {
    opts.policies = {
      CreateInvoice: {
        roles: ['billing'],
      },

      ReadInvoice: {
        permissions: ['invoice:read'],
      },
    }
  }),
)
```

The role strategy applies to `CreateInvoice`.

The permission strategy applies to `ReadInvoice`.

A role strategy does not reject `ReadInvoice` merely because `ReadInvoice` has
no `roles` configuration.

---

## Authorization and authentication

Authorization depends on the request identity.

The built-in authorization strategies obtain the identity from the request
context.

The flow is:

```text
Authentication
      │
      ▼
Request Identity
      │
      ▼
Authorization
      │
      ▼
Application request
```

If the request context is unavailable, the built-in authorization strategies
return an unauthorized result.

This means authentication and authorization have different responsibilities:

| Concern        | Responsibility                                       |
| -------------- | ---------------------------------------------------- |
| Authentication | Establish who the caller is                          |
| Authorization  | Decide whether that identity may execute the request |

For HTTP applications, authentication normally happens at the HTTP boundary
before the CQRS request reaches the authorization pipeline.

---

## Authorization pipeline

The authorization pipeline executes its configured strategies sequentially.

Conceptually:

```text
Request
  │
  ▼
Strategy 1
  │
  ├── fail → stop
  │
  ▼
Strategy 2
  │
  ├── fail → stop
  │
  ▼
Strategy 3
  │
  ├── fail → stop
  │
  ▼
next()
```

If a strategy fails, the pipeline immediately returns the failure.

If every strategy succeeds, the request continues to the next pipeline behavior.

This means authorization is fail-fast.

---

## Authorization order in the CQRS pipeline

The authorization pipeline follows the exception, logging, and performance
pipelines. Command idempotency, when enabled, precedes authorization; validation
and other configured behaviors follow it.

The effective high-level order is:

```text
Exception
   ↓
Logging
   ↓
Performance
   ↓
Idempotency (commands, when enabled)
   ↓
Authorization
   ↓
Validation
   ↓
Other command / query behaviors
   ↓
Handler
```

Idempotency applies to commands when configured. Authorization and validation
are common behaviors for both command and query pipelines; other behaviors
depend on the configured plugins.

As a result, authorization can reject a request before application validation is
executed.

This is useful when the request should not proceed into later application
processing unless the caller is authorized.

---

## Configure authorization with validation

Authorization and validation are separate concerns.

For example:

```ts
const app = new XenoAppBuilder()

app.addPipeline([
  withAuthZ((opts) => {
    opts.policies = {
      CreateInvoice: {
        tenantId: true,
        permissions: ['invoice:create'],
      },
    }
  }),
  withZodValidation((opts) => {
    opts.schemas = {
      CreateInvoice: CreateInvoiceSchema,
    }
  }),
])

const container = await app.build()
```

The responsibilities remain separate:

```text
Authorization
    =
    "May this identity execute CreateInvoice?"

Validation
    =
    "Is this CreateInvoice request structurally valid?"
```

Do not use authorization policies as a replacement for input validation.

---

## Authorization for queries

Authorization is not limited to commands.

Because the configured authorization pipeline is added to both the command and
query pipelines, the same policy mechanism can protect queries.

For example:

```ts
app.addPipeline(
  withAuthZ((opts) => {
    opts.policies = {
      GetInvoice: {
        tenantId: true,
        permissions: ['invoice:read'],
      },

      DeleteInvoice: {
        tenantId: true,
        permissions: ['invoice:delete'],
      },
    }
  }),
)
```

This allows read and write operations to have independent authorization
requirements.

---

## Resource ownership is not automatic

A common mistake is assuming that:

```ts
GetInvoice: {
  userId: true,
}
```

automatically prevents one user from reading another user's invoice.

It does not.

The built-in `userId` strategy verifies that the authenticated identity contains
a valid user ID.

It does not load the invoice and compare:

```text
invoice.ownerId
```

with:

```text
identity.userId
```

Likewise:

```ts
GetInvoice: {
  tenantId: true,
}
```

does not automatically compare an invoice's tenant with the authenticated
tenant.

For resource ownership or resource-level tenant isolation, use an
application-specific authorization strategy.

---

## Custom authorization strategies

Built-in policies cover common identity-based authorization requirements.

For application-specific rules, configure custom authorization strategies:

```ts
app.addPipeline(
  withAuthZ((opts) => {
    opts.customAuthorizationStrategy = [
      (container) => {
        return new InvoiceOwnershipStrategy(
          container.resolve(INVOICE_REPOSITORY),
          container.resolve(TOKENS.CONTEXT_ACCESSOR),
        )
      },
    ]
  }),
)
```

Custom strategies are added to the same authorization pipeline as the built-in
strategies.

Typical custom rules include:

- resource ownership;
- organization membership;
- subscription state;
- account status;
- resource-specific tenant isolation;
- business-specific access rules.

See [Custom Authorization](./custom) for the implementation details.

---

## A complete example

The following example combines identity, tenant, role, and permission checks:

```ts
const app = new XenoAppBuilder()

app.addPipeline(
  withAuthZ((opts) => {
    opts.policies = {
      CreateInvoice: {
        userId: true,
        tenantId: true,
        roles: ['admin', 'billing'],
        permissions: ['invoice:create'],
      },

      ReadInvoice: {
        tenantId: true,
        permissions: ['invoice:read', 'invoice:admin'],
      },

      DeleteInvoice: {
        tenantId: true,
        roles: ['admin'],
        permissions: ['invoice:delete'],
      },
    }
  }),
)

const container = await app.build()
```

The resulting rules are:

| Intent          | Requirements                                                        |
| --------------- | ------------------------------------------------------------------- |
| `CreateInvoice` | valid user + valid tenant + `admin` OR `billing` + `invoice:create` |
| `ReadInvoice`   | valid tenant + `invoice:read` OR `invoice:admin`                    |
| `DeleteInvoice` | valid tenant + `admin` + `invoice:delete`                           |

All configured requirements for an intent must pass.

---

## Common mistakes

### Authorization is not running

Make sure the CQRS pipeline is enabled:

```ts
app.addPipeline(
  withAuthZ((opts) => {
    opts.policies = {
      CreateInvoice: {
        permissions: ['invoice:create'],
      },
    }
  }),
)
```

Authorization is not enabled merely by defining an authentication provider.

---

### Authentication is configured but authorization is not

These are separate:

```text
Authentication
    ↓
creates Identity

Authorization
    ↓
checks Identity
```

Configure both when the application needs authenticated and protected
operations.

---

### Expecting `userId` to enforce ownership

This:

```ts
{
  userId: true,
}
```

only requires a valid authenticated user ID.

It does not enforce:

```text
identity.userId === resource.ownerId
```

Use a custom strategy for ownership rules.

---

### Expecting `tenantId` to enforce resource isolation

This:

```ts
{
  tenantId: true,
}
```

only requires a valid authenticated tenant ID.

It does not automatically scope database queries or compare a resource's tenant
ID.

Resource isolation must be enforced by the application's data access and
authorization rules.

---

### Expecting multiple roles to require all roles

This:

```ts
roles: ['admin', 'billing']
```

means:

```text
admin OR billing
```

If the application requires a more complex relationship between roles, implement
that rule explicitly.

---

### Expecting multiple permissions to require all permissions

This:

```ts
permissions: ['invoice:create', 'invoice:approve']
```

means:

```text
invoice:create OR invoice:approve
```

If both permissions are required simultaneously, use a custom authorization
rule.

---

## When to use built-in policies

Use built-in policies when the requirement can be expressed as:

```text
valid user identity
valid tenant identity
required role
required permission
```

Examples:

```ts
GetDashboard: {
  userId: true,
}
```

```ts
ListInvoices: {
  tenantId: true,
}
```

```ts
ManageUsers: {
  roles: ['admin'],
}
```

```ts
CreateInvoice: {
  permissions: ['invoice:create'],
}
```

Use a custom authorization strategy when the rule depends on application data or
business logic.

---

## Related documentation

- [Authorization Policies](./policies)
- [Custom Authorization](./custom)
- [Authentication Overview](../authentication/overview)
- [Authentication Middleware](../middleware/authentication)
- [Middleware Execution](../middleware/execution)
- [CQRS Overview](../../application/cqrs/overview)

---

---

<a id="source-5"></a>

# Source: `docs/security/middlewares/allow-method.md`

## Introduction

`MethodCheckMiddleware` checks a request's path and HTTP method against the
routes registered with `XenoAppBuilder.addRoutes()`. When the middleware
receives a method that is not allowed for the path, it stops the middleware
pipeline and returns `405 Method Not Allowed`.

The middleware derives its policy from route definitions rather than from a
separate allow-method configuration. For a rejected request, the response
includes an `Allow` header with the methods registered for that path.

In the normal HTTP server flow, the route matcher looks up the path and method
before the route handler invokes the middleware pipeline. If no route matches,
the HTTP adapter returns `404 Not Found`; the request does not reach
`MethodCheckMiddleware`.

Method checking and CORS are separate controls:

- `MethodCheckMiddleware` decides whether the request method is registered for
  the path.
- `cors.enabled` controls the regular CORS response middleware.
- `cors.enablePreflight` controls whether the OPTIONS middleware handles
  preflight requests.

## Enable Method Checking

Register routes and enable the middleware stack with `addMiddlewares()`:

```ts
import { HttpRouteParamHelper, XenoAppBuilder } from '@xeno-js/core'
import type { NativeRouteDefinition } from '@xeno-js/shared'
import type { MyRegistry } from './registry'
import type { CreateOrderPayload } from './order/application'

const ordersRoutes: NativeRouteDefinition<MyRegistry>[] = [
  {
    path: '/api/v1/orders',
    method: 'GET',
    handler: async (ctx) => {
      const input = await HttpRouteParamHelper.extractInput<{
        page?: number
        pageSize?: number
      }>(ctx.req, '/api/v1/orders')
      return ctx.handle('GET_ORDERS_CONTROLLER', input)
    },
  },
  {
    path: '/api/v1/orders/:id',
    method: 'GET',
    handler: async (ctx) => {
      const input = await HttpRouteParamHelper.extractInput<{ id: string }>(
        ctx.req,
        '/api/v1/orders/:id',
      )
      return ctx.handle('GET_ORDER_BY_ID_CONTROLLER', input)
    },
  },
  {
    path: '/api/v1/orders',
    method: 'POST',
    handler: async (ctx) => {
      const input = await HttpRouteParamHelper.extractInput<CreateOrderPayload>(
        ctx.req,
        '/api/v1/orders',
      )
      return ctx.handle('CREATE_ORDER_CONTROLLER', input)
    },
  },
]

const app = new XenoAppBuilder()

app.addRoutes(ordersRoutes)
app.addMiddlewares()

const application = await app.build()
```

This example registers `GET` and `POST` for `/api/v1/orders`, and `GET` for
`/api/v1/orders/:id`. `MyRegistry`, the controller tokens, and
`CreateOrderPayload` come from the application; define them to match the
services and input types in your project.

Xeno collects the methods from these route definitions when it builds the
middleware module. Register routes before calling `build()`; the builder
resolves queued modules during that call.

The middleware module adds `MethodCheckMiddleware` only when the application has
routes. Calling `addMiddlewares()` without registering routes does not add this
method check.

There is no `allowMethods` option in `addMiddlewares()`. To permit another
method for a path, register a route with that method.

## How Method Validation Works

When `MethodCheckMiddleware` receives a request, it:

1. Checks the request path and method against the registered route definitions.
2. Continues to the next middleware when the pair is registered.
3. Returns `405 Method Not Allowed` when the method is not allowed for the path.

For regular HTTP server requests, the route matcher first looks up a route by
path and method. If the caller requests a different, unregistered path (for
example, `GET /api/v1/customers`) or an unregistered method for that path, the
route matcher returns no match and the HTTP adapter responds with
`404 Not Found` and a route-not-found error. That request does not reach the
middleware pipeline. The `405` response applies when a request reaches
`MethodCheckMiddleware` and the middleware rejects its method.

The method registry groups methods by route path. It normalizes the path and
removes the query string or fragment before checking it. Dynamic route segments
using `:name` or `{name}` patterns are matched against a single path segment.

Xeno does not infer extra methods from a registered method. For example,
registering `GET` does not automatically register `HEAD`; add a route for each
method the application intends to accept.

If the middleware rejects a request, it returns an error response with the
`NOT_ALLOWED` error code and the rejected path. Its `Allow` header contains the
methods registered for the matching path, joined by commas.

## Middleware Order

The method check runs after request-context and origin validation, and after the
OPTIONS handler when that handler is enabled. It runs before the regular CORS
middleware, rate limiting, authentication, and CSRF middleware.

When `cors.enablePreflight` is enabled and routes exist, the OPTIONS middleware
handles an `OPTIONS` request first and returns its preflight response. Without
preflight handling, the route matcher can create a synthetic `OPTIONS` route for
a path that exists; the request then continues to method checking, where
`OPTIONS` must be allowed by the middleware's route registry.

Disabling `cors.enabled` does not disable method checking. Likewise, allowing a
request method does not configure CORS response headers or authorize the caller;
those responsibilities remain with their corresponding middleware and
application authorization policies.

## Common Issues

### A request through the middleware returns `405 Method Not Allowed`

Check that `addRoutes()` registers the path and method used by the request. The
method check uses the route definitions, not the CORS settings or HTTP server
configuration. In the normal HTTP server flow, a path/method pair that the route
matcher cannot find returns `404` before the middleware runs.

Also check dynamic route patterns and path spelling. The method registry strips
query strings and fragments and normalizes trailing slashes.

### A request to a different route returns `404 Not Found`

This is expected when the route matcher cannot find a registered route for the
requested path and method. Add the route with `addRoutes()` or correct the URL.
The HTTP adapter returns the route-not-found response before
`MethodCheckMiddleware` runs.

### Registering `GET` does not allow `HEAD`

The method registry uses the methods explicitly present in route definitions.
Register `HEAD` separately if the application needs to accept it.

### Disabling CORS did not change the `405` response

This is expected. `cors.enabled` controls the CORS response middleware; method
checking remains active when routes and the middleware module are configured.

### An OPTIONS request is rejected

For browser preflight requests, check that `cors.enablePreflight` is enabled and
routes are registered. Without preflight handling, a synthetic `OPTIONS` route
may reach method checking and be rejected unless `OPTIONS` is allowed for that
path. See [OPTIONS Middleware](./options-middleware) for preflight behavior.

## Related Documentation

- [OPTIONS Middleware](./options-middleware)
- [Allow-Origin Middleware](./allow-origin)
- [Security Overview](../overview)

---

---

<a id="source-6"></a>

# Source: `docs/security/middlewares/allow-origin.md`

## Introduction

`AllowOriginMiddleware` checks the request's `Origin` against the application's
allowed origins or a route-specific origin policy. If the origin is not allowed,
Xeno stops the middleware pipeline and returns `403 Forbidden`.

Origin validation and CORS response headers are related but separate:

- `AllowOriginMiddleware` decides whether the request may continue.
- `cors.enabled` controls whether the regular CORS middleware adds CORS response
  headers.
- `cors.enablePreflight` controls handling for preflight `OPTIONS` requests.

The origin validation middleware is added by `addMiddlewares()` and runs for
incoming requests regardless of `cors.enabled`.

## Configure the Application Allowlist

Set `cors.allowOrigins` with the origins allowed to call the application:

```ts
import { XenoAppBuilder } from '@xeno-js/core'

const app = new XenoAppBuilder()

app.addMiddlewares((options) => {
  options.cors = {
    enabled: true,
    allowOrigins: ['https://app.example.com', 'https://admin.example.com'],
  }
})

const application = await app.build()
```

An origin consists of the scheme, host, and optional port. Configure the origin
as the browser sends it, for example `https://app.example.com` or
`http://localhost:5173`.

When `cors.allowOrigins` is empty or omitted, Xeno derives the global allowlist
from `APP_URL`. If `APP_URL` is not configured, it uses
`http://localhost:<PORT>`, with `3000` as the default port.

The configured comparison trims surrounding whitespace and ignores letter case.
The origin must otherwise match the configured value.

## Configure a Route-Specific Policy

An individual route can define its own `security.allowOrigins` policy. When the
matching route defines this property, Xeno uses it instead of the
application-level list for that route:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import type { NativeRouteDefinition } from '@xeno-js/shared'

declare const reportsHandler: NativeRouteDefinition['handler']

const reportsRoute: NativeRouteDefinition = {
  path: '/api/reports',
  method: 'GET',
  security: {
    allowOrigins: ['https://admin.example.com'],
  },
  handler: reportsHandler,
}

const app = new XenoAppBuilder()
app.addRoutes([reportsRoute])
app.addMiddlewares()

const application = await app.build()
```

`reportsHandler` represents the handler already implemented by the application;
the example focuses on attaching the route security policy.

An empty route-specific list is still an explicit policy and allows no origin
values. It does not fall back to the application-level list.

## Wildcard and Subdomain Entries

The `'*'` entry allows any origin:

```ts
app.addMiddlewares((options) => {
  options.cors = {
    allowOrigins: ['*'],
  }
})
```

Use a wildcard only when the resource is intentionally available to every
origin. An allowed origin is not an authentication or authorization decision;
sensitive operations still require authentication and authorization.

The application-level `AllowOrigin` matcher also supports a single-label
subdomain suffix pattern such as `*.example.com`. It matches an origin such as
`https://app.example.com`, but not a nested subdomain such as
`https://dev.app.example.com`. This pattern does not constrain the scheme and
does not match an origin with a port suffix. For a narrow trust boundary, prefer
listing exact origins.

Route-specific `security.allowOrigins` uses direct list matching and supports
`'*'`; do not assume the subdomain pattern behavior applies to route-specific
policies.

## Requests Without an Origin

The global allow-origin matcher allows requests with no `Origin` header. This is
common for server-to-server requests and some non-browser clients.

A route-specific `security.allowOrigins` list behaves differently: the
middleware requires the missing origin (treated as an empty value) to match the
list. Since a normal allowlist does not contain an empty string, such a request
is rejected with `403 Forbidden`. Account for this if a route must serve both
browser and non-browser callers.

## How Validation Works

For each request, the middleware:

1. Reads the origin from the request context.
2. Checks the matched route's `security.allowOrigins` when that property is
   defined; otherwise checks the application-level allowlist.
3. Calls the next middleware when allowed.
4. Logs a warning and returns `403 Forbidden` when denied.

The middleware runs before the CORS response middleware, rate limiting,
authentication, and CSRF middleware. A rejected request does not reach those
later stages.

Allowing an origin only controls this origin gate. It does not guarantee that
the browser can read the response: the CORS response middleware and preflight
handling must also be configured correctly. Conversely, setting `cors.enabled`
to `false` does not bypass the origin gate.

The origin gate checks the policy for the incoming method. A browser preflight
arrives as `OPTIONS`; if there is no explicit `OPTIONS` route, the route matcher
does not apply the requested `GET` or `POST` route's security policy at this
stage. Configure the global `cors.allowOrigins` to permit the browser origin for
preflight requests as well.

## Common Issues

### A request is denied with `403 Forbidden`

Check the `Origin` value sent by the client, including its scheme and port, and
compare it with `cors.allowOrigins`. For a route-specific policy, check
`security.allowOrigins` on the matched route; when present, it takes precedence
over the global list.

If the request has no `Origin` header, inspect whether the route defines an
explicit allowlist. Such a route-specific list does not permit a missing origin
by default.

### The browser still reports a CORS error after the origin is allowed

Origin validation only decides whether the request continues. Check
`cors.enabled`, `cors.enablePreflight`, the configured credentials policy, and
any required headers or methods. See [OPTIONS Middleware](./options-middleware)
for preflight behavior.

### Local development works, but the deployed application rejects the frontend

Configure the deployed frontend's exact origin or set `APP_URL` to the intended
global default. When `APP_URL` is unset, Xeno derives the default from the
configured `PORT` (or `3000` if absent), which may not match the deployed
frontend's origin.

### A route's global allowlist seems ignored

Check the route's `security.allowOrigins`. If it is defined, Xeno uses that
route policy instead of the global list for the matching path and method.

For browser preflight requests, also check the global allowlist. The origin
middleware evaluates the `OPTIONS` method before preflight handling, so it may
not see the security policy attached to the requested `GET` or `POST` route.

## Related Documentation

- [OPTIONS Middleware](./options-middleware)
- [Request Context Middleware](./request-context-midleware)
- [Security Overview](../overview)

---

---

<a id="source-7"></a>

# Source: `docs/security/middlewares/authentication-middleware.md`

## Introduction

The authentication middleware runs at the HTTP boundary. It extracts a
credential from the incoming request, delegates credential validation to the
configured authentication service, and writes the resulting identity to the
current request context.

Authentication answers **who is making the request**. It does not decide whether
that identity can execute a command, query, or route. Use application
authorization policies for access decisions.

The flow separates transport from identity verification:

```text
HTTP request
    │
    ├── Token extractor: reads a credential from headers or cookies
    │
    ├── Authentication service: validates the credential or resolves a session
    │
    ├── ClaimsIdentityMapper: maps claims to Identity
    │
    └── RequestContext: makes the identity available downstream
```

## Configure Authentication

Register an authentication provider with `addAuth()` and enable the HTTP
middleware stack with `addMiddlewares()`:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withSupabase } from '@xeno-js/supabase'
import { ordersRoutes } from './order/order.route'

const app = new XenoAppBuilder()

app.addRoutes(ordersRoutes)
app.addAuth(
  withSupabase((options, config) => {
    options.url = config.getOrThrow('SUPABASE_URL')
    options.key = config.getOrThrow('SUPABASE_KEY')
  }),
)
app.addMiddlewares()

const application = await app.build()
```

`ordersRoutes` represents the application's route definitions. `addAuth()`
queues the authentication provider; `addMiddlewares()` queues the HTTP
middleware module. The builder resolves queued modules during `build()`.
Configuring an authentication provider without adding the middleware stack does
not authenticate incoming HTTP requests.

The provider supplies an authentication service and may supply a token
extractor. Without a provider, `addMiddlewares()` still installs
`AuthenticationMiddleware`, but Xeno uses `NoAuthGateKeeper`, which ignores any
supplied token and always establishes a guest identity. Do not treat this
configuration as authentication for protected endpoints.

## Credential Extraction

The middleware does not parse credentials itself. It calls the configured token
extractor with the request headers.

When no custom extractor is registered, Xeno uses `BearerTokenExtractor`. It
checks:

1. An `Authorization` header with the `Bearer` scheme.
2. The configured authentication cookie, if no bearer token was found.

The default cookie name is `sb-access-token`. `authCookieName` in the middleware
options changes this fallback when the middleware creates the default extractor.
An authentication integration can register its own extractor and cookie
handling; in that case, the integration's configuration controls extraction.

For example, a bearer token is sent as:

```http
Authorization: Bearer <access-token>
```

For Supabase-specific credential and cookie behavior, see
[Supabase Authentication](../authentication/supabase).

## Validate Credentials and Establish Identity

When the extractor returns a token, the configured gatekeeper calls the
authentication service's `authenticate(token)` method. If authentication
succeeds, Xeno maps the returned claims to an `Identity` and updates the request
context before continuing the middleware pipeline.

The default `ClaimsIdentityMapper` maps:

- `sub` to `Identity.userId`;
- `tenantId` to `Identity.tenantId`;
- `email`, `name`, `roles`, and `permissions` to the corresponding identity
  properties.

The mapper parses user and tenant identifiers as GUIDs. Provide valid GUID
strings when the application relies on those identifiers.

When the extractor returns no token, a configured gatekeeper calls `getUser()`
to resolve an existing session. If the service returns no user or fails to
resolve one, the gatekeeper uses the guest identity and continues. Thus, the
HTTP authentication middleware does not automatically require every request to
have an authenticated identity.

The identity is stored in the `RequestContext` and is available to downstream
middleware and application services. In the default Node.js context,
`AsyncLocalStorage` carries request-specific context through asynchronous
execution. Authentication services are request-scoped; do not retain them in
singleton services.

## Authentication Failures

If a supplied token fails authentication, the middleware:

1. Logs a warning containing the request path and authentication error.
2. Returns an error response using the authentication error's status, or
   `401 Unauthorized` when no status is supplied.
3. Includes the error code, message, request path, and correlation identifiers
   in the response.
4. Stops the pipeline without calling the next middleware.

A missing token is different from an invalid token. When no token is extracted,
Xeno attempts session lookup and then falls back to the guest identity; when a
supplied token is rejected, the middleware returns an error response.

## Require an Authenticated User

`AuthenticationMiddleware` establishes identity but does not, by itself, reject
guest requests. Decide at the application boundary which operations require an
authenticated user, tenant, role, or permission.

For example, an authorization policy can require a user identity or a role
before executing a command. Keep those access rules in the application
authorization pipeline rather than treating successful execution of the HTTP
authentication middleware as proof that the caller is authorized.

See [Authorization](../authorization/overview) for authorization strategies and
policies.

## Configure a Custom Authentication Service

Xeno supports custom providers through `addAuth()`. Configure an `authService`
and, when needed, a `tokenExtractor`:

```ts
import { XenoAppBuilder } from '@xeno-js/core'

// Implement these application-specific classes against Xeno's auth and extractor contracts.
const app = new XenoAppBuilder()

app.addAuth((options) => {
  options.authService = () => new MyAuthService()
  options.tokenExtractor = () => new MyTokenExtractor()
})

app.addMiddlewares()

const application = await app.build()
```

`MyAuthService` must implement Xeno's authentication service contract, and
`MyTokenExtractor` must implement
`IServiceExtractor<Request['headers'], Optional<string>>`. The auth service
factory receives the active service scope; Xeno registers the authentication
service as scoped. The token extractor is registered as a singleton, so it must
not capture request-scoped services.

The authentication service validates a supplied token through
`authenticate(token)`. When the extractor returns no token, the gatekeeper calls
`getUser()` to resolve a session. `isAuthenticated()` is part of the service
contract but is not called by the HTTP authentication middleware.

See [Authentication Overview](../authentication/overview) for the complete
service contract and [Custom Authentication](../authentication/custom) for
provider implementation details.

## Common Issues

### A route executes without a logged-in user

This is expected when no credential or session resolves to an authenticated
user. The gatekeeper assigns a guest identity and continues. Add application
authorization rules to operations that require an authenticated user.

### A valid-looking token returns `401 Unauthorized`

Check that the extractor reads the credential from the location used by the
client, that the authentication service accepts the token, and that the
configured provider and environment values are correct. A custom extractor must
return the raw credential expected by the authentication service.

### Authentication middleware ignores the supplied token

Check whether `addAuth()` configured the provider before `build()`. Without an
authentication provider, `NoAuthGateKeeper` deliberately ignores credentials and
always returns a guest identity.

### The token is read from the wrong cookie

Check the authentication integration's token extractor configuration.
`authCookieName` applies to the default bearer-token extractor created by the
middleware; a provider that registers its own extractor may use different cookie
options.

### Authentication succeeds but an operation is denied

Authentication only establishes identity. Inspect the application's
authorization policies, including any user, tenant, role, or permission
requirements.

## Related Documentation

- [Authentication Overview](../authentication/overview)
- [Supabase Authentication](../authentication/supabase)
- [Custom Authentication](../authentication/custom)
- [Authorization](../authorization/overview)
- [Request Context Middleware](./request-context-midleware)
- [Middleware Overview](./overview)
- [Security Overview](../overview)

---

---

<a id="source-8"></a>

# Source: `docs/security/middlewares/cookie-middleware.md`

## Introduction

Server-side authentication libraries may need to read cookies from the incoming
request and update cookies in the response. A login, token refresh, or session
update can produce one or more `Set-Cookie` values that the HTTP server must
return to the browser.

Xeno separates cookie management from response transport:

- The request-scoped `ISsrCookieHandler` reads incoming cookies and buffers
  cookie updates during the request.
- `CookieResponseMiddleware` collects the buffered values after the downstream
  middleware and route handler finish, then appends them to the response's
  `Set-Cookie` header.

The middleware does not create authentication cookies or decide their contents.
An SSR authentication integration or application code queues those changes
through the cookie handler.

This is separate from `CsrfCookieMiddleware`, which generates a CSRF token
cookie and validates tokens on state-changing requests.

## Enable SSR Cookie Handling

Set `isSSR` to `true` in `addMiddlewares()`. When using the Supabase SSR
integration, configure `options.ssr` in `withSupabase()` as well; that
integration registers the request-scoped cookie handler used by the response
middleware.

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withSupabase } from '@xeno-js/supabase'

const app = new XenoAppBuilder()

app.addAuth(
  withSupabase((options, config) => {
    options.url = config.getOrThrow('SUPABASE_URL')
    options.key = config.getOrThrow('SUPABASE_KEY')
    options.ssr = {
      cookieOptions: {
        path: '/',
        httpOnly: true,
        secure: true,
        sameSite: 'lax',
      },
    }
  }),
)

app.addMiddlewares((options) => {
  options.isSSR = true
})

const application = await app.build()
```

Use the SSR integration when the server must read and update Supabase session
cookies. The `options.ssr.cookieOptions` values are forwarded to the Supabase
SSR client and also configure defaults for cookies serialized by Xeno's default
handler. Keep `secure: true` in HTTPS deployments; for local HTTP development
only, configure `secure: false`.

`isSSR` enables `CookieResponseMiddleware`, but it does not register a cookie
handler by itself. A handler must be supplied by an auth integration or
registered in the request scope. The Supabase SSR integration configures a
default handler unless you provide a custom `options.ssr.cookieHandler`.

## Cookie Handler API

The request-scoped handler implements `ISsrCookieHandler`:

| Method                | Behavior                                                                                                        |
| --------------------- | --------------------------------------------------------------------------------------------------------------- |
| `getAll()`            | Reads the incoming `Cookie` header from the current network context and returns parsed `{ name, value }` pairs. |
| `setAll(cookies)`     | Queues cookie values and attributes to be added to the outgoing response.                                       |
| `getPendingCookies()` | Serializes and returns the queued cookies as `Set-Cookie` values.                                               |

In the Supabase SSR integration, the server client uses `getAll()` to read the
incoming browser session and calls `setAll()` when it needs to update cookies.
`CookieResponseMiddleware` then transports the queued values to the HTTP
response.

The handler uses the current request's service scope, so cookie state is
isolated per request. Avoid storing a request's pending cookies in a shared
singleton; use the scoped handler supplied by the SSR integration or implement a
custom `ISsrCookieHandler` with equivalent request isolation.

## How the Response Middleware Works

For each request, `CookieResponseMiddleware`:

1. Calls the next middleware and waits for the response.
2. Gets the active request scope from `IRequestContext`.
3. Resolves `TOKENS.COOKIE_HANDLER` and reads its pending cookie values.
4. Appends them to the response's `Set-Cookie` header, preserving existing
   cookies and other response headers.
5. Returns the updated response.

If the response already contains a `Set-Cookie` string, Xeno converts it to an
array before appending the pending values. Each cookie remains a separate
`Set-Cookie` value; do not combine multiple cookies into one comma-separated
header.

When there is no active scope or no pending cookie, the middleware leaves the
response unchanged. It also leaves the response unchanged if resolving the
handler or reading pending cookies fails. Errors from the downstream middleware
still propagate.

The middleware runs after the downstream response is produced. It can therefore
append queued cookies to normal responses and error responses that return
through the middleware chain. It cannot change a response that bypasses the Xeno
middleware pipeline.

## Default Cookie Serialization

Xeno's default `CookieHandler` reads request cookies from the network context
and serializes queued cookie values. Its serializer applies these defaults
unless overridden:

| Attribute  | Default                                 |
| ---------- | --------------------------------------- |
| `Path`     | `/`                                     |
| `HttpOnly` | `true`                                  |
| `Secure`   | `true` in production; otherwise `false` |
| `SameSite` | `Lax`                                   |

You can set default attributes in the Supabase SSR configuration:

```ts
options.ssr = {
  cookieOptions: {
    path: '/',
    httpOnly: true,
    secure: true,
    sameSite: 'lax',
    domain: 'example.com',
  },
}
```

The serializer also supports `maxAge`, `expires`, `domain`, `partitioned`, and
`priority`. Per-cookie values passed to `setAll()` override handler defaults.
Cookie names and values are URI-encoded when serialized; incoming values are
URI-decoded when possible.

Choose attributes to match the deployment:

- `httpOnly: true` prevents client-side JavaScript from reading authentication
  cookies.
- `secure: true` restricts cookie transmission to HTTPS.
- `sameSite: 'lax'` is the default; `'strict'` tightens cross-site sending and
  `'none'` permits cross-site use in browsers that support it. `SameSite=None`
  requires `Secure`.
- `domain` broadens which hosts receive the cookie. Omit it for a host-only
  cookie unless sharing across subdomains is an explicit requirement.

## Use a Custom Cookie Handler

The Supabase SSR configuration accepts a custom handler when the application
needs to integrate with its own request and response cookie system:

```ts
import type { ISsrCookieHandler } from '@xeno-js/core'
import { withSupabase } from '@xeno-js/supabase'

declare const applicationCookieHandler: ISsrCookieHandler

const supabaseAuth = withSupabase((options, config) => {
  options.url = config.getOrThrow('SUPABASE_URL')
  options.key = config.getOrThrow('SUPABASE_KEY')
  options.ssr = {
    cookieHandler: applicationCookieHandler,
  }
})
```

`applicationCookieHandler` represents an application-provided handler. Its
`getAll()` must read the current request's cookies, and its `setAll()` must
retain that request's cookie updates until `getPendingCookies()` returns them to
`CookieResponseMiddleware`. Since the Supabase plugin accepts a handler
instance, the application must ensure the handler uses request-aware state and
does not mix pending cookies across concurrent requests.

The Supabase integration uses the custom handler instead of registering its
default handler. Keep `isSSR: true` enabled so Xeno adds the response middleware
that appends the pending cookie values.

## Common Issues

### The response does not include updated authentication cookies

Check that `options.ssr` is configured in the authentication integration and
that `isSSR` is `true` in `addMiddlewares()`. Confirm that the handler's
`setAll()` is called and that the request has an active scope with
`TOKENS.COOKIE_HANDLER` registered.

### `isSSR: true` is set, but no cookies are sent

`isSSR` only enables `CookieResponseMiddleware`; it does not create a handler.
Configure the Supabase SSR integration or register a custom request-scoped
handler. The middleware returns the response unchanged when no scope or pending
cookies are available.

### Existing `Set-Cookie` values disappear

The middleware appends pending cookies and preserves existing response headers.
If a custom HTTP adapter or response mapper rewrites headers after Xeno returns
the response, verify that it preserves every `Set-Cookie` value as a separate
header.

### Cookies work locally but not in production

Check the `Secure`, `SameSite`, `Domain`, and `Path` attributes against the
deployed scheme and host. Secure cookies require HTTPS. Cross-site cookies
generally require `SameSite=None; Secure`; browser privacy settings may impose
additional restrictions.

### The browser does not send a cookie to a subdomain

Cookies are host-only unless a `Domain` attribute is set. Configure an
appropriate domain only when sharing across subdomains is required, and consider
the increased scope of that cookie.

### An authentication cookie is confused with the CSRF cookie

Authentication cookies are read and updated by the SSR auth integration through
`ISsrCookieHandler`. The CSRF cookie is a separate token cookie managed by
`CsrfCookieMiddleware`; see [CSRF Middleware](./csrf-middleware).

## Related Documentation

- [Supabase Authentication](../authentication/supabase)
- [Authentication Overview](../authentication/overview)
- [CSRF Middleware](./csrf-middleware)
- [Request Context Middleware](./request-context-midleware)
- [Security Overview](../overview)

---

---

<a id="source-9"></a>

# Source: `docs/security/middlewares/cors-middleware.md`

## Introduction

`CORSMiddleware` adds CORS response headers after the downstream middleware and
route handler return a response. It reads the request origin from the request
context and checks the matched route's CORS policy to determine the allowed
origin and credentials setting.

CORS headers and origin validation are separate responsibilities:

- `AllowOriginMiddleware` checks the request origin against the application
  allowlist or the route's `security.allowOrigins` policy. It can stop the
  request with `403 Forbidden`.
- `CORSMiddleware` adds `Access-Control-Allow-Origin`,
  `Access-Control-Allow-Credentials`, and `Vary: Origin` to the response when an
  origin is present or the route allows `'*'`.
- `OptionsMiddleware` handles browser preflight `OPTIONS` requests and
  advertises methods and headers.

The middleware stack runs these controls independently. Enabling or disabling
CORS response headers does not disable origin validation.

## Enable CORS Response Headers

`XenoAppBuilder` enables the CORS response middleware by default. Configure the
application-level credentials setting and allowed origins through
`addMiddlewares()`:

```ts
import { XenoAppBuilder } from '@xeno-js/core'

const app = new XenoAppBuilder()

app.addMiddlewares((options) => {
  options.cors = {
    enabled: true,
    enablePreflight: true,
    withCredentials: true,
    allowOrigins: ['https://app.example.com'],
  }
})

const application = await app.build()
```

When `cors.allowOrigins` is empty or omitted, Xeno uses `APP_URL` as the
application-level origin. If `APP_URL` is not configured, it uses
`http://localhost:<PORT>`, with `3000` as the default port.
`AllowOriginMiddleware` uses this allowlist to validate incoming requests.

`cors.withCredentials` controls the default value of
`Access-Control-Allow-Credentials`. `cors.enablePreflight` is a separate setting
that enables `OptionsMiddleware`; it does not control whether `CORSMiddleware`
adds headers.

## Set a Route-Specific CORS Policy

A route can define its own `security.allowOrigins` and
`security.withCredentials` values. The `withCors()` helper from `@xeno-js/core`
applies those values to a route definition:

```ts
import { HttpRouteParamHelper, withCors, XenoAppBuilder } from '@xeno-js/core'
import type { NativeRouteDefinition } from '@xeno-js/shared'
import type { MyRegistry } from './registry'

const orderRoute: NativeRouteDefinition<MyRegistry> = {
  path: '/api/v1/orders/:id',
  method: 'GET',
  handler: async (ctx) => {
    const input = await HttpRouteParamHelper.extractInput<{ id: string }>(
      ctx.req,
      '/api/v1/orders/:id',
    )
    return ctx.handle('GET_ORDER_BY_ID_CONTROLLER', input)
  },
}

const app = new XenoAppBuilder()

app.addRoutes([withCors(orderRoute, ['https://app.example.com'], true)])
app.addMiddlewares((options) => {
  options.cors = {
    enabled: true,
    enablePreflight: true,
    withCredentials: true,
    allowOrigins: ['https://app.example.com'],
  }
})

const application = await app.build()
```

`MyRegistry` and `GET_ORDER_BY_ID_CONTROLLER` represent application-specific
types and a controller token. `withCors()` returns the route with
`security.allowOrigins` and `security.withCredentials` set; its third argument
defaults to `true`.

When a route defines its own origin policy, `AllowOriginMiddleware` uses that
policy instead of the application-level list for that route. `CORSMiddleware`
uses the route's credential setting when it is defined; otherwise it falls back
to `cors.withCredentials`.

## Response Headers

When the request has an `Origin` header, or the route policy contains `'*'`, the
middleware adds these headers to the response:

```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
Vary: Origin
```

The middleware preserves headers already present on the response and merges the
CORS values into them.

For a route whose `security.allowOrigins` contains `'*'`:

- when credentials are disabled, Xeno returns `Access-Control-Allow-Origin: *`;
- when credentials are enabled, Xeno returns the request's `Origin` value
  instead of `*`.

For a request without an `Origin` header, the middleware does not add CORS
headers unless the route policy contains `'*'`. Non-browser clients can make
requests without an `Origin`; whether they are permitted is determined by the
origin-validation policy, not by these response headers.

## How the Middleware Works

For each request, `CORSMiddleware`:

1. Reads the origin from the current request context.
2. Calls the next middleware, allowing the route to produce a response.
3. Looks up the route policy for the request path and method.
4. Uses the route's `withCredentials` value when set; otherwise uses the
   application setting.
5. Adds CORS response headers when the request has an origin or the route allows
   `'*'`.

Because it adds headers after `next()` resolves, responses returned by
downstream middleware, including error responses, also receive CORS headers when
the origin or wildcard condition is met. A request rejected earlier by
`AllowOriginMiddleware` never reaches `CORSMiddleware`.

## Common Issues

### The browser reports a CORS error even though `cors.enabled` is true

Check that the request origin is allowed by `cors.allowOrigins` or the route's
`security.allowOrigins`. The origin-validation middleware runs before
`CORSMiddleware` and rejects disallowed origins with `403 Forbidden`.

For browser preflight requests, also check `cors.enablePreflight`, and verify
that the preflight response allows the requested method and headers. See
[OPTIONS Middleware](./options-middleware).

### The response has no CORS headers

Check that `cors.enabled` is true and that the request includes an `Origin`
header. Without an origin, CORS headers are added only when the matched route's
`security.allowOrigins` includes `'*'`.

### The response returns the request origin instead of `*`

This is expected when the route allows `'*'` and credentials are enabled.
Browsers do not accept a wildcard `Access-Control-Allow-Origin` value for
credentialed requests. Disable credentials only if the endpoint does not need
them.

### A route does not use the application-level origin list

Check whether the matched route defines `security.allowOrigins`. When present,
that list replaces the global list for origin validation on that route and also
informs the CORS response policy.

### The application allows the origin, but browser preflight fails

The regular CORS middleware adds headers to the response after the route runs;
it does not handle preflight requests. Enable `cors.enablePreflight` and
configure the allowed headers and route methods. See
[OPTIONS Middleware](./options-middleware).

## Related Documentation

- [Allow-Origin Middleware](./allow-origin)
- [OPTIONS Middleware](./options-middleware)
- [Security Overview](../overview)

---

---

<a id="source-10"></a>

# Source: `docs/security/middlewares/csrf-middleware.md`

## Introduction

Cross-Site Request Forgery (CSRF) occurs when a browser sends a user's
automatically attached credentials to an application because another site caused
the browser to submit a request. A request can therefore be authenticated even
when the user did not intentionally initiate the action.

CSRF protection adds proof that the request came from code that can read a token
issued for the authenticated user. Xeno uses a signed, user-bound token in a
cookie and requires the same token in a request header for state-changing
methods. A malicious site can often cause a browser to send cookies, but it
cannot normally read the application's CSRF cookie and copy its value into the
required header.

CSRF protection complements authentication; it does not identify the user or
grant permissions. Xeno enables it only when the application supplies `csrf`
configuration to `XenoAppBuilder.addMiddlewares()`.

## Configure CSRF Protection

Configure a server-side secret and, optionally, the cookie and header settings:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withSupabase } from '@xeno-js/supabase'
import { ordersRoutes } from './order/order.route'

const app = new XenoAppBuilder()

app.addRoutes(ordersRoutes)
app.addAuth(
  withSupabase((options, config) => {
    options.url = config.getOrThrow('SUPABASE_URL')
    options.key = config.getOrThrow('SUPABASE_KEY')
  }),
)
app.addMiddlewares((options, config) => {
  options.cors = {
    enabled: true,
    enablePreflight: true,
    withCredentials: true,
    allowOrigins: [config.getOrThrow('CLIENT_URL')],
  }
  options.csrf = {
    secret: config.getOrThrow('CSRF_SECRET'),
    cookieName: '__Host-xeno-csrf',
    headerName: 'x-csrf-token',
    cookieMaxAgeSeconds: 3600,
    sameSite: 'lax',
  }
})

const application = await app.build()
```

The route list and configuration keys in this example belong to the application.
Keep `CSRF_SECRET` on the server, load it from a secret manager or protected
environment configuration, and use a strong randomly generated value. Do not
send it to the browser or commit it to source control. All application instances
that validate the same user's tokens must use the same secret.

CSRF configuration options:

| Option                | Default            | Purpose                                                         |
| --------------------- | ------------------ | --------------------------------------------------------------- |
| `secret`              | Required           | Secret used to sign and validate user-bound tokens.             |
| `cookieName`          | `__Host-xeno-csrf` | Name of the CSRF cookie read from the incoming `Cookie` header. |
| `headerName`          | `x-csrf-token`     | Request header from which Xeno reads the submitted token.       |
| `cookieMaxAgeSeconds` | `3600`             | Cookie lifetime in seconds.                                     |
| `sameSite`            | `lax`              | Cookie `SameSite` value: `strict`, `lax`, or `none`.            |

When `csrf` is omitted, Xeno does not add either CSRF middleware. Adding CSRF
configuration enables both token-cookie handling and validation; there is no
separate `enabled` flag.

## How the Token Flow Works

Xeno registers two middleware stages when CSRF is configured:

```text
Request context
    │
    ├── AuthenticationMiddleware establishes identity
    │
    ├── CsrfCookieMiddleware
    │       └── after the response, sets a user-bound CSRF cookie if absent
    │
    └── CsrfMiddleware
            └── validates state-changing requests before continuing
```

The order matters: authentication runs before the CSRF middleware stages and
stores the identity in the `RequestContext`, so token generation and validation
use the authenticated `userId`. In the Node.js context, `AsyncLocalStorage`
carries this request-specific context through asynchronous middleware execution.

The normal browser flow is:

1. The user authenticates with the application.
2. The browser makes an authenticated request, typically a `GET` that loads the
   application or its session.
3. If the request context contains an authenticated `userId` and no CSRF cookie
   was received, `CsrfCookieMiddleware` generates a token and appends a
   `Set-Cookie` header to the response.
4. The client reads the CSRF cookie value.
5. For each `POST`, `PUT`, `PATCH`, or `DELETE`, the browser sends the cookie
   automatically and the client copies the same token into the configured
   header.
6. `CsrfMiddleware` checks the cookie, header, user identity, and token
   signature before calling the next middleware.

The cookie middleware runs after the downstream request finishes. A first
state-changing request without a token is therefore rejected; start by making an
authenticated request that can establish the CSRF cookie, then submit the state
change.

The cookie middleware does not replace an existing CSRF cookie when a user is
already authenticated. When a browser switches accounts, clear the old CSRF
cookie and make an authenticated request as the new user to receive a token
bound to that identity.

## How Xeno Validates Tokens

For `POST`, `PUT`, `PATCH`, and `DELETE`, Xeno performs these checks:

1. Reads the token from the configured request header and from the configured
   cookie.
2. Returns `403 Forbidden` if either value is missing.
3. Returns `403 Forbidden` if the header and cookie values differ.
4. Requires an authenticated identity with a `userId`.
5. Validates the token signature against that user ID and the configured secret.
6. Calls the next middleware only when every check succeeds.

Xeno generates a token from a random nonce and an HMAC-SHA256 signature bound to
the `userId`. Validation compares the supplied signature in constant time. The
token is stateless: the service does not need to store a separate token record
for each request.

Missing tokens, mismatched tokens, missing identity, or an invalid signature
produce `403 Forbidden`; the response error details distinguish the failure.
`GET`, `HEAD`, and `OPTIONS` are not checked by `CsrfMiddleware`.

## Send Tokens from a Browser

The CSRF cookie is not marked `HttpOnly`, so browser application code can read
its value and echo it in the request header. The cookie value is URI-encoded
when Xeno writes it; decode it before setting the header:

```ts
function readCookie(name: string): string | undefined {
  const prefix = `${name}=`
  const cookie = document.cookie
    .split(';')
    .map((part) => part.trim())
    .find((part) => part.startsWith(prefix))

  return cookie ? decodeURIComponent(cookie.slice(prefix.length)) : undefined
}

const csrfToken = readCookie('__Host-xeno-csrf')
if (!csrfToken) {
  throw new Error(
    'CSRF cookie is missing; make an authenticated request to establish it.',
  )
}

const response = await fetch('https://api.example.com/api/v1/orders', {
  method: 'POST',
  credentials: 'include',
  headers: {
    'Content-Type': 'application/json',
    'x-csrf-token': csrfToken,
  },
  body: JSON.stringify({ productId: 'item-123', quantity: 1 }),
})
```

Replace the URL and request body with the application's endpoint and input. The
header name in the client must match `csrf.headerName`; the cookie name must
match `csrf.cookieName`.

For cross-origin browser requests, configure CORS with the exact client origin
and credentials enabled, and use `credentials: 'include'` in `fetch`. Xeno
includes `x-csrf-token` in its standard preflight allow-header list.
Cross-origin cookies are also subject to browser cookie rules: the cookie is
always `Secure`, and a genuinely cross-site setup may require `sameSite: 'none'`
and HTTPS.

The default cookie is host-only because Xeno does not set a `Domain` attribute.
JavaScript on a different hostname cannot read that cookie, even when the
frontend and API are subdomains of the same site. Use a same-host deployment or
a same-origin frontend/API proxy when the browser client must read and echo the
cookie; configuring CORS alone does not grant JavaScript access to another
host's cookie.

## Configure Cookie Attributes

By default, Xeno writes a cookie with these attributes:

```http
Set-Cookie: __Host-xeno-csrf=<encoded-token>; Path=/; Secure; SameSite=lax; Max-Age=3600
```

The cookie uses `Path=/`, always includes `Secure`, and does not include
`HttpOnly` or `Domain`. Configure the token lifetime and `SameSite` behavior
through `csrf.cookieMaxAgeSeconds` and `csrf.sameSite`. The `__Host-` prefix is
intended for secure host-only cookies; use HTTPS in deployments that rely on
this cookie.

Choose `SameSite` based on the browser deployment:

- `lax` is the default and is suitable for many same-site applications.
- `strict` applies a tighter cross-site sending policy and may affect navigation
  flows.
- `none` allows cross-site cookie use in browsers, but requires `Secure`; Xeno
  always sets `Secure`.

`SameSite` is an additional browser control, not a replacement for token
validation. The middleware also requires a matching header token and a valid
user-bound signature.

## Protect or Exempt Individual Routes

When CSRF is configured, `POST`, `PUT`, `PATCH`, and `DELETE` routes require a
valid token by default. You do not need to add `withCsrf: true` to each
protected route.

An application can explicitly disable the CSRF check for a route with
`security.withCsrf: false` or the `withoutCsrf()` helper:

```ts
import { withoutCsrf } from '@xeno-js/core'
import type { NativeRouteDefinition } from '@xeno-js/shared'
import type { MyRegistry } from './registry'

declare const webhookRoute: NativeRouteDefinition<MyRegistry>

const routes: NativeRouteDefinition<MyRegistry>[] = [withoutCsrf(webhookRoute)]
```

`webhookRoute` represents a route already defined by the application. Exempt a
route only when its security model does not rely on ambient browser credentials,
such as a webhook authenticated by a separately verified signature. Do not
exempt a route merely to work around a missing or misconfigured CSRF token.

## Common Issues

### A state-changing request returns `403 Forbidden` with “CSRF token missing”

Check that the browser first made an authenticated request that established the
CSRF cookie. Then verify that the browser sends the cookie and the configured
header on the state-changing request. For cross-origin calls, include
`credentials: 'include'` and configure CORS credentials and the exact client
origin.

### The request returns “CSRF token mismatch”

The header and cookie must contain exactly the same decoded token. Confirm that
the client reads the configured cookie name, decodes its URI-encoded value, and
sends it using the configured header name.

### The request returns “CSRF token invalid”

Check that the request has an authenticated `userId`, that the token belongs to
the current identity, and that every application instance uses the same
`CSRF_SECRET`. A token issued to a different user or signed with a different
secret fails validation.

### The browser does not send the CSRF cookie

Check the cookie's `Secure`, `SameSite`, and `Max-Age` attributes and the
browser's cookie policy. Cross-origin requests need credentials enabled in both
the browser request and CORS response. Cross-site requests may require
`sameSite: 'none'` over HTTPS.

### The frontend cannot read the cookie

The CSRF cookie is host-only and Xeno does not configure a `Domain` attribute. A
page on a different hostname cannot access it through `document.cookie`; use a
same-host frontend/API setup or a same-origin proxy.
`Access-Control-Allow-Origin` does not change cookie domain scoping.

### A route does not require a CSRF token

Check that `csrf` is configured in `addMiddlewares()` and that the route does
not explicitly set `security.withCsrf` to `false` or use `withoutCsrf()`. Only
`POST`, `PUT`, `PATCH`, and `DELETE` are validated by this middleware.

## Related Documentation

- [Authentication Overview](../authentication/overview)
- [Allow-Origin Middleware](./allow-origin)
- [CORS Middleware](./cors-middleware)
- [OPTIONS Middleware](./options-middleware)
- [Security Overview](../overview)

---

---

<a id="source-11"></a>

# Source: `docs/security/middlewares/options-middleware.md`

## Introduction

Browsers send a CORS preflight request using the HTTP `OPTIONS` method before
some cross-origin requests. The request asks whether the target resource accepts
the intended method and headers.

Xeno.JS handles these requests with `OptionsMiddleware`. It returns a
`204 No Content` response with the allowed methods and headers instead of
forwarding that `OPTIONS` request through the rest of the middleware chain. The
middleware is configured through `XenoAppBuilder.addMiddlewares()`.

## Enable Preflight Handling

`XenoAppBuilder` enables preflight handling by default. It runs only when
preflight handling is enabled and at least one route has been configured with
`addRoutes()`:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import type { NativeRouteDefinition } from '@xeno-js/shared'

declare const routes: NativeRouteDefinition[]

const app = new XenoAppBuilder()

app.addRoutes(routes)
app.addMiddlewares((options) => {
  options.cors = {
    enablePreflight: true,
    enabled: true,
    allowOrigins: ['https://app.example.com'],
    allowHeaders: ['X-Request-Id', 'X-Client-Version'],
  }
})

const application = await app.build()
```

In this example, `routes` is the application's `NativeRouteDefinition[]`.
Register the real route definitions used by your application; without routes,
Xeno does not add `OptionsMiddleware` to the pipeline.

`cors.enablePreflight` and `cors.enabled` control separate middleware:

- `enablePreflight` enables the `OPTIONS` preflight handler.
- `enabled` enables the regular CORS middleware for other requests.

For browser-based cross-origin requests, configure both unless you intentionally
want preflight handling without the regular CORS middleware.

## Preflight Response

For an `OPTIONS` request, Xeno:

1. Reads `Access-Control-Request-Method`; if it is missing, it checks the route
   policy as though the requested method were `GET`.
2. Gets the allowed methods for the request path from the registered routes.
3. Checks the route's CORS policy for origin and credentials settings.
4. Returns `204 No Content` with `Access-Control-Allow-Methods` and
   `Access-Control-Allow-Headers`.

The allowed request headers include Xeno's standard headers and the headers
configured in `cors.allowHeaders`. Xeno removes duplicate entries before
returning them.

The middleware does not dynamically copy `Access-Control-Request-Headers` from
the browser. It returns the configured allow-header list, so add every
application-specific request header to `cors.allowHeaders`.

The origin and credential values come from the route policy when defined;
otherwise Xeno uses the middleware-level CORS configuration. A wildcard origin
is returned as `*` only when credentials are disabled. When credentials are
enabled, Xeno returns the request origin instead.

The `Allow-Origin` middleware runs before the OPTIONS handler and validates the
incoming origin against the route policy or configured allowed origins.

## Configure Allowed Methods and Headers

Allowed methods come from the methods registered for each route path. Make sure
that the route definitions include the method the browser intends to call. The
browser sends that method in the `Access-Control-Request-Method` header; the
OPTIONS response lists the methods registered for the path.

Add application-specific headers through `cors.allowHeaders`:

```ts
app.addMiddlewares((options) => {
  options.cors = {
    enabled: true,
    enablePreflight: true,
    allowOrigins: ['https://app.example.com'],
    allowHeaders: [
      'Content-Type',
      'Authorization',
      'X-Request-Id',
      'X-Client-Version',
    ],
  }
})
```

Xeno also adds standard headers such as `Content-Type`, `Authorization`,
`Origin`, and its request tracing headers. You do not need to repeat them unless
doing so improves clarity.

## Common Issues

### The `OPTIONS` request reaches the application handler

Check that `cors.enablePreflight` is `true` and that the application has
registered routes through `addRoutes()`. If either condition is missing, Xeno
does not add `OptionsMiddleware`.

### The browser reports that a method is not allowed

Check that the requested method is registered for the same path used by the
preflight request. Xeno builds the `Access-Control-Allow-Methods` value from the
registered route methods; `cors.allowHeaders` does not configure methods.

The preflight response advertises methods; it does not execute the requested
application route. The actual request must still be handled by the HTTP server
and route, and must pass the application's middleware and authorization checks.

### The browser reports that a request header is not allowed

Add the requested header name to `cors.allowHeaders`. Header names are returned
as a comma-separated list in `Access-Control-Allow-Headers`.

### The origin is missing or rejected

Check `cors.allowOrigins` and any route-specific CORS policy. When the
application-level origin list is empty, Xeno derives a default from `APP_URL`,
or from `http://localhost:<PORT>` when `APP_URL` is not set. An origin rejected
by the origin-validation middleware does not proceed to the OPTIONS handler.

### Credentialed requests fail with a wildcard origin

Browsers do not accept `Access-Control-Allow-Origin: *` for credentialed
requests. Configure explicit allowed origins and enable credentials where
required. Xeno responds with the request origin rather than `*` when credentials
are enabled.

### Preflight works, but the actual request still fails

Preflight handling does not replace the regular CORS middleware. Check
`cors.enabled`, the configured origin policy, and the actual response path. Also
ensure that the application route and HTTP server handle the requested method.

## Related Documentation

- [Security Overview](../overview)
- [Request Context Middleware](./request-context-midleware)
- [Rate Limiting](./rate-limit)

---

---

<a id="source-12"></a>

# Source: `docs/security/middlewares/overview.md`

## Introduction

HTTP middleware applies cross-cutting behavior to requests before and after
route handlers run. Examples include creating request context, checking origins,
authenticating credentials, limiting request rates, and validating CSRF tokens.

In Xeno.JS, `XenoAppBuilder.addMiddlewares()` queues and configures the built-in
HTTP middleware module. The builder resolves queued modules during `build()`,
creates a middleware pipeline, and connects it to the HTTP dispatcher.
Application code configures this stack through `MiddlewareConfig`;
`addMiddlewares()` is not an API for inserting an arbitrary middleware class
into the pipeline.

Middleware configuration belongs to the HTTP boundary. It does not replace route
registration, authentication service setup, or application-layer authorization.

## Add the Middleware Stack

Call `addMiddlewares()` before `build()`. Pass a setup callback to configure its
options:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withSupabase } from '@xeno-js/supabase'
import { ordersRoutes } from './order/order.route'

const app = new XenoAppBuilder()

app.addRoutes(ordersRoutes)
app.addAuth(
  withSupabase((options, config) => {
    options.url = config.getOrThrow('SUPABASE_URL')
    options.key = config.getOrThrow('SUPABASE_KEY')
  }),
)
app.addMiddlewares((options, config) => {
  options.cors = {
    enabled: true,
    enablePreflight: true,
    allowOrigins: [config.getOrThrow('CLIENT_URL')],
    allowHeaders: ['X-Client-Version'],
  }
  options.rateLimit = {
    enabled: true,
    maxRequests: 100,
    windowSeconds: 60,
  }
})

const application = await app.build()
```

`ordersRoutes` represents the application's `NativeRouteDefinition[]`.
`addRoutes()` and `addAuth()` queue their modules; `addMiddlewares()` queues the
middleware module. During `build()`, Xeno resolves queued modules by priority
and constructs the HTTP pipeline. Configure routes, authentication, and
middleware before building the application.

The builder queues the middleware module only once. Subsequent calls to
`addMiddlewares()` return the same builder without applying another setup
callback. Put the complete middleware configuration in a single call.

Without a callback, `addMiddlewares()` uses the builder's defaults. When
authentication is not configured, the stack still includes an authentication
middleware, but it uses `NoAuthGateKeeper` and establishes a guest identity
rather than authenticating credentials.

## Middleware Pipeline

When configured, the HTTP middleware pipeline runs in this order:

```text
RequestContextMiddleware
    ↓
CookieResponseMiddleware              (when isSSR is true)
    ↓
AllowOriginMiddleware
    ↓
OptionsMiddleware                    (when preflight is enabled and routes exist)
    ↓
MethodCheckMiddleware                (when routes exist)
    ↓
CORSMiddleware                       (when cors.enabled is true)
    ↓
RateLimitMiddleware                  (when rateLimit.enabled is true)
    ↓
AuthenticationMiddleware
    ↓
CsrfCookieMiddleware                 (when csrf is configured)
    ↓
CsrfMiddleware                       (when csrf is configured)
    ↓
Route handler
```

The pipeline unwinds after the route returns. Response middleware can then
append headers or cookies. For example, `CORSMiddleware` adds CORS response
headers after the downstream response is available, and
`CookieResponseMiddleware` appends cookies queued during the request.

Some stages act as request gates and stop processing when a policy fails:

- origin validation returns `403 Forbidden` when an origin is denied;
- method checking returns `405 Method Not Allowed` when the request reaches the
  middleware with a disallowed method;
- rate limiting returns `429 Too Many Requests` after the configured limit is
  exceeded;
- CSRF validation returns `403 Forbidden` when a required token is missing,
  mismatched, or invalid.

For regular HTTP server requests, the route matcher selects a route by path and
method before the handler invokes the middleware pipeline. If no route matches,
the HTTP adapter returns `404 Not Found`; that request does not reach the
pipeline.

## Middleware Options

`addMiddlewares()` receives a setup callback with the middleware options and the
application's `IConfigurationService`:

```ts
app.addMiddlewares((options, config) => {
  options.cors = {
    allowOrigins: [config.getOrThrow('CLIENT_URL')],
  }
})
```

The following options are available.

### CORS options

| Option                 | Default                                                                 | Behavior                                                                                                                                      |
| ---------------------- | ----------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `cors.enabled`         | `true`                                                                  | Adds `CORSMiddleware`, which adds CORS response headers after the request continues through the pipeline.                                     |
| `cors.enablePreflight` | `true`                                                                  | Adds `OptionsMiddleware` when routes exist. It handles browser `OPTIONS` preflight requests.                                                  |
| `cors.withCredentials` | `true`                                                                  | Sets the default `Access-Control-Allow-Credentials` value. A route-specific `security.withCredentials` policy takes precedence when defined.  |
| `cors.allowOrigins`    | `[]`                                                                    | Application-level origin allowlist. When empty or omitted, Xeno uses `APP_URL`, or `http://localhost:<PORT>` with `3000` as the default port. |
| `cors.allowHeaders`    | `Content-Type`, `Authorization`, `X-Requested-With`, `Accept`, `Origin` | Adds application-specific headers to the list returned by preflight handling.                                                                 |

`cors.allowOrigins` and `cors.enabled` control different behavior. The
origin-validation middleware checks requests independently of `cors.enabled`;
turning off CORS response headers does not bypass origin validation.

`cors.allowHeaders` is used by `OptionsMiddleware` to build
`Access-Control-Allow-Headers`. Xeno also includes standard headers such as
`x-csrf-token`, `Authorization`, `Content-Type`, `Origin`, request-tracing
headers, and other framework headers. It combines the built-in and configured
entries and removes duplicates. Add each application-specific request header,
such as `X-Client-Version`, to `allowHeaders`; Xeno does not dynamically copy
the browser's `Access-Control-Request-Headers`.

Allowed methods come from the methods registered with `addRoutes()`, not from a
CORS option. For cross-origin browser requests, the allowed origin, requested
method, and request headers must all be covered. See
[CORS Middleware](./cors-middleware), [Allow-Origin Middleware](./allow-origin),
and [OPTIONS Middleware](./options-middleware).

### Rate-limit options

| Option                    | Default | Behavior                                                         |
| ------------------------- | ------- | ---------------------------------------------------------------- |
| `rateLimit.enabled`       | `true`  | Enables the request rate limiter.                                |
| `rateLimit.maxRequests`   | `100`   | Maximum requests allowed for a rate-limit key during the window. |
| `rateLimit.windowSeconds` | `60`    | Rate-limit window duration in seconds.                           |

The rate limiter uses the configured cache. If rate limiting is enabled and no
cache service has been registered, Xeno configures its in-memory cache. An
in-memory cache is local to one process; multi-instance deployments need a
shared cache to share counters. See [Rate Limiting](./rate-limit).

### CSRF options

CSRF protection is disabled by default. Supplying `options.csrf` enables both
the cookie-issuing middleware and the state-changing request validator.

| Option                     | Default            | Behavior                                                                                                                     |
| -------------------------- | ------------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| `csrf.secret`              | Required           | Server-side secret used to sign and validate user-bound tokens. Keep it private and consistent across application instances. |
| `csrf.cookieName`          | `__Host-xeno-csrf` | Cookie that carries the CSRF token.                                                                                          |
| `csrf.headerName`          | `x-csrf-token`     | Request header that must echo the cookie token.                                                                              |
| `csrf.cookieMaxAgeSeconds` | `3600`             | CSRF cookie lifetime in seconds.                                                                                             |
| `csrf.sameSite`            | `lax`              | Cookie `SameSite` policy: `strict`, `lax`, or `none`.                                                                        |

The middleware validates `POST`, `PUT`, `PATCH`, and `DELETE` requests. It
requires matching cookie and header values, an authenticated `userId`, and a
valid signature. A route can explicitly opt out with `security.withCsrf: false`.
See [CSRF Middleware](./csrf-middleware).

### Request and cookie options

| Option            | Default           | Behavior                                                                                                                                                            |
| ----------------- | ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `isSSR`           | `false`           | Adds `CookieResponseMiddleware` to append pending SSR cookies from the request-scoped cookie handler to the response. It does not register a cookie handler itself. |
| `trustedIpHeader` | `undefined`       | Selects the header from which the client IP is extracted when the connecting proxy is trusted. The extractor otherwise checks `x-forwarded-for`, then `x-real-ip`.  |
| `trustedProxies`  | `[]`              | Exact socket IP addresses of trusted proxies. Xeno uses a forwarded IP only when the connecting socket address matches an entry.                                    |
| `authCookieName`  | `sb-access-token` | Cookie name used by the default bearer-token extractor when no custom token extractor is registered.                                                                |

Only list proxies you control. `trustedProxies` currently matches exact
addresses; it does not interpret CIDR ranges or `'*'`. See
[Request Context Middleware](./request-context-midleware) for client IP
resolution and [Cookie Response Middleware](./cookie-middleware) for SSR cookie
handling.

## Configuration Example

This example shows a typical browser-facing API configuration. Keep secrets in
protected application configuration, and replace the example application values
with the deployment's actual origins and routes:

```ts
app.addMiddlewares((options, config) => {
  options.cors = {
    enabled: true,
    enablePreflight: true,
    withCredentials: true,
    allowOrigins: [config.getOrThrow('CLIENT_URL')],
    allowHeaders: ['X-Client-Version'],
  }

  options.rateLimit = {
    enabled: true,
    maxRequests: 60,
    windowSeconds: 60,
  }

  options.csrf = {
    secret: config.getOrThrow('CSRF_SECRET'),
    cookieName: '__Host-xeno-csrf',
    headerName: 'x-csrf-token',
    cookieMaxAgeSeconds: 3600,
    sameSite: 'lax',
  }

  options.trustedIpHeader = 'x-forwarded-for'
  options.trustedProxies = ['10.0.0.5']
})
```

Set CSRF only when the application uses cookie-based credentials and requires
CSRF protection. Set trusted proxy options only when the listed proxy is the
trusted network peer that connects to the application.

## Common Issues

### Middleware settings seem to have no effect

Check that `addMiddlewares()` runs before `build()` and that the options are
configured in its callback. The builder queues the middleware module once; later
calls do not apply another setup callback.

### A custom request header fails during browser preflight

Add the header to `cors.allowHeaders`. The preflight handler advertises Xeno's
standard headers plus the configured list; it does not reflect arbitrary headers
from the incoming preflight request.

### Changing `cors.enabled` does not stop an origin rejection

This is expected. `AllowOriginMiddleware` performs request-origin validation
independently from `CORSMiddleware`, which adds response headers. Configure the
origin allowlist or route-specific policy rather than relying on `cors.enabled`.

### A route returns `404` instead of a middleware error

The route matcher selects the path and method before the handler enters the
middleware pipeline. Register the route with `addRoutes()` or correct the
requested path and method.

### All clients behind a proxy share one rate-limit key

Check that the connecting proxy's socket IP exactly matches an entry in
`trustedProxies` and that the proxy supplies the configured client IP header. Do
not trust arbitrary forwarded headers from untrusted clients.

## Middleware Guides

- [Allow-Method Middleware](./allow-method)
- [Allow-Origin Middleware](./allow-origin)
- [Cookie Response Middleware](./cookie-middleware)
- [CORS Middleware](./cors-middleware)
- [CSRF Middleware](./csrf-middleware)
- [OPTIONS Middleware](./options-middleware)
- [Rate Limiting](./rate-limit)
- [Request Context Middleware](./request-context-midleware)

## Related Documentation

- [Security Overview](../overview)
- [Authentication Overview](../authentication/overview)
- [App Builder](../../fundamentals/app-builder)

---

---

<a id="source-13"></a>

# Source: `docs/security/middlewares/rate-limit.md`

## Introduction

`RateLimitMiddleware` counts requests using the configured cache and returns
HTTP `429 Too Many Requests` when a client exceeds its limit. The limit is
configured through `XenoAppBuilder.addMiddlewares()`.

The middleware builds a cache key for each request resource. Depending on the
context data available, the key is scoped to the tenant and user, tenant and
client IP, user, or client IP. When no user or client IP is available, the
middleware cannot build a key and returns `503 Service Unavailable`.

## Configure the Limit

`XenoAppBuilder` enables rate limiting by default, with 100 requests per 60
seconds. Configure the maximum and time window through the middleware options:

```ts
import { XenoAppBuilder } from '@xeno-js/core'

const app = new XenoAppBuilder()

app.addMiddlewares((options) => {
  options.rateLimit = {
    enabled: true,
    maxRequests: 60,
    windowSeconds: 60,
  }
})

const application = await app.build()
```

`maxRequests` is the maximum number of requests allowed for a cache key during
the window. The middleware returns `429` when the request count exceeds that
number. The response includes a `Retry-After` header set to `windowSeconds`.

To disable rate limiting:

```ts
app.addMiddlewares((options) => {
  options.rateLimit = {
    enabled: false,
  }
})
```

## How Xeno Builds Rate-Limit Keys

The key builder checks the current request context in this order:

1. Tenant and user ID: `ratelimit:tenant:<tenantId>:user:<userId>:<path>`
2. Tenant and client IP: `ratelimit:tenant:<tenantId>:ip:<clientIp>:<path>`
3. User ID: `ratelimit:user:<userId>:<path>`
4. Client IP: `ratelimit:ip:<clientIp>:<path>`

The resource portion is the request path. Different paths therefore have
separate counters. The middleware stack runs rate limiting before
authentication, so do not assume that the rate-limit key will contain an
identity established by the authentication middleware for that same request. For
requests without an identity available at that point, the key uses the client
IP.

## Configure Client IP Behind a Reverse Proxy

When Xeno.JS runs behind a reverse proxy, `req.socket.remoteAddress` may
identify the proxy rather than the originating client. If the proxy IP is not
trusted, the client IP resolver uses that socket address. Many users can then
share one IP-based cache key and exhaust the same rate limit.

To use a forwarded client-IP header safely, configure the header name and the
exact socket IP address of the proxy that connects to the application:

```ts
app.addMiddlewares((options) => {
  options.trustedIpHeader = 'x-forwarded-for'
  options.trustedProxies = ['10.0.0.5']

  options.rateLimit = {
    enabled: true,
    maxRequests: 100,
    windowSeconds: 60,
  }
})
```

Replace `10.0.0.5` with the actual address observed as the incoming socket's
remote address in your deployment. `trustedProxies` is an exact list of
addresses; the current resolver does not support CIDR ranges or a wildcard. In
particular, `trustedProxies: ['*']` does **not** make every proxy trusted.

Only trust a proxy that you control and that reliably removes or overwrites
client-supplied forwarding headers. Otherwise a client may forge the forwarded
IP and evade IP-based limits. Follow your hosting provider's proxy-header and
network configuration when identifying which proxy address reaches your
application.

The core extractor uses `x-forwarded-for` by default if `trustedIpHeader` is
omitted, and falls back to `x-real-ip` when that header is absent. Setting
`trustedIpHeader` selects the header to extract; it does not by itself make a
proxy trusted.

## Common Issues

### All users behind the proxy are rate-limited together

Check that the socket IP reported by the application matches an entry in
`trustedProxies`. If it does not, the resolver uses the socket IP for all
requests arriving through that proxy. Also verify that the configured forwarded
header is populated by the proxy and contains the expected client address.

Do not use `['*']` as a workaround: it is treated as a literal IP address, not a
wildcard.

### Each application instance has a separate counter

If you do not configure a cache provider, Xeno.JS adds the default in-memory
cache when rate limiting is enabled. That cache is local to the application
process. In a deployment with multiple server instances, each instance can
maintain its own counter, so the effective limit is not shared globally.

Configure a shared cache provider, such as Redis, when instances need to use the
same counters. See [Redis Cache](../../cache/redis) for cache setup.

### A `503 Service Unavailable` response appears

The rate-limit middleware returns `503` when it cannot build a key because
neither a user identity nor a client IP is available in the request context.
Check the context and proxy setup, and ensure the request reaches the middleware
with a resolvable client IP.

### Legitimate users receive `429 Too Many Requests`

Review the configured `maxRequests` and `windowSeconds`, and confirm that
unrelated clients are not sharing the same resolved IP address. Also check
whether clients behind NAT or a corporate proxy legitimately share one public
IP; in that case, an IP-based key can apply a shared budget to those users.

## Related Documentation

- [Request Context Middleware](./request-context-midleware)
- [Context and Scopes](../../fundamentals/context-scopes)
- [Cache Overview](../../cache/overview)
- [Redis Cache](../../cache/redis)

---

---

<a id="source-14"></a>

# Source: `docs/security/middlewares/request-context-midleware.md`

## Introduction

The request context makes request-specific metadata available while Xeno.JS
processes an HTTP request. The core `RequestContextMiddleware` extracts metadata
from the request headers, resolves the client IP, creates the context, and runs
the rest of the middleware and route pipeline inside it.

The setup depends on the runtime:

- **Node.js HTTP application:** call `addMiddlewares()`. The core middleware
  module adds `RequestContextMiddleware`; the default Node context uses
  `AsyncLocalStorage`.
- **Vue application:** configure the context with
  `addContext(withPiniaContext())`. The Vue package provides a Pinia-backed
  implementation instead of the Node context.

These are different context implementations. `withPiniaContext()` configures the
context provider; it does not install the core HTTP middleware stack.

## Enable Request Context in Node.js

`XenoAppBuilder` uses the Node configuration service by default. Enable the HTTP
middleware stack with `addMiddlewares()`:

```ts
import { XenoAppBuilder } from '@xeno-js/core'

const app = new XenoAppBuilder()

app.addMiddlewares()

const application = await app.build()
```

During `build()`, the builder adds the default Node context if you have not
configured another context. The middleware module then creates the core
`RequestContextMiddleware` and places it first in the HTTP middleware pipeline.
You do not need to construct or register that middleware yourself.

The middleware creates correlation, request, and span identifiers when the
incoming request does not provide them. It also extracts request metadata and
the client IP before calling the next middleware. The Node context carries this
data through asynchronous work with `AsyncLocalStorage` and manages the service
scope for the request.

You can configure the rest of the HTTP middleware stack through the
`addMiddlewares()` callback:

```ts
import { XenoAppBuilder } from '@xeno-js/core'

const app = new XenoAppBuilder()

app.addMiddlewares((options) => {
  options.trustedProxies = ['10.0.0.5']
  options.trustedIpHeader = 'x-forwarded-for'
})

const application = await app.build()
```

Set trusted proxy options only when the application runs behind proxies you
control. Each `trustedProxies` entry must match a proxy's socket IP address; the
resolver does not interpret CIDR ranges. These options affect how the middleware
resolves the client IP; they do not enable request context by themselves.

## Configure Request Context in Vue

Vue applications use the context implementation from `@xeno-js/vue`. Select the
Vite configuration service and pass the plugin to `addContext()`:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withPiniaContext } from '@xeno-js/vue'

const app = new XenoAppBuilder(undefined, 'vite')

app.addContext(withPiniaContext())

const application = await app.build()
```

`withPiniaContext()` supplies a Vue-specific `IRequestContext` to the core
`ContextModule`. The implementation reads and updates request and identity state
through the Vue Pinia context store; it does not use Node's `AsyncLocalStorage`.

The builder does not substitute the Node context when configured for Vite. Add
the Vue context plugin before calling `build()`. If the Vue application also
hosts a Node HTTP server, configure that server's HTTP middleware separately;
`addContext(withPiniaContext())` only selects the context implementation.

Ensure Pinia is installed and active before the Vue context store is used.

The plugin accepts an optional request-ID reuse window:

```ts
app.addContext(withPiniaContext({ requestIdWindowMs: 3000 }))
```

The value configures the Vue context store. Omit the option to use the store's
default.

## Access the Current Context

After the context module has been configured, resolve the context accessor from
the built application:

```ts
const { contextAccessor } = application.getServices()

const context = contextAccessor.getContext()
```

For Node.js, the accessor reads the context active in the current asynchronous
execution. In Vue, the Vue implementation reads it from the Pinia context store.

See [Context and Scopes](../../fundamentals/context-scopes) for accessing
identity and scoped services, and [App Builder](../../fundamentals/app-builder)
for the application bootstrap lifecycle.

---

---

<a id="source-15"></a>

# Source: `docs/security/overview.md`

## Introduction

Xeno.JS provides security controls at two different boundaries:

- the **HTTP boundary**, where incoming requests are inspected and
  authenticated;
- the **application boundary**, where authenticated identities are authorized to
  execute application requests.

These responsibilities are deliberately separate.

```text
HTTP request
    │
    ├── Request context / metadata
    │
    ├── Origin validation
    │
    ├── OPTIONS handling
    │
    ├── HTTP method validation
    │
    ├── CORS
    │
    ├── Rate limiting
    │
    ├── Authentication
    │       └── credential → Identity
    │
    ├── CSRF validation
    │
    ▼
Application request
    │
    └── Authorization
            ├── user ID
            ├── tenant ID
            ├── roles
            ├── permissions
            └── custom strategies
```

The security documentation therefore follows the same separation:

```text
security/
├── authentication/
├── authorization/
├── middleware/
└── helpers/
```

---

## Configure security

### How do I enable Xeno.JS security middleware?

The HTTP security layer is enabled through `AppBuilder.addMiddlewares()`.

A minimal authenticated application looks like this:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withSupabase } from '@xeno-js/supabase'

const app = new XenoAppBuilder()

app.addAuth(
  withSupabase((options, config) => {
    options.url = config.get('SUPABASE_URL')!
    options.key = config.get('SUPABASE_KEY')!
  }),
)

app.addMiddlewares()

const application = await app.build()
```

There are two important calls:

```ts
app.addAuth(...)
```

configures the authentication service.

```ts
app.addMiddlewares(...)
```

installs the HTTP middleware stack and connects it to the authentication service
when authentication has been configured.

If `addMiddlewares()` is called without `addAuth()`, Xeno configures the
authentication middleware with a `NoAuthGateKeeper`, which returns a guest
identity instead of authenticating credentials.

---

## Authentication vs authorization

These two mechanisms solve different problems.

### Authentication

Authentication answers:

> Who is making this request?

The HTTP authentication middleware:

1. extracts a credential;
2. passes it to the configured gatekeeper;
3. authenticates the credential;
4. maps the resulting claims to an identity;
5. stores the identity in the request context.

The flow is:

```text
HTTP credential
      │
      ▼
Token extractor
      │
      ▼
AuthenticationMiddleware
      │
      ▼
GateKeeper
      │
      ▼
Authentication service
      │
      ▼
Identity
      │
      ▼
Request context
```

If authentication fails, `AuthenticationMiddleware` returns an error response
instead of continuing the middleware chain, using the authentication error's
status or `401 Unauthorized` by default.

See:

- [Authentication Overview](./authentication/overview)
- [Supabase Authentication](./authentication/supabase)
- [Custom Authentication](./authentication/custom)

---

## Authorization

Authorization answers:

> Is this authenticated identity allowed to execute this application request?

Authorization is configured through the CQRS pipeline:

```ts
import { withAuthZ } from '@xeno-js/core'

app.addPipeline(
  withAuthZ((opts) => {
    opts.policies = {
      CreateInvoice: {
        userId: true,
        tenantId: true,
        roles: ['admin', 'billing'],
        permissions: ['invoice:create'],
      },
    }
  }),
)
```

The authorization pipeline evaluates the configured strategies before passing
the request to the next application pipeline stage.

```text
Authenticated Identity
        │
        ▼
AuthorizationPipeline
        │
        ├── user ID strategy
        ├── tenant ID strategy
        ├── role strategy
        ├── permission strategy
        └── custom strategies
        │
        ▼
Application handler
```

Strategies run sequentially. If one fails, the authorization pipeline stops and
returns that failure.

See:

- [Authorization Overview](./authorization/overview)
- [Authorization Policies](./authorization/policies)
- [Custom Authorization](./authorization/custom)

---

## HTTP security middleware

`addMiddlewares()` builds a composite HTTP middleware containing the security
controls configured for the application.

The current middleware composition is:

```text
Request context
      │
      ▼
SSR cookie response handling  (when isSSR is enabled)
      │
      ▼
Origin validation
      │
      ▼
OPTIONS handling       (when CORS preflight is enabled and routes are configured)
      │
      ▼
HTTP method checks     (when routes are configured)
      │
      ▼
CORS                   (when enabled)
      │
      ▼
Rate limiting
      │
      ▼
Authentication
      │
      ▼
CSRF cookie generation and validation (when csrf is configured)
      │
      ▼
Application execution
```

Some stages are conditional.

| Control             | Enable by                                                              |
| ------------------- | ---------------------------------------------------------------------- |
| Request context     | `addMiddlewares()`                                                     |
| Origin validation   | `addMiddlewares()`                                                     |
| OPTIONS handling    | `cors.enablePreflight: true` and routes configured                     |
| CORS                | `cors.enabled: true`                                                   |
| HTTP method checks  | Routes supplied through `addRoutes()`                                  |
| Authentication      | `addMiddlewares()`; `addAuth()` supplies a real authentication service |
| Rate limiting       | `rateLimit.enabled: true`                                              |
| CSRF                | `csrf` configuration                                                   |
| SSR cookie response | `isSSR: true`                                                          |

The exact middleware chain is built by Xeno when the middleware module is
configured.

---

## Default security configuration

`XenoAppBuilder` starts with the following relevant middleware defaults:

```ts
{
  isSSR: false,

  rateLimit: {
    enabled: true,
    maxRequests: 100,
    windowSeconds: 60,
  },

  csrf: undefined,

  cors: {
    enabled: true,
    enablePreflight: true,
    withCredentials: true,
    allowOrigins: [],
    allowHeaders: ['Content-Type', 'Authorization', 'X-Requested-With', 'Accept', 'Origin'],
  },

  trustedIpHeader: undefined,

  trustedProxies: [],

  authCookieName: 'sb-access-token',
}
```

This means that some security behavior is already active without additional
configuration.

In particular:

- CORS is enabled;
- CORS preflight handling is enabled when routes are configured;
- credentials are enabled;
- origin validation is enabled;
- rate limiting has a default configuration;
- CSRF is not enabled until configured;
- route-specific method checks are enabled when routes are supplied through
  `addRoutes()`;
- `isSSR` enables the SSR cookie response middleware; it does not select an
  authentication token extractor.

---

## Configure authentication

Authentication is configured independently from the other HTTP controls.

For the built-in Supabase integration:

```ts
app.addAuth(
  withSupabase((options, config) => {
    options.url = config.get('SUPABASE_URL')!
    options.key = config.get('SUPABASE_KEY')!
  }),
)
```

The authentication service is registered before the middleware module is
configured.

When `addAuth()` has been called, `addMiddlewares()` connects
`AuthenticationMiddleware` to the configured authentication service.

Without `addAuth()`, the middleware uses `NoAuthGateKeeper`.

See [Authentication Overview](./authentication/overview).

---

## Configure CORS and origins

### Allowed origins

The middleware configuration accepts an explicit origin list:

```ts
app.addMiddlewares((options) => {
  options.cors = {
    allowOrigins: ['https://app.example.com'],
  }
})
```

If `options.cors.allowOrigins` is empty, Xeno derives the default origin from:

1. `APP_URL`, when configured;
2. otherwise `http://localhost:<PORT>`.

The origin is checked independently by the origin middleware.

Therefore:

```ts
options.cors.allowOrigins
```

controls **which request origins are accepted**, while:

```ts
options.cors.enabled
```

controls whether CORS response handling is enabled.

These are related but separate concerns.

---

### Credentials

Credentials are enabled by default:

```ts
options.cors = {
  withCredentials: true,
}
```

Avoid a wildcard allow-origin list when credentials are enabled. Xeno's origin
check accepts `'*'`; the CORS middleware responds with the requesting origin and
credentials enabled rather than rejecting that configuration. Prefer an explicit
allowlist:

```ts
options.cors = {
  allowOrigins: ['https://app.example.com'],
  withCredentials: true,
}
```

Use explicit origins when browser credentials are enabled.

---

## Configure CSRF protection

CSRF protection is opt-in.

Enable it by supplying `csrf` configuration:

```ts
app.addMiddlewares((options, config) => {
  options.csrf = {
    secret: config.get('CSRF_SECRET')!,
    cookieName: '__Host-xeno-csrf',
    headerName: 'x-csrf-token',
    cookieMaxAgeSeconds: 3600,
    sameSite: 'lax',
  }
})
```

The CSRF configuration supports:

```ts
interface CsrfConfig {
  secret: string
  cookieName?: string
  headerName?: string
  cookieMaxAgeSeconds?: number
  sameSite?: 'strict' | 'lax' | 'none'
}
```

When enabled, Xeno uses two middleware stages:

```text
CsrfCookieMiddleware
        │
        └── creates a user-bound CSRF cookie

CsrfMiddleware
        │
        └── validates state-changing requests
```

The CSRF middleware checks `POST`, `PUT`, `DELETE`, and `PATCH` requests. A
matching route policy with `withCsrf: false` exempts that route.

For these methods it requires:

1. a CSRF token in the configured request header;
2. a CSRF token in the configured cookie;
3. matching header and cookie values;
4. a valid token associated with the authenticated user.

The default cookie name is:

```text
__Host-xeno-csrf
```

The default token lifetime is 3600 seconds.

CSRF protection therefore complements authentication. The authentication
credential identifies the caller; the CSRF token protects cookie-authenticated
state-changing requests from forged requests.

See [CSRF Protection](./middleware/csrf).

---

## Configure rate limiting

Rate limiting is configured through:

```ts
app.addMiddlewares((options) => {
  options.rateLimit = {
    maxRequests: 30,
    windowSeconds: 60,
  }
})
```

The configuration controls:

- `maxRequests` — maximum number of requests;
- `windowSeconds` — duration of the rate-limit window.

When the limit is exceeded, the middleware returns:

```text
429 Too Many Requests
```

and includes:

```http
Retry-After: <window-seconds>
```

Rate limiting uses the configured atomic cache and a rate-limit key derived from
the request context and request path.

The default configuration is:

```ts
{
  enabled: true,
  maxRequests: 100,
  windowSeconds: 60,
}
```

Change this explicitly for production workloads according to the expected
traffic pattern.

See [Rate Limiting](./middleware/rate-limit).

---

## Restrict HTTP methods

Route-specific method validation is derived from the route definitions supplied
through `addRoutes()`:

```ts
app.addRoutes(routes)
```

When routes are configured, Xeno checks incoming methods against the methods
declared for each route.

OPTIONS preflight handling is configured under `cors` and is enabled by default:

```ts
app.addMiddlewares((options) => {
  options.cors = {
    enablePreflight: true,
  }
})
```

This is useful when browser clients require CORS preflight handling.

See [HTTP Methods](./middleware/methods).

---

## Configure trusted client IP information

Xeno can be configured to use a trusted IP header:

```ts
app.addMiddlewares((options) => {
  options.trustedIpHeader = 'x-forwarded-for'
  options.trustedProxies = ['10.0.0.10']
})
```

These settings affect how client IP information is resolved in the request
context.

Only configure proxy/header trust when the application infrastructure actually
controls the corresponding proxy path.

Do not blindly trust arbitrary client-supplied forwarding headers.

---

## SSR authentication

For server-side rendering applications, enable SSR behavior:

```ts
app.addMiddlewares((options) => {
  options.isSSR = true
})
```

`isSSR` enables the SSR cookie response middleware. It does not change the
default authentication extractor. Supabase SSR cookie extraction is configured
by the Supabase authentication integration; see
[Supabase Authentication](./authentication/supabase).

The normal extractor is:

```text
Authorization: Bearer <token>
        │
        ▼
BearerTokenExtractor
```

The Supabase SSR integration can process:

```text
Authorization header
        +
Supabase authentication cookies
```

including chunked authentication cookies.

See [Supabase Authentication](./authentication/supabase).

---

## Custom authentication

The authentication integration can be replaced with a custom authentication
service and credential extractor.

```ts
app.addAuth((options) => {
  options.tokenExtractor = (_scope) => new MyTokenExtractor()
  options.authService = (_scope) => new MyAuthService()
})
```

`authService` is required and is created per service scope. `tokenExtractor` is
optional and defaults to the bearer-and-cookie extractor; a custom extractor
factory is registered as a singleton.

The two components have separate responsibilities:

```text
Token extractor
    =
    "Where does the credential come from?"

Authentication service
    =
    "How is the credential authenticated?"
```

The resulting authenticated claims are mapped to the request identity.

See [Custom Authentication](./authentication/custom).

---

## Configure authorization

Authorization belongs to the application pipeline.

For example:

```ts
app.addPipeline(
  withAuthZ((opts) => {
    opts.policies = {
      CreateInvoice: {
        userId: true,
        tenantId: true,
        roles: ['admin', 'billing'],
        permissions: ['invoice:create'],
      },
    }
  }),
)
```

The policy can combine multiple strategy types.

```text
CreateInvoice
    │
    ├── user ID
    ├── tenant ID
    ├── roles
    └── permissions
```

Xeno creates only the strategy types required by the configured policies.

Custom authorization strategies can also be supplied.

Authorization runs in the CQRS/application pipeline, not in the HTTP middleware
stack.

See [Authorization Policies](./authorization/policies).

---

## User and tenant identity

Authentication creates the request identity.

Authorization strategies can then consume identity fields such as:

```text
userId
tenantId
roles
permissions
```

A policy such as:

```ts
{
  CreateInvoice: {
    tenantId: true,
  },
}
```

is an authorization check against the authenticated identity.

It should not be interpreted as automatic resource ownership checking.

For example, requiring a tenant ID does not by itself mean that Xeno compares:

```text
identity.tenantId
```

with:

```text
invoice.tenantId
```

Application-specific ownership rules belong in application authorization logic
or custom authorization strategies.

An intent without a matching policy requirement is not denied by default.
Configure a policy for each operation that must be restricted.

---

## Execute HTTP requests through the middleware stack

To execute a controller operation through the configured HTTP middleware, use
the application's `handle()` method:

```ts
const response = await app.handle(request, controllerToken, input)
```

`controllerToken` must be the registry key for a registered controller.
`handle()` dispatches through the HTTP middleware and requires middleware and an
HTTP adapter to have been registered. The chain applies HTTP concerns such as:

- authentication;
- CORS;
- origin validation;
- method checks;
- rate limiting;
- CSRF;
- request context.

Conceptually:

```text
HTTP request
      │
      ▼
app.handle(...)
      │
      ▼
configured middleware
      │
      ▼
application action
```

Do not bypass the configured middleware chain when an operation depends on HTTP
security behavior.

See [Middleware Execution](./middleware/execution).

---

## Security boundaries

Xeno's security features should be understood as separate controls.

| Mechanism                   | Protects                                                     |
| --------------------------- | ------------------------------------------------------------ |
| Authentication              | Establishes caller identity                                  |
| Authorization               | Controls application request access                          |
| Origin validation           | Restricts request origins                                    |
| CORS                        | Controls browser cross-origin response behavior              |
| HTTP method checks          | Restricts methods available for registered routes            |
| CSRF                        | Protects state-changing cookie-authenticated requests        |
| Rate limiting               | Limits request frequency                                     |
| Trusted proxy configuration | Controls which infrastructure may provide client-IP metadata |
| Sanitization                | Produces safer representations of untrusted values           |
| IP masking                  | Reduces exposure of client IP information in logs/metadata   |

No single mechanism replaces the others.

For example:

```text
Authentication ≠ Authorization
Authentication ≠ CSRF
CORS ≠ CSRF
Sanitization ≠ Validation
Rate limiting ≠ Authorization
```

---

## Recommended configuration

A typical secured application can compose the controls explicitly:

```ts
import { XenoAppBuilder, withAuthZ } from '@xeno-js/core'
import { withSupabase } from '@xeno-js/supabase'

const app = new XenoAppBuilder()

app.addAuth(
  withSupabase((options, config) => {
    options.url = config.get('SUPABASE_URL')!
    options.key = config.get('SUPABASE_KEY')!

    options.redirectTo = config.get('APP_URL')

    options.ssr = {
      cookieOptions: {
        path: '/',
        httpOnly: true,
        sameSite: 'lax',
        secure: true,
      },
    }
  }),
)

app.addMiddlewares((options, config) => {
  options.isSSR = true
  options.cors = {
    allowOrigins: [config.get('APP_URL')!],
    withCredentials: true,
  }

  options.rateLimit = { maxRequests: 30, windowSeconds: 60 }

  options.csrf = {
    secret: config.get('CSRF_SECRET')!,
    cookieName: '__Host-xeno-csrf',
    headerName: 'x-csrf-token',
    cookieMaxAgeSeconds: 3600,
    sameSite: 'lax',
  }
})

app.addPipeline(
  withAuthZ((opts) => {
    opts.policies = {
      CreateInvoice: {
        userId: true,
        tenantId: true,
        permissions: ['invoice:create'],
      },
    }
  }),
)

const application = await app.build()
```

This produces a security flow of:

```text
HTTP request
    │
    ▼
SSR cookie response (when enabled)
    │
    ▼
Origin validation
    │
    ▼
OPTIONS handling, method checks, then CORS
    │
    ▼
Rate limiting
    │
    ▼
Authentication
    │
    ▼
CSRF validation
    │
    ▼
Request identity
    │
    ▼
Authorization
    │
    ▼
Application handler
```

The exact controls used by an application should match its authentication
transport, browser usage, proxy topology, and application authorization
requirements.

---

## Troubleshooting

### Authentication is not being enforced

Check that both calls exist:

```ts
app.addAuth(...)
app.addMiddlewares(...)
```

`addAuth()` registers the authentication service.

`addMiddlewares()` installs the HTTP authentication middleware.

If `addAuth()` is omitted, Xeno uses `NoAuthGateKeeper`.

If application code uses the authentication service without HTTP middleware,
register it with `addAuth()` and resolve it from an active service scope. The
service is scoped; resolving it directly from the root container is not valid.

---

### CSRF protection is not running

Check that `csrf` is configured:

```ts
app.addMiddlewares((options) => {
  options.csrf = {
    secret: '...',
  }
})
```

CSRF is not enabled merely because authentication is enabled.

---

### CORS rejects the request

Check:

```ts
options.cors?.allowOrigins
```

and:

```ts
options.cors?.withCredentials
```

Prefer explicit allowed origins when credentials are enabled.

If `cors.allowOrigins` is empty, check `APP_URL` and `PORT`, because Xeno
derives a default origin from them.

---

### Requests are unexpectedly rate limited

Check:

```ts
options.rateLimit
```

The default is:

```ts
{
  enabled: true,
  maxRequests: 100,
  windowSeconds: 60,
}
```

For a higher-throughput API, configure the limit explicitly.

---

### Authorization does not run

Authentication and authorization are separate.

Make sure the application has configured the pipeline:

```ts
app.addPipeline(
  withAuthZ((opts) => {
    opts.policies = {
      CreateInvoice: {
        roles: ['admin'],
      },
    }
  }),
)
```

Authentication alone creates an identity; it does not automatically authorize
every application request.

---

## Security documentation

### Docs Authentication

- [Authentication Overview](./authentication/overview)
- [Supabase Authentication](./authentication/supabase)
- [Custom Authentication](./authentication/custom)

### Docs Authorization

- [Authorization Overview](./authorization/overview)
- [Authorization Policies](./authorization/policies)
- [Custom Authorization](./authorization/custom)

### HTTP middleware

- [Middleware Overview](./middleware/overview)
- [Authentication Middleware](./middleware/authentication)
- [CSRF Protection](./middleware/csrf)
- [CORS](./middleware/cors)
- [HTTP Methods](./middleware/methods)
- [Rate Limiting](./middleware/rate-limit)
- [Extractors](./middleware/extractors)
- [Middleware Execution](./middleware/execution)

### Helpers

- [HTTP Helpers](./helpers/http)
- [Sanitization Helpers](./helpers/sanitize)

---
