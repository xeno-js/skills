# Runtime, dependency injection and context

Technical reference assembled from the website documentation. Each section
identifies its original Markdown source. Website frontmatter and the repeated
sponsorship footer are omitted; body content and fenced examples are retained.

---

## Contents

- [`docs/dependency-injection/captive-dependencies.md`](#source-1)
- [`docs/dependency-injection/dependency-graph.md`](#source-2)
- [`docs/dependency-injection/lifetimes/scoped.md`](#source-3)
- [`docs/dependency-injection/lifetimes/singleton.md`](#source-4)
- [`docs/dependency-injection/lifetimes/transient.md`](#source-5)
- [`docs/dependency-injection/registration.md`](#source-6)
- [`docs/dependency-injection/resolution.md`](#source-7)
- [`docs/dependency-injection/service-container.md`](#source-8)
- [`docs/fundamentals/app-builder.md`](#source-9)
- [`docs/fundamentals/context-scopes.md`](#source-10)
- [`docs/fundamentals/dependency-injection.md`](#source-11)
- [`docs/fundamentals/overview.md`](#source-12)
- [`docs/fundamentals/xeno-application.md`](#source-13)
- [`docs/fundamentals/xeno-registry.md`](#source-14)

<a id="source-1"></a>

# Source: `docs/dependency-injection/captive-dependencies.md`

## Introduction

A **captive dependency** happens when a service with a longer lifetime holds or
requires a dependency with a shorter lifetime.

The most important case in Xeno.JS is:

```text
Singleton
    ↓
Scoped
```

A singleton lives for the lifetime of the service container, while a scoped
service belongs to a specific logical scope.

The container explicitly rejects a scoped dependency when it is resolved while a
singleton is being constructed. This check also applies when the resolution
reaches the scoped service through a transient dependency.

## When this problem occurs

You can encounter a captive dependency when you register a singleton that
resolves a scoped service:

```ts
services.addSingleton('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('REQUEST_CONTEXT'))
})
```

If `REQUEST_CONTEXT` is registered as scoped, resolving `USER_SERVICE` requires
the container to create a singleton that depends on a scoped instance.

That dependency is invalid because the singleton could outlive the scope that
created the scoped dependency.

Xeno.JS detects this situation and throws a captive dependency error instead of
allowing the singleton construction to capture the scoped instance.

## Understand the lifetime relationship

A useful way to reason about service lifetimes is:

```text
Longer lifetime
      │
      ▼
  Singleton
      │
      ▼
    Scoped
      │
      ▼
   Transient
Shorter lifetime
```

The important rule is not simply "never mix lifetimes".

The problem is that a longer-lived service must not capture a shorter-lived
dependency.

For example:

```text
Singleton → Scoped
```

is invalid.

By contrast:

```text
Scoped → Singleton
```

is valid because the singleton remains available for the entire lifetime of the
scope.

## How Xeno.JS detects the problem

While a singleton factory is running, the container marks the resolution context
as singleton. If resolution in that context reaches a scoped registration,
directly or through transient registrations, it throws a captive dependency
error. The check is specific to singleton construction; it does not validate
every lifetime combination or every possible root-scope use.

When a singleton resolves a scoped service, the container throws:

```text
[DI Captive Dependency Error]: Attempted to resolve a scoped service 'REQUEST_CONTEXT' from a singleton context. This can lead to captive dependencies. Resolution path: USER_SERVICE -> REQUEST_CONTEXT
```

The exact token names and resolution path depend on your registrations.

The resolution path is useful because it tells you which dependency chain caused
the problem.

For example:

```text
USER_SERVICE
    ↓
USER_REPOSITORY
    ↓
REQUEST_CONTEXT
```

If `USER_SERVICE` is a singleton and `REQUEST_CONTEXT` is scoped, the resolution
chain contains the problematic relationship. The container reports the path
reached during resolution.

## Fix a singleton that depends on a scoped service

The simplest solution is usually to change the dependent service to scoped.

### Before

```ts
services.addSingleton('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('REQUEST_CONTEXT'))
})

services.addScoped('REQUEST_CONTEXT', () => {
  return new RequestContext()
})
```

This creates:

```text
USER_SERVICE [singleton]
       ↓
REQUEST_CONTEXT [scoped]
```

### After

```ts
services.addScoped('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('REQUEST_CONTEXT'))
})

services.addScoped('REQUEST_CONTEXT', () => {
  return new RequestContext()
})
```

The dependency graph is now:

```text
USER_SERVICE [scoped]
       ↓
REQUEST_CONTEXT [scoped]
```

Both services belong to the same scope.

This is appropriate when `UserService` represents request-level or
operation-level state.

## Keep the singleton when it should be application-wide

Sometimes the singleton is intentionally long-lived.

For example:

```ts
services.addSingleton('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('REQUEST_CONTEXT'))
})
```

If `UserService` genuinely needs request-specific state, changing the dependency
to another lifetime may hide the underlying design problem.

Instead, remove the scoped dependency from the singleton.

For example, make the singleton depend only on long-lived services:

```ts
services.addSingleton('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('USER_REPOSITORY_FACTORY'))
})
```

Then provide request-specific dependencies at the operation boundary:

```text
Singleton
    │
    └── long-lived dependency

Request scope
    │
    ├── RequestContext
    ├── Repository
    └── UserService
```

The exact design depends on what the service actually needs, but the key
requirement is the same: the singleton must not capture a scoped instance.

## Check indirect dependencies

The problem does not have to be directly visible in the registration.

For example:

```ts
services.addSingleton('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('USER_REPOSITORY'))
})

services.addTransient('USER_REPOSITORY', (scope) => {
  return new UserRepository(scope.resolve('REQUEST_CONTEXT'))
})

services.addScoped('REQUEST_CONTEXT', () => {
  return new RequestContext()
})
```

The graph is:

```text
USER_SERVICE [singleton]
       ↓
USER_REPOSITORY [transient]
       ↓
REQUEST_CONTEXT [scoped]
```

The transient factory preserves the singleton resolution context, so resolving
`REQUEST_CONTEXT` still triggers the captive-dependency check. A scoped
intermediate dependency would be rejected as soon as the container tried to
resolve that registration, before running its factory.

When Xeno.JS reports the resolution path, use it to trace the registrations from
the singleton to the scoped service.

## Valid lifetime combinations

The lifetime of the dependency matters relative to the lifetime of the service
that consumes it.

### Singleton → Singleton

Valid:

```text
Singleton
    ↓
Singleton
```

Example:

```ts
services.addSingleton('CONFIGURATION', () => {
  return new Configuration()
})

services.addSingleton('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('CONFIGURATION'))
})
```

Both services can live for the lifetime of the container.

### Scoped → Singleton

Valid:

```text
Scoped
    ↓
Singleton
```

Example:

```ts
services.addSingleton('LOGGER', () => {
  return new Logger()
})

services.addScoped('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('LOGGER'))
})
```

The scoped service can safely use the longer-lived singleton.

### Scoped → Scoped

Valid:

```text
Scoped
    ↓
Scoped
```

Both services are resolved within the same active scope.

```ts
services.addScoped('USER_REPOSITORY', (scope) => {
  return new UserRepository(scope.resolve('REQUEST_CONTEXT'))
})

services.addScoped('REQUEST_CONTEXT', () => {
  return new RequestContext()
})
```

### Transient → Scoped

A transient service can depend on a scoped service when the transient is
resolved inside an active scope:

```text
Transient
    ↓
Scoped
```

For example:

```ts
services.addScoped('REQUEST_CONTEXT', () => {
  return new RequestContext()
})

services.addTransient('USER_HELPER', (scope) => {
  return new UserHelper(scope.resolve('REQUEST_CONTEXT'))
})
```

When the transient is resolved through an active operation scope, its factory
receives that scope and resolves the scoped instance belonging to it.

Do not resolve this transient through root services: a transient factory
resolved there receives the root scope, and a nested scoped resolution is stored
in that root scope rather than being rejected as a singleton captive dependency.
That can make scope-specific state live beyond the intended operation.

## Avoid fixing the error by changing everything to singleton

A common workaround is to change the scoped dependency into a singleton:

```ts
services.addSingleton('REQUEST_CONTEXT', () => {
  return new RequestContext()
})
```

This may remove the lifetime error, but it can introduce a different application
bug if `REQUEST_CONTEXT` contains request-specific state.

Instead, first determine what the dependency represents.

Use a scoped lifetime when the service contains state that belongs to a logical
scope, such as:

- request-specific state
- transaction-specific state
- per-operation state
- contextual data

Use a singleton when the service is genuinely safe to share across the
application's container lifetime.

## Use the active scope correctly

Scoped services must be resolved through an active `IServiceScope`.

For example:

```ts
await application.runInScope(async (services) => {
  const userRepository = services.USER_REPOSITORY
  await userRepository.findById('123')
})
```

`runInScope()` creates the request context and scope for the operation and
disposes the scope when the operation completes.

Do not resolve a scoped dependency directly from root services:

```ts
const userRepository = application.getServices().USER_REPOSITORY
```

Xeno.JS rejects this with:

```text
[DI Container Error]: Scoped service 'USER_REPOSITORY' requires an active scope.
```

The captive dependency problem is different: it occurs when a singleton is being
constructed and tries to resolve that scoped dependency.

## Diagnose a captive dependency error

When you see:

```text
[DI Captive Dependency Error]
```

check the following.

### 1. Find the singleton

Look at the first service in the resolution path.

For example:

```text
USER_SERVICE -> USER_REPOSITORY -> REQUEST_CONTEXT
```

If `USER_SERVICE` is registered as singleton, start there:

```ts
services.addSingleton('USER_SERVICE', ...)
```

### 2. Follow every `resolve()`

Inspect the registration:

```ts
services.addSingleton('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('USER_REPOSITORY'))
})
```

Then inspect `USER_REPOSITORY`:

```ts
services.addScoped('USER_REPOSITORY', (scope) => {
  return new UserRepository(scope.resolve('REQUEST_CONTEXT'))
})
```

You can reconstruct the dependency graph directly from the registration
factories.

### 3. Check the lifetime of the dependency

Find the dependency registration:

```ts
services.addScoped('REQUEST_CONTEXT', ...)
```

If the chain starts from a singleton and eventually reaches a scoped service,
you have the lifetime conflict.

### 4. Decide which lifetime represents the actual use case

Do not change a lifetime only to silence the error.

Ask:

- Should this service exist once for the application?
- Should it exist once per scope?
- Should a new instance be created on each resolution?
- Does it contain request-specific or transaction-specific state?

Then change the registration or dependency design accordingly.

## Complete example

The following example shows a valid scoped dependency graph:

```ts
import { XenoAppBuilder } from '@xeno-js/core'

const app = new XenoAppBuilder().addServices((services) => {
  services.addSingleton('LOGGER', () => {
    return new Logger()
  })

  services.addScoped('REQUEST_CONTEXT', () => {
    return new RequestContext()
  })

  services.addScoped('USER_REPOSITORY', (scope) => {
    return new UserRepository(
      scope.resolve('REQUEST_CONTEXT'),
      scope.resolve('LOGGER'),
    )
  })

  services.addScoped('USER_SERVICE', (scope) => {
    return new UserService(
      scope.resolve('USER_REPOSITORY'),
      scope.resolve('LOGGER'),
    )
  })
})

await app.build()
```

The resulting dependency graph is:

```text
USER_SERVICE [scoped]
    │
    ├── USER_REPOSITORY [scoped]
    │       │
    │       └── REQUEST_CONTEXT [scoped]
    │
    └── LOGGER [singleton]
```

This is valid because:

- `USER_SERVICE` is scoped.
- `USER_REPOSITORY` is scoped.
- `REQUEST_CONTEXT` is scoped.
- `LOGGER` is singleton.
- No singleton captures a scoped service.

## Common problems

### Singleton depends directly on scoped service

**Error:**

```text
[DI Captive Dependency Error]: Attempted to resolve a scoped service 'REQUEST_CONTEXT' from a singleton context.
```

**Check:**

```ts
services.addSingleton('MY_SERVICE', (scope) => {
  return new MyService(scope.resolve('REQUEST_CONTEXT'))
})
```

**Fix:**

Make `MY_SERVICE` scoped if it genuinely requires request-scoped state:

```ts
services.addScoped('MY_SERVICE', (scope) => {
  return new MyService(scope.resolve('REQUEST_CONTEXT'))
})
```

Or redesign the singleton so it does not retain the scoped dependency.

### Singleton indirectly depends on scoped service

**Graph:**

```text
Singleton
    ↓
Singleton
    ↓
Scoped
```

The dependency can be hidden several registrations deep.

**Fix:** follow the resolution path from the error and inspect each registration
in the chain.

### Scoped service resolved without a scope

**Error:**

```text
[DI Container Error]: Scoped service 'USER_REPOSITORY' requires an active scope.
```

**Fix:**

Resolve the service through an active `IServiceScope`:

```ts
await application.runInScope(async (services) => {
  const repository = services.USER_REPOSITORY
})
```

### Changing a scoped service to singleton breaks request isolation

If a service contains request-specific state, do not change its lifetime to
singleton simply to remove a captive dependency.

Keep the service scoped and adjust the dependent service instead.

## Checklist

When you encounter a captive dependency:

- [ ] Read the complete resolution path from the error.
- [ ] Find the singleton at the beginning of the dependency chain.
- [ ] Inspect each `scope.resolve(...)` in the chain.
- [ ] Find the scoped service that is being captured.
- [ ] Check whether the dependent service should actually be scoped.
- [ ] If the dependent service must remain singleton, remove its direct
      dependency on scoped state.
- [ ] Do not change a service to singleton merely to suppress the error.
- [ ] Verify that request and transaction isolation are preserved.
- [ ] Rebuild the application and resolve the dependency again.

## Related docs

- [Service Container](./service-container) — Learn where services are registered
  and resolved.
- [Registration](./registration) — Learn how to register services with explicit
  lifetimes.
- [Singleton](./lifetimes/singleton) — Learn how singleton services behave.
- [Scoped](./lifetimes/scoped) — Learn how scoped services behave.
- [Transient](./lifetimes/transient) — Learn how transient services behave.
- [Resolution](./resolution) — Learn how to resolve registered services.
- [Dependency Graph](./dependency-graph) — Learn how to model dependencies
  between services.

---

---

<a id="source-2"></a>

# Source: `docs/dependency-injection/dependency-graph.md`

## Introduction

When your application contains several services, each service can depend on
other registered services.

In Xeno.JS, you define these relationships explicitly in your dependency
injection registrations.

For example:

```text
UserController
      |
      v
   UserService
      |
      v
 UserRepository
      |
      v
 UserDataSource
```

The dependency graph is expressed directly in the registration code:

```ts
services.addScoped('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('USER_REPOSITORY'))
})

services.addScoped('USER_REPOSITORY', (scope) => {
  return new UserRepository(scope.resolve('USER_DATA_SOURCE'))
})
```

You do not need decorators, runtime scanning, or a separate graph configuration.

## Before you start

You should already have:

- a Xeno.JS application created with `XenoAppBuilder`
- the services you want to connect
- service registrations for their dependencies
- a clear lifetime for each service

For application-level registration, use `AppBuilder.addServices()`:

```ts
const app = new XenoAppBuilder().addServices((services) => {
  // service registrations
})

await app.build()
```

`addServices()` provides the service container during application configuration,
where you can register singleton, scoped, and transient services. `build()`
returns the configured runtime application, not the container.

See [Registration](./registration) for the registration API and
[Lifetimes](./lifetimes/singleton) for service lifetime details.

## Define a dependency between services

Declare registrations during application composition, then resolve dependencies
from the dependent service's factory. The factory resolves its dependencies when
the service is requested, so registration order is not the dependency
relationship itself.

For example, suppose `UserService` needs `UserRepository`.

```ts
services.addScoped('USER_REPOSITORY', (scope) => {
  return new UserRepository(scope.resolve('USER_DATA_SOURCE'))
})

services.addScoped('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('USER_REPOSITORY'))
})
```

The resulting dependency relationship is:

```text
USER_SERVICE
     |
     v
USER_REPOSITORY
     |
     v
USER_DATA_SOURCE
```

The `scope` passed to every registration factory is the mechanism used to
resolve dependencies.

## Register multiple dependencies

A service can depend on more than one service.

For example:

```ts
services.addScoped('USER_SERVICE', (scope) => {
  return new UserService(
    scope.resolve('USER_REPOSITORY'),
    scope.resolve('USER_VALIDATOR'),
    scope.resolve('LOGGER'),
  )
})
```

The graph is:

```text
             ┌──> USER_REPOSITORY
             |
USER_SERVICE ├──> USER_VALIDATOR
             |
             └──> LOGGER
```

Each `resolve()` call adds another dependency to the service's composition.

Keep the dependencies explicit in the factory:

```ts
services.addScoped('USER_SERVICE', (scope) => {
  const repository = scope.resolve('USER_REPOSITORY')
  const validator = scope.resolve('USER_VALIDATOR')
  const logger = scope.resolve('LOGGER')

  return new UserService(repository, validator, logger)
})
```

This form can be useful when the registration becomes large or when you want to
make the dependency list easier to inspect.

## Build a multi-level dependency graph

Dependencies can continue through several levels.

For example:

```ts
services.addScoped('USER_CONTROLLER', (scope) => {
  return new UserController(scope.resolve('USER_SERVICE'))
})

services.addScoped('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('USER_REPOSITORY'))
})

services.addScoped('USER_REPOSITORY', (scope) => {
  return new UserRepository(scope.resolve('USER_DATA_SOURCE'))
})

services.addScoped('USER_DATA_SOURCE', () => {
  return new UserDataSource()
})
```

The resulting graph is:

```text
USER_CONTROLLER
       |
       v
 USER_SERVICE
       |
       v
USER_REPOSITORY
       |
       v
USER_DATA_SOURCE
```

This makes the composition of the application visible in the registration code.

## Keep dependency direction explicit

When designing a service graph, register dependencies according to the direction
in which your application needs them.

For example:

```text
Controller
    |
    v
Application Service
    |
    v
Repository
    |
    v
Data Source
```

The controller depends on the application service.

The application service depends on the repository.

The repository depends on the data source.

The dependency direction is encoded directly in the factories:

```ts
services.addScoped('USER_CONTROLLER', (scope) => {
  return new UserController(scope.resolve('USER_SERVICE'))
})

services.addScoped('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('USER_REPOSITORY'))
})

services.addScoped('USER_REPOSITORY', (scope) => {
  return new UserRepository(scope.resolve('USER_DATA_SOURCE'))
})
```

If a dependency points in an unexpected direction, review the service boundary
before adding another registration.

## Use built-in Xeno.JS services as dependencies

Xeno.JS exposes framework services through registry tokens when the relevant
modules are configured.

For example, a service can depend on the request context or mediator:

```ts
services.addTransient('USER_CONTROLLER', (scope) => {
  return new UserController(
    scope.resolve('REQUEST_CONTEXT'),
    scope.resolve('MEDIATOR'),
  )
})
```

The built-in tokens are typed through the application registry, so the token
determines the type returned by `resolve()`.

Examples of framework service tokens used by the registrations in this
documentation include:

- `REQUEST_CONTEXT`
- `MEDIATOR`
- `LOGGER`
- `CACHE`
- `CONFIGURATION_SERVICE`
- `SERVICE_SCOPE_ACCESSOR`
- `UNIT_OF_WORK`

Only resolve a token when the corresponding registration is available in your
application configuration.

## Register dependencies through modules

For larger applications, related registrations can be grouped in a module
instead of placing everything in one `addServices()` callback.

A module receives the same service container and can configure its own services.

Conceptually:

```text
Application
    |
    +-- User Module
    |     |
    |     +-- UserController
    |     +-- UserService
    |     +-- UserRepository
    |
    +-- Order Module
          |
          +-- OrderService
          +-- OrderRepository
```

Each module can own the registrations required for its feature.

This keeps the composition root manageable while keeping dependencies explicit.

See [Registration](./registration) for service registration and the module
documentation for application module composition.

## Avoid circular dependencies

A circular dependency exists when services eventually depend on themselves.

For example:

```text
SERVICE_A
   |
   v
SERVICE_B
   |
   v
SERVICE_A
```

The registrations might look like this:

```ts
services.addTransient('SERVICE_A', (scope) => {
  return new ServiceA(scope.resolve('SERVICE_B'))
})

services.addTransient('SERVICE_B', (scope) => {
  return new ServiceB(scope.resolve('SERVICE_A'))
})
```

Xeno.JS detects the circular dependency while resolving the services and throws
an error similar to:

```text
[DI Circular Dependency Error]: Detected circular dependency while resolving 'SERVICE_A'. Resolution path: SERVICE_A -> SERVICE_B -> SERVICE_A
```

### Fix a circular dependency

Do not try to work around the error by adding another `resolve()` call.

Instead, identify why the two services require each other.

For example, if:

```text
UserService -> NotificationService -> UserService
```

is required only because both services need a small shared operation, extract
that operation into a separate service:

```text
        +------------------+
        | SharedService    |
        +------------------+
           ^            ^
           |            |
     UserService   NotificationService
```

Then register the new dependency explicitly:

```ts
services.addScoped('SHARED_SERVICE', () => {
  return new SharedService()
})

services.addScoped('USER_SERVICE', (scope) => {
  return new UserService(
    scope.resolve('SHARED_SERVICE'),
    scope.resolve('NOTIFICATION_SERVICE'),
  )
})

services.addScoped('NOTIFICATION_SERVICE', (scope) => {
  return new NotificationService(scope.resolve('SHARED_SERVICE'))
})
```

If the two services genuinely need each other, review the application boundary
rather than attempting to hide the cycle inside the container.

See [Captive Dependencies](./captive-dependencies) for lifetime-related
dependency problems.

## Respect lifetime boundaries

The dependency graph also includes the lifetime of each service.

For example:

```text
Singleton
    |
    v
Scoped
```

is not a valid dependency relationship in Xeno.JS.

A singleton is created at the container level, while a scoped service belongs to
an active scope. Allowing the singleton to retain the scoped service would make
the scoped instance outlive its intended scope.

While constructing a singleton, the container rejects a scoped dependency
reached directly or through transient registrations and throws a captive
dependency error. This check is specific to singleton construction; it does not
validate every lifetime combination or root-scope use.

For example:

```text
[DI Captive Dependency Error]: Attempted to resolve a scoped service 'REQUEST_SERVICE' from a singleton context. This can lead to captive dependencies. Resolution path: ...
```

If a singleton needs data that is specific to a scope, change the design so that
the scoped value is resolved within the appropriate scope rather than stored by
the singleton.

See [Singleton](./lifetimes/singleton), [Scoped](./lifetimes/scoped), and
[Captive Dependencies](./captive-dependencies).

## Use the service scope for scoped dependencies

If a service depends on a scoped service, resolve it through the scope for the
intended execution. In application code, use `getScopedServices()` within an
existing execution or `runInScope()` to create a scope for a new operation.

For example:

```ts
services.addScoped('REQUEST_DATA', () => {
  return new RequestData()
})

services.addScoped('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('REQUEST_DATA'))
})
```

Both services are resolved within the same logical scope:

```text
Request scope
     |
     +--> USER_SERVICE
              |
              +--> REQUEST_DATA
```

This allows `REQUEST_DATA` to remain scoped to the current application
execution. Do not resolve a transient that depends on `REQUEST_DATA` through
root services: its factory receives the root scope, so a nested scoped
resolution is associated with that root scope rather than the intended request
or operation scope.

See [Scoped](./lifetimes/scoped) and [Resolution](./resolution).

## Inspect the graph from your registrations

Xeno.JS does not require a separate dependency graph declaration.

When investigating a service, start from its registration:

```ts
services.addScoped('USER_SERVICE', (scope) => {
  return new UserService(
    scope.resolve('USER_REPOSITORY'),
    scope.resolve('LOGGER'),
  )
})
```

The immediate dependencies are:

```text
USER_SERVICE
   ├──> USER_REPOSITORY
   └──> LOGGER
```

Then inspect the registrations for those dependencies:

```ts
services.addScoped('USER_REPOSITORY', (scope) => {
  return new UserRepository(scope.resolve('USER_DATA_SOURCE'))
})
```

Now the graph becomes:

```text
USER_SERVICE
   ├──> USER_REPOSITORY
   │       |
   │       └──> USER_DATA_SOURCE
   |
   └──> LOGGER
```

This is the practical way to trace a dependency chain in Xeno.JS: follow each
`resolve()` from the service registration to the next registration.

## Common problems

### Registration not found

If a dependency is referenced but has not been registered:

```ts
services.addScoped('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('USER_REPOSITORY'))
})
```

and `USER_REPOSITORY` has no registration, Xeno.JS throws:

```text
[DI Container Error]: Registration not found for token 'USER_REPOSITORY'. Ensure the service is registered before resolving.
```

Check that:

1. the dependency has been registered;
2. the token is spelled correctly;
3. the registration is included in the application build;
4. the module containing the registration is configured.

### Duplicate registration

A token cannot be registered more than once in the same container.

For example:

```ts
services.addScoped('USER_SERVICE', () => {
  return new UserService()
})

services.addScoped('USER_SERVICE', () => {
  return new UserService()
})
```

causes:

```text
[DI Container Error]: The token 'USER_SERVICE' is already registered in the container.
```

Keep a single registration for each token in a container.

If multiple implementations are required, use different tokens and choose
explicitly which service each consumer depends on.

### Scoped service requires an active scope

Resolving a scoped registration directly from root services is invalid:

```text
[DI Container Error]: Scoped service 'USER_SERVICE' requires an active scope.
```

Resolve scoped services within an active operation scope instead:

```ts
await application.runInScope(async (services) => {
  const service = services.USER_SERVICE
  await service.execute()
})
```

In request-driven code, use `getScopedServices()` to resolve through the
request's active scope rather than creating an unrelated scope. Also avoid
resolving a transient that depends on scoped services through root services: its
factory receives the root scope, so a nested scoped resolution belongs to that
root scope.

See [Resolution](./resolution).

### Circular dependency

If the container reports:

```text
[DI Circular Dependency Error]
```

follow the resolution path included in the error.

For example:

```text
SERVICE_A -> SERVICE_B -> SERVICE_C -> SERVICE_A
```

Inspect those registrations and remove the cycle by changing the service
boundary or extracting a shared dependency.

### Singleton depends on scoped service

If the error contains:

```text
[DI Captive Dependency Error]
```

check the lifetimes along the reported resolution path.

A singleton should not directly resolve a scoped service during its
construction.

See [Captive Dependencies](./captive-dependencies).

## Complete example

The following example defines a small application graph with a controller,
application service, repository, and data source:

```ts
import { XenoAppBuilder } from '@xeno-js/core'

class UserDataSource {
  async findById(id: string) {
    return {
      id,
      name: 'Ada',
    }
  }
}

class UserRepository {
  constructor(private readonly dataSource: UserDataSource) {}

  findById(id: string) {
    return this.dataSource.findById(id)
  }
}

class UserService {
  constructor(private readonly repository: UserRepository) {}

  findById(id: string) {
    return this.repository.findById(id)
  }
}

class UserController {
  constructor(private readonly service: UserService) {}

  handle(id: string) {
    return this.service.findById(id)
  }
}

const app = new XenoAppBuilder().addServices((services) => {
  services.addScoped('USER_DATA_SOURCE', () => {
    return new UserDataSource()
  })

  services.addScoped('USER_REPOSITORY', (scope) => {
    return new UserRepository(scope.resolve('USER_DATA_SOURCE'))
  })

  services.addScoped('USER_SERVICE', (scope) => {
    return new UserService(scope.resolve('USER_REPOSITORY'))
  })

  services.addTransient('USER_CONTROLLER', (scope) => {
    return new UserController(scope.resolve('USER_SERVICE'))
  })
})

const application = await app.build()
```

The dependency graph is:

```text
USER_CONTROLLER
       |
       v
 USER_SERVICE
       |
       v
USER_REPOSITORY
       |
       v
USER_DATA_SOURCE
```

The important part is that every edge in the graph is visible in the
registration code.

## Dependency graph checklist

When adding a new service, verify:

- [ ] The service has a registration.
- [ ] Every dependency has a registration.
- [ ] Each dependency is resolved explicitly from the factory scope.
- [ ] The service lifetime matches how it is used.
- [ ] Scoped dependencies are resolved inside an active scope.
- [ ] There is no circular dependency.
- [ ] A singleton does not depend directly on a scoped service.
- [ ] The dependency direction matches the application's boundaries.
- [ ] The registrations are included in the application build.

## Related docs

- [Service Container](./service-container) — understand the container used to
  register and resolve services.
- [Registration](./registration) — register singleton, scoped, and transient
  services.
- [Singleton](./lifetimes/singleton) — configure services shared by the
  container.
- [Scoped](./lifetimes/scoped) — configure services tied to a logical scope.
- [Transient](./lifetimes/transient) — create a new instance for each
  resolution.
- [Resolution](./resolution) — resolve services from the container or an active
  scope.
- [Captive Dependencies](./captive-dependencies) — fix invalid lifetime
  relationships.

---

---

<a id="source-3"></a>

# Source: `docs/dependency-injection/lifetimes/scoped.md`

## Introduction

Use a **scoped** service when you need one service instance for each logical
execution scope, while keeping different scopes isolated from each other.

A common use case is request-level state:

- one repository instance per request
- one transaction state per request
- one unit of work per request
- request-specific services
- other dependencies that must not be shared across concurrent executions

Xeno.JS exposes scoped services through `addScoped()` and resolves them through
`IServiceScope`. After `build()`, use `getScopedServices()` inside an existing
scope or `runInScope()` to execute work in a new one.

## Before you start

You need:

- an `XenoAppBuilder`
- a service token included in the application's registry
- a service implementation
- a service factory

Custom tokens must be represented in your application's registry so TypeScript
can associate the token with its service type.

For example:

```ts
type UserRepository = {
  findById(id: string): Promise<unknown>
}
```

## Register a scoped service

Register the service with `addScoped()` inside `AppBuilder.addServices()`:

```ts
const app = new XenoAppBuilder().addServices((services) => {
  services.addScoped('USER_REPOSITORY', () => {
    return new UserRepository()
  })
})

const application = await app.build()
```

The important part is:

```ts
services.addScoped('USER_REPOSITORY', () => {
  return new UserRepository()
})
```

The factory receives the current service scope, so dependencies can be resolved
from that scope.

## Register a scoped service with dependencies

Use the factory argument to resolve other services:

```ts
const app = new XenoAppBuilder().addServices((services) => {
  services.addScoped('USER_REPOSITORY', (scope) => {
    return new UserRepository(scope.resolve('USER_DATA_SOURCE'))
  })
})

const application = await app.build()
```

This keeps the service graph explicit.

The dependency is resolved from the same scope that is creating
`USER_REPOSITORY`.

## Resolve a scoped service

`XenoAppBuilder.build()` returns the runtime application, not the service
container. Do not try to resolve a scoped service through its root services:

```ts
const application = await app.build()
const repository = application.getServices().USER_REPOSITORY
```

This is invalid for a scoped registration because it has no active scope.

For an operation outside an existing request or execution, use `runInScope()`:

```ts
const repository = await application.runInScope(async (services) => {
  return services.USER_REPOSITORY
})
```

The callback receives services resolved through the new scope. The operation
owns that scope's lifetime.

## Reuse the same instance within a scope

A scoped service is created once per scope.

```ts
const sameInstance = await application.runInScope(async (services) => {
  const first = services.USER_REPOSITORY
  const second = services.USER_REPOSITORY
  return first === second
})

console.log(sameInstance) // true
```

The two resolutions return the same instance because they use the same scope.

## Get a different instance in another scope

Different scopes have independent scoped-service instances:

```ts
const repositoryA = await application.runInScope(
  async (services) => services.USER_REPOSITORY,
)
const repositoryB = await application.runInScope(
  async (services) => services.USER_REPOSITORY,
)

console.log(repositoryA === repositoryB) // false
```

This is the main difference between `scoped` and `singleton`:

| Lifetime  | Instance reuse                   |
| --------- | -------------------------------- |
| Singleton | One instance per container       |
| Scoped    | One instance per scope           |
| Transient | New instance for each resolution |

## Create a complete scoped service graph

A typical application can combine scoped services and their dependencies:

```ts
class UserDataSource {
  async findById(id: string) {
    return { id }
  }
}

class UserRepository {
  constructor(private readonly dataSource: UserDataSource) {}

  async findById(id: string) {
    return this.dataSource.findById(id)
  }
}

const app = new XenoAppBuilder().addServices((services) => {
  services.addScoped('USER_DATA_SOURCE', () => {
    return new UserDataSource()
  })

  services.addScoped('USER_REPOSITORY', (scope) => {
    return new UserRepository(scope.resolve('USER_DATA_SOURCE'))
  })
})

const application = await app.build()

const user = await application.runInScope(async (services) => {
  const repository = services.USER_REPOSITORY
  return repository.findById('user-123')
})
```

Both `USER_DATA_SOURCE` and `USER_REPOSITORY` belong to the same scope.

If another scope resolves `USER_REPOSITORY`, Xeno.JS creates another scoped
graph for that scope.

## Use scoped services for request-level state

A scoped lifetime is useful when a service should be isolated to a logical
request or execution.

For example:

```ts
class RequestState {
  private readonly values = new Map<string, unknown>()

  set(key: string, value: unknown) {
    this.values.set(key, value)
  }

  get<T>(key: string): T | undefined {
    return this.values.get(key) as T | undefined
  }
}
```

Register it as scoped:

```ts
const app = new XenoAppBuilder().addServices((services) => {
  services.addScoped('REQUEST_STATE', () => {
    return new RequestState()
  })
})
```

Each scope gets its own `RequestState`.

This prevents request-specific state from being shared between unrelated
executions.

## Scoped services and request execution

Xeno.JS provides a request context that exposes the current service scope.

During request execution, the request context creates a service scope and makes
it available through the current execution context.

Application code that needs the active scope can use the request-context
abstraction rather than creating an unrelated scope.

Inside application code, use the application's scoped service API rather than
reaching into the root container:

```ts
const { USER_REPOSITORY: repository } = application.getScopedServices()
```

This is particularly useful when a service must participate in the same scope as
the rest of the current request.

If there is no active scope, `getScopedServices()` throws. For work that needs
its own scope, call `runInScope()` instead.

## Scope disposal

`runInScope()` owns the scope lifecycle and disposes the scope when the
operation completes, including when the callback fails:

```ts
await application.runInScope(async (services) => {
  const repository = services.USER_REPOSITORY
  await repository.findById('user-123')
})
```

When the scope is disposed, the container disposes its tracked disposable
instances in reverse creation order. It also tracks disposable transient
instances created through that scope. A disposed scope cannot resolve more
services.

## Prefer request-managed scopes for requests

If your application already uses Xeno.JS request context, do not create an
additional scope for every service resolution.

Instead, resolve services from the active request scope.

For example:

```ts
const { unitOfWork, USER_REPOSITORY: repository } =
  application.getScopedServices()
```

This allows all scoped dependencies participating in the request to share the
same scope.

## Scoped services and transient dependencies

A scoped service can resolve transient dependencies:

```ts
services.addTransient('USER_MAPPER', () => {
  return new UserMapper()
})

services.addScoped('USER_REPOSITORY', (scope) => {
  return new UserRepository(scope.resolve('USER_MAPPER'))
})
```

The repository remains scoped, while each resolution of the mapper follows
transient lifetime semantics. The repository factory resolves the mapper when
the repository is first created in that scope; later resolutions of the
repository return the cached repository instance.

Each separate resolution of the transient dependency creates a new instance.

## Scoped services and singleton dependencies

A scoped service can use a singleton dependency when that dependency is
appropriate for sharing across scopes:

```ts
services.addSingleton('MY_LOGGER', () => {
  return new Logger()
})

services.addScoped('USER_REPOSITORY', (scope) => {
  return new UserRepository(scope.resolve('MY_LOGGER'))
})
```

The repository is different for each scope, while the logger is shared by the
container.

The reverse relationship requires more care: a singleton must not depend on a
scoped service.

See [Captive Dependencies](../captive-dependencies).

## Common error: resolving a scoped service from the root

If a scoped registration is resolved from the root container, for example
through `getServices()`:

```ts
const repository = application.getServices().USER_REPOSITORY
```

Xeno.JS throws:

```text
[DI Container Error]: Scoped service 'USER_REPOSITORY' requires an active scope.
```

### Fix

Run the operation in a scope:

```ts
const repository = await application.runInScope(
  async (services) => services.USER_REPOSITORY,
)
```

For request-bound code, use the active request scope instead.

## Common error: no active scope

Calling `getScopedServices()` when no request or execution scope is active:

```ts
application.getScopedServices()
```

throws an error because there is no active scope. For work outside a request or
other execution, create the scope through `runInScope()`:

```ts
await application.runInScope(async (services) => {
  const repository = services.USER_REPOSITORY
  // Use the repository within this scope.
})
```

## Common error: using the wrong scope

Avoid starting a separate `runInScope()` operation inside code that is already
executing within a request scope:

```ts
await application.runInScope(async (services) => {
  // This operation has a separate scope from the current request.
})
```

A separate scope has its own scoped instances.

If the operation should participate in the current request, resolve services
from the active scope with `getScopedServices()` instead.

## When to use scoped lifetime

Use `scoped` when:

- state must be isolated per request or execution
- multiple services in one request should share the same instance
- a resource should live for the duration of a scope
- a repository or unit of work should not be shared globally
- concurrent requests must not share mutable service state

Typical examples include:

```text
Request
  └── Service Scope
      ├── UnitOfWork
      ├── TransactionState
      ├── UserRepository
      └── RequestState
```

The exact services in a scope depend on your application.

## When not to use scoped lifetime

Do not use `scoped` merely because a service is used by several classes.

Choose the lifetime based on how the service's instance should be shared:

- use **singleton** when one shared instance should live with the container
- use **scoped** when one instance should exist per logical scope
- use **transient** when every resolution should create a new instance

See [Singleton](./singleton) and [Transient](./transient) for the other lifetime
options.

## Scoped lifetime checklist

Before registering a service as scoped, verify:

- [ ] The service needs one instance per logical scope.
- [ ] Different scopes must receive different instances.
- [ ] The service token exists in the application registry.
- [ ] The service is registered with `addScoped()`.
- [ ] Dependencies are resolved through the provided scope.
- [ ] The service is resolved through an active scope, such as with
      `getScopedServices()` or `runInScope()`.
- [ ] Request-bound code uses the active request scope when appropriate.
- [ ] Scoped work runs through the request scope or `runInScope()` so its scope
      is disposed after use.
- [ ] The service does not require singleton lifetime.

## Related docs

- [Service Container](../service-container)
- [Registration](../registration)
- [Singleton](./singleton)
- [Transient](./transient)
- [Resolution](../resolution)
- [Dependency Graph](../dependency-graph)
- [Captive Dependencies](../captive-dependencies)

---

---

<a id="source-4"></a>

# Source: `docs/dependency-injection/lifetimes/singleton.md`

## Introduction

Use a **singleton** when your application needs one shared service instance for
the lifetime of the dependency injection container.

A singleton is useful for services that are safe to share across the
application, such as stateless infrastructure clients, configuration-related
services, caches, or other resources whose lifetime should not be tied to an
individual request or scope.

## Before you start

You need:

- an Xeno.JS application using `XenoAppBuilder`
- a service token available in your application registry
- a service class or factory function to instantiate

Xeno.JS uses explicit dependency injection. You register the service yourself
instead of relying on decorators or automatic dependency discovery.

If you are adding a custom service token, make sure it is part of your
application's registry.

## Register a singleton

Register a singleton with `addSingleton()` inside `AppBuilder.addServices()`:

```ts
import { XenoAppBuilder } from '@xeno-js/core'

const app = new XenoAppBuilder().addServices((services) => {
  services.addSingleton('USER_SERVICE', () => {
    return new UserService()
  })
})

await app.build()
```

The important part is:

```ts
services.addSingleton('USER_SERVICE', () => {
  return new UserService()
})
```

The factory creates the service instance. Xeno.JS creates that instance when the
service is first resolved and reuses it for later resolutions.

## Resolve the singleton

`build()` returns the runtime application, not the service container. Resolve
the singleton from its root services:

```ts
const application = await app.build()
const userService = application.getServices().USER_SERVICE
```

Resolving the same token again returns the same instance:

```ts
const first = application.getServices().USER_SERVICE
const second = application.getServices().USER_SERVICE

console.log(first === second) // true
```

This is the defining behavior of the singleton lifetime.

## Add dependencies to a singleton

Singleton factories receive the container's root service scope. Use it to
resolve dependencies that can be resolved from the root; the singleton factory
does not run in a request scope.

For example:

```ts
services.addSingleton('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('USER_REPOSITORY'))
})
```

The dependency graph is therefore explicit in the registration:

```text
USER_SERVICE
    |
    v
USER_REPOSITORY
```

You do not need constructor decorators or runtime dependency scanning.

## Register a singleton with multiple dependencies

A singleton can resolve several registered services:

```ts
services.addSingleton('USER_SERVICE', (scope) => {
  return new UserService(
    scope.resolve('USER_REPOSITORY'),
    scope.resolve(TOKENS.LOGGER),
    scope.resolve(TOKENS.CONFIGURATION_SERVICE),
  )
})
```

Keep the registration responsible for composing the service. The service itself
should receive the dependencies it needs through its constructor or factory API.

## Singleton and service scopes

Creating a service scope does **not** create another singleton instance.

For example:

```ts
const serviceA = await application.runInScope(
  async (services) => services.USER_SERVICE,
)
const serviceB = await application.runInScope(
  async (services) => services.USER_SERVICE,
)

console.log(serviceA === serviceB) // true
```

Singletons are shared by the container, while scoped services have a separate
instance for each scope.

This distinction is important when an application uses request or operation
scopes.

For more information about scoped services, see [Scoped](./scoped).

## When to use a singleton

Use a singleton when the service should be shared across the application's
container lifetime.

Typical candidates include:

- stateless application services that are safe to share
- reusable infrastructure clients
- caches
- configuration services
- loggers
- factories
- immutable or internally synchronized resources
- resources that should have one instance for the application

The important requirement is that the service must be safe to share.

For example, a service containing mutable state specific to one HTTP request
should generally not be registered as a singleton.

## When not to use a singleton

Do not use a singleton simply because creating one instance is convenient.

A different lifetime may be more appropriate when the service contains:

- request-specific state
- user-specific state
- transaction-specific state
- scope-specific resources
- mutable state that must not be shared between requests

For request- or operation-specific dependencies, use a scoped lifetime instead.

For services that should be recreated for every resolution, use transient
lifetime.

See:

- [Scoped](./scoped)
- [Transient](./transient)

## Singleton dependencies

A singleton can depend on services with lifetimes that are compatible with its
lifetime.

For example, a singleton depending on another singleton is valid:

```ts
services.addSingleton(TOKENS.CONFIGURATION_SERVICE, () => {
  return new ConfigurationService()
})

services.addSingleton('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve(TOKENS.CONFIGURATION_SERVICE))
})
```

Both services are shared by the container.

### Avoid singleton → scoped dependencies

A singleton must not depend on a scoped service.

For example:

```ts
services.addScoped('REQUEST_SERVICE', () => {
  return new RequestService()
})

services.addSingleton('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('REQUEST_SERVICE'))
})
```

This is invalid because `USER_SERVICE` lives for the lifetime of the container
while `REQUEST_SERVICE` belongs to an individual scope. The container also
rejects a scoped dependency reached transitively while constructing a singleton,
for example through a transient dependency.

Xeno.JS detects this situation and throws a captive dependency error.

If the service needs request-specific or scope-specific state, consider
registering the consuming service as scoped as well.

See [Captive Dependencies](../captive-dependencies).

## Singleton disposal

If a singleton implements the disposable contract, the service container tracks
it in the root scope. When `ServiceContainer.dispose()` runs, it disposes
tracked instances in reverse creation order, then clears the singleton instances
and registrations. A disposal error is logged and does not prevent the container
from attempting to dispose the remaining tracked instances.

For example:

```ts
class ExternalClient {
  async dispose(): Promise<void> {
    await this.closeConnection()
  }

  async closeConnection(): Promise<void> {
    // Release external resources.
  }
}
```

Register it normally:

```ts
services.addSingleton('EXTERNAL_CLIENT', () => {
  return new ExternalClient()
})
```

The `XenoApplication` runtime API shown in this documentation does not expose a
container-disposal method. Dispose the `ServiceContainer` when you manage that
container directly; do not treat the value returned by `build()` as the
container.

## Complete example

The following example registers a shared `UserService` and gives it a repository
dependency.

```ts
import { XenoAppBuilder } from '@xeno-js/core'

class UserRepository {
  async findById(id: string) {
    return {
      id,
      name: 'Ada',
    }
  }
}

class UserService {
  constructor(private readonly repository: UserRepository) {}

  async findById(id: string) {
    return this.repository.findById(id)
  }
}

const app = new XenoAppBuilder().addServices((services) => {
  services.addSingleton('USER_REPOSITORY', () => {
    return new UserRepository()
  })

  services.addSingleton('USER_SERVICE', (scope) => {
    return new UserService(scope.resolve('USER_REPOSITORY'))
  })
})

const application = await app.build()

const first = application.getServices().USER_SERVICE
const second = application.getServices().USER_SERVICE

console.log(first === second) // true
```

The resulting dependency graph is:

```text
USER_SERVICE
    |
    v
USER_REPOSITORY
```

Both services use the singleton lifetime, so the same instances are reused by
the container.

## Troubleshooting

### `The token '...' is already registered`

A service token can only have one registration in the container.

Check that the same token is not registered twice:

```ts
services.addSingleton('USER_SERVICE', () => {
  return new UserService()
})

// Do not register USER_SERVICE again.
```

If a module already registers the service, remove the duplicate application
registration or use a different token.

### `Registration not found for token '...'`

The dependency has not been registered before it is resolved.

For example:

```ts
services.addSingleton('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('USER_REPOSITORY'))
})
```

requires `USER_REPOSITORY` to be registered:

```ts
services.addSingleton('USER_REPOSITORY', () => {
  return new UserRepository()
})
```

Check the token spelling and make sure the registration is part of the
application configuration.

### `Scoped service '...' requires an active scope`

A scoped service cannot be resolved directly from the root container.

If the service is intentionally scoped, resolve it through an active scope,
using `getScopedServices()` during an existing execution or `runInScope()` for a
new operation.

If the service is intended to be application-wide, reconsider whether it should
be registered as a singleton.

### `DI Captive Dependency Error`

This usually means a singleton is trying to resolve a scoped service:

```text
Singleton
    |
    v
Scoped service
```

Change the lifetime of the consuming service when it needs scope-specific state,
or move the scoped dependency out of the singleton.

See [Captive Dependencies](../captive-dependencies).

### The singleton contains request-specific state

If the service stores data that belongs to an individual request, user,
transaction, or operation, singleton lifetime is probably not appropriate.

Move that state into a scoped service or another request-specific abstraction.

## Singleton checklist

Before registering a service as singleton, verify:

- [ ] The service should have one instance for the container lifetime.
- [ ] The service is safe to share between scopes.
- [ ] The service does not store request-specific state.
- [ ] Its dependencies use compatible lifetimes.
- [ ] All dependency tokens are registered.
- [ ] The service is registered only once.
- [ ] Disposable resources are cleaned up when the container is disposed.

## Related docs

- [Dependency Injection](../overview)
- [Service Container](../service-container)
- [Registration](../registration)
- [Scoped](./scoped)
- [Transient](./transient)
- [Resolution](../resolution)
- [Dependency Graph](../dependency-graph)
- [Captive Dependencies](../captive-dependencies)

---

---

<a id="source-5"></a>

# Source: `docs/dependency-injection/lifetimes/transient.md`

## Introduction

Use a transient service when you want **a new instance every time the service is
resolved**.

Transient services are useful for lightweight services that do not need to
preserve instance state between resolutions.

In Xeno.JS, register a transient service with `addTransient()`:

```ts
services.addTransient('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('USER_REPOSITORY'))
})
```

Every call to `resolve('USER_SERVICE')` creates a new `UserService` instance.

## Before you start

You need:

- an `XenoAppBuilder` to configure the service container;
- a service token registered in your application registry;
- a factory that creates the service instance.

For an application built with `XenoAppBuilder`, register application services
with `addServices()`:

```ts
const app = new XenoAppBuilder().addServices((services) => {
  services.addTransient('USER_SERVICE', (scope) => {
    return new UserService(scope.resolve('USER_REPOSITORY'))
  })
})

const application = await app.build()
```

`build()` returns the runtime application, not the service container. Use its
service APIs to resolve registered services.

## Register a transient service

Use `addTransient()` with:

1. the service token;
2. a factory function that creates the service.

```ts
services.addTransient('USER_SERVICE', () => {
  return new UserService()
})
```

The factory receives the current `IServiceScope`:

```ts
services.addTransient('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('USER_REPOSITORY'))
})
```

Use the scope to resolve dependencies instead of constructing them manually
inside the service.

## Resolve a transient service

After registration and application build, resolve a transient without scoped
dependencies from the application's root services:

```ts
const application = await app.build()
const userService = application.getServices().USER_SERVICE
```

Each resolution creates a new instance:

```ts
const first = application.getServices().USER_SERVICE
const second = application.getServices().USER_SERVICE

console.log(first === second) // false
```

This is the defining behavior of the transient lifetime.

## Create a new instance for every resolution

Choose transient lifetime when the service should not be shared between
resolutions.

For example:

```ts
class RequestOperation {
  public readonly id = crypto.randomUUID()
}

services.addTransient('REQUEST_OPERATION', () => {
  return new RequestOperation()
})
```

Resolving the service twice produces two different instances:

```ts
const first = application.getServices().REQUEST_OPERATION
const second = application.getServices().REQUEST_OPERATION

console.log(first.id)
console.log(second.id)

console.log(first === second) // false
```

This is different from a singleton, where both resolutions return the same
instance.

## Register a transient service with dependencies

Transient services can depend on other registered services.

```ts
class UserRepository {
  findById(id: string) {
    // ...
  }
}

class UserService {
  constructor(private readonly repository: UserRepository) {}

  async findUser(id: string) {
    return this.repository.findById(id)
  }
}
```

Register both services:

```ts
const app = new XenoAppBuilder().addServices((services) => {
  services.addScoped('USER_REPOSITORY', () => {
    return new UserRepository()
  })

  services.addTransient('USER_SERVICE', (scope) => {
    return new UserService(scope.resolve('USER_REPOSITORY'))
  })
})

const application = await app.build()
```

The transient `USER_SERVICE` gets its dependencies through the scope supplied to
its factory. `build()` returns the runtime application, not the container.

## Transient services and dependency lifetimes

A transient service can depend on services with different lifetimes.

For example:

```text
Transient
├── Singleton
└── Scoped
```

The important distinction is that the **transient service itself is recreated on
every resolution**. Its dependencies keep the lifetime with which they were
registered.

### Transient → Singleton

A transient service can resolve a singleton:

```ts
services.addSingleton('LOGGER', () => {
  return new Logger()
})

services.addTransient('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('LOGGER'))
})
```

Every `USER_SERVICE` resolution creates a new `UserService`, while `LOGGER`
remains shared according to its singleton lifetime.

### Transient → Scoped

A transient service can also resolve a scoped service when it is resolved inside
an active scope:

```ts
services.addScoped('REQUEST_STATE', () => {
  return new RequestState()
})

services.addTransient('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('REQUEST_STATE'))
})
```

Within the same service scope:

```text
resolve USER_SERVICE
        │
        ├── new UserService
        │
        └── REQUEST_STATE
              │
              └── same scoped instance for this scope
```

The transient service is recreated on every resolution, while the scoped
dependency follows the lifetime of the active scope. Resolve it through an
active request or operation scope to keep that boundary explicit.

Do not resolve this transient through root services when it depends on a scoped
service. The transient factory receives the root scope in that case, so the
scoped dependency would be associated with the root scope rather than a request
or operation scope.

See [Scoped](./scoped) for request and scope-based service lifetimes.

## Complete example

The following example registers a repository as scoped and a service as
transient:

```ts
import { XenoAppBuilder } from '@xeno-js/core'

class UserRepository {
  async findById(id: string) {
    return {
      id,
      name: 'Ada',
    }
  }
}

class UserService {
  constructor(private readonly repository: UserRepository) {}

  async findUser(id: string) {
    return this.repository.findById(id)
  }
}

const app = new XenoAppBuilder().addServices((services) => {
  services.addScoped('USER_REPOSITORY', () => {
    return new UserRepository()
  })

  services.addTransient('USER_SERVICE', (scope) => {
    return new UserService(scope.resolve('USER_REPOSITORY'))
  })
})

const application = await app.build()
```

When the transient service is resolved repeatedly:

```ts
await application.runInScope(async (services) => {
  const first = services.USER_SERVICE
  const second = services.USER_SERVICE

  console.log(first === second) // false
})
```

Each resolution creates a new `UserService`.

## Transient services in a request scope

A transient service does not create or own a scope.

When a transient service is resolved from an active scope, its factory receives
that scope and can resolve scoped dependencies from it:

```ts
await application.runInScope(async (services) => {
  const first = services.USER_SERVICE
  const second = services.USER_SERVICE

  console.log(first === second) // false
})
```

The transient instances differ, while any scoped dependency they resolve is
shared within this operation's scope. Each dependency retains its registered
lifetime independently of `USER_SERVICE`.

## Transient services and disposal

If a transient service exposes a `dispose()` method, Xeno.JS tracks the instance
for disposal.

For example:

```ts
class TemporaryResource {
  async dispose() {
    // Release the resource.
  }
}

services.addTransient('TEMPORARY_RESOURCE', () => {
  return new TemporaryResource()
})
```

When the service is resolved through a scope, the transient instance can be
disposed when that scope is disposed:

```ts
await application.runInScope(async (services) => {
  const resource = services.TEMPORARY_RESOURCE
  // Use resource within this operation.
})
```

The scope tracks disposable transient instances and disposes them when the
operation completes. Transient instances resolved from root services are tracked
by the root scope and disposed when the container is disposed.

If the service does not need cleanup, no `dispose()` method is required.

## Transient vs singleton

Use a **transient** service when each resolution should produce a new instance:

```ts
services.addTransient('USER_OPERATION', () => {
  return new UserOperation()
})
```

Use a **singleton** when the same instance should be reused:

```ts
services.addSingleton('USER_OPERATION', () => {
  return new UserOperation()
})
```

The difference is observable directly from resolution:

```ts
const first = application.getServices().USER_OPERATION
const second = application.getServices().USER_OPERATION

console.log(first === second)
```

With transient lifetime:

```text
resolve → instance A
resolve → instance B
resolve → instance C
```

With singleton lifetime:

```text
resolve ─┐
resolve ─┼→ same instance
resolve ─┘
```

See [Singleton](./singleton) for shared service instances.

## Transient vs scoped

Transient and scoped services are both created more than once, but they follow
different rules.

| Lifetime  | Instance reuse                          |
| --------- | --------------------------------------- |
| Transient | New instance for every resolution       |
| Scoped    | One instance per service scope          |
| Singleton | One instance for the container lifetime |

For example, with a single scope:

```ts
await application.runInScope(async (services) => {
  const transientA = services.TRANSIENT_SERVICE
  const transientB = services.TRANSIENT_SERVICE

  const scopedA = services.SCOPED_SERVICE
  const scopedB = services.SCOPED_SERVICE

  console.log(transientA === transientB) // false
  console.log(scopedA === scopedB) // true
})
```

Use transient when you need a new instance for every resolution.

Use scoped when you need one instance shared by resolutions inside the same
logical scope.

See [Scoped](./scoped) for scoped lifetime behavior.

## Common problems

### The service is not registered

If the token has not been registered, resolution fails with:

```text
[DI Container Error]: Registration not found for token 'USER_SERVICE'. Ensure the service is registered before resolving.
```

Check that the service is registered:

```ts
services.addTransient('USER_SERVICE', () => {
  return new UserService()
})
```

Also make sure the registration is part of the application configuration before
calling `build()`.

### The token is already registered

A token can only have one registration in the container.

Registering the same token again throws:

```text
[DI Container Error]: The token 'USER_SERVICE' is already registered in the container.
```

Check for duplicate registrations in:

- `addServices()`;
- custom modules;
- framework modules;
- application bootstrap configuration.

If the service should have different behavior, use a different token rather than
registering the same token twice.

### Resolve transient services with scoped dependencies in an execution scope

Consider:

```ts
services.addScoped('REQUEST_STATE', () => {
  return new RequestState()
})

services.addTransient('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('REQUEST_STATE'))
})
```

Resolve `USER_SERVICE` within an execution scope:

```ts
await application.runInScope(async (services) => {
  const userService = services.USER_SERVICE
  // Use userService within this scope.
})
```

`runInScope()` ensures the scoped dependency belongs to the operation's scope.
The container rejects resolving a scoped token directly from root services:

```ts
application.getServices().REQUEST_STATE
```

When a transient service is resolved directly from root services, its factory
receives the root scope. If it resolves a scoped dependency, that dependency is
therefore associated with the root scope rather than a request or operation
scope. Avoid relying on this for request-specific services; use the active
request scope or `runInScope()` instead.

See [Scoped](./scoped).

### The service unexpectedly shares state

If two resolutions return the same instance, verify that the service was
registered as transient:

```ts
services.addTransient('USER_SERVICE', () => {
  return new UserService()
})
```

If it was registered with `addSingleton()`, the same instance is intentionally
reused.

If it was registered with `addScoped()`, the same instance is intentionally
reused within the active scope.

## When to use transient

Use transient lifetime when:

- each resolution should produce an independent instance;
- the service does not need shared instance state;
- the service is lightweight to construct;
- the service represents an operation or short-lived object;
- sharing one instance could introduce unwanted state between consumers.

Typical examples include:

- operation-specific services;
- lightweight application services;
- request-independent helper objects;
- factories or adapters whose instances should not be reused.

## When not to use transient

Avoid transient lifetime when you explicitly need one shared instance.

For application-wide shared state or resources, consider a singleton:

```ts
services.addSingleton('MY_LOGGER', () => {
  return new Logger()
})
```

For state that should be shared only within a logical scope, consider scoped
lifetime:

```ts
services.addScoped('REQUEST_STATE', () => {
  return new RequestState()
})
```

Do not choose a lifetime only because the service is small or large. Choose it
according to the required instance reuse behavior.

## Checklist

Before registering a service as transient, verify:

- [ ] A service token exists in the application registry.
- [ ] The service is registered with `addTransient()`.
- [ ] The factory creates a new instance.
- [ ] Dependencies are resolved through the factory's `scope`.
- [ ] Any scoped dependency is resolved within an active scope.
- [ ] The service does not require shared instance state.
- [ ] Disposable resources are released when the owning scope is disposed.

## Related docs

- [Service Container](../service-container) — Learn where services are
  registered and resolved.
- [Registration](../registration) — Learn how to register services and choose a
  lifetime.
- [Singleton](./singleton) — Share one service instance across the container.
- [Scoped](./scoped) — Share one service instance within a logical scope.
- [Resolution](../resolution) — Learn how to resolve registered services.
- [Dependency Graph](../dependency-graph) — Learn how services depend on one
  another.
- [Captive Dependencies](../captive-dependencies) — Learn how incompatible
  service lifetimes can cause dependency problems.

---

---

<a id="source-6"></a>

# Source: `docs/dependency-injection/registration.md`

## Introduction

Register application services explicitly with Xeno.JS dependency injection.

A registration connects:

- a **token** used to identify the service
- a **factory** used to create the service
- a **lifetime** that controls how the created instance is reused

The basic registration API is:

```ts
services.addSingleton(token, factory)
services.addScoped(token, factory)
services.addTransient(token, factory)
```

Xeno.JS does not require decorators or runtime service scanning. You register
the services you want to make available to the application.

## Before you start

You need:

- a Xeno.JS application using `XenoAppBuilder`
- a service implementation
- a token available in the application registry
- any dependencies required by the service

For most applications, register application services through
`AppBuilder.addServices()`.

## Register a service

Use `addServices()` to configure the application's service container.

```ts
import { XenoAppBuilder } from '@xeno-js/core'

const app = new XenoAppBuilder()

app.addServices((services) => {
  services.addScoped('USER_REPOSITORY', (scope) => {
    return new UserRepository()
  })
})

await app.build()
```

`addServices()` receives the service container and returns the `XenoAppBuilder`,
so registrations can be chained with other application configuration.

The registration callback is applied when `build()` bootstraps the application.

## Choose a registration lifetime

Xeno.JS provides three registration methods:

| Method           | Instance behavior                       |
| ---------------- | --------------------------------------- |
| `addSingleton()` | One instance for the container lifetime |
| `addScoped()`    | One instance per service scope          |
| `addTransient()` | A new instance for each resolution      |

Choose the lifetime based on how the service should behave. The lifetime itself
is not part of the token; it is defined when the service is registered.

For detailed lifetime guidance, see:

- [Singleton](./lifetimes/singleton)
- [Scoped](./lifetimes/scoped)
- [Transient](./lifetimes/transient)

## Register a singleton

Use `addSingleton()` when the application should reuse the same service
instance.

```ts
app.addServices((services) => {
  services.addSingleton('USER_MAPPER', () => {
    return new UserMapper()
  })
})
```

The factory is called when the service is first resolved. Subsequent resolutions
reuse the same instance.

Typical examples include services that are intentionally shared across the
application, such as stateless infrastructure services or reusable
configuration-oriented components.

## Register a scoped service

Use `addScoped()` when the service should have one instance within a logical
service scope.

```ts
app.addServices((services) => {
  services.addScoped('USER_REPOSITORY', (scope) => {
    return new UserRepository()
  })
})
```

A scoped service must be resolved through an active scope.

For request-oriented applications, scoped services are commonly used for
dependencies whose lifetime should follow the current request or execution
scope.

```ts
const application = await app.build()

await application.runInScope(async (services) => {
  const repository = services.USER_REPOSITORY
})
```

Do not resolve a scoped service directly from the root container:

```ts
const application = await app.build()

// Invalid: USER_REPOSITORY is scoped.
application.getServices().USER_REPOSITORY
```

For more information, see [Scoped](./lifetimes/scoped).

## Register a transient service

Use `addTransient()` when a new service instance should be created for each
resolution.

```ts
app.addServices((services) => {
  services.addTransient('USER_SERVICE', () => {
    return new UserService()
  })
})
```

Transient services are useful when the service does not need to retain instance
state between resolutions.

For more information, see [Transient](./lifetimes/transient).

## Register a service with dependencies

Service factories receive the current service scope.

Use that scope to resolve dependencies explicitly:

```ts
app.addServices((services) => {
  services.addScoped('USER_REPOSITORY', (scope) => {
    return new UserRepository(scope.resolve('USER_DATA_SOURCE'))
  })

  services.addScoped('USER_SERVICE', (scope) => {
    return new UserService(scope.resolve('USER_REPOSITORY'))
  })
})
```

This creates the following dependency relationship:

```text
USER_SERVICE
    │
    └── USER_REPOSITORY
            │
            └── USER_DATA_SOURCE
```

The dependency is visible directly in the registration code.

You do not need a decorator or constructor metadata to tell Xeno.JS how the
dependency should be obtained.

## Register a complete service graph

A typical application can register multiple services in one `addServices()`
call:

```ts
import { XenoAppBuilder } from '@xeno-js/core'

const app = new XenoAppBuilder()

app.addServices((services) => {
  services.addSingleton('USER_MAPPER', () => {
    return new UserMapper()
  })

  services.addScoped('USER_DATA_SOURCE', (scope) => {
    return new UserDataSource(scope.resolve('DB_CONTEXT'))
  })

  services.addScoped('USER_REPOSITORY', (scope) => {
    return new UserRepository(
      scope.resolve('USER_DATA_SOURCE'),
      scope.resolve('USER_MAPPER'),
    )
  })

  services.addScoped('USER_SERVICE', (scope) => {
    return new UserService(scope.resolve('USER_REPOSITORY'))
  })
})

const application = await app.build()
```

The important part is that every dependency used by a factory must itself be
registered and available under the expected token. `build()` returns the
configured runtime application, not the service container.

## Register services with custom application tokens

Xeno.JS uses a typed application registry to associate service tokens with their
service types.

The registry includes built-in service tokens such as:

- `CONFIGURATION_SERVICE`
- `LOGGER`
- `MEDIATOR`
- `REQUEST_CONTEXT`
- `CACHE`
- `CONCURRENCY_SERVICE`
- `DB_CONTEXT`
- `HTTP_ADAPTER`

Custom application services can extend the application registry when your
application needs additional typed tokens.

The registry is what allows calls such as:

```ts
scope.resolve('USER_SERVICE')
```

to be checked against the registered token type by TypeScript.

Keep custom tokens stable and descriptive. A token should identify the service
contract or role rather than an implementation detail.

For example:

```ts
'USER_REPOSITORY'
```

is preferable to:

```ts
'POSTGRES_USER_REPOSITORY_V2'
```

when the application depends on the repository abstraction rather than its
concrete implementation.

## Register an existing service implementation

Factories are regular TypeScript functions, so the implementation can be
constructed directly:

```ts
services.addScoped('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('USER_REPOSITORY'))
})
```

The factory can also contain configuration required to construct the service:

```ts
services.addSingleton('USER_CLIENT', (scope) => {
  const configuration = scope.resolve('CONFIGURATION_SERVICE')

  return new UserClient({
    baseUrl: configuration.get('USER_API_URL'),
  })
})
```

Keep the factory focused on composition. Business behavior should remain in the
service itself.

## Register services through a module

If a feature contains several related services, you can package their
registrations in an Xeno.JS module.

A module receives the service container during configuration:

```ts
import type { IModule, IServiceContainer } from '@xeno-js/core'

export class UserModule implements IModule {
  async configure(container: IServiceContainer): Promise<void> {
    container.addScoped('USER_REPOSITORY', (scope) => {
      return new UserRepository(scope.resolve('USER_DATA_SOURCE'))
    })

    container.addScoped('USER_SERVICE', (scope) => {
      return new UserService(scope.resolve('USER_REPOSITORY'))
    })
  }
}
```

Register the module with the application:

```ts
const app = new XenoAppBuilder()

app.addModule('UserModule', async () => {
  return new UserModule()
})

await app.build()
```

Use direct `addServices()` registrations for application-level composition and
modules when a feature owns a cohesive group of registrations.

## Do not register the same token twice

Each token can have one registration in a service container.

This is invalid:

```ts
services.addScoped('USER_SERVICE', () => {
  return new UserService()
})

services.addTransient('USER_SERVICE', () => {
  return new UserService()
})
```

The second registration throws a container error:

```text
[DI Container Error]: The token 'USER_SERVICE' is already registered in the container.
```

If you need different implementations, give them different tokens or change the
existing registration rather than registering the same token twice.

## Make sure every dependency is registered

If a factory resolves a token that has no registration, resolution fails.

For example:

```ts
services.addScoped('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('USER_REPOSITORY'))
})
```

If `USER_REPOSITORY` has not been registered, Xeno.JS reports:

```text
[DI Container Error]: Registration not found for token 'USER_REPOSITORY'. Ensure the service is registered before resolving.
```

Check the registration graph:

```text
USER_SERVICE
    │
    └── USER_REPOSITORY
```

Then make sure both nodes are registered.

## Avoid resolving scoped services from the root container

A scoped registration requires an active scope.

This is invalid:

```ts
services.addScoped('USER_REPOSITORY', () => {
  return new UserRepository()
})

const application = await app.build()

application.getServices().USER_REPOSITORY
```

Xeno.JS reports:

```text
[DI Container Error]: Scoped service 'USER_REPOSITORY' requires an active scope.
```

Resolve it within an operation scope instead:

```ts
await application.runInScope(async (services) => {
  const repository = services.USER_REPOSITORY
})
```

If the service is intentionally application-wide, consider whether
`addSingleton()` is the appropriate lifetime instead.

## Avoid circular registrations

A registration graph must have a valid dependency path.

For example:

```text
SERVICE_A
    │
    └── SERVICE_B
            │
            └── SERVICE_A
```

This creates a circular dependency:

```ts
services.addScoped('SERVICE_A', (scope) => {
  return new ServiceA(scope.resolve('SERVICE_B'))
})

services.addScoped('SERVICE_B', (scope) => {
  return new ServiceB(scope.resolve('SERVICE_A'))
})
```

Xeno.JS detects the cycle during resolution and reports a circular dependency
error.

To fix it, change the dependency direction or extract the shared responsibility
into another service.

For dependency-graph guidance, see [Dependency Graph](./dependency-graph).

## Avoid invalid lifetime combinations

Registration lifetime affects which dependencies a service can safely use.

A singleton should not capture a scoped service:

```ts
services.addSingleton('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('REQUEST_SERVICE'))
})

services.addScoped('REQUEST_SERVICE', () => {
  return new RequestService()
})
```

This creates a lifetime mismatch:

```text
Singleton
USER_SERVICE
    │
    └── Scoped
        REQUEST_SERVICE
```

While constructing a singleton, the container detects this situation and reports
a captive dependency error. The same check applies when the resolution reaches a
scoped registration through a transient dependency. It does not validate every
lifetime combination or root-scope use.

If the dependency is request- or scope-specific, consider making the consuming
service scoped as well:

```ts
services.addScoped('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('REQUEST_SERVICE'))
})
```

See [Captive Dependencies](./captive-dependencies) for more details.

## Build the application after registration

`addServices()` configures the builder. The registrations are applied when the
application is built.

```ts
const app = new XenoAppBuilder()

app.addServices((services) => {
  services.addScoped('USER_SERVICE', () => {
    return new UserService()
  })
})

const application = await app.build()
```

`build()` returns the configured runtime application, not the service container.
Resolve root services from the application:

```ts
const userService = application.getServices().USER_SERVICE
```

For scoped services, run the operation in a scope:

```ts
await application.runInScope(async (services) => {
  const userService = services.USER_SERVICE
})
```

## Troubleshooting

### "The token is already registered"

Check whether the same token is registered more than once.

```text
[DI Container Error]: The token 'USER_SERVICE' is already registered in the container.
```

Search your application modules and `addServices()` callbacks for the token.

### "Registration not found"

The factory is resolving a token that has not been registered.

```text
[DI Container Error]: Registration not found for token 'USER_REPOSITORY'. Ensure the service is registered before resolving.
```

Check that:

1. the token is spelled correctly
2. the registration exists
3. the registration is part of the application being built
4. the dependency is available before the service is resolved

### "Scoped service requires an active scope"

The service was registered with `addScoped()` but was resolved from the root
container.

Use:

```ts
await application.runInScope(async (services) => {
  const service = services.USER_SERVICE
})
```

instead of:

```ts
application.getServices().USER_SERVICE
```

### TypeScript does not accept my token

Check the application's registry.

Xeno.JS types service registrations and resolutions through the registry, so a
custom token must be represented in the registry used by the application.

Do not work around a registry error by casting the container to an unrelated
type. Extend the application's registry so the token and service type remain
connected.

### A service works in one place but fails in another

Check the service lifetime and resolution context.

In particular:

- `singleton` services can be resolved from root services
- `transient` services can be resolved from root or scoped services; if they
  depend on scoped services, resolve them inside the intended operation scope
- `scoped` services require an active scope

If the service depends on scoped services, also check the lifetime of the
consuming service.

## Registration checklist

Before considering a registration complete, verify:

- [ ] The service has a clear token.
- [ ] The token exists in the application registry.
- [ ] The correct lifetime is selected.
- [ ] Every dependency used by the factory is registered.
- [ ] Scoped dependencies are resolved from an active scope.
- [ ] No token is registered more than once.
- [ ] The dependency graph contains no circular dependency.
- [ ] A singleton does not depend on a scoped service.
- [ ] The application calls `build()` after configuring registrations.

## Related docs

- [Service Container](../dependency-injection/service-container)
- [Singleton](../dependency-injection/lifetimes/singleton)
- [Scoped](../dependency-injection/lifetimes/scoped)
- [Transient](../dependency-injection/lifetimes/transient)
- [Resolution](../dependency-injection/resolution)
- [Dependency Graph](../dependency-injection/dependency-graph)
- [Captive Dependencies](../dependency-injection/captive-dependencies)

---

---

<a id="source-7"></a>

# Source: `docs/dependency-injection/resolution.md`

## Introduction

If you have already registered a service and need to use it, the runtime
application provides separate APIs for root and scoped resolution.

The method is type-safe: the service token determines the type returned by the
registry.

Use:

- `application.getServices()` for singleton or transient services that can be
  resolved from the root. Resolve a transient with scoped dependencies inside
  the intended operation scope.
- `application.getScopedServices()` inside an existing request or execution
  scope.
- `application.runInScope()` when an operation needs a new scope.

## Before you start

You need:

1. A service registered in the container.
2. The service token used by that registration.
3. An active scope if the service uses the `scoped` lifetime.

For example:

```ts
const app = new XenoAppBuilder().addServices((services) => {
  services.addSingleton('USER_SERVICE', () => {
    return new UserService()
  })
})

const application = await app.build()
```

`build()` returns the runtime application, not the service container. Resolve
the service from its root services:

```ts
const userService = application.getServices().USER_SERVICE
```

## Resolve a service with `XenoAppBuilder`

`XenoAppBuilder` exposes a root resolution convenience:

```ts
resolve<K extends keyof TRegistry>(token: K): TRegistry[K]
```

After `build()`, prefer the runtime application's `getServices()` API:

```ts
const app = new XenoAppBuilder().addServices((services) => {
  services.addSingleton('USER_SERVICE', () => {
    return new UserService()
  })
})

const application = await app.build()
const userService = application.getServices().USER_SERVICE

await userService.findById('123')
```

`XenoAppBuilder.resolve()` delegates to the root service container. It cannot
resolve scoped registrations because they require an active scope.

### When to use it

Use `app.resolve()` only for root-resolvable services when code intentionally
has access to the builder. For application runtime code, use
`application.getServices()`.

For request-scoped application code, prefer resolving through the active scope
instead.

## Resolve from the runtime application

After building the application, `build()` returns the configured runtime
application, not the `IServiceContainer`.

```ts
const app = new XenoAppBuilder().addServices((services) => {
  services.addSingleton('USER_SERVICE', () => {
    return new UserService()
  })
})

const application = await app.build()

const userService = application.getServices().USER_SERVICE
```

The application proxy uses the registry to associate each property with its
service type. The builder also exposes `resolve()` for root-container
registrations, but the built application is the runtime API.

## Resolve a service from a scope

Scoped services cannot be resolved directly from the root container.

Register the service as scoped:

```ts
const app = new XenoAppBuilder().addServices((services) => {
  services.addScoped('REQUEST_SERVICE', () => {
    return new RequestService()
  })
})

const application = await app.build()
```

This does **not** work:

```ts
const requestService = application.getServices().REQUEST_SERVICE
```

Xeno.JS throws:

```text
[DI Container Error]: Scoped service 'REQUEST_SERVICE' requires an active scope.
```

Run the operation within a scope instead:

```ts
await application.runInScope(async (services) => {
  const requestService = services.REQUEST_SERVICE
  await requestService.execute()
})
```

`runInScope()` creates the request context and resolution scope for the
operation, then disposes the scope when the operation completes.

## Resolve dependencies inside a service factory

Service factories receive the current `IServiceScope`.

This allows a service to explicitly resolve its dependencies:

```ts
services.addScoped('USER_REPOSITORY', (scope) => {
  return new UserRepository(
    scope.resolve('USER_DATA_SOURCE'),
    scope.resolve('USER_MAPPER'),
  )
})
```

The dependency graph is therefore explicit in the registration:

```text
USER_REPOSITORY
├── USER_DATA_SOURCE
└── USER_MAPPER
```

You do not need decorators or runtime dependency scanning.

## Resolve different service lifetimes

The result of `resolve()` depends on the registered lifetime.

### Singleton

A singleton returns the same instance from the same container:

```ts
const first = application.getServices().CONFIGURATION_SERVICE
const second = application.getServices().CONFIGURATION_SERVICE

console.log(first === second)
// true
```

### Scoped

A scoped service returns the same instance within one scope:

```ts
await application.runInScope(async (services) => {
  const first = services.REQUEST_SERVICE
  const second = services.REQUEST_SERVICE

  console.log(first === second)
  // true
})
```

A separate `runInScope()` call creates a different scope and therefore a
different scoped instance. Do not retain a scoped service beyond the callback
that owns its scope.

### Transient

A transient service creates a new instance for every resolution:

```ts
const sameInstance = await application.runInScope(async (services) => {
  const first = services.USER_FACTORY
  const second = services.USER_FACTORY
  return first === second
})

console.log(sameInstance)
// false
```

For a transient without scoped dependencies, `getServices()` can also resolve it
from the root. If it depends on scoped services, resolve it through the intended
execution scope.

See [Transient](./lifetimes/transient) for details.

## Resolve a scoped service from the current execution

When a request or other application execution already has an active scope, use
`getScopedServices()` to resolve services through it:

```ts
const { REQUEST_SERVICE: requestService } = application.getScopedServices()

await requestService.execute()
```

`getScopedServices()` throws if there is no active scope. For work outside an
existing request or execution, use `runInScope()`:

```ts
await application.runInScope(async (services) => {
  const requestService = services.REQUEST_SERVICE
  await requestService.execute()
})
```

The scope is disposed when the `runInScope()` operation completes. Do not retain
scoped services for use after that execution ends.

## Resolve built-in Xeno.JS services

Xeno.JS also registers services identified by its built-in registry tokens.

For example:

```ts
const { mediator, logger, requestContext } = application.getServices()
```

The registry associates each token with its corresponding service type.

For example, `MEDIATOR` resolves to the mediator contract, while
`REQUEST_CONTEXT` resolves to the request-context contract.

This means TypeScript can infer the result of the resolution:

```ts
const { mediator } = application.getServices()

await mediator.send(command)
```

No manual type cast is required.

## Resolve a service by its custom token

Custom services can extend the application registry so their tokens remain
type-safe.

For example:

```ts
interface UserServiceRegistry extends XenoRegistry {
  USER_SERVICE: UserService
}
```

Use the application-specific registry as the generic parameter when creating the
builder:

```ts
const app = new XenoAppBuilder<UserServiceRegistry>()
```

Once the token is part of the registry, the same resolution API applies:

```ts
const userService = application.getServices().USER_SERVICE
```

The returned value is typed as `UserService`.

## Common problems

### Registration not found

If you resolve a token that has not been registered:

```ts
const userService = application.getServices().USER_SERVICE
```

Xeno.JS throws:

```text
[DI Container Error]: Registration not found for token 'USER_SERVICE'. Ensure the service is registered before resolving.
```

Check that:

1. The service is registered.
2. The registration runs before resolution.
3. The token is spelled exactly the same in registration and resolution.
4. The module containing the registration has been configured before the service
   is resolved.

For application-level registrations, prefer configuring them through
`addServices()`:

```ts
const app = new XenoAppBuilder().addServices((services) => {
  services.addScoped('USER_SERVICE', () => {
    return new UserService()
  })
})

const application = await app.build()
```

### Scoped service requires an active scope

If you see:

```text
[DI Container Error]: Scoped service 'USER_SERVICE' requires an active scope.
```

the service was resolved from the root container even though it was registered
as scoped.

Use `runInScope()` when the operation needs its own scope:

```ts
await application.runInScope(async (services) => {
  const userService = services.USER_SERVICE
})
```

When already inside request execution, resolve through the active scope with
`getScopedServices()`.

### Scope has been closed

A disposed scope cannot be used for further resolution.

`runInScope()` disposes its scope when the callback completes. Do not retain
scoped services and use them after the execution has ended; their owning scope
has been disposed and cannot perform further resolutions. A disposed
`IServiceScope` throws:

```text
[DI Container Error]: Unable to resolve 'USER_SERVICE'. The scope has been closed.
```

Do not retain a scoped service or scope beyond the lifetime in which it is
valid.

### Duplicate registration

A token can only be registered once in a container.

For example:

```ts
services.addSingleton('USER_SERVICE', () => new UserService())

services.addScoped('USER_SERVICE', () => new UserService())
```

throws:

```text
[DI Container Error]: The token 'USER_SERVICE' is already registered in the container.
```

Choose one registration for the token or use a different token.

## Complete example

The following example registers a repository and a service, then resolves the
service from a request scope.

```ts
import { XenoAppBuilder } from '@xeno-js/core'

const app = new XenoAppBuilder().addServices((services) => {
  services.addScoped('USER_REPOSITORY', (scope) => {
    return new UserRepository(scope.resolve('USER_DATA_SOURCE'))
  })

  services.addScoped('USER_SERVICE', (scope) => {
    return new UserService(scope.resolve('USER_REPOSITORY'))
  })
})

const application = await app.build()

const user = await application.runInScope(async (services) => {
  const userService = services.USER_SERVICE
  return userService.findById('123')
})
```

The important part is that the resolution follows the lifetime boundary:

```text
application
   |
   +-- runInScope()
          |
          +-- USER_SERVICE
                 |
                 +-- USER_REPOSITORY
                        |
                        +-- USER_DATA_SOURCE
```

The services registered as scoped are resolved within the same scope.

## Choosing the resolution API

| Situation                                          | Use                               |
| -------------------------------------------------- | --------------------------------- |
| Resolve a root-level singleton or transient        | `application.getServices()`       |
| Resolve a service in an active request scope       | `application.getScopedServices()` |
| Create a scope for an operation                    | `application.runInScope()`        |
| Resolve a dependency inside a registration factory | `scope.resolve(token)`            |

The key rule is simple:

> **Resolve scoped services through an active scope.**

## Checklist

Before resolving a service, verify:

- [ ] The service has been registered.
- [ ] You are using the same token used during registration.
- [ ] The application has been built when using `XenoAppBuilder`.
- [ ] You are using an active scope for scoped services.
- [ ] You are not resolving a service from a disposed scope.
- [ ] The custom token is included in the application registry when using custom
      services.

## Related docs

- [Service Container](./service-container) — understand the container used to
  register and resolve services.
- [Registration](./registration) — register services with Xeno.JS.
- [Singleton](./lifetimes/singleton) — register services with one shared
  instance.
- [Scoped](./lifetimes/scoped) — register services with one instance per scope.
- [Transient](./lifetimes/transient) — create a new instance on every
  resolution.
- [Dependency Graph](./dependency-graph) — understand how registered services
  depend on each other.
- [Captive Dependencies](./captive-dependencies) — troubleshoot invalid lifetime
  dependencies.

---

---

<a id="source-8"></a>

# Source: `docs/dependency-injection/service-container.md`

## Introduction

Xeno.JS uses a **service container** to register and resolve the dependencies
used by your application.

The container gives you explicit dependency injection without requiring
decorators, runtime scanning, or implicit dependency discovery.

Use it when you need to:

- register application services;
- connect an interface or token to an implementation;
- define how long a service instance should live;
- resolve dependencies from other services;
- share dependencies across application components;
- keep dependency configuration in your application composition root.

For most applications, you interact with the container through `XenoAppBuilder`
rather than creating a `ServiceContainer` directly.

## Before you start

You need `@xeno-js/core` installed:

```bash
npm install @xeno-js/core
```

The usual application entry point is an `XenoAppBuilder`:

```ts
import { XenoAppBuilder } from '@xeno-js/core'

const app = new XenoAppBuilder()
```

`XenoAppBuilder` owns the application's service container and exposes the
registration APIs used during composition. `build()` returns the configured
runtime application, not the container.

---

## Register Services

Register application services with `XenoAppBuilder.addServices()`.

The callback receives the service registration API:

```ts
import { XenoAppBuilder } from '@xeno-js/core'

const app = new XenoAppBuilder()

app.addServices((services) => {
  services.addScoped('USER_REPOSITORY', (scope) => {
    return new UserRepository(scope.resolve('USER_DATA_SOURCE'))
  })
})
```

The registration consists of:

1. a **token** that identifies the service;
2. a **factory** that creates the service;
3. a **lifetime** that determines how the created instance is reused.

The factory receives the current service scope, so dependencies can be resolved
explicitly:

```ts
services.addScoped('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('USER_REPOSITORY'))
})
```

This makes the dependency relationship visible directly in the registration.

### Register multiple services

You can register multiple services in the same `addServices()` callback:

```ts
const app = new XenoAppBuilder()

app.addServices((services) => {
  services.addScoped('USER_REPOSITORY', (scope) => {
    return new UserRepository(scope.resolve('USER_DATA_SOURCE'))
  })

  services.addScoped('USER_SERVICE', (scope) => {
    return new UserService(scope.resolve('USER_REPOSITORY'))
  })

  services.addTransient('USER_CONTROLLER', (scope) => {
    return new UserController(scope.resolve('USER_SERVICE'))
  })
})
```

`addServices()` is chainable:

```ts
const app = new XenoAppBuilder().addServices((services) => {
  services.addScoped('USER_REPOSITORY', (scope) => {
    return new UserRepository(scope.resolve('USER_DATA_SOURCE'))
  })
})
```

---

## Choose a Service Lifetime

Every registration must use one of the available lifetimes:

| Method           | Instance behavior                | Typical use                            |
| ---------------- | -------------------------------- | -------------------------------------- |
| `addSingleton()` | One instance for the container   | Shared application services            |
| `addScoped()`    | One instance per scope           | Request or operation-specific services |
| `addTransient()` | New instance for each resolution | Lightweight, stateless services        |

For details, see:

- [Singleton](./lifetimes/singleton)
- [Scoped](./lifetimes/scoped)
- [Transient](./lifetimes/transient)

---

## Resolve a Service

`XenoAppBuilder.resolve()` can resolve a registered root-level service:

```ts
const app = new XenoAppBuilder().addServices((services) => {
  services.addSingleton('CONFIG_SERVICE', () => {
    return new ConfigService()
  })
})

const config = app.resolve('CONFIG_SERVICE') //`XenoAppBuilder.resolve()` before build()

await app.build()

const config = app.getServices().CONFIG_SERVICE // `getServices()` after build()
```

`XenoAppBuilder.resolve()` delegates resolution to the root service container.

Use this builder method for root-level resolution during composition, such as
singleton and transient services without scoped dependencies. In runtime code,
use `getServices()` on the application returned by `build()`.

Scoped services are different: they require an active scope.

---

## Resolve Dependencies Inside a Factory

The registration factory receives a service scope.

Use that scope to resolve the dependencies required to construct your service:

```ts
services.addScoped('USER_SERVICE', (scope) => {
  const repository = scope.resolve('USER_REPOSITORY')
  const logger = scope.resolve('LOGGER')

  return new UserService(repository, logger)
})
```

The scope supplied to a factory depends on the registration and resolution:
singleton factories receive the root scope, while scoped and transient factories
receive the scope used to resolve them.

This is the standard Xeno.JS pattern for explicit dependency injection.

You do not need to manually construct the dependency graph at every call site:

```ts
// Avoid constructing the dependency graph manually.
const repository = new UserRepository(...)
const logger = new Logger(...)
const service = new UserService(repository, logger)
```

Instead, register the graph once:

```ts
services.addScoped('USER_SERVICE', (scope) => {
  return new UserService(
    scope.resolve('USER_REPOSITORY'),
    scope.resolve('LOGGER'),
  )
})
```

Then resolve the service where it is needed.

---

## Use a Service Scope

A service scope is required when working with scoped services.

`build()` returns the runtime application, not the service container. For
application work that needs scoped services, use `runInScope()`:

```ts
const application = await app.build()

await application.runInScope(async (services) => {
  const repository = services.USER_REPOSITORY
})
```

`runInScope()` creates the request context and service scope for the operation,
then disposes the scope when the operation completes.

Within the same scope:

```ts
await application.runInScope(async (services) => {
  const first = services.USER_REPOSITORY
  const second = services.USER_REPOSITORY

  console.log(first === second) // true
})
```

A different scope receives a different scoped instance:

```ts
const repository1 = await application.runInScope(
  async (services) => services.USER_REPOSITORY,
)
const repository2 = await application.runInScope(
  async (services) => services.USER_REPOSITORY,
)

console.log(repository1 === repository2) // false
```

For request-scoped application code, Xeno.JS can manage the active scope through
its context system. See [Context & Scopes](../context-scopes).

---

## Resolve a Scoped Service from the Application

When an execution already has an active scope, use `getScopedServices()`:

```ts
const { USER_REPOSITORY: repository } = application.getScopedServices()
```

If there is no active scope, `getScopedServices()` throws. For work outside an
existing request or execution, use `runInScope()` as shown above. See
[Context & Scopes](../context-scopes) for more information about active scopes.

---

## Build the Application

Service registrations added through `XenoAppBuilder.addServices()` are applied
when the application is built.

A typical bootstrap looks like this:

```ts
import { XenoAppBuilder } from '@xeno-js/core'

const app = new XenoAppBuilder()

app.addServices((services) => {
  services.addScoped('USER_REPOSITORY', (scope) => {
    return new UserRepository(scope.resolve('USER_DATA_SOURCE'))
  })

  services.addScoped('USER_SERVICE', (scope) => {
    return new UserService(scope.resolve('USER_REPOSITORY'))
  })
})

const application = await app.build()
```

After `build()` completes, use the returned runtime application to access
registered services. It is not the service container.

---

## Complete Example

The following example shows a small dependency graph:

```text
USER_CONTROLLER
       |
       v
USER_SERVICE
       |
       v
USER_REPOSITORY
       |
       v
USER_DATA_SOURCE
```

Register the services explicitly:

```ts
import { XenoAppBuilder } from '@xeno-js/core'

const app = new XenoAppBuilder()

app.addServices((services) => {
  services.addScoped('USER_DATA_SOURCE', () => {
    return new UserDataSource()
  })

  services.addScoped('USER_REPOSITORY', (scope) => {
    return new UserRepository(scope.resolve('USER_DATA_SOURCE'))
  })

  services.addScoped('USER_SERVICE', (scope) => {
    return new UserService(scope.resolve('USER_REPOSITORY'))
  })

  services.addTransient('USER_CONTROLLER', (scope) => {
    return new UserController(scope.resolve('USER_SERVICE'))
  })
})

const application = await app.build()
```

The dependency graph is defined entirely through registrations.

When `USER_CONTROLLER` is resolved in a scope, its factory resolves
`USER_SERVICE`, which resolves `USER_REPOSITORY`, which resolves
`USER_DATA_SOURCE`:

```ts
await application.runInScope(async (services) => {
  const controller = services.USER_CONTROLLER
})
```

---

## Use Custom Registries

Xeno.JS uses a typed registry to associate service tokens with their TypeScript
types.

This means registrations and resolutions are type-checked.

For example, if your registry contains:

```ts
type ApplicationRegistry = {
  USER_REPOSITORY: UserRepository
  USER_SERVICE: UserService
}
```

then:

```ts
services.addScoped('USER_REPOSITORY', () => {
  return new UserRepository()
})
```

and:

```ts
const service = app.resolve('USER_SERVICE')
```

are checked against the registered token types.

This allows the dependency injection API to remain explicit while preserving
TypeScript type safety.

For applications extending the Xeno.JS registry, see
[Registration](./registration).

---

## Registering the Same Token Twice

A token can only have one registration in a container.

This is invalid:

```ts
services.addScoped('USER_SERVICE', () => {
  return new UserService()
})

services.addSingleton('USER_SERVICE', () => {
  return new UserService()
})
```

The second registration throws a container error similar to:

```text
[DI Container Error]: The token 'USER_SERVICE' is already registered in the container.
```

If you need different implementations, use different tokens.

---

## Missing Registrations

Resolving a token that has not been registered fails.

For example:

```ts
const app = new XenoAppBuilder()

app.addServices((services) => {
  services.addScoped('USER_SERVICE', () => {
    return new UserService()
  })
})

const application = await app.build()

app.resolve('USER_REPOSITORY')
```

The container reports:

```text
[DI Container Error]: Registration not found for token 'USER_REPOSITORY'. Ensure the service is registered before resolving.
```

When this happens, check:

1. the token used by `resolve()`;
2. the token used during registration;
3. that the registration is inside `addServices()`;
4. that the token is registered before attempting resolution.

Token names must match exactly.

---

## Scoped Service Resolution Errors

This is invalid:

```ts
services.addScoped('USER_REPOSITORY', () => {
  return new UserRepository()
})

const repository = app.resolve('USER_REPOSITORY')
```

A scoped service requires an active scope.

The container reports:

```text
[DI Container Error]: Scoped service 'USER_REPOSITORY' requires an active scope.
```

Use an application scope instead:

```ts
const application = await app.build()

await application.runInScope(async (services) => {
  const repository = services.USER_REPOSITORY
})
```

Or, when an application scope already exists:

```ts
const { USER_REPOSITORY: repository } = application.getScopedServices()
```

---

## Circular Dependencies

The container detects circular dependencies during resolution.

For example:

```ts
services.addTransient('SERVICE_A', (scope) => {
  return new ServiceA(scope.resolve('SERVICE_B'))
})

services.addTransient('SERVICE_B', (scope) => {
  return new ServiceB(scope.resolve('SERVICE_A'))
})
```

Resolving `SERVICE_A` produces a circular dependency error:

```text
[DI Circular Dependency Error]: Detected circular dependency while resolving 'SERVICE_A'.
```

If this happens, inspect the dependency graph and remove the cycle.

See [Dependency Graph](./dependency-graph) for strategies for structuring
service dependencies.

---

## Captive Dependencies

A singleton must not depend on a scoped service.

For example:

```ts
services.addScoped('REQUEST_SERVICE', () => {
  return new RequestService()
})

services.addSingleton('APPLICATION_SERVICE', (scope) => {
  return new ApplicationService(scope.resolve('REQUEST_SERVICE'))
})
```

Resolving the singleton causes a captive dependency error.

```text
[DI Captive Dependency Error]: Attempted to resolve a scoped service 'REQUEST_SERVICE' from a singleton context.
```

If you encounter this error, review the lifetimes of the services involved.

See [Captive Dependencies](./captive-dependencies).

---

## Dispose Scoped Services

`runInScope()` disposes the scope when the scoped operation completes.
Disposable scoped and transient instances are tracked by the scope used to
resolve them. Singleton instances are tracked by the root scope.

`ServiceContainer.dispose()` disposes the root scope, then clears its singleton
instances and registrations. This API belongs to a directly owned
`ServiceContainer`; `XenoAppBuilder.build()` returns a runtime application, not
that container. The application API shown here does not expose
`ServiceContainer.dispose()`.

The container disposes tracked instances in reverse creation order. If disposal
of one instance fails, it logs the error and continues disposing the others.

---

## When to Use the Service Container Directly

Most application code should use `XenoAppBuilder.addServices()` for
registration.

Use the underlying container directly when you are building infrastructure,
modules, or integration code that receives an `IServiceContainer`.

For example:

```ts
async function configureModule(container: IServiceContainer) {
  container.addScoped('USER_REPOSITORY', (scope) => {
    return new UserRepository(scope.resolve('USER_DATA_SOURCE'))
  })
}
```

Application code generally does not need to instantiate or manage
`ServiceContainer` manually.

The normal application flow is:

```text
AppBuilder
    |
    v
addServices()
    |
    v
Service registration
    |
    v
build()
    |
    v
Service resolution
```

---

## Troubleshooting

### `Registration not found for token`

Check that the service has been registered using exactly the same token:

```ts
services.addScoped('USER_SERVICE', ...)
```

must be resolved with:

```ts
await application.runInScope(async (services) => {
  const service = services.USER_SERVICE
})
```

not a different token.

### `Scoped service requires an active scope`

The service was registered with `addScoped()` but was resolved from the root
container.

Use:

```ts
await application.runInScope(async (services) => {
  const service = services.USER_SERVICE
})
```

or use `application.getScopedServices()` when an active scope already exists.

### `already registered`

The same token has been registered more than once.

Remove the duplicate registration or give the implementations different tokens.

### `DI Circular Dependency Error`

Two or more services depend on each other through their registration factories.

Inspect the resolution path in the error and remove the dependency cycle.

### `DI Captive Dependency Error`

A singleton is trying to resolve a scoped service.

Review the resolution path and keep scoped work outside the singleton's
construction path. A transient dependency does not make a scoped dependency safe
when the transient is being resolved as part of singleton construction.

### Scoped service used outside its operation

Do not retain a scoped service beyond the `runInScope()` callback that created
it. The scope closes when that operation completes.

---

## Service Container Checklist

Before considering your dependency injection setup complete:

- [ ] Every service has a unique registration token.
- [ ] Every service uses the appropriate lifetime.
- [ ] Dependencies are resolved explicitly inside registration factories.
- [ ] Scoped services are only resolved from an active scope.
- [ ] Resolve root services through the runtime application's `getServices()`.
- [ ] Resolve scoped services through `getScopedServices()` only in an active
      scope, or use `runInScope()`.
- [ ] There are no circular dependencies.
- [ ] Singletons do not depend on scoped services.
- [ ] Use `runInScope()` so each application scope is closed when its work
      completes.
- [ ] Scoped services are used only during the scope that created them.

## Related Documentation

- [Dependency Injection](./overview)
- [Registration](./registration)
- [Singleton](./lifetimes/singleton)
- [Scoped](./lifetimes/scoped)
- [Transient](./lifetimes/transient)
- [Resolution](./resolution)
- [Dependency Graph](./dependency-graph)
- [Captive Dependencies](./captive-dependencies)
- [Context & Scopes](../context-scopes)
- [App Builder](../app-builder)

---

---

<a id="source-9"></a>

# Source: `docs/fundamentals/app-builder.md`

## Introduction

`XenoAppBuilder` is the composition root of a Xeno.JS application.

Use it to configure the application before startup:

- application services
- custom modules
- context
- middleware
- CQRS pipelines
- cache
- database
- authentication
- logging
- HTTP infrastructure
- concurrency services

The typical lifecycle is:

```text
AppBuilder
    ↓
configure
    ↓
build()
    ↓
configured runtime application
    ↓
application runtime
```

## Create an Application

Create an `XenoAppBuilder` and configure the capabilities your application
needs:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import type { XenoRegistry } from '@xeno-js/shared'

interface AppRegistry extends XenoRegistry {
  greeting: {
    greet(): string
  }
}

const app = new XenoAppBuilder<AppRegistry>()

app.addServices((services) => {
  services.addSingleton('greeting', () => ({
    greet: () => 'Hello Xeno!',
  }))
})

const application = await app.build()

const { greeting } = app.getServices()

console.log(greeting.greet())
```

There are three main steps:

1. Create the builder.
2. Configure the application.
3. Call `build()`.

Configuration methods return the same `XenoAppBuilder`, so they can also be
chained.

## Register Services

Use `addServices()` to register application-specific dependencies. Its setup
callback receives the service container and configuration service. The method's
optional second argument sets the module priority, which defaults to `99`.

Custom service tokens used with `addServices()` must also appear in the registry
type passed to `XenoAppBuilder`.

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import type { XenoRegistry } from '@xeno-js/shared'

interface UserRegistry extends XenoRegistry {
  USER_REPOSITORY: UserRepository
  USER_SERVICE: UserService
}

const app = new XenoAppBuilder<UserRegistry>()

app.addServices((services) => {
  services.addSingleton('USER_REPOSITORY', () => {
    return new UserRepository()
  })

  services.addScoped('USER_SERVICE', (container) => {
    return new UserService(container.resolve('USER_REPOSITORY'))
  })
})

await app.build()
```

In the example, `services` is the application's service container.

Choose a lifetime based on how long an instance should be reused:

- `addSingleton()` creates one instance shared by the container.
- `addScoped()` reuses an instance within a service scope. Resolve it through a
  scope; resolving a scoped registration from the builder's root container
  throws.
- `addTransient()` creates a new instance for each resolution.

Avoid injecting a scoped service into a singleton: the container rejects this
captive dependency. Disposable instances are disposed with their owning scope or
container.

For service lifetimes and dependency registration, see the dependency injection
documentation.

## Add a Custom Module

Use `addModule()` when a feature needs to register several related services or
perform its own configuration.

```ts
const app = new XenoAppBuilder<UserRegistry>()

app.addModule('UsersModule', async () => ({
  async configure(container) {
    container.addScoped('USER_SERVICE', () => {
      return new UserService()
    })
  },
}))

await app.build()
```

The module factory is executed during `build()`.

An optional configuration object can be passed as the third argument. A fourth
argument sets the module priority; the default is `50`, and modules run in
ascending priority order:

```ts
app.addModule(
  'UsersModule',
  async () => ({
    async configure(container, options) {
      container.addScoped('USER_SERVICE', () => {
        return new UserService(options)
      })
    },
  }),
  {
    enabled: true,
  },
  40,
)
```

Use `addRoutes()` to queue route definitions for the route module. Route
registration runs during bootstrap at priority `5`.

HTTP server integrations are supplied as plugins to `addHttpServer()`; their
specific setup is defined by the integration package.

## Enable Context

If no context has been configured, `build()` adds the default context module
when using the Node configuration service. The builder does not configure
context in its constructor.

For a Node application, the default is added automatically:

```ts
const app = new XenoAppBuilder()
```

This does not configure context in the constructor; the default Node context is
queued by `build()` if you have not called `addContext()`.

You can also configure it explicitly:

```ts
const app = new XenoAppBuilder()

app.addContext()
```

When using the Vite configuration service, configure an explicit context plugin
with `addContext()`; `build()` does not substitute the Node context in that
environment.

The constructor optionally accepts a service container as its first argument.
Its second argument selects the configuration service: it defaults to Node, and
also accepts `'vite'` or a custom `IConfigurationService` instance. Select Vite
when composing a browser application and provide its context plugin explicitly.

## Configure Middleware

Use `addMiddlewares()` to configure Xeno.JS middleware:

```ts
const app = new XenoAppBuilder()

app.addMiddlewares((options) => {
  options.cors = {
    enabled: true,
    withCredentials: true,
  }
})

await app.build()
```

Middleware configuration can also enable authentication-related behavior when
authentication has been configured.

### Automatic Dependencies

During configuration, `addMiddlewares()` queues the middleware module. During
`build()`, that module adds a logger if one has not been registered. The default
middleware settings enable rate limiting, which also configures the default
in-memory cache if no cache has been registered. Context is added by the build
process for Node configurations, as described above.

You do not need to call `addLogger()` or `addCache()` merely to use the default
middleware configuration. Add either explicitly when you need to customize its
provider or settings.

If you want to configure them explicitly, configure them before or after
`addMiddlewares()`:

```ts
const app = new XenoAppBuilder()

app.addLogger((options) => {
  options.console = true
})

app.addCache((options) => {
  options.inMemory = true
})

app.addMiddlewares((options) => {
  options.cors = {
    enabled: true,
  }
})
```

## Configure the CQRS Pipeline

Use `addPipeline()` to register one or more CQRS pipeline plugins:

```ts
import { XenoAppBuilder, withQueryCaching } from '@xeno-js/core'

const app = new XenoAppBuilder()

app.addPipeline(withQueryCaching())

await app.build()
```

Plugins configure pipeline behavior, including:

- performance
- authorization
- validation
- command and query behaviors

For example, `withQueryCaching()` enables query caching:

```ts
app.addPipeline(withQueryCaching())
```

### Dependencies configured by the CQRS module

The CQRS module adds a logger if none has been registered. It also adds the
default in-memory cache if query caching or command idempotency is enabled and
no cache has been registered. If context is not explicitly configured, the build
process adds the default Node context; Vite requires an explicit context plugin.

For example:

```ts
import { XenoAppBuilder, withQueryCaching } from '@xeno-js/core'

const app = new XenoAppBuilder()

app.addPipeline(withQueryCaching())
```

This enables query caching and its cache dependency; it does not turn on
unrelated pipeline behaviors.

If you need custom logging or cache configuration, configure those capabilities
explicitly:

```ts
const app = new XenoAppBuilder()

app.addLogger((options) => {
  options.console = true
})

app.addCache((options) => {
  options.inMemory = true
})

app.addPipeline(withQueryCaching())
```

## Configure the Cache

Use `addCache()` to configure application caching:

```ts
const app = new XenoAppBuilder()

app.addCache((options) => {
  options.inMemory = true
})
```

The default `XenoAppBuilder` cache configuration uses the in-memory
implementation:

```ts
const app = new XenoAppBuilder()

app.addCache()
```

The cache can also be configured with Redis.

See [Cache Overview](../cache/overview) and [Redis Cache](../cache/redis) for
the complete cache configuration.

## Configure the Database

`addDb()` accepts a database plugin. The plugin configures and queues the
provider-specific database module; `addDb()` does not accept a configuration
callback directly.

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import type { DbPlugin } from '@xeno-js/shared'

declare const databasePlugin: DbPlugin

const app = new XenoAppBuilder()
app.addDb(databasePlugin)
```

Here, `databasePlugin` is provided by the database integration selected by the
application. Its options and initialization behavior are provider-specific.

Configure the database once at the application composition root and inject the
required database services into application components.

## Configure Authentication

Use `addAuth()` to configure authentication:

```ts
import { XenoAppBuilder } from '@xeno-js/core'

const app = new XenoAppBuilder()

app.addAuth((options, config) => {
  // ApplicationAuthService is the application's implementation of the auth contract.
  options.authService = (scope) => new ApplicationAuthService(scope)
})
```

The callback configures `AuthModuleOptions`; `authService` is required.
Authentication integration details belong to the service implementation or its
provider package.

## Configure Logging

Use `addLogger()` to configure logging:

```ts
const app = new XenoAppBuilder()

app.addLogger((options) => {
  options.console = true
})
```

The setup callback configures the core logger settings, including:

- log level;
- console output;
- custom logger clients.

`addLogger()` also accepts one logger plugin or an array of logger plugins.
Provider-specific options belong to those plugins; the callback above only
configures the core logger settings.

See [Logging Overview](/docs/application/logging/overview) and
[Sentry Vue](/docs/application/logging/sentry-vue) for provider-specific
configuration.

## Configure HTTP Clients

Use `addHttpClient()` to register HTTP clients. It accepts an HTTP plugin or a
setup callback. The callback receives a `HttpConfig` whose `clients` entries
contain a name, an `IHttpClient` factory, and an optional resilience factory.

HTTP clients are infrastructure dependencies; registering an HTTP client does
not configure an inbound server.

See [HTTP Client and Resilience](/docs/http/client) for Axios and custom client
configuration, resolution, and resilience policies.

## Configure the HTTP Server

Use `addHttpServer()` with an HTTP server plugin from the selected transport
integration. Use `addRoutes()` to queue route definitions. Server and transport
options are integration-specific.

## Add the Concurrency Service

Use `addConcurrencyService()` when application code needs the Xeno.JS
concurrency service. It registers a singleton available through
`TOKENS.CONCURRENCY_SERVICE`:

```ts
import { XenoAppBuilder, TOKENS } from '@xeno-js/core'

const app = new XenoAppBuilder()

app.addConcurrencyService()

await app.build()

const { concurrencyService } = app.getServices()
```

This service is separate from the command retry behavior configured with a CQRS
pipeline plugin.

## Resolve Services

`XenoAppBuilder` exposes `resolve()` as a convenience for resolving a registered
dependency:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import type { XenoRegistry } from '@xeno-js/shared'

interface AppRegistry extends XenoRegistry {
  greeting: {
    greet(): string
  }
}

const app = new XenoAppBuilder<AppRegistry>()

app.addServices((services) => {
  services.addSingleton('greeting', () => ({
    greet: () => 'Hello Xeno!',
  }))
})

await app.build()

const { greeting } = app.getServices()

console.log(greeting.greet())
```

`build()` does not return the `ServiceContainer`; it returns the configured
runtime application instance. Use the builder's `resolve()` method for
root-container registrations:

```ts
const greeting = app.resolve('GREETING_SERVICE')
```

Root resolution cannot retrieve scoped services. Resolve those from an active
service scope instead. For application code, prefer dependency injection rather
than repeatedly reaching back into `XenoAppBuilder`.

## Build the Application

Call `build()` when configuration is complete:

```ts
const application = await app.build()
```

`build()` initializes queued modules according to their priority and returns the
configured runtime application instance (by default, a `XenoApplication`). It
does not expose the container as its return value.

The important behavior is:

1. Modules are ordered by priority.
2. Each module is initialized.
3. If all modules initialize successfully, the configured runtime application
   instance is returned.

If module initialization fails, `build()` throws a bootstrap error identifying
the module that failed.

For example:

```text
Bootstrap failed at [CacheModule]: ...
```

The module failure is logged and wrapped in a bootstrap error; the original
error is preserved as its cause.

## Build Only Once

After a successful build, `XenoAppBuilder` keeps the build promise and returns
the same runtime application instance on subsequent calls:

```ts
const first = await app.build()
const second = await app.build()

console.log(first === second)
// true
```

If a build fails, the builder clears the failed promise, disposes and recreates
its container, and a subsequent `build()` call can retry after that reset.
Application bootstrap should normally happen once during startup.

## Module Ordering

AppBuilder queues modules with priorities and initializes them in ascending
order. Integrations do not declare dependency relationships automatically;
choose a priority when a custom module must run before or after another module.
The source-defined priorities include:

| Priority | Module                   |
| -------: | ------------------------ |
|        0 | Context                  |
|        1 | Logger                   |
|        2 | Cache                    |
|        2 | Authentication           |
|        4 | Middleware               |
|        5 | CQRS and Routes          |
|       40 | Concurrency Service      |
|       50 | Custom modules (default) |
|       99 | Application services     |

The HTTP client module and custom modules use priority `50` unless another
priority is specified. Database and HTTP server plugins can queue
provider-specific modules, so their priorities depend on the integration.

`addServices()` uses priority `99` by default, so its registrations run after
the built-in modules listed above.

## A Typical Composition Root

A real application can combine the capabilities it needs:

```ts
import { XenoAppBuilder, withQueryCaching } from '@xeno-js/core'
import type { DbPlugin, XenoRegistry } from '@xeno-js/shared'

interface AppRegistry extends XenoRegistry {
  USER_REPOSITORY: UserRepository
  USER_SERVICE: UserService
}

declare const databasePlugin: DbPlugin<AppRegistry>

const app = new XenoAppBuilder<AppRegistry>()

app.addLogger((options) => {
  options.console = true
})

app.addCache()

app.addPipeline(withQueryCaching())

// databasePlugin is supplied by the selected database integration.
app.addDb(databasePlugin)

app.addServices((services) => {
  services.addScoped('USER_REPOSITORY', (container) => {
    return new UserRepository(container.resolve('DB_CONTEXT'))
  })

  services.addScoped('USER_SERVICE', (container) => {
    return new UserService(container.resolve('USER_REPOSITORY'))
  })
})

const application = await app.build()
```

The composition root now makes the application's infrastructure and services
explicit:

```text
AppBuilder
    │
    ├── Context
    ├── Logging
    ├── Cache
    ├── CQRS Pipeline
    ├── Database integration
    └── Application Services
             │
             ▼
configured runtime application
```

## XenoAppBuilder and ServiceContainer

The builder and its internal service container have different responsibilities:

| Component          | Responsibility                            |
| ------------------ | ----------------------------------------- |
| `XenoAppBuilder`   | Compose and bootstrap the application     |
| `ServiceContainer` | Register and resolve runtime dependencies |

Use `XenoAppBuilder` during application composition:

```ts
const app = new XenoAppBuilder<UserRegistry>()

app.addServices((services) => {
  services.addScoped('USER_SERVICE', () => {
    return new UserService()
  })
})

await app.build()
```

The builder retains the dependency-injection container internally; `build()`
returns the runtime application rather than that container. Resolve root
registrations through the builder:

```ts
import { XenoAppBuilder, TOKENS } from '@xeno-js/core'

const app = new XenoAppBuilder()
app.addLogger()
await app.build()

const { logger } = app.getServices()
```

`XenoAppBuilder` is the composition API and uses `ServiceContainer` internally
for dependency registration and resolution. A scoped service cannot be resolved
through the root builder; it needs an active scope. The returned runtime
instance is discussed separately in the XenoApplication documentation.

## XenoAppBuilder and the Registry

`XenoAppBuilder` is generic over the application's registry. Add custom service
tokens to the registry so that registration and resolution remain type-safe:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import type { XenoRegistry } from '@xeno-js/shared'

interface AppRegistry extends XenoRegistry {
  GREETING_SERVICE: {
    greet(): string
  }
}

const app = new XenoAppBuilder<AppRegistry>()
```

The registry associates dependency tokens with their expected types.

This makes calls such as:

```ts
import { TOKENS } from '@xeno-js/core'

const logger = app.resolve(TOKENS.LOGGER)
```

type-safe when `TOKENS.LOGGER` is part of the registry.

See [Xeno Registry](./xeno-registry) for the registry model.

## When to Use XenoAppBuilder

Use `XenoAppBuilder` when defining the application's composition root.

Typical responsibilities include:

- registering application services;
- enabling application modules;
- configuring infrastructure;
- configuring context;
- configuring middleware;
- configuring CQRS pipelines;
- configuring logging;
- configuring authentication;
- configuring databases;
- configuring caches;
- configuring HTTP infrastructure.

Do not use `XenoAppBuilder` as a general-purpose runtime service locator.

Once the application is built, application components should receive their
dependencies through dependency injection.

## The Key Idea

`XenoAppBuilder` answers one practical question:

> **How is this application composed?**

The answer should be visible in one place:

```ts
const app = new XenoAppBuilder()

app.addLogger(...)
app.addCache(...)
app.addPipeline(...)
app.addDb(...)
app.addServices(...)

await app.build()
```

Configure the application's capabilities and dependencies at the composition
root, then hand execution to the configured runtime.

## Related Documentation

- [Service Container](../dependency-injection/service-container)
- [Registration](../dependency-injection/registration)
- [Resolution](../dependency-injection/resolution)
- [Xeno Registry](./xeno-registry)
- [CQRS Pipelines](../application/pipelines/overview)
- [Cache Overview](../cache/overview)
- [Redis Cache](../cache/redis)
- [Observability Overview](../observability/overview)

---

---

<a id="source-10"></a>

# Source: `docs/fundamentals/context-scopes.md`

## Introduction

Xeno.JS uses **request context** and **service scopes** to make request-specific
state and scoped dependencies available during an application execution.

Use this page when you need to:

- access the current request context;
- access the current user identity;
- resolve a scoped service from application code;
- use `getScopedServices()` or `runInScope()`;
- create or use a service scope explicitly;
- understand when scoped services are available;
- access context from commands, queries, handlers, or transport code.

## Before you start

You should already have:

- an [`XenoAppBuilder`](./app-builder);
- services registered in the Xeno.JS container;
- an understanding of [Dependency Injection](./dependency-injection).

With the Node configuration service, `XenoAppBuilder` adds the default context
module during `build()` when none has been configured. With the Vite
configuration service, configure a context plugin explicitly.

After `build()`, use the returned `XenoApplication` to access services.
`getServices()` resolves root services, `getScopedServices()` resolves services
through the active scope, and `runInScope()` runs an operation in a new scope.

In the examples below, `application` refers to the value returned by
`await builder.build()`. The service proxy exposes registry keys as properties.
For example, the built-in logger key is `logger`, so it can be accessed by its
registry name or by the token value:

```ts
import { TOKENS } from '@xeno-js/shared'

const { logger } = application.getServices()
const loggerByToken = application.getServices().LOGGER
```

---

## Understand Context and Scopes

There are two related concepts:

### Request context

The request context stores information associated with the current application
execution.

You can use it to access:

- the current `RequestContext`;
- the current user identity;
- the current service scope;
- the current network context.

### Service scope

A service scope provides the resolution boundary for scoped services.

A scoped service gets its own instance within a scope. Resolving the same scoped
service again through that scope returns the instance associated with that
scope.

Scoped services must be resolved through an active scope.

For example:

```ts
container.addScoped('USER_REPOSITORY', (scope) => {
  return new UserRepository(scope.resolve('USER_DATA_SOURCE'))
})
```

The service can then be resolved from the active scope.

---

## Access the Current Request Context

If you need the current `RequestContext`, resolve the context accessor:

```ts
const { contextAccessor } = application.getServices()

const context = contextAccessor.getContext()
```

`getContext()` returns the current context when execution is running inside an
active request context.

Outside an active context, it returns `undefined`.

Use the context accessor when application code needs request-specific
information without receiving the context explicitly through every method call.

---

## Access the Current Identity

To access the current authenticated identity:

```ts
const { identityAccessor } = application.getServices()

const identity = identityAccessor.getIdentity()
```

The result is `undefined` when there is no active identity.

You can also update the identity associated with the current context:

```ts
const { requestContext } = application.getServices()

requestContext.updateIdentity(identity)
```

This updates the identity for the current context execution.

---

## Access User Context in a Handler

Handlers extending `BaseHandler` can access the current user context through
`_getCurrentContext()`:

```ts
export class CreateUserHandler extends BaseHandler<CreateUserCommand, User> {
  protected async executeAsync(
    request: CreateUserCommand,
  ): Promise<ResultType<User>> {
    const userContext = this._getCurrentContext()

    // Use the current user context here.

    return Result.ok({
      id: crypto.randomUUID(),
      email: request.email,
      name: request.name,
    })
  }
}
```

This is useful when the handler needs user-specific information as part of a use
case.

For general request context access, use the context/accessor APIs. For
handler-specific user context, use the `BaseHandler` API.

---

## Register a Scoped Service

Register services with scoped lifetime using `addScoped()`:

```ts
const app = new XenoAppBuilder().addServices((container) => {
  container.addScoped('USER_REPOSITORY', (scope) => {
    return new UserRepository(scope.resolve('USER_DATA_SOURCE'))
  })
})
```

A scoped service is associated with the active service scope.

This makes scoped lifetime useful for services that should share state during
one application execution without becoming global singletons.

---

## Resolve a Scoped Service

Use the application returned by `build()` to resolve scoped services.

### Resolve through the active scope

When a request or another execution has already established a scope, use
`getScopedServices()`:

```ts
const { USER_REPOSITORY: repository } = application.getScopedServices()
```

`getScopedServices()` throws if there is no active scope. It returns the
registered services through that scope; the property names are the keys in the
application registry.

### Start a new scoped operation

For work outside an existing request or execution, use `runInScope()`. The
callback receives the services resolved through the new scope:

```ts
const result = await application.runInScope(async (services) => {
  const repository = services.USER_REPOSITORY
  // Use the repository within this scope.
})
```

`runInScope()` creates the request context and service scope for the operation,
then disposes the scope when the operation completes. Use `getScopedServices()`
inside an existing active scope; use `runInScope()` when the operation needs its
own scope.

---

## Use a Scope in a Service

If a factory receives a service scope, resolve its dependencies from that scope:

```ts
container.addScoped('USER_SERVICE', (scope) => {
  return new UserService(
    scope.resolve('USER_REPOSITORY'),
    scope.resolve(TOKENS.LOGGER),
  )
})
```

This keeps dependency resolution inside the lifetime boundary of the service
being created.

You do not need to manually create a scope just to resolve dependencies in a
normal DI factory.

---

## Scoped Service Lifetime

A scoped service is cached within its service scope.

For example:

```ts
container.addScoped('REQUEST_STATE', () => {
  return new RequestState()
})
```

Within the same scope:

```ts
const first = scope.resolve('REQUEST_STATE')
const second = scope.resolve('REQUEST_STATE')
```

`first` and `second` refer to the same scoped instance.

A different scope gets a different instance.

When the scope is disposed, its scoped services are disposed as part of the
scope lifecycle when they support disposal.

---

## Singleton, Scoped, and Transient

Xeno.JS supports three common service lifetimes:

| Lifetime  | Instance behavior                          |
| --------- | ------------------------------------------ |
| Singleton | One instance for the application container |
| Scoped    | One instance per service scope             |
| Transient | A new instance for each resolution         |

Use scoped lifetime when a service should live for the duration of an
application execution and share its instance with other services resolved from
that scope.

For example:

```ts
container.addSingleton('CONFIGURATION_SERVICE', () => {
  return new ConfigurationService()
})

container.addScoped('UNIT_OF_WORK', (scope) => {
  return new UnitOfWork(scope.resolve(TOKENS.DB_CONTEXT))
})

container.addTransient('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('USER_REPOSITORY'))
})
```

See [Dependency Injection](./dependency-injection) for registration and lifetime
details.

---

## Create a Scope Manually

The `XenoApplication` returned by `build()` does not expose the container's
`createScope()` method. Use `runInScope()` to create and dispose a scope for
application work:

```ts
await application.runInScope(async (services) => {
  const service = services.USER_SERVICE
  // Use the service within this scope.
})
```

Use this for executions that are not already running inside an existing request
context.

---

## Use `runInScope()`

`runInScope()` creates the request context and service scope, passes the scoped
services proxy to the callback, and disposes the scope when the callback
completes:

```ts
await application.runInScope(async (services) => {
  const service = services.USER_SERVICE
  // Use the service within this scope.
})
```

Prefer the existing application/request scope when one is already active. Start
a new scope only when the execution boundary requires one.

---

## Run Code with a Request Context

Use `runInScope()` to establish a context and service scope for an asynchronous
execution:

```ts
const result = await application.runInScope(async (services) => {
  const service = services.USER_SERVICE
  // Run the operation that uses the service.
})
```

During the callback:

- the request context is available;
- the service scope is available;
- scoped services can be resolved;
- context accessors can retrieve the current context and scope.

When the callback completes, the scope is disposed.

Use `IRequestContext.runAsync()` when working directly with the request-context
contract; for application code, `runInScope()` provides the scope and services
together.

---

## Update the Current Identity

If authentication or another application boundary establishes a new identity
during execution, update the current context:

```ts
const { requestContext } = application.getServices()

requestContext.updateIdentity(identity)
```

Subsequent context and identity access within the same execution can observe the
updated identity.

---

## Access Network Context

Network information associated with the current request can be accessed through
the request context:

```ts
const { networkContextAccessor } = application.getServices()

const networkContext = networkContextAccessor.getNetworkContext()
```

The value is available only when the current execution has an associated network
context.

Use this API when application or infrastructure code needs request network
metadata rather than accessing transport-specific request objects directly.

---

## Context Outside an Active Execution

Context accessors are safe to call when no execution context is active.

For example:

```ts
const context = application.getServices().contextAccessor.getContext()

if (!context) {
  // No active context.
}
```

Similarly:

```ts
const scope = application.getServices().serviceScopeAccessor.getScope()

if (!scope) {
  // No active service scope.
}
```

`getScopedServices()` behaves differently: it requires an active scope and
throws when one is not available.

This distinction is important:

```ts
// Optional access
const scope = application.getServices().serviceScopeAccessor.getScope()

// Required scoped resolution
const service = application.getScopedServices().USER_SERVICE
```

Use the accessor when absence is a valid condition. Use `getScopedServices()`
when the operation requires a scoped service in an existing scope, or
`runInScope()` when it must create one.

---

## Context in Background Jobs

Background jobs do not necessarily start inside an existing request context.

If a job needs scoped services, establish an execution boundary before resolving
them.

For example:

```ts
await application.runInScope(async (services) => {
  const service = services.USER_SERVICE
  // Run the background operation using this scoped service.
})
```

This gives the job its own request context and service scope, and disposes the
scope when the execution finishes.

Do not rely on a request scope remaining available after the request execution
has completed.

---

## Context in Commands and Queries

Commands and queries normally execute inside an application scope established by
the surrounding execution.

Handlers can therefore use their injected dependencies normally:

```ts
export class FindUserHandler extends BaseHandler<FindUserQuery, User> {
  constructor(
    userContextFactory: UserContextFactory,
    private readonly userRepository: IUserRepository,
  ) {
    super(userContextFactory)
  }

  protected async executeAsync(
    request: FindUserQuery,
  ): Promise<ResultType<User>> {
    const userContext = this._getCurrentContext()

    return this.userRepository.findById(request.userId, userContext)
  }
}
```

For handlers, prefer constructor injection for declared dependencies.

Use `getScopedServices()` at integration or composition boundaries when an
execution has already established the scope.

See:

- [Creating Commands](../application/cqrs/command)
- [Creating Queries](../application/cqrs/query)
- [Creating and Registering Handlers](../application/cqrs/handler)

---

## When to Use `getScopedServices()`

`getScopedServices()` is useful when application code sits at the boundary
between a transport and the Xeno.JS application layer and needs a service from
the active scope.

For example:

```ts
const controller = application.getScopedServices().FIND_USER_CONTROLLER

// Use the controller within the active scope.
```

This method requires an active scope. For work outside an existing scope, use
`runInScope()` instead.

---

## Troubleshooting

### `Active service scope is required to execute Scoped service`

You called `getScopedServices()` without an active service scope.

Use `runInScope()` when the operation needs to create its own scope:

```ts
await application.runInScope(async (services) => {
  const service = services.USER_SERVICE
  // Run the operation using this scoped service.
})
```

If the code is already inside a request execution, verify that the
request/context middleware has established the Xeno.JS context before the
service is resolved.

### Scoped service cannot be resolved from the root container

Do not resolve a scoped service directly from the root container:

```ts
// Incorrect for a scoped service
application.getServices().USER_SERVICE
```

Use an active scope or create one for the operation:

```ts
application.getScopedServices().USER_SERVICE
```

or:

```ts
await application.runInScope(async (services) => {
  const service = services.USER_SERVICE
  // Run the operation using this scoped service.
})
```

### `getScope()` returns `undefined`

There is no active request/service context at the point where the accessor is
called.

Check the execution boundary and establish a context when the operation requires
scoped services.

### Service registration is missing

If the scope exists but the requested token is not registered, normal container
resolution fails.

Verify that the service is registered before the application is built:

```ts
const app = new XenoAppBuilder().addServices((container) => {
  container.addScoped('USER_SERVICE', (scope) => {
    return new UserService(scope.resolve('USER_REPOSITORY'))
  })
})

await app.build()
```

### Scope has already been disposed

A service scope is valid only during its lifetime.

Do not keep a scope or scoped service and use it after the scope has completed.

For asynchronous work, keep the work inside the execution that owns the scope.

---

## Best Practices

### Use dependency injection for handler dependencies

Prefer:

```ts
constructor(
  private readonly userRepository: IUserRepository,
) {}
```

over resolving application dependencies dynamically inside the handler.

### Use `getScopedServices()` or `runInScope()` at application boundaries

Use `getScopedServices()` when a transport, adapter, or other boundary is
already running in a scope. Use `runInScope()` when the operation needs a new
scope.

### Do not resolve scoped services from the root container

Always use an active scope for scoped dependencies.

### Do not manually create scopes unnecessarily

If Xeno.JS already established a scope for the current execution, reuse it.

### Let `runInScope()` manage the scope lifetime

`runInScope()` disposes the scope when the callback completes. Keep
scoped-service work inside that callback.

### Keep context access transport-independent

Application code should consume context and identity through Xeno.JS APIs rather
than depending directly on Fastify, Express, or another transport's request
object.

---

## Context and Scopes Checklist

Before shipping context-dependent code, verify:

- [ ] scoped services are registered with `addScoped()`;
- [ ] scoped services are resolved through an active scope;
- [ ] `getScopedServices()` is used only when an active scope exists;
- [ ] `runInScope()` is used to create a scope for work outside an existing
      execution;
- [ ] request context exists before resolving scoped services;
- [ ] context accessors handle `undefined` when no context is expected;
- [ ] scoped-service work completes before the `runInScope()` callback returns;
- [ ] scoped services are not used after their scope is disposed;
- [ ] handlers use dependency injection for their declared dependencies;
- [ ] transport-specific request objects do not leak into application services
      unnecessarily.

---

## Related Documentation

- [Fundamentals Overview](./overview)
- [Dependency Injection](./dependency-injection)
- [App Builder](./app-builder)
- [Creating Commands](../application/cqrs/command)
- [Creating Queries](../application/cqrs/query)
- [Creating and Registering Handlers](../application/cqrs/handler)
- [Application Overview](../application/overview)

---

---

<a id="source-11"></a>

# Source: `docs/fundamentals/dependency-injection.md`

## Introduction

Xeno.JS uses **explicit dependency injection** to compose application services.

The dependency graph is defined through TypeScript code rather than discovered
through decorators or runtime constructor metadata.

The basic model is:

```text
Application Registry
        │
        ▼
 Service Container
        │
        ▼
 Registration Factory
        │
        ▼
 Active Service Scope
        │
        ├── resolve dependency A
        ├── resolve dependency B
        └── create service
```

This makes dependency construction visible at the composition boundary.

---

## What dependency injection means in Xeno.JS

Dependency injection separates two responsibilities:

1. defining a service
2. deciding how that service receives its dependencies

For example, a service can remain an ordinary TypeScript class:

```ts
class UserService {
  constructor(private readonly repository: UserRepository) {}

  async getUser(id: string) {
    return this.repository.findById(id)
  }
}
```

Xeno.JS does not need to inspect this constructor at runtime.

Instead, the composition is declared explicitly:

```ts
container.addTransient('USER_SERVICE', (scope) => {
  const repository = scope.resolve('USER_REPOSITORY')

  return new UserService(repository)
})
```

The dependency is visible exactly where the service is composed.

---

## Explicit factories instead of hidden injection

The central difference is that Xeno.JS uses **factory-based dependency
registration**.

A registration has three parts:

```ts
container.addTransient('USER_SERVICE', (scope) => {
  const repository = scope.resolve('USER_REPOSITORY')

  return new UserService(repository)
})
```

The first argument is the token:

```ts
'USER_SERVICE'
```

The second argument is the factory:

```ts
;(scope) => {
  const repository = scope.resolve('USER_REPOSITORY')

  return new UserService(repository)
}
```

The factory receives the current `IServiceScope`.

Dependencies are then resolved explicitly from that scope.

This is intentionally different from a container that scans constructors and
automatically determines their dependencies.

---

## Injection tokens

Xeno.JS uses registry keys as dependency injection tokens.

The `ApplicationRegistry` defines the relationship between a token and its
service type.

Conceptually:

```ts
interface ApplicationRegistry {
  USER_REPOSITORY: UserRepository
  USER_SERVICE: UserService
}
```

The container can then use those keys for typed resolution:

```ts
const service = container.resolve('USER_SERVICE')
```

The TypeScript type of `service` is derived from the registry.

This means the token and the resolved service type remain connected at compile
time.

---

## Extending the registry

Applications can extend the Xeno registry with their own services.

For example:

```ts
type AppRegistry extends XenoRegistry {
    USER_REPOSITORY: UserRepository
    USER_SERVICE: UserService
  }

```

The resulting registry becomes the type contract for the application's Service
Container.

```ts
const container = new ServiceContainer<AppRegistry>()
```

Now registrations and resolutions are checked against the application registry.

```ts
container.addSingleton('USER_REPOSITORY', () => new UserRepository())
```

And:

```ts
const repository = container.resolve('USER_REPOSITORY')
```

The registry therefore acts as the compile-time contract for dependency
injection.

---

## Registering dependencies

Xeno.JS provides three registration methods:

```ts
container.addSingleton(...)
container.addTransient(...)
container.addScoped(...)
```

Each method receives:

1. a registry token
2. a factory function

### Singleton

```ts
container.addSingleton('USER_REPOSITORY', () => new UserRepository())
```

The factory is evaluated once and the resulting instance is reused.

### Transient

```ts
container.addTransient('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('USER_REPOSITORY'))
})
```

A new instance is produced on every resolution.

### Scoped

```ts
container.addScoped(
  'USER_SERVICE',
  (scope) => new UserService(scope.resolve('USER_REPOSITORY')),
)
```

A scoped instance is created once within each scope.

For more details, see [Service Lifetimes](./service-lifetimes).

---

## Resolving dependencies

The `resolve()` method is the explicit dependency resolution mechanism.

From a container:

```ts
const logger = container.resolve('logger')
```

From a scope:

```ts
const scope = container.createScope()

const service = scope.resolve('USER_SERVICE')
```

The difference is important.

The root container can resolve singleton and transient services.

A scoped service requires an active scope:

```ts
container.addScoped(
  'USER_SERVICE',
  (scope) => new UserService(scope.resolve('USER_REPOSITORY')),
)
```

This is invalid:

```ts
container.resolve('USER_SERVICE')
```

Xeno.JS throws an error indicating that the scoped service requires an active
scope.

The correct form is:

```ts
const scope = container.createScope()

const service = scope.resolve('USER_SERVICE')
```

---

## Dependency chains

Dependencies can resolve other dependencies.

For example:

```ts
container.addSingleton('USER_REPOSITORY', () => new UserRepository())

container.addTransient('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('USER_REPOSITORY'))
})

container.addTransient('USER_CONTROLLER', (scope) => {
  return new UserController(scope.resolve('USER_SERVICE'))
})
```

The resulting graph is:

```text
USER_CONTROLLER
       │
       ▼
USER_SERVICE
       │
       ▼
USER_REPOSITORY
```

The container resolves the graph recursively through the factories.

---

## Circular dependencies

Dependency graphs can contain accidental cycles.

For example:

```text
Service A
   │
   ▼
Service B
   │
   ▼
Service A
```

Xeno.JS tracks the current resolution path.

If the same token is encountered again during the active resolution chain, the
container throws a circular dependency error.

For example:

```ts
container.addTransient('SERVICE_A', (scope) => {
  scope.resolve('SERVICE_B')

  return new ServiceA()
})

container.addTransient('SERVICE_B', (scope) => {
  scope.resolve('SERVICE_A')

  return new ServiceB()
})
```

Resolving `SERVICE_A` produces a resolution path similar to:

```text
SERVICE_A -> SERVICE_B -> SERVICE_A
```

This is preferable to allowing the cycle to fail later through a less
descriptive runtime error.

---

## Captive dependencies

A **captive dependency** occurs when a service with a longer lifetime attempts
to hold a dependency with a shorter lifetime.

The important case explicitly detected by the current Xeno.JS container is:

```text
Singleton
    │
    ▼
Scoped
```

For example:

```ts
container.addScoped(
  'USER_SERVICE',
  (scope) => new UserService(scope.resolve('USER_REPOSITORY')),
)

container.addSingleton('GLOBAL_SERVICE', (scope) => {
  return new GlobalService(scope.resolve('USER_SERVICE'))
})
```

The container rejects this resolution.

The reason is that the singleton would capture a scoped instance beyond the
intended scope boundary.

Xeno.JS reports this as a captive dependency error and includes the resolution
path.

---

## Dependency lifetime rules

The lifetime of the dependency is part of the dependency graph.

A simplified model is:

```text
Singleton
   │
   ├── Singleton       ✓
   ├── Transient       ✓
   └── Scoped          ✗
```

For scoped and transient services, Xeno.JS does not apply the same
singleton-to-scoped rejection.

For example, a scoped service can resolve a transient service:

```ts
container.addTransient('VALIDATOR', () => new Validator())

container.addScoped('USER_SERVICE', (scope) => {
  const validator = scope.resolve('VALIDATOR')

  return new UserService(validator)
})
```

This is a valid composition.

The important rule is that a singleton must not capture a scoped dependency.

---

## Scopes and dependency injection

Scopes provide the lifetime boundary for scoped dependencies.

```ts
const scope = container.createScope()
```

A scoped service is cached within that scope:

```ts
const first = scope.resolve('USER_SERVICE')

const second = scope.resolve('USER_SERVICE')
```

Both resolutions return the same instance within the scope.

Another scope receives another instance:

```ts
const scopeA = container.createScope()
const scopeB = container.createScope()

const serviceA = scopeA.resolve('USER_SERVICE')

const serviceB = scopeB.resolve('USER_SERVICE')
```

The instances are isolated:

```text
Container
   │
   ├── Scope A
   │     └── USER_SERVICE #1
   │
   └── Scope B
         └── USER_SERVICE #2
```

Singleton instances remain shared between these scopes.

---

## Dependency injection and request execution

Scopes are particularly useful when application execution has a logical boundary
such as an HTTP request.

The general execution model can be represented as:

```text
Incoming Request
       │
       ▼
Create Scope
       │
       ▼
Resolve Application Services
       │
       ▼
Execute Command / Query
       │
       ▼
Dispose Scope
```

The Service Container itself does not require HTTP to exist.

The same composition model can be used by other application entry points where a
logical scope is useful.

---

## Resolution context

The Service Container tracks the current resolution stack and active lifetime
while it synchronously invokes registration factories. It uses this state to
detect circular dependencies and singleton-to-scoped captive dependencies, then
clears the resolution context when the resolution call completes.

`AsyncLocalStorage` belongs to the request-context implementation, not to the
Service Container's dependency-resolution mechanism.

---

## Dependency injection and disposal

Xeno.JS can track disposable services.

A service implementing the disposable contract can participate in container or
scope cleanup.

For example:

```ts
class DatabaseConnection {
  async dispose() {
    await this.close()
  }
}
```

Registered as a scoped service:

```ts
container.addScoped('DB_CONNECTION', () => new DatabaseConnection())
```

When the scope is disposed, tracked disposable resources are disposed.

```ts
await scope.dispose()
```

The same principle applies to container-level resources such as singleton
instances.

```ts
await container.dispose()
```

This makes service lifetime and resource ownership part of the same composition
model.

---

## Dependency injection through XenoAppBuilder

Most applications do not need to instantiate `ServiceContainer` manually.

`XenoAppBuilder` exposes dependency registration through `addServices()`. Add
custom service keys to the application's registry so that registrations and
resolutions remain typed:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import type { XenoRegistry } from '@xeno-js/shared'

interface AppRegistry extends XenoRegistry {
  USER_REPOSITORY: UserRepository
  USER_SERVICE: UserService
}

const builder = new XenoAppBuilder<AppRegistry>()

builder.addServices((container) => {
  container.addSingleton('USER_REPOSITORY', () => new UserRepository())

  container.addScoped('USER_SERVICE', (scope) => {
    return new UserService(scope.resolve('USER_REPOSITORY'))
  })
})
```

The registrations are queued as part of application composition.

They are applied during:

```ts
const application = await builder.build()
```

`build()` returns the runtime `XenoApplication`, not the `ServiceContainer`.
Resolve root registrations through `getServices()` using the registry keys:

```ts
const { USER_REPOSITORY } = application.getServices()
```

Resolve `USER_SERVICE` through an active scope. For work outside an existing
scope, use `runInScope()`:

```ts
await application.runInScope(async (services) => {
  const service = services.USER_SERVICE
  // Use the service within this scope.
})
```

`getScopedServices()` provides the same registry-key access when the current
execution already has an active scope.

---

## Custom modules and dependency injection

For larger applications, dependency registrations can be grouped into modules.

```ts
builder.addModule('UsersModule', async () => ({
  async configure(container) {
    container.addSingleton('USER_REPOSITORY', () => new UserRepository())

    container.addTransient('USER_SERVICE', (scope) => {
      return new UserService(scope.resolve('USER_REPOSITORY'))
    })
  },
}))
```

This keeps related registrations together.

A module can therefore become the composition boundary for a feature:

```text
UsersModule
   │
   ├── UserRepository
   ├── UserService
   └── other feature dependencies
```

---

## Explicit DI vs decorator-driven DI

Xeno.JS deliberately keeps dependency registration explicit.

There is no requirement to annotate classes such as:

```ts
@Injectable()
class UserService {}
```

or to rely on constructor reflection to discover dependencies.

Instead:

```ts
container.addTransient(
  'USER_SERVICE',
  (scope) => new UserService(scope.resolve('USER_REPOSITORY')),
)
```

The advantage of this model is that the dependency graph is visible in normal
TypeScript code.

It also means there is less runtime metadata involved in dependency discovery.

---

## What Xeno.JS does not infer

Because the model is explicit, Xeno.JS does not need to infer:

- which constructor parameters are dependencies
- which class should implement an interface
- which decorator metadata describes a service
- which lifetime should be assigned automatically

The application declares these decisions at the composition boundary.

For example:

```ts
container.addScoped('USER_SERVICE', (scope) => {
  const repository = scope.resolve('USER_REPOSITORY')

  return new UserService(repository)
})
```

The code itself describes the composition.

---

## Dependency injection architecture

The complete model is:

```text
                 Xeno Registry
                      │
                      ▼
               Typed DI Contract
                      │
                      ▼
              Service Container
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
      Singleton    Scoped     Transient
          │           │           │
          └───────────┼───────────┘
                      ▼
               Service Factory
                      │
                      ▼
                Service Scope
                      │
                      ▼
              Application Service
```

This makes dependency injection part of the application architecture rather than
a separate magic layer.

---

## Best practices

### Keep registration at the composition boundary

Prefer:

```ts
app.addServices((container) => {
  // registrations
})
```

over constructing infrastructure dependencies inside business logic.

### Resolve dependencies explicitly

Prefer:

```ts
const repository = scope.resolve('USER_REPOSITORY')
```

over hidden global access.

### Choose lifetimes deliberately

Use:

- `singleton` for shared application-wide resources
- `scoped` for state that belongs to one logical execution scope
- `transient` for short-lived services that should be recreated on resolution

### Avoid singleton-to-scoped dependencies

Do not create:

```text
Singleton → Scoped
```

because the singleton can outlive the scope.

### Keep the registry aligned with the application

The registry should describe the services that the application actually exposes
through dependency injection.

### Use modules as the application grows

When registrations become feature-specific, move them into modules rather than
allowing a single composition root to become unmanageable.

---

## Summary

Xeno.JS dependency injection is based on four ideas:

1. **Typed registry** — tokens and service types are connected through
   TypeScript.
2. **Explicit factories** — dependencies are resolved explicitly from the active
   scope.
3. **Explicit lifetimes** — singleton, scoped, and transient behavior is
   declared at registration time.
4. **Runtime validation** — the container detects missing registrations,
   circular dependencies, and singleton-to-scoped captive dependencies.

The central pattern is simple:

```ts
container.addTransient('USER_SERVICE', (scope) => {
  return new UserService(scope.resolve('USER_REPOSITORY'))
})
```

There is no hidden dependency discovery here.

The application defines the dependency graph explicitly, and Xeno.JS provides
the runtime that manages that graph.

---

---

<a id="source-12"></a>

# Source: `docs/fundamentals/overview.md`

## What You Will Learn

The Fundamentals section covers the following concepts:

- **XenoRegistry** — the type contract for built-in and application-specific
  service keys and types.
- **App Builder** — the entry point for composing and configuring a Xeno.JS
  application.
- **Service Container** — the runtime container responsible for resolving
  application dependencies.
- **Dependency Injection** — how services and dependencies are explicitly
  provided and resolved.
- **Service Lifetimes** — how services live and how their lifetime affects
  application behavior.
- **Modules** — how application functionality can be organized and composed into
  reusable units.
- **Context & Scopes** — how scoped execution and contextual state are managed
  during application execution.

---

## The Composition Model

A Xeno.JS application is assembled rather than implicitly discovered.

At startup, the application composition defines the services, modules,
infrastructure integrations, middleware, pipelines, and other capabilities that
are available to the application.

A simplified model looks like this:

```text
Application
    │
    ▼
XenoRegistry (type contract)
    │
    ▼
App Builder
    │
    ├── Services
    ├── Modules
    ├── Middleware
    ├── Pipelines
    └── Infrastructure
    │
    ▼
Service Container
    │
    ├── Singleton services
    ├── Scoped services
    └── Transient services
    │
    ▼
Application Execution
    │
    └── Context & Scopes
```

The exact configuration depends on the application, but the underlying model
remains explicit: the application defines what it needs and how those
dependencies are composed.

---

## Xeno Registry

The **XenoRegistry** is a TypeScript type that defines the service keys and
types available to typed dependency-injection APIs.

It combines built-in service keys with application-specific keys. The registry
does not store runtime registrations or service instances; the Service Container
manages those at runtime.

The App Builder and Service Container use this type contract so that
registrations and resolutions can be checked against the declared service types.

See [Xeno Registry](./xeno-registry) for more information.

---

## App Builder

The **App Builder** is the main composition entry point for a Xeno.JS
application.

It lets the application queue and configure modules, services, context,
middleware, pipelines, and infrastructure integrations. `build()` initializes
the queued modules and returns the configured runtime application; the builder
retains the Service Container internally.

Typical configuration can include:

- application context
- middleware
- pipelines
- database integrations
- cache
- authentication
- logging
- application services

This makes the composition root visible in one place instead of distributing
application setup across unrelated parts of the codebase.

See [App Builder](./app-builder) for more information.

---

## Service Container

During application composition, Xeno.JS uses a **Service Container** to manage
registrations and resolve dependencies.

The container invokes registration factories as dependencies are resolved and
respects each service's lifetime.

This creates a clear distinction between:

1. **registration** — describing what the application provides;
2. **composition** — configuring those registrations;
3. **resolution** — obtaining the required dependency at runtime.

See [Dependency Injection](./dependency-injection) for more information about
the Service Container and its registration and resolution APIs.

---

## Dependency Injection

Dependency Injection is one of the mechanisms used to keep application
dependencies explicit.

Instead of a component creating its own infrastructure dependencies, the
application's composition can register an implementation under a typed service
key and provide the resolved dependency to that component.

This separates the responsibility of **using a dependency** from the
responsibility of **composing the dependency**.

See [Dependency Injection](./dependency-injection) for more information.

---

## Service Lifetimes

Not every dependency should live for the same amount of time.

Xeno.JS supports explicit service lifetimes such as:

- **Singleton** — one instance shared by the application container.
- **Scoped** — one instance associated with an execution scope.
- **Transient** — a new instance created when the dependency is resolved.

Choosing a lifetime is therefore part of application composition, rather than an
implementation detail hidden from the developer.

---

## Modules

As an application grows, registering every capability directly in the main
composition root can become difficult to maintain.

**Modules** provide a way to group related application configuration and
registrations into cohesive units.

A module can encapsulate the composition required for a particular capability
while still participating in the application's overall composition model.

The [App Builder](./app-builder) page explains how to add and configure a custom
module.

---

## Context & Scopes

Application execution often requires contextual information and scoped
dependencies that should be available within a particular execution boundary.

Xeno.JS provides a request context and service scopes for application execution.
A scope defines where scoped dependencies are shared; context accessors expose
execution-specific information such as the current `RequestContext` and
identity.

For work outside an existing execution, `runInScope()` creates a context and
service scope for the operation and disposes the scope when the operation
completes.

See [Context and Scopes](./context-scopes) for more information.

---

## How the Pieces Fit Together

The Fundamentals concepts are closely related:

```text
                 XenoRegistry (type)
                       │
                       ▼
                  App Builder
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       Modules      Services    Pipelines
          │            │
          │            ▼
          │     Dependency Injection
          │            │
          └────────────┤
                       ▼
                Service Container
                       │
              ┌────────┴────────┐
              ▼                 ▼
          Lifetimes          Scopes
                                │
                                ▼
                         Application Context
```

Together, these mechanisms provide the foundation on which the other Xeno.JS
capabilities are composed.

The **Application** section builds on these fundamentals to explain how
application logic is executed through commands, queries, mediation, pipelines,
and behaviors.

---

## Where to Go Next

If you are new to Xeno.JS, the recommended order is:

1. [Xeno Registry](./xeno-registry)
2. [App Builder](./app-builder)
3. [Dependency Injection](./dependency-injection)
4. [Context and Scopes](./context-scopes)

These concepts form the foundation for understanding the rest of the framework.

---

---

<a id="source-13"></a>

# Source: `docs/fundamentals/xeno-application.md`

## Introduction

`XenoApplication` is the runtime API used after the application has been
composed. `XenoAppBuilder.build()` initializes the queued modules and returns
the configured application. Unless a custom application factory is configured,
the builder returns an `XenoApplication` backed by its service container.

The builder and the application have different responsibilities:

| Component         | Responsibility                                      |
| ----------------- | --------------------------------------------------- |
| `XenoAppBuilder`  | Configures and initializes the application modules. |
| `XenoApplication` | Exposes services and operations used at runtime.    |

Use the [App Builder](./app-builder) to configure the application. Use the
returned application to access registered services, run scoped work, dispatch
requests, or start the configured HTTP server.

## Application lifecycle

The runtime application is obtained after composition:

```text
XenoAppBuilder
      │
      ├── Configure services and integrations
      ├── Queue application modules
      ▼
    build()
      │
      ├── Initialize modules
      ▼
XenoApplication
      │
      ├── Access services
      ├── Run scoped operations
      └── Handle requests or listen
```

`build()` is asynchronous. Once it succeeds, use its return value as the
application runtime. It does not return the Service Container.

## Use the application returned by bootstrap

The `main.ts` example obtains the application from its bootstrap function, then
resolves services and start the server:

```ts
import 'dotenv/config'
import { bootstrap } from './bootstrap'

async function main() {
  const app = await bootstrap()
  const { logger, configurationService } = app.getServices()

  logger.info('Initializing app...')

  const port = configurationService.getNumber('PORT', 3000)

  try {
    await app.listen({
      port,
      host: 'localhost',
      onListening: (address) => {
        logger.info(
          `Xeno Node server is running on http://${address.host}:${address.port}`,
        )
      },
    })
  } catch {
    process.exit(1)
  }
}

main()
```

The bootstrap function's implementation is application-specific; this example
uses only the contract visible at the call site. `listen()` returns a promise,
so awaiting it lets the `try`/`catch` handle a rejected server startup.

## Access root services

Call `getServices()` to access services resolved from the application's root
container:

```ts
const { logger, configurationService } = app.getServices()
```

The returned object is a typed proxy: reading a property resolves the
corresponding registry key from the container. Root access is appropriate for
services that can be resolved outside a scope. A scoped service must instead be
resolved within an active service scope.

See [Dependency Injection](./dependency-injection) for registration and lifetime
details.

## Access services in an active scope

When a request or other execution has already established a scope, call
`getScopedServices()`:

```ts
const { logger } = app.getScopedServices()
logger.info('Operation in active scope')
```

The proxy resolves service keys through the active scope. If no scope is active,
`getScopedServices()` throws an error. For work that needs its own scope, use
`runInScope()` instead.

## Run a scoped operation

`runInScope()` creates a request context and service scope for an operation. Its
callback receives a proxy that resolves services through that scope:

```ts
await app.runInScope(async (services) => {
  services.logger.info('Run operation in scope')
})
```

The callback must return a promise. An optional second argument can provide
partial request context, such as identity, network, or tracing information.

The context execution owns the scope lifecycle. The scope is disposed when the
operation completes, including when the callback fails. See
[Context and Scopes](./context-scopes) for more about scoped service access.

## Handle an incoming request

`handle()` dispatches an operation for an incoming Xeno request:

Its generic contract accepts an `XenoIncomingRequest`, a token constrained to
`keyof TRegistry`, and optional input; it returns a
`Promise<ResponseDto<TOutput>>`.

The token identifies the controller operation in the application registry;
`input` is optional. The method adapts the incoming request and delegates to the
HTTP dispatcher, returning a `ResponseDto` in a promise.

Use this method only when the required HTTP adapter, dispatcher, and middleware
services are registered. For a network-facing server, configure the selected
HTTP integration and use `listen()`.

## Start the HTTP server

`listen()` resolves the registered HTTP server, creates its options, and awaits
the server's listen operation:

```ts
await app.listen({
  port: 3000,
  host: 'localhost',
})
```

The HTTP server must be configured during application composition. The
`NativeListenOptions` accepted by the method are passed through the server
options factory; transport-specific setup belongs to the selected integration.

## Choose the appropriate API

| Need                                         | API                   |
| -------------------------------------------- | --------------------- |
| Resolve a root service                       | `getServices()`       |
| Resolve a service in an existing scope       | `getScopedServices()` |
| Create a scope for an operation              | `runInScope()`        |
| Dispatch an incoming request to an operation | `handle()`            |
| Start the configured HTTP server             | `listen()`            |

Do not treat `XenoApplication` as the composition root. Configure dependencies
and integrations with `XenoAppBuilder`, then use the returned application for
runtime work.

## Summary

`XenoApplication` is the default runtime object returned by
`XenoAppBuilder.build()`. It provides distinct APIs for root service resolution,
scoped service resolution, scoped operations, request dispatch, and server
startup. Choose the API that matches the execution boundary so scoped services
remain inside an active scope.

---

---

<a id="source-14"></a>

# Source: `docs/fundamentals/xeno-registry.md`

## Introduction

`XenoRegistry` is a TypeScript type that defines the keys and service types
available to the application's typed dependency-injection APIs.

It combines the built-in `ApplicationRegistry` with application-specific service
keys. It does not store runtime registrations or service instances; the Service
Container handles those at runtime.

The type works with the [App Builder](./app-builder) and the
[Service Container](../dependency-injection/service-container) to check
registrations and resolutions against the application contract.

## Why a Registry?

As an application grows, it can have many different dependencies:

- application services
- repositories
- data sources
- infrastructure services
- clients
- configuration
- authentication services
- loggers
- caches

Those dependencies need stable identifiers so that they can be registered and
resolved consistently.

The application registry type makes those keys and their service types explicit.
It is a compile-time contract, not a runtime store of registrations.

The relationship is:

```text
XenoRegistry Type
    │
    ├── Built-in service keys and types
    └── Application-specific service keys and types
             │
             ▼
       Type-checks registration
             │
             ▼
       Service Container
             │
             └── Runtime registrations and instances
```

The registry type describes **which service keys and types are valid**.

The Service Container is responsible for **registering and resolving
dependencies at runtime**.

## Registry vs. Container

The Registry and the Service Container have different responsibilities.

| Component         | Responsibility                                            |
| ----------------- | --------------------------------------------------------- |
| XenoRegistry      | Defines the compile-time service keys and their types     |
| App Builder       | Composes the application and its registrations            |
| Service Container | Stores registrations and resolves dependencies at runtime |

This separation keeps application composition explicit.

`XenoRegistry` is a type, not a separate runtime object. It constrains the keys
and service types accepted by the container; the container itself stores the
registrations and resolves the instances.

## The Xeno Registry

`XenoRegistry` is a type alias over the built-in `ApplicationRegistry`, with an
optional type parameter for application-specific service keys and value types.

A project can define an application-specific type using this alias. It is not a
class and is not constructed at runtime.

For example:

```ts
import type { XenoRegistry } from '@xeno-js/shared'

interface InventoryRegistry extends XenoRegistry {
  STOCK_REPOSITORY: {
    getAvailableQuantity(sku: string): number
  }
  RESERVATION_SERVICE: {
    reserve(sku: string, quantity: number): void
  }
}
```

The first two generic parameters represent the database context and transaction
types used by the built-in registry. They default to `unknown`. The third
parameter adds application-specific keys and their value types.

The registry is a compile-time shape. Runtime registrations still belong to the
builder and its container. The third type parameter adds keys that do not
replace keys already defined by `ApplicationRegistry`.

An application may keep this type in a dedicated `registry.ts` module:

```text
src/
├── bootstrap.ts
└── registry.ts
```

The file name and project layout are conventions; the type can be declared
wherever the application keeps its composition contracts.

## Application-Specific Tokens

A registry becomes useful when an application introduces dependencies that are
specific to its own architecture.

For example, an inventory application can add a repository key and its contract
to the registry type:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import type { XenoRegistry } from '@xeno-js/shared'

interface StockRepository {
  getAvailableQuantity(sku: string): number
}

interface InventoryRegistry extends XenoRegistry {
  STOCK_REPOSITORY: StockRepository
}

const builder = new XenoAppBuilder<InventoryRegistry>()

const availableStock = new Map<string, number>([['SKU-100', 8]])

builder.addServices((services) => {
  services.addSingleton('STOCK_REPOSITORY', () => ({
    getAvailableQuantity: (sku) => availableStock.get(sku) ?? 0,
  }))
})

const application = await builder.build()
const stockRepository = application.getServices().STOCK_REPOSITORY
const availableQuantity = stockRepository.getAvailableQuantity('SKU-100')
```

The registry type associates `STOCK_REPOSITORY` with `StockRepository`. The
builder registers a runtime implementation under that key, and `getServices()`
exposes it using the same registry key.

Built-in services use the keys declared in `ApplicationRegistry`. When the
logger is registered, `TOKENS.LOGGER` has the value `logger`, so both forms
access the same service:

```ts
import { TOKENS } from '@xeno-js/shared'

const { logger } = application.getServices()
const loggerByToken = application.getServices().LOGGER
```

Conceptually:

```text
          Registry
             │
     ┌───────┴────────┐
     ▼                ▼
 STOCK_REPOSITORY   StockRepository
     key               type
     └───────┬────────┘
             ▼
       App Composition
             │
             ▼
      Concrete Services
```

This lets TypeScript check that registration and resolution use the declared
service contract while the composition layer supplies its implementation.

## The Registry as Part of the Composition Root

The registry belongs to the application's composition model.

A typical Xeno.JS application has a flow similar to:

```text
registry.ts (type)
     │
     ▼
bootstrap.ts (runtime registrations)
     │
     ▼
App Builder
     │
     ▼
Service Container
     │
     ▼
Application
```

The registry type defines the application's service keys and types.

The bootstrap process uses the type while composing the application and
registering runtime implementations.

This keeps dependency configuration close to the composition root instead of
requiring individual application components to know how their dependencies are
created.

## Registry and Dependency Injection

The Registry is closely related to Dependency Injection, but the two concepts
should not be confused.

**Dependency Injection** describes how dependencies are provided to a component.

The **Registry** provides a consistent way to identify those dependencies within
the application's composition model.

For example:

```text
                Registry
                   │
                   ▼
           "STOCK_REPOSITORY"
                   │
                   ▼
            Service Container
                   │
                   ▼
          StockRepository implementation
                   │
                   ▼
             Application Service
```

The application service does not need to know how the repository was
constructed.

The composition layer registers an implementation under the typed service key.

For more information about resolving dependencies, see
[Dependency Injection](./dependency-injection).

## Registry and Infrastructure

The registry type can describe an application-facing contract independently of
its infrastructure implementation.

For example, an application may define a repository contract while
infrastructure provides a concrete database implementation.

The composition can then connect the two:

```text
Application
    │
    ▼
Repository Contract
    │
    │ exposed through the application service key
    ▼
Registry Service Key
    │
    ▼
Infrastructure Registration
    │
    ▼
Database Repository
```

The composition layer can then register an implementation that satisfies the
declared contract. The registry type constrains its key and service shape; it
does not select or instantiate that implementation.

## Customizing the Registry

The registry type should describe application-specific service keys and their
types rather than runtime state.

A useful rule is:

> **Use the registry type to describe service keys and contracts, not to store
> runtime state.**

Runtime state belongs to the appropriate service, context, scope, or other
application mechanism.

Keeping the type focused makes the application contract easier to review.

## Registry in a Project

An application can keep its registry type in a `registry.ts` file and import it
from the composition module.

For example:

```text
my-app/
├── src/
│   ├── bootstrap.ts
│   └── registry.ts
```

This is an organizational convention, not a required Xeno.JS runtime file or
folder layout. The registry type can be declared wherever the application keeps
its composition contracts.

## Summary

`XenoRegistry` is the compile-time contract for the built-in and
application-specific service keys used by typed registration and resolution.

It provides:

- type-safe service keys and corresponding service types;
- an extension point for application-specific service contracts;
- compile-time integration with the App Builder and Service Container.

The resulting model is explicit:

```text
Registry
   │
   ▼
App Builder
   │
   ▼
Service Container
   │
   ▼
Resolved Dependencies
```

Once the Registry is understood, the next step is to understand how the
application composition is assembled through the [App Builder](./app-builder).

---
