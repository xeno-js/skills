# Application, CQRS and pipelines

Technical reference assembled from the website documentation. Each section
identifies its original Markdown source. Website frontmatter and the repeated
sponsorship footer are omitted; body content and fenced examples are retained.

---

## Contents

- [`docs/application/cqrs/command.md`](#source-1)
- [`docs/application/cqrs/handler.md`](#source-2)
- [`docs/application/cqrs/overview.md`](#source-3)
- [`docs/application/cqrs/query.md`](#source-4)
- [`docs/application/cqrs/zod-schema.md`](#source-5)
- [`docs/application/overview.md`](#source-6)
- [`docs/application/app-error.md`](#source-10)
- [`docs/application/result.md`](#source-11)
- [`docs/application/pipelines/enrichment-identity.md`](#source-7)
- [`docs/application/pipelines/exception.md`](#source-8)
- [`docs/application/pipelines/validation.md`](#source-9)

<a id="source-1"></a>

# Source: `docs/application/cqrs/command.md`

## Introduction

Use a Command to represent an application operation that the Xeno.JS CQRS
pipeline sends to its Handler. A Command can also declare cache keys or patterns
to invalidate when query caching is enabled.

A Command consists of:

1. a class extending `Command`;
2. an `intent` identifying the Command;
3. the data required by the operation;
4. a Handler registered with the same intent;
5. optionally, validation and other Command pipeline features.

## Before you start

You need:

- an `XenoAppBuilder`;
- the CQRS pipeline enabled with `addPipeline()`;
- a Command class;
- a Handler registered with the Command intent;
- execution within an active service scope (or use of `runInScope()` outside an
  existing request).

If you want validation, you also need a Zod schema.

## Enable CQRS

`addPipeline()` registers the CQRS infrastructure used to execute Commands and
Queries.

```ts
import { XenoAppBuilder } from '@xeno-js/core'

const app = new XenoAppBuilder().addPipeline()

await app.build()
```

Pass pipeline plugins to `addPipeline()` to enable optional behaviors such as
validation, idempotency, concurrency, or performance tracking. The CQRS module
also includes exception handling and logging behaviors.

---

## Create a Command

Extend the `Command` class and pass the Command intent to `super()`.

```ts
import { Command } from '@xeno-js/core'

export class CreateUserCommand extends Command<User> {
  constructor(
    public readonly email: string,
    public readonly name: string,
  ) {
    super('CreateUserCommand')
  }
}
```

The `Command` base class automatically sets:

```ts
command.type === REQUEST_TYPE.COMMAND
```

Define the data required by the application operation. The optional second
constructor argument can also declare cache keys or patterns to invalidate after
a successful Command, when query caching is enabled. These must match the stored
keys, including their context:

```ts
super('CreateUserCommand', {
  keys: ['users:list:public'],
})
```

This key matches a query whose logical key is `users:list`, whose `isUserScoped`
is `false`, and whose active identity has no tenant. For tenant or user-scoped
keys, use the configured key builder as described in
[Cache invalidation for Commands](#add-cache-invalidation-to-a-command).

The `intent` identifies the Command and is used to resolve its Handler and any
intent-specific pipeline configuration.

In this example:

```ts
super('CreateUserCommand')
```

means that the Handler must be registered with:

```text
CreateUserCommand
```

as its service token.

## Define the Command data

Command properties should contain the input required by the application
operation.

For example:

```ts
export class CreateUserCommand extends Command<User> {
  constructor(
    public readonly email: string,
    public readonly name: string,
  ) {
    super('CreateUserCommand')
  }
}
```

The resulting Command contains:

```ts
{
  intent: 'CreateUserCommand',
  type: 'COMMAND',
  email: 'john@example.com',
  name: 'John Doe',
}
```

You do not need to manually add `intent` or `type` to the class.

---

## Add Command validation

If the Command accepts external input, add a validation plugin to
`addPipeline()` to validate it before the Handler runs.

First enable validation:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withZodValidation } from '@xeno-js/zod'

const app = new XenoAppBuilder().addPipeline(
  withZodValidation((options) => {
    options.schemas = {}
  }),
)

await app.build()
```

Then register the schema using the Command intent as the key.

For example:

```ts
import { z } from 'zod'
import { XenoAppBuilder } from '@xeno-js/core'
import { withZodValidation } from '@xeno-js/zod'

const createUserSchema = z.object({
  intent: z.literal('CreateUserCommand'),
  type: z.literal('COMMAND'),
  email: z.string().email(),
  name: z.string().min(1),
})

const app = new XenoAppBuilder().addPipeline(
  withZodValidation((options) => {
    options.schemas = {
      CreateUserCommand: createUserSchema,
    }
  }),
)

await app.build()
```

The schema key must match:

```ts
super('CreateUserCommand')
```

The validation pipeline uses the Command's `intent` to select the schema.

### Keep schemas next to the Command

For larger applications, keep the Command and its schema together:

```text
src/
└── application/
    └── users/
        ├── create-user.command.ts
        └── create-user.schema.ts
```

For example:

```ts
// create-user.schema.ts

import { z } from 'zod'

export const createUserSchema = z.object({
  intent: z.literal('CreateUserCommand'),
  type: z.literal('COMMAND'),
  email: z.string().email(),
  name: z.string().min(1),
})
```

Then register it during application bootstrap:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withZodValidation } from '@xeno-js/zod'

import { createUserSchema } from './application/users/create-user.schema'

const app = new XenoAppBuilder().addPipeline(
  withZodValidation((options) => {
    options.schemas = {
      CreateUserCommand: createUserSchema,
    }
  }),
)

await app.build()
```

For more details about validation behavior and custom validation, see
[Validation](../pipelines/validation).

---

## Create the Command Handler

A Command must have a Handler registered with the same intent.

For the previous example:

```ts
super('CreateUserCommand')
```

the Handler must be registered as:

```text
CreateUserCommand
```

A Handler extends `BaseHandler` and implements `executeAsync()`.

```ts
import { BaseHandler } from '@xeno-js/core'
import { Result, type ResultType } from '@xeno-js/shared'

import type { CreateUserCommand, User } from './create-user.command'

export class CreateUserHandler extends BaseHandler<CreateUserCommand, User> {
  protected async executeAsync(
    request: CreateUserCommand,
  ): Promise<ResultType<User>> {
    const user = {
      id: crypto.randomUUID(),
      email: request.email,
      name: request.name,
    }

    return Result.ok(user)
  }
}
```

See [Creating Handlers](./handler) for the complete Handler configuration and
dependency injection patterns.

---

## Register the Handler

Register the Handler in the service container using the Command intent.
`BaseHandler` requires a user-context factory; the context module provides it
under `TOKENS.USER_CONTEXT_FACTORY`.

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { TOKENS } from '@xeno-js/shared'

import { CreateUserHandler } from './application/users/create-user.handler'

const app = new XenoAppBuilder()
  .addServices((container) => {
    container.addTransient('CreateUserCommand', (scope) => {
      return new CreateUserHandler(scope.resolve(TOKENS.USER_CONTEXT_FACTORY))
    })
  })
  .addPipeline()

await app.build()
```

The registration token:

```ts
'CreateUserCommand'
```

must match the Command intent:

```ts
super('CreateUserCommand')
```

If the names do not match, Xeno.JS cannot resolve the Handler when the Command
is executed.

### Register Handler dependencies

If your Handler requires application services, resolve them from the active
service scope.

For example:

```ts
export class CreateUserHandler extends BaseHandler<CreateUserCommand, User> {
  constructor(
    identityFactory: IFactory<void, UserContext>,
    private readonly userRepository: UserRepository,
  ) {
    super(identityFactory)
  }

  protected async executeAsync(
    request: CreateUserCommand,
  ): Promise<ResultType<User>> {
    const user = await this.userRepository.create({
      email: request.email,
      name: request.name,
    })

    return Result.ok(user)
  }
}
```

Register the dependencies explicitly:

```ts
import { TOKENS } from '@xeno-js/shared'

const app = new XenoAppBuilder()
  .addServices((container) => {
    container.addScoped(
      'USER_REPOSITORY',
      (scope) => new UserRepository(scope.resolve(TOKENS.DB_CONTEXT)),
    )

    container.addTransient('CreateUserCommand', (scope) => {
      return new CreateUserHandler(
        scope.resolve(TOKENS.USER_CONTEXT_FACTORY),
        scope.resolve('USER_REPOSITORY'),
      )
    })
  })
  .addPipeline()

await app.build()
```

Xeno.JS uses explicit dependency registration. The Handler's dependencies are
resolved from the service scope passed to the factory.

For more information, see
[Dependency Injection](../../fundamentals/dependency-injection).

---

## Execute a Command

Once the Command and Handler are registered, execute the Command through the
mediator inside an active service scope. For non-HTTP work, `runInScope()`
creates that scope and disposes it when the operation completes.

```ts
import { TOKENS } from '@xeno-js/shared'

const command = new CreateUserCommand('john@example.com', 'John Doe')

const application = await app.build()
const result = await application.runInScope(async (services) => {
  const mediator = services[TOKENS.MEDIATOR]
  return mediator.send(command, new AbortController().signal)
})
```

The mediator executes the Command pipeline and eventually resolves the Handler
using:

```ts
command.intent
```

For this Command:

```ts
command.intent === 'CreateUserCommand'
```

so the mediator resolves the Handler registered with:

```ts
'CreateUserCommand'
```

The returned value is a `ResultType<TResponse>`.

Handle the result according to your application's error-handling conventions.

---

## Complete example

A simple Command implementation can be organized as follows:

```text
src/
├── application/
│   └── users/
│       ├── create-user.command.ts
│       ├── create-user.handler.ts
│       └── create-user.schema.ts
└── bootstrap.ts
```

### Command

```ts
// application/users/create-user.command.ts

import { Command } from '@xeno-js/core'

export interface User {
  id: string
  email: string
  name: string
}

export class CreateUserCommand extends Command<User> {
  constructor(
    public readonly email: string,
    public readonly name: string,
  ) {
    super('CreateUserCommand')
  }
}
```

### Schema

```ts
// application/users/create-user.schema.ts

import { z } from 'zod'

export const createUserSchema = z.object({
  intent: z.literal('CreateUserCommand'),
  type: z.literal('COMMAND'),
  email: z.string().email(),
  name: z.string().min(1),
})
```

### Handler

```ts
// application/users/create-user.handler.ts

import { BaseHandler } from '@xeno-js/core'
import { Result, type ResultType } from '@xeno-js/shared'

import type { CreateUserCommand, User } from './create-user.command'

export class CreateUserHandler extends BaseHandler<CreateUserCommand, User> {
  protected async executeAsync(
    request: CreateUserCommand,
  ): Promise<ResultType<User>> {
    const user: User = {
      id: crypto.randomUUID(),
      email: request.email,
      name: request.name,
    }

    return Result.ok(user)
  }
}
```

### Application bootstrap

```ts
// bootstrap.ts

import { XenoAppBuilder } from '@xeno-js/core'
import { TOKENS } from '@xeno-js/shared'
import { withZodValidation } from '@xeno-js/zod'

import { createUserSchema } from './application/users/create-user.schema'
import { CreateUserHandler } from './application/users/create-user.handler'

const builder = new XenoAppBuilder()
  .addServices((container) => {
    container.addTransient('CreateUserCommand', (scope) => {
      return new CreateUserHandler(scope.resolve(TOKENS.USER_CONTEXT_FACTORY))
    })
  })
  .addPipeline(
    withZodValidation((options) => {
      options.schemas = {
        CreateUserCommand: createUserSchema,
      }
    }),
  )

const application = await builder.build()
await application.runInScope(async (services) => {
  const mediator = services[TOKENS.MEDIATOR]
  return mediator.send(
    new CreateUserCommand('john@example.com', 'John Doe'),
    new AbortController().signal,
  )
})
```

The important relationship is:

```text
CreateUserCommand
       │
       │ intent = "CreateUserCommand"
       ▼
"CreateUserCommand" service registration
       │
       ▼
CreateUserHandler
```

The same intent connects the Command to its Handler and to its validation
schema.

---

## Add cache invalidation to a Command

When a Command changes data used by a cached Query, declare the affected stored
keys in `cacheInvalidation`. Enable the behavior with `withQueryCaching()`:

```ts
import { XenoAppBuilder, withQueryCaching } from '@xeno-js/core'

const application = await new XenoAppBuilder()
  .addPipeline(withQueryCaching())
  .build()
```

Add your Handler registrations before building the application. If no cache is
registered, Xeno.JS configures the default in-memory cache. `addCache()` alone
does not enable command invalidation.

The optional `ICommand.cacheInvalidation` property accepts `keys?: string[]` and
`patterns?: string[]`. When extending `Command`, pass that object as the second
argument to `super()`:

```ts
import { Command } from '@xeno-js/core'
import type { ICacheKeyBuilder } from '@xeno-js/core'

interface User {
  id: string
  name: string
}

export class UpdateUserCommand extends Command<User> {
  constructor(
    public readonly userId: string,
    public readonly name: string,
    keyBuilder: ICacheKeyBuilder,
  ) {
    super('UpdateUserCommand', {
      keys: [keyBuilder.buildContextualKey(`user:${userId}`)],
    })
  }
}
```

For user ID `42`, this key matches a Query with `cacheKey: 'user:42'` and
`isUserScoped: false`, provided the Query and Command use the same tenant
context. Resolve the key builder through `TOKENS.CACHE_KEY_BUILDER` while the
appropriate identity is active, then pass it to the Command constructor. Account
for identity enrichment that may change the context later. For a user-scoped
Query, use `buildUserScopedKey()` with the matching identity instead.

The pipeline passes declared keys and patterns directly to the cache provider.
It does not add tenant or user context automatically. A key for the current
identity does not invalidate other users' entries; list and user-scoped caches
may need additional, carefully scoped keys or patterns.

The pipeline attempts invalidation after the downstream Command execution
returns a successful `Result`. Failed Commands and thrown errors do not trigger
it. If a cache removal fails, Xeno.JS logs a warning, stops the remaining
removals, and returns the successful Command result. Invalidation is not atomic
with the write, so plan for stale data and concurrent reads.

See
[Command Cache Invalidation Pipeline](/docs/application/pipelines/cache-invalidation)
for an example with a Query, scoped keys, pattern behavior, and troubleshooting.

---

## Add idempotency to a Command

Idempotency is a Command-specific pipeline feature.

Add the `withIdempotency` plugin to the Command pipeline:

```ts
import { XenoAppBuilder, withIdempotency } from '@xeno-js/core'

const app = new XenoAppBuilder().addPipeline(
  withIdempotency((options) => {
    options.lockTtlSeconds = 300
    options.processedTtlSeconds = 86400
  }),
)

await app.build()
```

The idempotency mechanism uses the active network context's `requestId` to
identify an operation; it does not read an ID from the Command object. The
options also support per-intent settings through `intentRegistry`.

See [Idempotency](../pipelines/idempotency) for the complete configuration and
behavior.

---

## Add concurrency handling to a Command

Concurrency retry handling is also Command-specific.

```ts
import { XenoAppBuilder, withConcurrency } from '@xeno-js/core'

const app = new XenoAppBuilder().addPipeline(
  withConcurrency((options) => {
    options.maxRetries = 3
    options.delayConfig = {
      baseDelayMs: 100,
      maxJitterMs: 50,
    }
  }),
)

await app.build()
```

See [Concurrency](../pipelines/concurrency) for the complete configuration and
retry behavior.

---

## Add other pipeline features

Commands can also use the common CQRS pipeline features:

- [Validation](../pipelines/validation)
- [Exception handling](../pipelines/exception)
- [Logging](../pipelines/logging)
- [Performance tracking](../pipelines/performance)
- [Idempotency](../pipelines/idempotency)
- [Concurrency](../pipelines/concurrency)
- [Command Cache Invalidation](/docs/application/pipelines/cache-invalidation)

Add the corresponding pipeline plugins to `addPipeline()` where required.
`addPipeline()` configures the CQRS module and does not take a configuration
callback.

---

## Troubleshooting

### `HANDLER_NOT_FOUND`

If executing a Command produces a `HANDLER_NOT_FOUND` error, check that:

1. the Command has an intent;
2. the Handler is registered in the service container;
3. the registration token exactly matches the Command intent;
4. the Handler exposes a `handle()` method through `BaseHandler`;
5. `addPipeline()` has been called;
6. the application has been built with `await app.build()`.

For example:

```ts
super('CreateUserCommand')
```

must correspond to:

```ts
import { TOKENS } from '@xeno-js/shared'

container.addTransient(
  'CreateUserCommand',
  (scope) => new CreateUserHandler(scope.resolve(TOKENS.USER_CONTEXT_FACTORY)),
)
```

### Validation is not running

Check that:

1. `addPipeline(withZodValidation(...))` is configured;
2. the Command runs inside an active service scope;
3. the schema is registered under the exact Command intent;
4. the Command is executed through the mediator;
5. the schema validates the Command request shape.

For example:

```ts
withZodValidation((options) => {
  options.schemas = {
    CreateUserCommand: createUserSchema,
  }
})
```

must match:

```ts
super('CreateUserCommand')
```

### Cached Query results remain stale

Check that `withQueryCaching()` is enabled, the Command runs through the
mediator and returns a successful `Result`, and the declared keys match the
actual stored keys. Check cache warnings and ensure the declared patterns are
supported by the provider. See
[Command Cache Invalidation](/docs/application/pipelines/cache-invalidation) for
tenant and user key formats.

### The Handler is not receiving the expected data

The Command class defines the data passed to the Handler:

```ts
export class CreateUserCommand extends Command<User> {
  constructor(
    public readonly email: string,
    public readonly name: string,
  ) {
    super('CreateUserCommand')
  }
}
```

The Handler receives the same Command instance:

```ts
protected async executeAsync(
  request: CreateUserCommand,
) {
  request.email
  request.name
}
```

If validation is enabled, remember that the schema validates the request object
before the Handler executes.

---

## Related documentation

- [CQRS Overview](./overview)
- [Creating Queries](./query)
- [Creating Handlers](./handler)
- [Validation](../pipelines/validation)
- [Exception Handling](../pipelines/exception)
- [Logging](../pipelines/logging)
- [Performance Tracking](../pipelines/performance)
- [Idempotency](../pipelines/idempotency)
- [Concurrency](../pipelines/concurrency)
- [Command Cache Invalidation](/docs/application/pipelines/cache-invalidation)
- [Dependency Injection](../../fundamentals/dependency-injection)

---

---

<a id="source-2"></a>

# Source: `docs/application/cqrs/handler.md`

## Introduction

A handler contains the application logic that executes a Command or Query.

In Xeno.JS, create a handler by extending `BaseHandler`, implement
`executeAsync()`, and register the handler in the application service container.

The handler is associated with a Command or Query through its `intent`.

## Before you start

You need:

- a Xeno.JS `XenoAppBuilder`;
- a Command or Query;
- a handler extending `BaseHandler`;
- the CQRS pipeline enabled with `addPipeline()`;
- a handler registration whose token matches the request `intent`.

For example:

```ts
const app = new XenoAppBuilder().addPipeline()
```

`XenoAppBuilder` configures the application context during build, so
`BaseHandler` can access the current user context when needed. `BaseHandler`
requires the user-context factory in its constructor; the context module
provides it under `TOKENS.USER_CONTEXT_FACTORY`.

## Create a Handler

Extend `BaseHandler` with:

1. the request type;
2. the response type.

Then implement `executeAsync()`.

For example, for a `CreateUserCommand`:

```ts
import { Result, type ResultType } from '@xeno-js/shared'
import { BaseHandler } from '@xeno-js/core'

import type { CreateUserCommand } from './create-user.command'

interface User {
  id: string
  email: string
  name: string
}

export class CreateUserHandler extends BaseHandler<CreateUserCommand, User> {
  protected async executeAsync(
    request: CreateUserCommand,
  ): Promise<ResultType<User>> {
    const user: User = {
      id: crypto.randomUUID(),
      email: request.email,
      name: request.name,
    }

    return Result.ok(user)
  }
}
```

`executeAsync()` receives the Command or Query that was sent through the
mediator.

Return a `ResultType<TResponse>` using the Xeno.JS `Result` API.

## Implement `executeAsync()`

`BaseHandler` requires subclasses to implement:

```ts
protected abstract executeAsync(
  request: TRequest,
  signal?: AbortSignal,
): Promise<ResultType<TResponse>>
```

The request is the Command or Query being processed.

The response type is defined by the generic parameters passed to `BaseHandler`.

For example:

```ts
export class FindUserHandler extends BaseHandler<FindUserQuery, User> {
  protected async executeAsync(
    request: FindUserQuery,
  ): Promise<ResultType<User>> {
    // Execute the application use case.
  }
}
```

### Cancellation

The public `handle()` method receives an `AbortSignal` and checks whether the
operation has already been aborted before executing the handler.

The current `BaseHandler` implementation passes that signal from `handle()` into
`executeAsync()`.

So, the `signal` parameter declared by `executeAsync()` is automatically
populated by `BaseHandler`.

## Inject Dependencies

Handlers can receive application dependencies through their constructor.

For example:

```ts
import type { ResultType } from '@xeno-js/shared'
import type { IFactory, UserContext } from '@xeno-js/shared'
import { BaseHandler } from '@xeno-js/core'

import type { CreateUserCommand } from './create-user.command'

export interface UserRepository {
  create(email: string, name: string): Promise<User>
}

interface User {
  id: string
  email: string
  name: string
}

export class CreateUserHandler extends BaseHandler<CreateUserCommand, User> {
  constructor(
    private readonly userRepository: UserRepository,
    userContextFactory: IFactory<void, UserContext>,
  ) {
    super(userContextFactory)
  }

  protected async executeAsync(
    request: CreateUserCommand,
  ): Promise<ResultType<User>> {
    const user = await this.userRepository.create(request.email, request.name)

    return Result.ok(user)
  }
}
```

The dependencies are supplied when the handler is registered with the service
container.

This keeps dependency construction outside the handler itself.

## Register a Handler

Register handlers through `XenoAppBuilder.addServices()`.

The registration token must match the request `intent`.

For example, if the command uses:

```ts
super('CreateUserCommand')
```

register the handler as:

```ts
import { TOKENS } from '@xeno-js/shared'

container.addTransient('CreateUserCommand', (scope) => {
  return new CreateUserHandler(
    scope.resolve('USER_REPOSITORY'),
    scope.resolve(TOKENS.USER_CONTEXT_FACTORY),
  )
})
```

The important relationship is:

```text
CreateUserCommand
      │
      │ intent = "CreateUserCommand"
      ▼
"CreateUserCommand"
      │
      ▼
CreateUserHandler
```

The intent and registration token must match exactly.

## Register a Handler with `XenoAppBuilder`

A complete application configuration can look like this:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { TOKENS } from '@xeno-js/shared'

const app = new XenoAppBuilder()
  .addServices((container) => {
    container.addTransient('CreateUserCommand', (scope) => {
      return new CreateUserHandler(
        scope.resolve('USER_REPOSITORY'),
        scope.resolve(TOKENS.USER_CONTEXT_FACTORY),
      )
    })
  })
  .addPipeline()

await app.build()
```

`addServices()` gives you access to the service container so you can register
the handler and its dependencies.

`addPipeline()` enables the CQRS execution pipeline required to process Commands
and Queries through the mediator.

## Register Handler Dependencies

Register dependencies before resolving them from the handler factory.

For example:

```ts
import { TOKENS } from '@xeno-js/shared'

const app = new XenoAppBuilder()
  .addServices((container) => {
    container.addScoped('USER_REPOSITORY', (scope) => {
      return new UserRepository(scope.resolve('USER_DATA_SOURCE'))
    })

    container.addTransient('CreateUserCommand', (scope) => {
      return new CreateUserHandler(
        scope.resolve('USER_REPOSITORY'),
        scope.resolve(TOKENS.USER_CONTEXT_FACTORY),
      )
    })
  })
  .addPipeline()

await app.build()
```

Choose the handler lifetime according to its dependencies and application
requirements.

A handler that has no state of its own is commonly suitable for transient
registration, while dependencies with request or scope-specific state should be
resolved through the active service scope.

## Access the Current User Context

`BaseHandler` provides the protected `_getCurrentContext()` method.

Use it when the handler needs the current `UserContext`.

For example:

```ts
export class GetProfileHandler extends BaseHandler<GetProfileQuery, Profile> {
  protected async executeAsync(
    request: GetProfileQuery,
  ): Promise<ResultType<Profile>> {
    const context = this._getCurrentContext()

    const userId = context.userId
    const tenantId = context.tenantId

    // Load data for the current user and tenant.

    return Result.ok({
      userId,
      tenantId,
    })
  }
}
```

`BaseHandler` requires the user-context factory in its constructor. The
application context registers it under `TOKENS.USER_CONTEXT_FACTORY`.

When registering the handler manually, resolve it from the service scope:

```ts
import { TOKENS } from '@xeno-js/shared'

scope.resolve(TOKENS.USER_CONTEXT_FACTORY)
```

This dependency is registered by Xeno.JS's application context setup.

## Connect a Handler to a Command

A Command defines its intent when it extends `Command`.

For example:

```ts
import { Command } from '@xeno-js/core'

export class CreateUserCommand extends Command<User> {
  constructor(
    public readonly email: string,
    public readonly name: string,
  ) {
    super('CreateUserCommand')
  }
}
```

Register the corresponding handler with the same intent:

```ts
import { TOKENS } from '@xeno-js/shared'

container.addTransient('CreateUserCommand', (scope) => {
  return new CreateUserHandler(scope.resolve(TOKENS.USER_CONTEXT_FACTORY))
})
```

The mediator can then associate the request with the registered handler.

## Connect a Handler to a Query

Queries use the same mechanism.

For example:

```ts
import { Query } from '@xeno-js/core'

export class FindUserQuery extends Query<User> {
  constructor(public readonly userId: string) {
    super('FindUserQuery', {
      cacheKey: `users:${userId}`,
      ttl: 60,
      bypassCache: false,
      consistentRead: false,
      isUserScoped: false,
    })
  }
}
```

Register the handler using the same intent:

```ts
import { TOKENS } from '@xeno-js/shared'

container.addTransient('FindUserQuery', (scope) => {
  return new FindUserHandler(
    scope.resolve('USER_REPOSITORY'),
    scope.resolve(TOKENS.USER_CONTEXT_FACTORY),
  )
})
```

For query-specific caching configuration, see [Caching](../pipelines/caching).

## Execute the Handler Through the Mediator

Handlers are normally not called directly. The mediator resolves the pipeline
and handler from the active service scope, so execute requests within an
application scope.

Send a Command through the mediator while an application service scope is
active:

```ts
const result = await mediator.send(
  new CreateUserCommand('john@example.com', 'John Doe'),
  new AbortController().signal,
)
```

Or execute a Query while an application service scope is active:

```ts
const result = await mediator.query(
  new FindUserQuery('user-123'),
  new AbortController().signal,
)
```

The mediator uses the request `intent` to resolve the corresponding handler from
the active service scope.

This means that the following three pieces must agree:

```text
Command / Query
      │
      └── intent
             │
             ▼
       Handler registration
             │
             └── Handler
```

## Complete Command Handler Example

A minimal application can be organized like this:

```text
src/
├── application/
│   └── users/
│       ├── create-user.command.ts
│       └── create-user.handler.ts
└── app.ts
```

### `create-user.command.ts`

```ts
import { Command } from '@xeno-js/core'

export interface User {
  id: string
  email: string
  name: string
}

export class CreateUserCommand extends Command<User> {
  constructor(
    public readonly email: string,
    public readonly name: string,
  ) {
    super('CreateUserCommand')
  }
}
```

### `create-user.handler.ts`

```ts
import { Result, type ResultType } from '@xeno-js/shared'
import { BaseHandler } from '@xeno-js/core'

import type { CreateUserCommand, User } from './create-user.command'

export class CreateUserHandler extends BaseHandler<CreateUserCommand, User> {
  protected async executeAsync(
    request: CreateUserCommand,
  ): Promise<ResultType<User>> {
    const user: User = {
      id: crypto.randomUUID(),
      email: request.email,
      name: request.name,
    }

    return Result.ok(user)
  }
}
```

### `app.ts`

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { TOKENS } from '@xeno-js/shared'

import { CreateUserHandler } from './application/users/create-user.handler'

const app = new XenoAppBuilder()
  .addServices((container) => {
    container.addTransient('CreateUserCommand', (scope) => {
      return new CreateUserHandler(scope.resolve(TOKENS.USER_CONTEXT_FACTORY))
    })
  })
  .addPipeline()

await app.build()
```

Once the application is built, the handler is available to the CQRS execution
flow.

## Create a Handler with a Repository

A more typical application handler delegates persistence to a repository:

```ts
import type { IFactory, UserContext } from '@xeno-js/shared'

export class CreateUserHandler extends BaseHandler<CreateUserCommand, User> {
  constructor(
    private readonly userRepository: UserRepository,
    userContextFactory: IFactory<void, UserContext>,
  ) {
    super(userContextFactory)
  }

  protected async executeAsync(
    request: CreateUserCommand,
  ): Promise<ResultType<User>> {
    const user = await this.userRepository.create({
      email: request.email,
      name: request.name,
    })

    return Result.ok(user)
  }
}
```

Register both services:

```ts
import { TOKENS } from '@xeno-js/shared'

const app = new XenoAppBuilder()
  .addServices((container) => {
    container.addScoped('USER_REPOSITORY', (scope) => {
      return new UserRepository(scope.resolve('USER_DATA_SOURCE'))
    })

    container.addTransient('CreateUserCommand', (scope) => {
      return new CreateUserHandler(
        scope.resolve('USER_REPOSITORY'),
        scope.resolve(TOKENS.USER_CONTEXT_FACTORY),
      )
    })
  })
  .addPipeline()

await app.build()
```

This keeps the handler focused on the application use case while the repository
remains responsible for persistence.

## Handle Errors

A handler returns a `ResultType<TResponse>`.

For successful execution:

```ts
return Result.ok(user)
```

For an application failure, return a failed `Result` using the appropriate
Xeno.JS error:

```ts
return Result.fail(
  AppError.validationError(request.intent, 'User could not be created.'),
)
```

The exact error should describe the failure that occurred in the application use
case.

Pipeline behaviors can also affect the final result before or around handler
execution. See the pipeline documentation for validation, authorization,
idempotency, concurrency, and other cross-cutting behavior.

## Troubleshooting

### `HANDLER_NOT_FOUND`

If the application reports that the handler cannot be found, check:

1. the handler is registered with `addServices()`;
2. `await app.build()` is called;
3. the registration token exactly matches the request `intent`;
4. the handler factory returns an object with a `handle()` method;
5. all dependencies required by the handler are registered.

For example:

```ts
super('CreateUserCommand')
```

requires:

```ts
import { TOKENS } from '@xeno-js/shared'

container.addTransient(
  'CreateUserCommand',
  (scope) => new CreateUserHandler(scope.resolve(TOKENS.USER_CONTEXT_FACTORY)),
)
```

`CreateUserCommand` and `createUserCommand` are different tokens.

### `SCOPE_NOT_AVAILABLE`

If the request cannot be processed because no service scope is available, make
sure the Command or Query is being executed through the normal Xeno.JS
application execution context.

Do not resolve and execute handlers manually outside the application scope when
using the mediator.

### `PIPELINE_NOT_AVAILABLE`

If the mediator reports that the CQRS pipeline is unavailable, ensure that the
application includes:

```ts
const app = new XenoAppBuilder().addPipeline()
```

before:

```ts
await app.build()
```

### Handler dependency cannot be resolved

If a handler constructor dependency cannot be resolved, verify that the
dependency is registered before the handler is created:

```ts
import { TOKENS } from '@xeno-js/shared'

.addServices((container) => {
  container.addScoped(
    'USER_REPOSITORY',
    (scope) => new UserRepository(
      scope.resolve('USER_DATA_SOURCE'),
    ),
  )

  container.addTransient(
    'CreateUserCommand',
    (scope) => new CreateUserHandler(
      scope.resolve('USER_REPOSITORY'),
      scope.resolve(TOKENS.USER_CONTEXT_FACTORY),
    ),
  )
})
```

The token passed to `scope.resolve()` must correspond to a registered service.

## Handler Checklist

Before running your application, verify:

- [ ] the handler extends `BaseHandler`;
- [ ] the request type matches the Command or Query;
- [ ] the response type matches the request response;
- [ ] `executeAsync()` is implemented;
- [ ] required dependencies are declared in the constructor;
- [ ] the handler is registered with `addServices()`;
- [ ] the registration token exactly matches the request `intent`;
- [ ] `addPipeline()` is enabled;
- [ ] the application is built with `await app.build()`;
- [ ] the Command is sent with `mediator.send()`;
- [ ] the Query is executed with `mediator.query()`.

## Related Documentation

- [CQRS Overview](./overview)
- [Creating Commands](./command)
- [Creating Queries](./query)
- [Validation Pipeline](../pipelines/validation)
- [Caching](../pipelines/caching)
- [Idempotency](../pipelines/idempotency)
- [Concurrency](../pipelines/concurrency)
- [Dependency Injection](../../fundamentals/dependency-injection)
- [App Builder](../../fundamentals/app-builder)

---

---

<a id="source-3"></a>

# Source: `docs/application/cqrs/overview.md`

## Introduction

Use the CQRS features in Xeno.JS to build application operations as **Commands**
and **Queries**, and execute them through their corresponding **Handlers**.

The Application module provides the main building blocks you need:

- **Commands** for operations that perform application actions;
- **Queries** for operations that retrieve data;
- **Handlers** for executing Commands and Queries;
- **Mediator** for sending Commands and executing Queries;
- **Pipeline behaviors** for cross-cutting concerns such as exception handling
  and logging, with optional plugins for validation, performance monitoring,
  caching, idempotency, authorization, and concurrency.

If you already know which task you need to perform, use the corresponding guide
below.

## Choose what you need to do

| I need to...                      | Go to                                                      |
| --------------------------------- | ---------------------------------------------------------- |
| Create a Command                  | [Create a Command](./command)                              |
| Create a Query                    | [Create a Query](./query)                                  |
| Create a Command or Query Handler | [Create a Handler](./handler)                              |
| Add validation                    | [Enable Validation](../pipelines/validation)               |
| Add logging                       | [Configure Logging](../pipelines/logging)                  |
| Track slow Commands and Queries   | [Configure Performance Tracking](../pipelines/performance) |
| Add idempotency to Commands       | [Configure Idempotency](../pipelines/idempotency)          |
| Handle concurrency conflicts      | [Configure Concurrency](../pipelines/concurrency)          |
| Cache Query results               | [Enable Caching](../pipelines/caching)                     |
| Handle exceptions                 | [Handle Exceptions](../pipelines/exception)                |

## Before you start

You typically need:

- an `XenoAppBuilder` instance;
- a Command or Query;
- a corresponding Handler;
- the mediator to execute the request within an active service scope.

If you are building an application from scratch, start with
[Create a Command](./command) or [Create a Query](./query).

## Commands

Use a Command when the application needs to perform an operation.

Examples include:

- creating a user;
- updating an order;
- deleting a resource;
- changing application state;
- triggering an application action.

Create a Command by following:

[Create a Command](./command)

Execute the Command through the mediator inside an active service scope. For
non-HTTP work, use `runInScope()` to create and dispose the scope:

```ts
const result = await mediator.send(command, signal)
```

The Command is associated with a Handler using its request intent.

For the complete registration and Handler workflow, see
[Create a Handler](./handler).

### Add Command-specific features

The CQRS module includes exception handling and logging behaviors. Add plugins
to enable optional common pipeline features, such as:

- validation;
- authorization;
- performance monitoring.

Commands can also use Command-specific features:

- idempotency;
- concurrency retry handling.

For example, add `withIdempotency` to configure idempotency:

```ts
import { XenoAppBuilder, withIdempotency } from 'xeno-js'

const app = new XenoAppBuilder().addPipeline(
  withIdempotency((options) => {
    options.lockTtlSeconds = 30
    options.processedTtlSeconds = 300
  }),
)

await app.build()
```

See [Configure Idempotency](../pipelines/idempotency) for the complete
configuration.

Add `withConcurrency` to `addPipeline()` to configure retries for concurrency
conflicts. See [Configure Concurrency](../pipelines/concurrency).

## Queries

Use a Query when the application needs to retrieve data.

Examples include:

- retrieving a user;
- loading an order;
- searching products;
- loading a paginated list;
- retrieving application data for a screen.

Create a Query by following:

[Create a Query](./query)

Execute the Query through the mediator inside an active service scope:

```ts
const result = await mediator.query(query, signal)
```

The Query is associated with a Handler using its request intent.

For the complete Handler workflow, see [Create a Handler](./handler).

### Cache Query results

Query caching is available through the Query pipeline.

Configure a cache and add the query-caching plugin:

```ts
import { XenoAppBuilder, withQueryCaching } from 'xeno-js'

const app = new XenoAppBuilder().addCache().addPipeline(withQueryCaching())

await app.build()
```

A cacheable Query defines its caching options when it is created.

For example:

```ts
export class GetUserQuery extends Query<User> {
  constructor(public readonly userId: string) {
    super('GetUserQuery', {
      cacheKey: `user:${userId}`,
      ttl: 60,
      bypassCache: false,
      consistentRead: false,
      isUserScoped: false,
    })
  }
}
```

For the complete caching workflow, see [Enable Caching](../pipelines/caching).

For the complete Query creation workflow, including Query parameters and
schemas, see [Create a Query](./query).

## Handlers

Every Command or Query needs a Handler that performs the actual application
operation.

Create a Handler by extending `BaseHandler` and implementing `executeAsync()`:

```ts
export class GetUserHandler extends BaseHandler<GetUserQuery, User> {
  protected async executeAsync(
    request: GetUserQuery,
  ): Promise<ResultType<User>> {
    // execute the application operation
  }
}
```

The Handler receives the request and can use its dependencies to perform the
operation.

See [Create a Handler](./handler) for:

- extending `BaseHandler`;
- injecting dependencies;
- implementing `executeAsync()`;
- accessing the current user context;
- registering the Handler;
- connecting the Handler to a Command or Query.

## Execute Commands and Queries

Commands and Queries are executed through different mediator methods. The
mediator requires an active service scope to resolve the pipeline and Handler;
use `runInScope()` for work outside an existing request.

### Execute a Command

```ts
const result = await mediator.send(command, signal)
```

Use `send()` for Commands.

### Execute a Query

```ts
const result = await mediator.query(query, signal)
```

Use `query()` for Queries.

Both operations return the result produced by the corresponding Handler or by a
pipeline behavior that completes the request before the Handler runs.

For example, a cached Query can return a cached result without executing its
Handler.

## Configure pipeline features

CQRS requests can use common pipeline features without putting that logic
directly into every Handler.

### Validation

Enable validation when Commands or Queries need input validation.

See:

[Enable Validation](../pipelines/validation)

The validation guide explains:

- how to configure validation;
- how to register Zod schemas;
- how to add custom validation;
- what happens when validation fails.

For creating the actual Query or Command schema, use:

- [Create a Command](./command)
- [Create a Query](./query)

### Logging

Use the logging pipeline to record Command and Query execution.

See:

[Configure Logging](../pipelines/logging)

### Performance tracking

Use the performance pipeline to detect slow Command and Query executions.

See:

[Configure Performance Tracking](../pipelines/performance)

### Exception handling

Use the exception pipeline to handle errors raised during request execution.

See:

[Handle Exceptions](../pipelines/exception)

### Idempotency

Use idempotency when a Command must not be processed more than once for the same
request ID.

Add `withIdempotency()` to `addPipeline()` and configure its options in the
plugin callback.

See:

[Configure Idempotency](../pipelines/idempotency)

Idempotency applies to Commands.

### Concurrency

Use concurrency retry handling when Commands can fail because of concurrency
conflicts.

Add `withConcurrency()` to `addPipeline()` and configure its options in the
plugin callback.

See:

[Configure Concurrency](../pipelines/concurrency)

This feature applies to Commands.

### Caching

Use query caching when repeated Query executions can reuse a previously stored
result.

Configure a cache and enable query caching with its plugin:

```ts
import { XenoAppBuilder, withQueryCaching } from 'xeno-js'

const app = new XenoAppBuilder().addCache().addPipeline(withQueryCaching())

await app.build()
```

Then define `cacheOptions` on the Query. `addPipeline()` accepts pipeline
plugins, not a configuration callback.

See:

[Enable Caching](../pipelines/caching)

## A typical CQRS workflow

A typical application flow looks like this:

```text
Create Command or Query
        │
        ▼
Create its Handler
        │
        ▼
Register the Handler
        │
        ▼
Configure required pipelines
        │
        ▼
Execute through the mediator
        │
        ▼
Receive the Result
```

For a Query with caching:

```text
Create Query
    │
    ▼
Define cacheOptions
    │
    ▼
Create Query Handler
    │
    ▼
addCache() + withQueryCaching()
    │
    ▼
mediator.query(...)
    │
    ├── cached result ──► return result
    │
    └── cache miss
            │
            ▼
       Query Handler
            │
            ▼
       store result
            │
            ▼
       return result
```

For a Command:

```text
Create Command
    │
    ▼
Create Command Handler
    │
    ▼
Configure Command pipelines
    │
    ▼
mediator.send(...)
    │
    ▼
Command Handler
    │
    ▼
Result
```

## Where to go next

Choose the guide that matches your task:

### Create a Command

[Create a Command](./command)

Use this when you need to define an operation that changes application state or
performs an application action.

### Create a Query

[Create a Query](./query)

Use this when you need to retrieve application data.

This guide also covers the Query-specific configuration used by features such as
caching.

### Create a Handler

[Create a Handler](./handler)

Use this when you need to implement or register the Handler for a Command or
Query.

### Configure application pipelines

Go to the relevant pipeline guide:

- [Enable Validation](../pipelines/validation)
- [Configure Logging](../pipelines/logging)
- [Configure Performance Tracking](../pipelines/performance)
- [Configure Idempotency](../pipelines/idempotency)
- [Configure Concurrency](../pipelines/concurrency)
- [Enable Caching](../pipelines/caching)
- [Handle Exceptions](../pipelines/exception)

---

---

<a id="source-4"></a>

# Source: `docs/application/cqrs/query.md`

## Introduction

Use a Query when your application needs to retrieve data without performing a
command operation.

A Xeno.JS Query:

1. extends the `Query` base class;
2. defines the input parameters required by the use case;
3. provides cache options;
4. is associated with a handler using its `intent`;
5. can optionally be validated;
6. can optionally participate in Query caching;
7. is executed through the mediator.

## Before you start

You need:

- an `XenoAppBuilder` instance;
- CQRS pipeline configuration through `addPipeline()`;
- a Query class;
- a Query handler;
- a service scope available when the Query is executed.

If you use validation, you also need a Zod schema or a custom validation
strategy.

If you use Query caching, add the `withQueryCaching()` plugin to the Query
pipeline.

## Enable CQRS

Configure CQRS through `XenoAppBuilder.addPipeline()`.

The minimal configuration is:

```ts
import { XenoAppBuilder } from '@xeno-js/core'

const app = new XenoAppBuilder().addPipeline()

const application = await app.build()
```

`addPipeline()` registers the CQRS infrastructure required to execute Commands
and Queries.

It also initializes the default application services used by the CQRS pipeline.

## Create a Query

Extend the `Query` class from `@xeno-js/core`.

```ts
import { Query } from '@xeno-js/core'

export interface User {
  id: string
  email: string
  name: string
}

export class GetUserQuery extends Query<User> {
  constructor(public readonly userId: string) {
    super('GetUserQuery', {
      cacheKey: `user:${userId}`,
      ttl: 60,
      bypassCache: false,
      consistentRead: false,
      isUserScoped: false,
    })
  }
}
```

The first argument passed to `super()` is the Query `intent`.

The intent is important because Xeno.JS uses it to identify the Query and
resolve the corresponding handler.

The `Query` base class also sets the request type to `REQUEST_TYPE.QUERY`.

## Define Query parameters

Define the data required by the Query directly on the Query class.

For example:

```ts
export class GetUserQuery extends Query<User> {
  constructor(public readonly userId: string) {
    super('GetUserQuery', {
      cacheKey: `user:${userId}`,
      ttl: 60,
      bypassCache: false,
      consistentRead: false,
      isUserScoped: false,
    })
  }
}
```

Keep the Query focused on the input required by the use case.

The handler is responsible for executing the use case and returning the result.

## Configure Query caching

A Query always receives an `ICacheableOptions` object.

The available options are:

| Option           | Description                                                     |
| ---------------- | --------------------------------------------------------------- |
| `cacheKey`       | Key used to identify the cached Query result.                   |
| `ttl`            | Optional cache lifetime.                                        |
| `bypassCache`    | When `true`, skips reading the cache for this request.          |
| `consistentRead` | When `true`, skips reading the cache and performs a fresh read. |
| `isUserScoped`   | When `true`, scopes the cache key to the current user.          |

For example:

```ts
export class GetUserQuery extends Query<User> {
  constructor(public readonly userId: string) {
    super('GetUserQuery', {
      cacheKey: `user:${userId}`,
      ttl: 60,
      bypassCache: false,
      consistentRead: false,
      isUserScoped: false,
    })
  }
}
```

### Choose a cache key

The cache key should identify the actual input that affects the Query result.

For example:

```ts
cacheKey: `user:${userId}`
```

For a Query with multiple parameters:

```ts
cacheKey: `users:${page}:${pageSize}:${status}`
```

Avoid using a static key when different Query parameters can produce different
results.

### Set a TTL

Use `ttl` to control how long a successful Query result remains cached.

```ts
ttl: 60
```

The `ICacheableOptions` contract documents `ttl` as seconds.

If you omit `ttl`, the configured cache implementation determines the effective
TTL behavior.

### Bypass the cache

Use `bypassCache` when a particular Query execution should not use an existing
cached value:

```ts
bypassCache: true
```

The Query still executes normally.

If the Query succeeds, its result can still be written to the cache.

### Use a consistent read

Use `consistentRead` when the Query should perform a fresh read instead of
returning an existing cached value:

```ts
consistentRead: true
```

Like `bypassCache`, this skips the cache read.

A successful result can still update the cache.

### Use user-scoped caching

Set `isUserScoped` to `true` when the Query result depends on the current user:

```ts
isUserScoped: true
```

For example:

```ts
export class GetCurrentUserProfileQuery extends Query<User> {
  constructor() {
    super('GetCurrentUserProfileQuery', {
      cacheKey: 'profile',
      ttl: 60,
      bypassCache: false,
      consistentRead: false,
      isUserScoped: true,
    })
  }
}
```

Use user-scoped caching for data that must not be shared between users.

## Enable Query caching

Query caching is enabled by adding the `withQueryCaching()` plugin to
`addPipeline()`.

Configure it as follows:

```ts
import { XenoAppBuilder, withQueryCaching } from '@xeno-js/core'

const app = new XenoAppBuilder().addPipeline(withQueryCaching())

await app.build()
```

When the plugin is enabled and no cache is already registered, the CQRS module
configures the in-memory cache by default. To use another provider, configure it
through `addCache()`.

You do not need to add `QueryCachingPipeline` manually.

### Configure the cache provider

`XenoAppBuilder` provides an in-memory cache configuration by default.

You can explicitly enable the in-memory cache:

```ts
const app = new XenoAppBuilder()
  .addCache((config) => {
    config.inMemory = true
  })
  .addPipeline(withQueryCaching())

await app.build()
```

A Redis configuration can also be supplied through `addCache()`:

```ts
import { XenoAppBuilder, withQueryCaching } from '@xeno-js/core'
import { withRedis } from '@xeno-js/redis'

const app = new XenoAppBuilder()
  .addCache(
    withRedis((options, config) => {
      options.host = config.get('REDIS_HOST') ?? 'localhost'
      options.port = config.getNumber('REDIS_PORT') ?? 6379
      options.options = {
        ...options.options,
        username: config.getOrThrow('REDIS_USERNAME'),
        password: config.getOrThrow('REDIS_PASSWORD'),
      }
    }),
  )
  .addPipeline(withQueryCaching())

await app.build()
```

`withRedis()` is provided by `@xeno-js/redis`; its options are adapter-specific.
`withQueryCaching()` comes from `@xeno-js/core`.

If you explicitly configure `addCache()`, provide a valid cache strategy. With
no explicit cache registration, enabling query caching uses the in-memory
provider.

## Create a Query schema

Queries can use the same validation pipeline used by Commands.

Define a Zod schema whose key matches the Query `intent`.

For the Query:

```ts
export class GetUserQuery extends Query<User> {
  constructor(public readonly userId: string) {
    super('GetUserQuery', {
      cacheKey: `user:${userId}`,
      ttl: 60,
      bypassCache: false,
      consistentRead: false,
      isUserScoped: false,
    })
  }
}
```

the validation schema can be:

```ts
import { z } from 'zod'

export const getUserQuerySchema = z.object({
  intent: z.literal('GetUserQuery'),
  type: z.literal('QUERY'),
  userId: z.string().min(1),
})
```

The schema is registered using the Query intent:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withZodValidation } from '@xeno-js/zod'

const app = new XenoAppBuilder().addPipeline(
  withZodValidation((options) => {
    options.schemas = {
      GetUserQuery: getUserQuerySchema,
    }
  }),
)
```

See [Validation Pipeline](../pipelines/validation) for the validation
configuration and behavior.

For the complete Query-specific schema workflow, keep the schema next to the
Query:

```text
src/
└── application/
    └── users/
        ├── get-user.query.ts
        ├── get-user.query.schema.ts
        └── get-user.handler.ts
```

## Create the Query Handler

A Query needs a handler registered under the same `intent`.

Extend `BaseHandler` and implement `executeAsync()`:

```ts
import { AppError } from '@xeno-js/shared'
import type { ResultType } from '@xeno-js/shared'
import { Result } from '@xeno-js/shared'

import { BaseHandler } from '@xeno-js/core'
import type { User } from './user.types'
import { GetUserQuery } from './get-user.query'

export class GetUserHandler extends BaseHandler<GetUserQuery, User> {
  protected async executeAsync(
    request: GetUserQuery,
  ): Promise<ResultType<User>> {
    const user = await this.findUser(request.userId)

    if (!user) {
      return Result.fail(AppError.notFound('GetUserHandler', 'User not found'))
    }

    return Result.ok(user)
  }

  private async findUser(userId: string): Promise<User | undefined> {
    // Load the user from your application service or repository.
    return undefined
  }
}
```

For handler registration and dependency injection, see
[Creating Handlers](./handler).

## Register the Query Handler

The handler registration token must exactly match the Query `intent`.

For the Query:

```ts
super('GetUserQuery', {
  cacheKey: `user:${userId}`,
  ttl: 60,
  bypassCache: false,
  consistentRead: false,
  isUserScoped: false,
})
```

register the handler using `GetUserQuery`:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { TOKENS } from '@xeno-js/shared'

const app = new XenoAppBuilder()
  .addServices((container) => {
    container.addTransient('GetUserQuery', (scope) => {
      return new GetUserHandler(scope.resolve(TOKENS.USER_CONTEXT_FACTORY))
    })
  })
  .addPipeline()

await app.build()
```

The string must match exactly.

```text
Query intent
    ↓
GetUserQuery
    ↓
registered handler
```

If the intent and registration token differ, Xeno.JS cannot resolve the handler.

## Execute a Query

Queries are executed through the mediator with `mediator.query()`. The mediator
resolves the Query pipeline and Handler from the active service scope.

```ts
import { TOKENS } from '@xeno-js/shared'

const query = new GetUserQuery('user-123')

const result = await application.runInScope(async (services) => {
  const mediator = services[TOKENS.MEDIATOR]
  return mediator.query(query, new AbortController().signal)
})
```

The result is a `ResultType<TResponse>`.

Check the result before using its value:

```ts
if (result.isOk()) {
  const user = result.getValueOrThrow()

  console.log(user)
}
```

## Complete example

A complete Query setup can look like this:

```text
src/
└── application/
    └── users/
        ├── get-user.query.ts
        ├── get-user.query.schema.ts
        ├── get-user.handler.ts
        └── user.types.ts
└── bootstrap.ts
```

### `user.types.ts`

```ts
export interface User {
  id: string
  email: string
  name: string
}
```

### `get-user.query.ts`

```ts
import { Query } from '@xeno-js/core'

import type { User } from './user.types'

export class GetUserQuery extends Query<User> {
  constructor(public readonly userId: string) {
    super('GetUserQuery', {
      cacheKey: `user:${userId}`,
      ttl: 60,
      bypassCache: false,
      consistentRead: false,
      isUserScoped: false,
    })
  }
}
```

### `get-user.query.schema.ts`

```ts
import { z } from 'zod'

export const getUserQuerySchema = z.object({
  intent: z.literal('GetUserQuery'),
  type: z.literal('QUERY'),
  userId: z.string().min(1),
})
```

### `get-user.handler.ts`

```ts
import type { ResultType } from '@xeno-js/shared'
import { Result } from '@xeno-js/shared'

import { BaseHandler } from '@xeno-js/core'

import { GetUserQuery } from './get-user.query'
import type { User } from './user.types'

export class GetUserHandler extends BaseHandler<GetUserQuery, User> {
  protected async executeAsync(
    request: GetUserQuery,
  ): Promise<ResultType<User>> {
    const user: User = {
      id: request.userId,
      email: 'john@example.com',
      name: 'John Doe',
    }

    return Result.ok(user)
  }
}
```

### `bootstrap.ts`

```ts
import { XenoAppBuilder, withQueryCaching } from '@xeno-js/core'
import { TOKENS } from '@xeno-js/shared'
import { withZodValidation } from '@xeno-js/zod'

import { GetUserHandler } from './application/users/get-user.handler'
import { GetUserQuery } from './application/users/get-user.query'
import { getUserQuerySchema } from './application/users/get-user.query.schema'

const app = new XenoAppBuilder()
  .addServices((container) => {
    container.addTransient('GetUserQuery', (scope) => {
      return new GetUserHandler(scope.resolve(TOKENS.USER_CONTEXT_FACTORY))
    })
  })
  .addPipeline([
    withQueryCaching(),
    withZodValidation((options) => {
      options.schemas = {
        GetUserQuery: getUserQuerySchema,
      }
    }),
  ])

const application = await app.build()

const query = new GetUserQuery('user-123')

const result = await application.runInScope(async (services) => {
  const mediator = services[TOKENS.MEDIATOR]
  return mediator.query(query, new AbortController().signal)
})

if (result.isOk()) console.log(result.getValueOrThrow())
```

This setup provides:

- a Query with typed input and output;
- a registered Query handler;
- Zod validation;
- Query caching;
- a cache key based on the Query parameter;
- execution through the mediator.

## Understand Query caching behavior

When Query caching is enabled and the Query has a `cacheKey`, Xeno.JS behaves as
follows:

```text
Query
  │
  ├── cache read
  │      │
  │      ├── hit → return cached result
  │      │
  │      └── miss
  │
  ├── execute handler
  │
  └── successful result → write to cache
```

A cache hit returns the cached result without executing the handler.

A successful cache miss stores the handler result.

Failed Query results are not written to the cache.

### Cache errors do not fail the Query

If a cache read fails, Xeno.JS logs a warning and continues with the Query
execution.

If a cache write fails, Xeno.JS logs a warning and returns the Query result
normally.

This means the cache is not required to be available for the Query handler to
produce its result after a cache read/write failure.

## Skip the cache for one Query

To force a fresh read:

```ts
export class GetUserQuery extends Query<User> {
  constructor(public readonly userId: string) {
    super('GetUserQuery', {
      cacheKey: `user:${userId}`,
      ttl: 60,
      bypassCache: true,
      consistentRead: false,
      isUserScoped: false,
    })
  }
}
```

`bypassCache: true` prevents an existing cached value from being returned.

The Query still executes and, if successful, its result can be cached.

For a semantically explicit fresh read, use:

```ts
consistentRead: true
```

Both options prevent the cache from being read.

## Troubleshooting

### The Query handler is not being called

Check:

1. the Query extends `Query`;
2. the `intent` is correct;
3. the handler is registered with exactly the same intent;
4. `addPipeline()` is configured;
5. the Query is executed through `mediator.query()`.

For example:

```ts
super('GetUserQuery', {
  cacheKey: `user:${userId}`,
  ttl: 60,
  bypassCache: false,
  consistentRead: false,
  isUserScoped: false,
})
```

must be registered as:

```ts
import { TOKENS } from '@xeno-js/shared'

container.addTransient(
  'GetUserQuery',
  (scope) => new GetUserHandler(scope.resolve(TOKENS.USER_CONTEXT_FACTORY)),
)
```

### Query caching is not working

Check:

1. `withQueryCaching()` is passed to `addPipeline()`;
2. `addPipeline()` is called;
3. the Query has a non-empty `cacheKey`;
4. if `addCache()` is configured explicitly, it has a valid cache strategy
   (otherwise query caching uses the default in-memory cache);
5. the Query parameters are represented in the cache key.

For example:

```ts
.addPipeline(withQueryCaching())
```

and:

```ts
super('GetUserQuery', {
  cacheKey: `user:${userId}`,
  ttl: 60,
  bypassCache: false,
  consistentRead: false,
  isUserScoped: false,
})
```

### Every Query executes even when caching is configured

Check whether caching is enabled:

```ts
withQueryCaching()
```

Configuring a cache provider alone does not add Query caching to the Query
pipeline.

### Validation is not running

Check:

1. `addPipeline()` includes `withZodValidation(...)`;
2. the plugin registers a schema;
3. the schema is registered using the exact Query intent;
4. the Query is executed through the mediator.

For example:

```ts
withZodValidation((options) => {
  options.schemas = {
    GetUserQuery: getUserQuerySchema,
  }
})
```

The key must match:

```ts
super('GetUserQuery', ...)
```

See [Validation Pipeline](../pipelines/validation) for validation-specific
troubleshooting.

### The wrong cached result is returned

Check the `cacheKey`.

Every value that can change the Query result should be represented in the key.

For example, avoid:

```ts
cacheKey: 'users'
```

when the Query depends on:

```ts
page
pageSize
status
```

Instead use a key that includes those parameters:

```ts
cacheKey: `users:${page}:${pageSize}:${status}`
```

If the result depends on the current user, also consider:

```ts
isUserScoped: true
```

## Related documentation

- [CQRS Overview](./overview) — Find the Query, Command, and Handler guides.
- [Creating Commands](./command) — Create and execute Commands.
- [Creating Handlers](./handler) — Create and register Query and Command
  handlers.
- [Validation Pipeline](../pipelines/validation) — Validate Queries and Commands
  with Zod or custom validation.
- [Caching Pipeline](../pipelines/caching) — Configure and understand Query
  caching.
- [Performance Pipeline](../pipelines/performance) — Track slow Query
  executions.
- [Logging Pipeline](../pipelines/logging) — Configure CQRS logging.
- [Exception Pipeline](../pipelines/exception) — Handle exceptions during Query
  execution.

---

---

<a id="source-5"></a>

# Source: `docs/application/cqrs/zod-schema.md`

## Introduction

When you validate Xeno.JS commands and queries with Zod, use `ZodUtils` to
create the schema instead of manually rebuilding the request metadata.

`ZodUtils` is exported by `@xeno-js/zod` and provides:

- `createCommandSchema()` for commands;
- `createQuerySchema()` for queries.

Both helpers combine your application-specific fields with the Xeno.JS base
schema and return a strict Zod object.

This means you do not need to manually add:

- `intent`;
- `type`;
- query `cacheOptions`.

## Before you start

You need:

- `@xeno-js/core`;
- `@xeno-js/zod`;
- `zod`;
- a Command or Query;
- the Zod validation pipeline enabled;
- a schema registered under the same `intent` as the request.

For validation configuration, see
[Validation Pipeline](../pipelines/validation).

---

## Create a Command Schema

Use `ZodUtils.createCommandSchema()` when creating a schema for a `Command`.

```ts
import { z } from 'zod'
import { ZodUtils } from '@xeno-js/zod'

const createUserSchema = ZodUtils.createCommandSchema('CreateUserCommand', {
  email: z.string().email(),
  name: z.string().min(1),
})
```

The second argument is the command-specific Zod shape.

You should **not** wrap it in `z.object()`.

### What ZodUtils adds

The resulting schema includes the fields required by the Xeno.JS command
contract:

```ts
{
  intent: z.literal('CreateUserCommand'),
  type: z.literal(REQUEST_TYPE.COMMAND),
  email: z.string().email(),
  name: z.string().min(1),
}
```

You only define the fields belonging to your command.

This avoids repeating request metadata in every schema.

---

## Create a Query Schema

Use `ZodUtils.createQuerySchema()` for a `Query`.

```ts
import { z } from 'zod'
import { ZodUtils } from '@xeno-js/zod'

const getUserSchema = ZodUtils.createQuerySchema('GetUserQuery', {
  userId: z.string().uuid(),
})
```

The generated schema includes:

```ts
{
  intent: z.literal('GetUserQuery'),
  type: z.literal(REQUEST_TYPE.QUERY),
  cacheOptions: {
    cacheKey: z.string().min(1),
    ttl: z.number().int().positive().optional(),
    bypassCache: z.boolean().optional(),
    consistentRead: z.boolean().optional(),
    isUserScoped: z.boolean()
  },
  userId: z.string().uuid(),
}
```

You therefore do not need to add `intent`, `type`, or `cacheOptions` to your
application-specific shape.

---

## Add Query Cache Options

A Xeno.JS `Query` contains `cacheOptions`.

For example:

```ts
import { Query } from '@xeno-js/core'

export class GetUserQuery extends Query<User> {
  constructor(public readonly userId: string) {
    super('GetUserQuery', {
      cacheKey: `user:${userId}`,
      ttl: 60,
      bypassCache: false,
      consistentRead: false,
      isUserScoped: false,
    })
  }
}
```

The generated query schema validates these cache options automatically.

### Available cache options

| Property         | Required | Description                                     |
| ---------------- | -------- | ----------------------------------------------- |
| `cacheKey`       | Yes      | Key used to identify the cached result.         |
| `ttl`            | No       | Cache lifetime in seconds.                      |
| `bypassCache`    | No       | Bypasses a cached value for the request.        |
| `consistentRead` | No       | Requests a read that bypasses the cached value. |
| `isUserScoped`   | Yes      | Whether the cache is user-scoped.               |

`cacheKey` must be a non-empty string.

`ttl`, when provided, must be a positive integer.

---

## Add Only Application-Specific Fields

The purpose of `ZodUtils` is to let the schema focus on the data defined by your
command or query.

For example, given:

```ts
export class CreateUserCommand extends Command<User> {
  constructor(
    public readonly email: string,
    public readonly name: string,
  ) {
    super('CreateUserCommand')
  }
}
```

create the schema with:

```ts
const createUserSchema = ZodUtils.createCommandSchema('CreateUserCommand', {
  email: z.string().email(),
  name: z.string().min(1),
})
```

Do not repeat:

```ts
intent: z.literal('CreateUserCommand'),
type: z.literal(...),
```

Those fields are provided by `ZodUtils`.

---

## Do Not Pass a Complete `z.object()`

`ZodUtils` expects a `ZodRawShape`, not an already-created Zod object.

Use:

```ts
const schema = ZodUtils.createCommandSchema('CreateUserCommand', {
  email: z.string().email(),
  name: z.string().min(1),
})
```

Not:

```ts
const schema = ZodUtils.createCommandSchema(
  'CreateUserCommand',
  z.object({
    email: z.string().email(),
    name: z.string().min(1),
  }),
)
```

The helper creates and extends the base Zod object itself.

---

## Register the Schema

Once the schema is created, register it under the same command or query
`intent`.

For example:

```ts
import { z } from 'zod'
import { XenoAppBuilder } from '@xeno-js/core'
import { ZodUtils, withZodValidation } from '@xeno-js/zod'

const createUserSchema = ZodUtils.createCommandSchema('CreateUserCommand', {
  email: z.string().email(),
  name: z.string().min(1),
})

const getUserSchema = ZodUtils.createQuerySchema('GetUserQuery', {
  userId: z.string().uuid(),
})

const app = new XenoAppBuilder().addPipeline(
  withZodValidation((options) => {
    options.schemas = {
      CreateUserCommand: createUserSchema,
      GetUserQuery: getUserSchema,
    }
  }),
)
```

The schema registry is keyed by request `intent`.

Therefore:

```ts
super('CreateUserCommand')
```

must correspond to:

```ts
CreateUserCommand: createUserSchema
```

Likewise:

```ts
super('GetUserQuery', ...)
```

must correspond to:

```ts
GetUserQuery: getUserSchema
```

---

## Complete Command Example

A command schema can be kept next to the command:

```text
src/
└── application/
    └── users/
        ├── create-user.command.ts
        └── create-user.schema.ts
```

### `create-user.command.ts`

```ts
import { Command } from '@xeno-js/core'

export interface User {
  id: string
  email: string
  name: string
}

export class CreateUserCommand extends Command<User> {
  constructor(
    public readonly email: string,
    public readonly name: string,
  ) {
    super('CreateUserCommand')
  }
}
```

### `create-user.schema.ts`

```ts
import { z } from 'zod'
import { ZodUtils } from '@xeno-js/zod'

export const createUserSchema = ZodUtils.createCommandSchema(
  'CreateUserCommand',
  {
    email: z.string().email(),
    name: z.string().min(1),
  },
)
```

### `bootstrap.ts`

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withZodValidation } from '@xeno-js/zod'
import { createUserSchema } from './application/users/create-user.schema'

const app = new XenoAppBuilder().addPipeline(
  withZodValidation((options) => {
    options.schemas = {
      CreateUserCommand: createUserSchema,
    }
  }),
)

await app.build()
```

---

## Complete Query Example

For a query, define the query-specific fields and let `ZodUtils` add the request
and cache fields.

```text
src/
└── application/
    └── users/
        ├── get-user.query.ts
        └── get-user.schema.ts
```

### `get-user.query.ts`

```ts
import { Query } from '@xeno-js/core'

export interface User {
  id: string
  email: string
  name: string
}

export class GetUserQuery extends Query<User> {
  constructor(public readonly userId: string) {
    super('GetUserQuery', {
      cacheKey: `user:${userId}`,
      ttl: 60,
      bypassCache: false,
      consistentRead: false,
      isUserScoped: false,
    })
  }
}
```

### `get-user.schema.ts`

```ts
import { z } from 'zod'
import { ZodUtils } from '@xeno-js/zod'

export const getUserSchema = ZodUtils.createQuerySchema('GetUserQuery', {
  userId: z.string().uuid(),
})
```

### `src/bootstrap.ts`

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withZodValidation } from '@xeno-js/zod'
import { getUserSchema } from './application/users/get-user.schema'

const app = new XenoAppBuilder().addPipeline(
  withZodValidation((options) => {
    options.schemas = {
      GetUserQuery: getUserSchema,
    }
  }),
)

await app.build()
```

---

## Understand Strict Validation

`ZodUtils` calls `.strict()` on the resulting schema.

This means properties that are not part of the generated schema are rejected.

For example:

```ts
const schema = ZodUtils.createCommandSchema('CreateUserCommand', {
  email: z.string().email(),
})
```

The schema does not accept an arbitrary additional property such as:

```ts
{
  email: 'john@example.com',
  unexpectedField: true,
}
```

Define every application-specific property that the request is expected to
contain.

You should also avoid manually adding the base properties because `ZodUtils`
already provides them.

---

## Troubleshooting

### `intent` validation fails

Check that the intent used by the request exactly matches the intent passed to
`ZodUtils`.

For example:

```ts
super('CreateUserCommand')
```

must use:

```ts
ZodUtils.createCommandSchema('CreateUserCommand', ...)
```

Intent matching is exact.

---

### `type` validation fails

Do not manually define the `type` field.

Use the correct helper:

```ts
ZodUtils.createCommandSchema(...)
```

for commands and:

```ts
ZodUtils.createQuerySchema(...)
```

for queries.

The helpers add the appropriate request type automatically.

---

### Query validation fails on `cacheOptions`

Check that the query provides a `cacheOptions` object with a non-empty:

```ts
cacheKey
```

For example:

```ts
super('GetUserQuery', {
  cacheKey: `user:${userId}`,
})
```

If `ttl` is provided, it must be a positive integer.

---

### Validation rejects an unexpected property

The generated schema is strict.

Make sure the property is included in the shape passed to `ZodUtils`.

For example:

```ts
ZodUtils.createQuerySchema('GetUserQuery', {
  userId: z.string().uuid(),
})
```

If the request contains another application-specific property, add it to the
shape.

---

### The schema is not being used

Check:

1. the Zod validation pipeline is enabled;
2. the schema is registered under the request `intent`;
3. the request's `intent` exactly matches the registry key;
4. the request is executed through the configured mediator;
5. `ZodUtils` is imported from `@xeno-js/zod`.

See [Validation Pipeline](../pipelines/validation) for the complete validation
configuration.

---

## When to Use `ZodUtils`

Use `ZodUtils` whenever you create a Zod schema for a Xeno.JS CQRS command or
query.

It keeps the schema definition focused on application data:

```ts
ZodUtils.createCommandSchema('CreateUserCommand', {
  email: z.string().email(),
  name: z.string().min(1),
})
```

instead of duplicating Xeno.JS request metadata:

```ts
z.object({
  intent: z.literal('CreateUserCommand'),
  type: ...,
  email: z.string().email(),
  name: z.string().min(1),
})
```

For queries, it also supplies the base `cacheOptions` schema.

---

## Related Documentation

- [Creating Commands](./command)
- [Creating Queries](./query)
- [Validation Pipeline](../pipelines/validation)
- [Creating Handlers](./handler)

---

---

<a id="source-6"></a>

# Source: `docs/application/overview.md`

## Introduction

The Application module provides the tools you need to define and execute application use cases with **CQRS** and **pipeline behaviors**.

Use this section when you need to:

- create a command;
- create a query;
- create a handler;
- validate commands or queries;
- add logging around application requests;
- monitor execution performance;
- add idempotency to commands;
- configure concurrency retries;
- enable query processing features such as caching;
- handle exceptions consistently.

For operation outcomes and application errors, see
[Result](/docs/application/result) and [AppError](/docs/application/app-error).

## Before you start

Application features are composed through `XenoAppBuilder`. Pass pipeline plugins to `addPipeline()` to configure the corresponding cross-cutting behavior.

For example, add performance tracking to the command and query pipelines:

```ts
import { XenoAppBuilder, withPerformance } from '@xeno-js/core'

const app = await new XenoAppBuilder()
  .addPipeline(withPerformance())
  .build()
```

Configure a plugin through its setup callback when you need to override its defaults:

```ts
import { XenoAppBuilder, withPerformance } from '@xeno-js/core'

const app = await new XenoAppBuilder()
  .addPipeline(withPerformance((options) => {
    options.thresholdMs = 500
    options.intentThresholdMs = {
      CREATE_USER_HANDLER: { thresholdMs: 300 },
    }
  }))
  .build()
```

Call `build()` after configuring the required plugins. The exact options depend on the plugin; each feature page documents its configuration and prerequisites.

### Create a command

To create a command, see:

**[Create a Command](./cqrs/command)**

The guide covers:

- defining the command;
- defining its input;
- adding validation;
- sending the command;
- connecting it to a handler.

### Create a query

To create a query, see:

**[Create a Query](./cqrs/query)**

The guide covers:

- defining the query;
- defining its parameters;
- adding validation;
- executing the query;
- connecting it to a handler.

### Create a handler

To implement the application logic for a command or query, see:

**[Create a Handler](./cqrs/handler)**

The guide covers:

- extending `BaseHandler`;
- implementing the handler method;
- receiving dependencies;
- accessing the current user context when required;
- registering the handler;
- connecting the handler to its command or query.

## Pipelines

Pipeline behaviors let you add cross-cutting behavior to commands and queries without putting that behavior directly into every handler.

Choose the feature you need:

| Task                                 | Documentation                                   |
| ------------------------------------ | ----------------------------------------------- |
| Handle application exceptions        | [Exception Pipeline](./pipelines/exception)     |
| Add request logging                  | [Logging Pipeline](./pipelines/logging)         |
| Monitor execution time               | [Performance Pipeline](./pipelines/performance) |
| Validate commands and queries        | [Validation Pipeline](./pipelines/validation)   |
| Prevent duplicate command processing | [Idempotency Pipeline](./pipelines/idempotency) |
| Retry concurrency conflicts          | [Concurrency Pipeline](./pipelines/concurrency) |
| Enable query caching                 | [Caching Pipeline](./pipelines/caching)         |

### Validation

If you need to validate a command or query before its handler runs:

**[Enable Validation](./pipelines/validation)**

The validation guide explains:

- how to configure validation in `XenoAppBuilder`;
- how to use Zod schemas;
- where to define schemas;
- how to configure custom validation;
- what happens when validation fails.

For the command and query definitions used by the examples, see:

- [Create a Command](./cqrs/command)
- [Create a Query](./cqrs/query)

### Logging

To configure logging around command and query execution:

**[Configure Logging](./pipelines/logging)**

### Performance

To monitor command and query execution time:

**[Configure Performance Tracking](./pipelines/performance)**

### Idempotency

To prevent duplicate processing of commands:

**[Configure Idempotency](./pipelines/idempotency)**

### Concurrency

To configure concurrency retries for commands:

**[Configure Concurrency](./pipelines/concurrency)**

### Caching

To enable query caching:

**[Enable Caching](./pipelines/caching)**

### Exception handling

To configure or troubleshoot exception handling for application requests:

**[Configure Exception Handling](./pipelines/exception)**

## CQRS

Use the CQRS section when you are creating application requests and their handlers.

| Task                         | Documentation                      |
| ---------------------------- | ---------------------------------- |
| Understand the CQRS workflow | [CQRS Overview](./cqrs/overview)   |
| Create a command             | [Create a Command](./cqrs/command) |
| Create a query               | [Create a Query](./cqrs/query)     |
| Create a handler             | [Create a Handler](./cqrs/handler) |

### Typical workflow

A typical application use case follows this path:

```text
Create Command or Query
        │
        ▼
Add input validation if required
        │
        ▼
Create the Handler
        │
        ▼
Register the Handler
        │
        ▼
Configure required pipelines
        │
        ▼
Execute the Command or Query
```

For example, when creating a new command:

```text
Create Command
      │
      ├── Add validation
      │      └── Validation Pipeline
      │
      └── Create Handler
             └── Register Handler
```

The individual guides contain the implementation details required for each step.

## Where should I start?

Use the task that matches what you are trying to do:

- **I need a command** → [Create a Command](./cqrs/command)
- **I need a query** → [Create a Query](./cqrs/query)
- **I need a handler** → [Create a Handler](./cqrs/handler)
- **I need validation** → [Validation Pipeline](./pipelines/validation)
- **I need logging** → [Logging Pipeline](./pipelines/logging)
- **I need performance tracking** → [Performance Pipeline](./pipelines/performance)
- **I need idempotency** → [Idempotency Pipeline](./pipelines/idempotency)
- **I need concurrency retries** → [Concurrency Pipeline](./pipelines/concurrency)
- **I need caching** → [Caching Pipeline](./pipelines/caching)
- **I need exception handling** → [Exception Pipeline](./pipelines/exception)

### Troubleshooting

If a command or query does not execute as expected, check:

1. the required pipeline plugin is passed to `XenoAppBuilder.addPipeline()`;
2. the plugin's required configuration is provided;
3. the command or query is connected to a handler;
4. the handler is registered in the application;
5. the request is executed through the configured mediator;
6. any feature-specific prerequisites documented in the relevant pipeline guide are satisfied.

If the problem is specific to a feature, start from its dedicated guide rather than troubleshooting the entire Application module.

## Related documentation

- [Create a Command](./cqrs/command)
- [Create a Query](./cqrs/query)
- [Create a Handler](./cqrs/handler)
- [CQRS Overview](./cqrs/overview)
- [Exception Pipeline](./pipelines/exception)
- [Performance Pipeline](./pipelines/performance)
- [Logging Pipeline](./pipelines/logging)
- [Validation Pipeline](./pipelines/validation)
- [Idempotency Pipeline](./pipelines/idempotency)
- [Concurrency Pipeline](./pipelines/concurrency)
- [Caching Pipeline](./pipelines/caching)

---

---

# Source: `docs/application/pipelines/enrichment-identity.md`

## Introduction

Use the Identity Enrichment Pipeline to enrich the current request identity
before command and query execution continues.

This is useful when authentication gives you a base identity and you want to
attach additional metadata (for example, tenant, roles, flags, or profile
details) before authorization and handler logic run.

When enabled, Xeno.JS:

1. reads the current identity from request context;
2. executes the configured enrichment strategy;
3. continues with the next pipeline behavior.

If no strategy is configured, Xeno.JS uses a noop strategy that does not modify
identity.

## Before you start

You need:

- an `XenoAppBuilder` instance;
- the CQRS pipeline enabled with `addPipeline()`;
- authentication configured if you expect an initial identity to be present;
- a custom `IStrategy<Identity>` if you need enrichment logic.

For context and identity APIs, see
[Context and Scopes](../../fundamentals/context-scopes).

## Enable identity enrichment

Enable the plugin with `withEnrichmentIdentity()`:

```ts
import { XenoAppBuilder, withEnrichmentIdentity } from '@xeno-js/core'

const app = new XenoAppBuilder().addPipeline(withEnrichmentIdentity())

const application = await app.build()
```

You do not need to instantiate `IdentityEnrichmentPipeline` manually.

## Configure a custom enrichment strategy

Use `withEnrichmentIdentity()` to provide a custom strategy:

```ts
import type { Identity, IStrategy, ResultType } from '@xeno-js/shared'
import { Result, XenoAppBuilder, withEnrichmentIdentity } from '@xeno-js/core'

class AttachTenantStrategy implements IStrategy<Identity> {
  public async execute(identity: Identity): Promise<ResultType<void>> {
    identity.tenantId = 'tenant-acme'
    identity.roles = [...(identity.roles ?? []), 'member']
    return Result.ok()
  }
}

const app = new XenoAppBuilder().addPipeline(
  withEnrichmentIdentity((options) => {
    options.strategy = new AttachTenantStrategy()
  }),
)

const application = await app.build()
```

The strategy receives the current `Identity` object from request context (or an
empty object when no identity exists yet).

## Understand execution order

Identity enrichment is added as a **common** pipeline behavior:

- it runs for **commands**;
- it runs for **queries**.

In CQRS pipeline ordering, enrichment runs before authorization.

Conceptually:

```text
Exception
  ↓
Logging
  ↓
Identity enrichment
  ↓
Authorization
  ↓
Command/Query handler
```

## Behavior when no strategy is configured

If the plugin is enabled without a custom strategy, Xeno.JS uses
`NoopIdentityEnrichmentStrategy`.

That default strategy always returns success and leaves identity unchanged.

## Common use cases

Use identity enrichment when you need to:

- normalize identity fields before authorization;
- map provider-specific claims to application-level claims;
- attach tenant information resolved from identity attributes;
- add derived flags used by downstream policies or handlers.

Keep the strategy focused on identity transformation. Prefer authorization
policies for allow/deny decisions.

## Best practices

- Keep enrichment deterministic and fast.
- Do not perform long-running I/O inside the strategy when avoidable.
- Treat missing identity fields explicitly.
- Keep authorization rules in the authorization layer; enrich identity data
  here.

## Related documentation

- [Logging Pipeline](./logging)
- [Validation Pipeline](./validation)
- [Idempotency Pipeline](./idempotency)
- [Context and Scopes](../../fundamentals/context-scopes)
- [Creating Commands](../cqrs/command)
- [Creating Queries](../cqrs/query)

---

---

<a id="source-8"></a>

# Source: `docs/application/pipelines/exception.md`

## Introduction

The `ExceptionPipeline` is a CQRS behavior that handles values thrown while
executing the next command or query behavior, including the handler.

It returns a failed `Result` for both existing `AppError` instances and other
thrown values. It does not itself write logs or create an HTTP response.

`XenoAppBuilder.addPipeline()` enables the CQRS module, which adds
`ExceptionPipeline` as a common behavior for commands and queries. You do not
configure it separately.

## Enable the Exception Pipeline

Before using the CQRS pipeline, configure it through `XenoAppBuilder`:

```ts
import { XenoAppBuilder } from '@xeno-js/core'

const app = new XenoAppBuilder().addPipeline()

await app.build()
```

`addPipeline()` queues the CQRS module, which is initialized during `build()`.

The Exception Pipeline is registered automatically as part of this
configuration.

There is no separate setting such as:

```ts
config.exception = {
  enabled: true,
}
```

## How exception handling works

When a command or query runs through the configured CQRS pipeline, the
`ExceptionPipeline` awaits the next behavior. This continues through the
remaining behaviors and the handler. If that execution throws, the behavior
converts the thrown value according to its type.

### Existing `AppError`

If the thrown value is already an `AppError`, Xeno.JS preserves that error.

For example:

```ts
throw AppError.create({
  code: 'USER_NOT_FOUND',
  message: 'User not found',
  status: 404,
  name: 'GetUserQuery',
})
```

The resulting `Result.fail` contains the same `AppError` instance.

This preserves the error instance and its application-defined metadata.

### Other thrown values

If the thrown value is not an `AppError` (including non-`Error` values), Xeno.JS
creates a system error.

For example:

```ts
throw new Error('Unexpected database failure')
```

The Exception Pipeline converts it into an `AppError` with:

- code: `SYSTEM_ERROR`;
- status: `500` (`STATUS_CODES.INTERNAL_SERVER_ERROR`);
- the CQRS request's `intent` as the error name;
- the original thrown value as the `cause`.

The original thrown value is therefore available as the `cause` of the resulting
`AppError`. The status is error metadata; this behavior does not itself create
or send an HTTP response.

## Use `AppError` for expected application errors

Use `AppError` when the error is part of the expected behavior of your
application.

For example, a use case may reject an operation because a requested resource
does not exist:

```ts
import { AppError } from '@xeno-js/shared'

export class GetUserHandler {
  async execute(userId: string) {
    const user = await this.repository.findById(userId)

    if (!user) {
      throw AppError.create({
        code: 'USER_NOT_FOUND',
        message: 'User not found',
        status: 404,
        name: 'GetUserQuery',
      })
    }

    return user
  }
}
```

The Exception Pipeline preserves this error instead of replacing it with a
generic system error.

This means callers can distinguish application errors from unexpected failures.

## Handle unexpected errors

You do not need to wrap every unexpected exception manually.

For example:

```ts
export class CreateUserHandler {
  async execute(request: CreateUserRequest) {
    const user = await this.repository.create(request)

    return user
  }
}
```

If an unexpected exception is thrown by the repository or another dependency:

```ts
throw new Error('Database connection failed')
```

the Exception Pipeline converts it into a failed `Result` containing a
`SYSTEM_ERROR` with status `500`.

The original thrown value is retained as the `cause`.

## Result of exception handling

When the next behavior or handler throws, the Exception Pipeline returns a
failed `Result` rather than propagating that thrown value from this pipeline
call.

Conceptually:

```text
Command / Query
      │
      ▼
Application execution
      │
      ├── success ───────────────► Result.ok(...)
      │
      └── exception
            │
            ├── AppError
            │      └─────────────► Result.fail(existing AppError)
            │
            └── other thrown value
                   └─────────────► Result.fail(
                                      AppError(
                                        SYSTEM_ERROR,
                                        status = 500,
                                        cause = original value
                                      )
                                    )
```

This gives command and query dispatch a consistent failed `Result` for
application errors and other thrown values that reach this behavior.

## Throwing an `AppError`

When creating an application-specific error, provide a meaningful error code,
message, status, and request name.

```ts
import { AppError } from '@xeno-js/shared'

throw AppError.create({
  code: 'ORDER_ALREADY_CANCELLED',
  message: 'The order has already been cancelled.',
  status: 409,
  name: 'CancelOrderCommand',
})
```

The error remains an `AppError` when it reaches the caller.

## Unexpected errors

For unexpected errors, normal JavaScript errors can be thrown directly:

```ts
throw new Error('Unexpected failure')
```

Xeno.JS converts the error into a system `AppError`.

You should not manually convert every unexpected exception into `SYSTEM_ERROR`.
The Exception Pipeline performs this conversion for you.

## Command and query execution

The CQRS module adds `ExceptionPipeline` as a common behavior at priority `0`,
so it applies to both command and query pipelines. The logging behavior is
registered separately; exception conversion itself does not log.

This means the behavior is available to:

- commands;
- queries;
- their handlers;
- dependencies called while those handlers or subsequent behaviors execute.

You therefore do not need separate exception handling configuration for commands
and queries.

For creating the request itself, see:

- [Creating Commands](../cqrs/command)
- [Creating Queries](../cqrs/query)
- [Creating Handlers](../cqrs/handler)

## Troubleshooting

### Exceptions are not being converted to `Result.fail`

Check that the application has enabled the CQRS pipeline:

```ts
const app = new XenoAppBuilder().addPipeline()

await app.build()
```

Without `addPipeline()`, the CQRS pipeline module is not initialized.

### My custom `AppError` becomes a `SYSTEM_ERROR`

Make sure the value being thrown is an actual `AppError`:

```ts
throw AppError.create({
  code: 'USER_NOT_FOUND',
  message: 'User not found',
  status: 404,
  name: 'GetUserQuery',
})
```

The Exception Pipeline preserves errors that are instances of `AppError`.

### My thrown value produces status 500

If the thrown value is not an `AppError`, the Exception Pipeline converts it to
a system error with status `500`.

If the error represents an expected application condition, create and throw an
appropriate `AppError` instead.

For example:

```ts
throw AppError.create({
  code: 'RESOURCE_NOT_FOUND',
  message: 'Resource not found',
  status: 404,
  name: 'GetResourceQuery',
})
```

### I need to inspect the original thrown value

Unexpected thrown values are stored as the `cause` of the generated `AppError`.

When handling the failed result, inspect the generated `AppError` and its
`cause`. Logging is provided separately by the configured logging behavior.

## Related documentation

- [Application Overview](../overview)
- [CQRS Overview](../cqrs/overview)
- [Creating Commands](../cqrs/command)
- [Creating Queries](../cqrs/query)
- [Creating Handlers](../cqrs/handler)
- [Logging Pipeline](./logging)
- [Performance Pipeline](./performance)

---

---

<a id="source-9"></a>

# Source: `docs/application/pipelines/validation.md`

## Introduction

Use the Validation Pipeline to validate commands and queries before they reach
the handler.

Xeno.JS supports two validation approaches:

- the Zod integration from `@xeno-js/zod`;
- custom validation strategies.

The pipeline is used for both commands and queries.

> **Note:** This page explains how to enable and register validation. For
> creating the Zod schema itself, see the CQRS documentation for
> [Commands](../cqrs/command) and [Queries](../cqrs/query).

## Before you start

You need:

- an `XenoAppBuilder` instance;
- a Command or Query;
- a request `intent`;
- a Zod schema registered for that intent, or a custom validation strategy.

If you are creating the request and its schema, start with:

- [Create a Command](../cqrs/command)
- [Create a Query](../cqrs/query)

## Enable validation

Add validation plugins to `XenoAppBuilder.addPipeline()`. The Zod plugin is
provided by `@xeno-js/zod`; the generic validation plugin is provided by
`@xeno-js/core`.

You do not need to register `ValidationPipeline` manually.

For Zod validation:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withZodValidation } from '@xeno-js/zod'

const app = new XenoAppBuilder()
  .addPipeline(
    withZodValidation((options) => {
      options.schemas = {}
    }),
  )
  .build()
```

For custom validation, pass strategy instances to `withValidation()`:

```ts
import { XenoAppBuilder, withValidation } from '@xeno-js/core'

const app = new XenoAppBuilder()
  .addPipeline(
    withValidation((strategies) => {
      strategies.push(validateUser)
    }),
  )
  .build()
```

The configured validation strategy is added to the common pipeline, so it
applies to both commands and queries.

## Use the Zod integration

The `@xeno-js/zod` package provides a Zod-based validator that you configure
during application bootstrap with `withZodValidation()`.

The configuration is a map where:

- the key is the request `intent`;
- the value is the corresponding Zod schema.

The key must match the value exposed as the request's `intent`.

### Register a Zod schema

For example, assume a command uses the intent:

```ts
CREATE_USER_HANDLER
```

Register its schema during bootstrap:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withZodValidation } from '@xeno-js/zod'

import { createUserSchema } from './application/users/create-user.schema'

const app = new XenoAppBuilder()
  .addPipeline(
    withZodValidation((options) => {
      options.schemas = {
        CREATE_USER_HANDLER: createUserSchema,
      }
    }),
  )
  .build()
```

The schema itself belongs with the command or query it validates.

For example:

```text
src/
├── application/
│   └── users/
│       ├── create-user.command.ts
│       ├── create-user.handler.ts
│       └── create-user.schema.ts
└── app.ts
```

The schema can be defined in `create-user.schema.ts`:

```ts
import { z } from 'zod'

export const createUserSchema = z.object({
  email: z.email(),
  name: z.string().min(2),
})
```

The details of how the schema is associated with a Command or Query are
documented in the corresponding CQRS guides:

- [Create a Command](../cqrs/command)
- [Create a Query](../cqrs/query)

### Register multiple schemas

You can register schemas for multiple commands and queries in the same bootstrap
configuration:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withZodValidation } from '@xeno-js/zod'

const app = new XenoAppBuilder()
  .addPipeline(
    withZodValidation((options) => {
      options.schemas = {
        CREATE_USER_HANDLER: createUserSchema,
        UPDATE_USER_HANDLER: updateUserSchema,
        GET_USER_HANDLER: getUserSchema,
      }
    }),
  )
  .build()
```

Each request is validated against the schema registered for its `intent`.

## Understand schema lookup

The Zod validator uses the request `intent` to find its schema.

For example:

```text
Request
  intent: CREATE_USER_HANDLER
        │
        ▼
withZodValidation({ schemas })
        │
        └── CREATE_USER_HANDLER → createUserSchema
```

This means the registration key must match the request intent exactly.

If the request has:

```ts
{
  intent: 'CREATE_USER_HANDLER',
}
```

the schema must be registered as:

```ts
schemas: {
  CREATE_USER_HANDLER: createUserSchema,
}
```

The validation strategy passes the request intent and request to the configured
validator service. The behavior for an intent with no registered schema depends
on that validator service; this specific behavior cannot be verified from the
current context.

## Add custom validation

Use `withValidation()` when the validation logic cannot or should not be
expressed as a Zod schema.

A custom validation strategy implements:

```ts
IStrategy<IRequest, boolean>
```

and exposes:

```ts
execute(request)
```

The method returns a result. Return a failed `Result` to stop pipeline
execution; the pipeline treats any successful result as validation success.

### Basic custom validator

For example:

```ts
import type { IRequest, IStrategy } from '@xeno-js/shared'
import { AppError, Result } from '@xeno-js/shared'

const validateUser: IStrategy<IRequest, boolean> = {
  async execute(request) {
    if (!('email' in request) || typeof request.email !== 'string') {
      return Result.fail(
        AppError.validationError(request.intent, 'Email is required'),
      )
    }

    return Result.ok(true)
  },
}
```

Register the strategy instance during bootstrap:

```ts
import { XenoAppBuilder, withValidation } from '@xeno-js/core'

const app = new XenoAppBuilder()
  .addPipeline(
    withValidation((strategies) => {
      strategies.push(validateUser)
    }),
  )
  .build()
```

`withValidation()` accepts strategy instances in its setup callback; the
provided plugin does not supply a service scope or create a strategy from a
scope-aware factory. The exact approach for injecting dependencies into a
strategy cannot be verified from the current context.

## Register multiple custom validators

You can register more than one custom validation strategy:

```ts
import { XenoAppBuilder, withValidation } from '@xeno-js/core'

const app = new XenoAppBuilder()
  .addPipeline(
    withValidation((strategies) => {
      strategies.push(validateUser, validateAccount, validateBusinessRules)
    }),
  )
  .build()
```

Strategies in the array passed to a single `withValidation()` call are executed
in array order.

The validation pipeline awaits each strategy in order. If a strategy returns a
failed `Result`, validation stops immediately and the remaining strategies are
not executed.

The request also does not continue to the next pipeline stage or handler.

## Combine Zod and custom validation

Zod and custom strategies can be used together.

For example:

```ts
import { XenoAppBuilder, withValidation } from '@xeno-js/core'
import { withZodValidation } from '@xeno-js/zod'

const app = new XenoAppBuilder()
  .addPipeline(
    withZodValidation((options) => {
      options.schemas = {
        CREATE_USER_HANDLER: createUserSchema,
      }
    }),
    withValidation((strategies) => {
      strategies.push(validateUser)
    }),
  )
  .build()
```

This is useful when:

- Zod validates the structure and basic constraints of the request;
- custom validation checks application-specific rules.

For example:

```text
CREATE_USER_HANDLER
        │
        ▼
 Validation strategies
        │
        ▼
     Handler
```

If any validation strategy fails, execution stops at that point. The exact
ordering between separate validation plugins depends on pipeline assembly and is
not specified by the implementation context provided here.

## What happens when validation succeeds?

When every configured validation strategy returns a successful `Result`, the
request continues through the remaining pipeline and eventually reaches its
handler.

Conceptually:

```text
Command / Query
      │
      ▼
Validation
      │
      ├── failed → Result.fail(...)
      │
      └── passed
            │
            ▼
       next pipeline
            │
            ▼
          Handler
```

The validation pipeline does not modify a successful request result. It allows
execution to continue.

## What happens when validation fails?

A failed validation returns a failed `Result`.

The next pipeline stage is not executed, and the handler is not called.

For a Zod validation failure, the schema strategy converts the validator
service's error into an `AppError.validationError`. The strategy prefixes the
message with `Validation failed:`.

The strategy prefixes the validator service's error message, but its exact
formatting cannot be verified from the current context.

## Validation for commands and queries

The same validation configuration is used for both commands and queries.

For example:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withZodValidation } from '@xeno-js/zod'

const app = new XenoAppBuilder()
  .addPipeline(
    withZodValidation((options) => {
      options.schemas = {
        CREATE_USER_HANDLER: createUserSchema,
        GET_USER_HANDLER: getUserSchema,
      }
    }),
  )
  .build()
```

Both requests can be validated through the same pipeline configuration:

```text
Command
  │
  └── Validation
        └── CREATE_USER_HANDLER → createUserSchema

Query
  │
  └── Validation
        └── GET_USER_HANDLER → getUserSchema
```

For request-specific schema definitions, see:

- [Create a Command](../cqrs/command)
- [Create a Query](../cqrs/query)

## Troubleshooting

### Validation is not running

Check the following:

1. `addPipeline()` is called on the `XenoAppBuilder`;
2. `withZodValidation()` or `withValidation()` is passed to `addPipeline()`;
3. the Zod schema is registered under the correct request `intent`;
4. the request is executed through the Xeno.JS mediator;
5. the request reaches the configured application pipeline.

### The Zod schema is never applied

Check the schema registry:

```ts
withZodValidation((options) => {
  options.schemas = {
    CREATE_USER_HANDLER: createUserSchema,
  }
})
```

The schema lookup uses the request `intent`. Check that it exactly matches the
registry key:

```ts
request.intent === 'CREATE_USER_HANDLER'
```

The two values must match exactly.

The behavior when no schema exists for an intent depends on the validator
service and cannot be verified from the current context.

### The custom validator is not running

Check that the strategy is registered through:

```ts
withValidation((strategies) => {
  strategies.push(validateUser)
})
```

Also verify that:

- the setup callback adds an `IStrategy` instance to the array;
- the strategy's `execute()` returns a successful or failed result;
- the plugin is passed to `addPipeline()` before `build()`.

### One custom validator prevents the others from running

This is expected when the previous validator returns a failed `Result`.

Validation stops at the first failure. This prevents the request from continuing
to the handler or to subsequent validation strategies.

## Related docs

- [Application Overview](../overview)
- [Create a Command](../cqrs/command)
- [Create a Query](../cqrs/query)
- [Create a Handler](../cqrs/handler)
- [Exception Pipeline](./exception)
- [Performance Pipeline](./performance)
- [Logging Pipeline](./logging)

---

# Source: `docs/application/app-error.md`

## Introduction

`AppError` is an `Error` with a machine-readable `code`, an HTTP `status`,
and application-defined context. Xeno.JS uses it as the default error type
of `ResultType<T>`.

Create an error with a named factory or with `AppError.create()`. Return it
in a failed [Result](/docs/application/result), or throw it when an exception is appropriate.
Creating an `AppError` does not log it or send an HTTP response.

```ts
import { AppError, Result, type ResultType } from '@xeno-js/shared'

interface Order {
  id: string
}

function getOrder(id: string): ResultType<Order> {
  if (id !== '42') {
    return Result.fail(AppError.notFound('getOrder', 'Order does not exist'))
  }

  return Result.ok({ id })
}
```

## Error properties

| Property | Meaning |
| --- | --- |
| `message` | Error message inherited from `Error`. Named factories use a standard message from `ERROR_CODE_MESSAGES`. |
| `code` | String identifying the error category or application condition. |
| `status` | Numeric HTTP status metadata. No response is sent when the error is created. |
| `name` | Name supplied by the caller, often identifying the operation or component. |
| `cause` | Original error or other diagnostic context. Its type is `unknown` and it can be `undefined`. |
| Additional properties | Application context copied from the payload, such as a `header` dictionary. |

For factories that accept `name` and `message`, the supplied message becomes
the message of a new `Error` stored in `cause`. It does not replace the
standard `AppError.message`.

```ts
import { AppError } from '@xeno-js/shared'

const error = AppError.notFound('GetOrder', 'Order 42 does not exist')

console.log(error.code)    // NOT_FOUND
console.log(error.status)  // 404
console.log(error.message) // errors.not_found

if (error.cause instanceof Error) {
  console.log(error.cause.message) // Order 42 does not exist
}
```

Treat `cause` and custom properties as potentially sensitive. Inspect or log
them according to your application's policy; do not assume they are safe to
send to a client.

## Named factories

Named factories return an `AppError`; they do not throw it. The table shows
the code and status each factory assigns:

| Factory | Code | Status | Cause or additional metadata |
| --- | --- | --- | --- |
| `badRequest(name, message)` | `BAD_REQUEST` | 400 | `Error(message)` |
| `validationError(name, message)` | `VALIDATION_FAILED` | 400 | `Error(message)` |
| `unauthorized(name, message)` | `UNAUTHORIZED` | 401 | `Error(message)` and a `WWW-Authenticate` Bearer header |
| `authFailed(name, message)` | `AUTHENTICATION_FAILED` | 401 | `Error(message)` |
| `forbidden(name, message)` | `FORBIDDEN` | 403 | `Error(message)` |
| `notFound(name, message)` | `NOT_FOUND` | 404 | `Error(message)` |
| `notSupported(name, message)` | `METHOD_NOT_ALLOWED` | 405 | `Error(message)` |
| `conflict(name, message)` | `CONFLICT` | 409 | `Error(message)` |
| `aborted(name)` | `ABORTED` | 499 | An `Error` describing a closed client connection |
| `internalServerError(name, err)` | `SYSTEM_ERROR` | 500 | The supplied `err`, which can be any value |

The factories use `ERROR_CODES`, `ERROR_CODE_MESSAGES`, and `STATUS_CODES`
from `@xeno-js/shared`. Choose a factory based on the condition your
application detected. For example, `unauthorized` and `authFailed` both use
status 401, but only `unauthorized` adds the Bearer challenge header.

```ts
import { AppError } from '@xeno-js/shared'

const error = AppError.conflict('CreateOrder', 'Order already exists')
```

## Create a custom error

Use `AppError.create()` when the built-in factories do not express your
condition. Supply `message`, `code`, `status`, `name`, and `cause`; the
payload type requires `cause`, although its value can be `undefined`.
Additional properties are copied onto the instance.

```ts
import { AppError } from '@xeno-js/shared'

const error = AppError.create({
  code: 'ORDER_ALREADY_SHIPPED',
  message: 'The order has already shipped.',
  status: 409,
  name: 'CancelOrder',
  cause: undefined,
  orderId: '42',
})
```

Custom properties have type `unknown` when read through `AppError`. Check
their type before using them:

```ts
if (typeof error.orderId === 'string') {
  console.log(error.orderId)
}
```

`AppError` has a private constructor. Use `create()` or a named factory
instead of `new AppError(...)`. Keep custom `message`, `code`, and any
response headers safe for their intended audience.

## Return or throw an AppError

Return an error when failure is part of the operation's result contract:

```ts
import { AppError, Result, type ResultType } from '@xeno-js/shared'

function cancelOrder(shipped: boolean): ResultType<void> {
  if (shipped) {
    return Result.fail(AppError.conflict('cancelOrder', 'Order already shipped'))
  }

  return Result.ok()
}
```

To throw, use `throw` with a returned error or call `AppError.throw()` with
the complete custom payload:

```ts
import { AppError } from '@xeno-js/shared'

throw AppError.forbidden('CancelOrder', 'The user cannot cancel this order')
```

```ts
import { AppError } from '@xeno-js/shared'

AppError.throw({
  code: 'ORDER_ALREADY_SHIPPED',
  message: 'The order has already shipped.',
  status: 409,
  name: 'CancelOrder',
  cause: undefined,
})
```

`AppError.throw()` returns `never` because it always throws. Neither form
automatically produces a failed result outside an exception handler.

## Handle cancellation

`AppError.throwIfAborted(signal, name)` accepts an `AbortSignal`, `null`, or
`undefined`. It throws `AppError.aborted(name)` only when the signal exists
and is already aborted:

```ts
import { AppError } from '@xeno-js/shared'

function prepareOrder(signal?: AbortSignal): void {
  AppError.throwIfAborted(signal, 'prepareOrder')
  // Continue with work that respects the signal.
}
```

This checks the signal at the time of the call. It does not subscribe to
later aborts or cancel work on its own. Pass the signal to the underlying
operations and check it at appropriate boundaries.

## CQRS and HTTP boundaries

The [Exception Pipeline](/docs/application/pipelines/exception) catches values thrown by the
next CQRS behavior or handler. It returns an existing `AppError` in a failed
result. Other thrown values become a `SYSTEM_ERROR` with status `500`, and
the original value is kept as `cause`.

The behavior does not itself log or send a response. A controller can map a
failed result with `this.fail(result.getErrorOrThrow())`. The helper uses
the error's code, message, and status; it does not automatically include
`cause` in the response. It also reads an optional `header` property before
merging headers supplied to `fail()`.

Only provide client-safe response details and headers. See
[API Responses](/docs/http/api-response) for the response shape, header
precedence, and handling of successful results.

## Troubleshooting

### My custom message is not in the HTTP response

Named factories store their `message` argument in `cause`. The standard
`AppError.message` is used by `this.fail()`, which does not expose `cause`
automatically. Pass client-safe details explicitly when needed.

### My error became `SYSTEM_ERROR`

The CQRS exception behavior preserves actual `AppError` instances. Other
thrown values, including plain objects with similar properties, are wrapped
as system errors. Use a factory or `AppError.create()` for an application
error that must retain its code.

### My aborted operation continued

`throwIfAborted()` checks the current state once. Ensure the underlying work
receives the signal or check it again before later work.

---

# Source: `docs/application/result.md`

## Introduction

`Result<TValue, TError>` represents the outcome of an operation. A result is
either successful, with an optional value, or failed, with an error. Callers
can inspect the outcome and decide how to handle it.

`ResultType<T, E>` is an alias for `Result<T, E>` whose error type defaults
to `AppError`. Xeno.JS uses it in contracts such as CQRS handlers and
repositories. Both types and the `Result` class are available from
`@xeno-js/shared`.

```ts
import { AppError, Result, type ResultType } from '@xeno-js/shared'

interface User {
  id: string
}

function findUser(id: string): ResultType<User> {
  if (id !== '42') {
    return Result.fail(AppError.notFound('findUser', 'User does not exist'))
  }

  return Result.ok({ id })
}
```

Creating a failed result does not throw. The caller receives the `AppError`
and can propagate it, inspect it, or map it at a boundary.

## Create a successful result

Use `Result.ok(value)` when an operation succeeds:

```ts
import { Result } from '@xeno-js/shared'

const result = Result.ok({ id: '42', name: 'Ada' })
```

The value is optional. For an operation without a return value, use
`Result.ok()`:

```ts
import { Result, type ResultType } from '@xeno-js/shared'

function complete(): ResultType<void> {
  return Result.ok()
}
```

A successful result can also contain `undefined` when its declared value
type is not `void`: `Result.ok<T>()` accepts no value. `isOk()` confirms the
outcome, not the presence of a value. `null`, `false`, `0`, and an empty
string are values and do not indicate failure.

## Create a failed result

Pass an error to `Result.fail(error)`:

```ts
import { AppError, Result, type ResultType } from '@xeno-js/shared'

function reserve(quantity: number): ResultType<number> {
  if (quantity <= 0) {
    return Result.fail(
      AppError.validationError('reserve', 'Quantity must be positive'),
    )
  }

  return Result.ok(quantity)
}
```

`Result` does not require the error to be an `AppError`. Set a different
error type where your contract needs one:

```ts
import { Result, type ResultType } from '@xeno-js/shared'

type ParseError = { code: 'INVALID_NUMBER' }

function parseCount(input: string): ResultType<number, ParseError> {
  const count = Number(input)
  if (!Number.isInteger(count)) {
    return Result.fail({ code: 'INVALID_NUMBER' })
  }

  return Result.ok(count)
}
```

When using explicit generic arguments on `Result.fail`, the first argument
is the success type and the second is the error type:
`Result.fail<number, ParseError>({ code: 'INVALID_NUMBER' })`.

## Inspect an outcome

Check `isOk()` before reading the matching side of the result.
`getValueOrThrow()` throws a new `Error` on failure;
`getErrorOrThrow()` throws a new `Error` on success.

```ts
const result = findUser('42')

if (!result.isOk()) {
  const error = result.getErrorOrThrow()
  console.error(error.code, error.message)
} else {
  const user = result.getValueOrThrow()
  if (user === undefined) {
    throw new Error('findUser succeeded without a user')
  }

  console.log(user.id)
}
```

This example uses `findUser` from the introduction. The check for
`undefined` is appropriate because that operation promises a user when it
succeeds. Other operations may intentionally return no value.

`isOk()` returns a boolean rather than a TypeScript type predicate. It does
not narrow the result into separate success and failure types, and
`getValueOrThrow()` always returns `TValue | undefined`. If it is called on
a failed result, it throws `Cannot get the value of a failed result.`, not
the stored error. To obtain the original error, call `getErrorOrThrow()`.

## Propagate a failure

When one operation depends on another, return its error before continuing:

```ts
import { Result, type ResultType } from '@xeno-js/shared'

interface User {
  id: string
}

function loadUser(id: string): ResultType<User> {
  // Application-specific implementation.
  return Result.ok({ id })
}

function getUserId(id: string): ResultType<string> {
  const userResult = loadUser(id)

  if (!userResult.isOk()) {
    return Result.fail(userResult.getErrorOrThrow())
  }

  const user = userResult.getValueOrThrow()
  if (user === undefined) {
    throw new Error('loadUser succeeded without a user')
  }

  return Result.ok(user.id)
}
```

This creates another failed `Result` containing the same error. It does not
wrap or log the error. The caller still needs to decide where to handle it.

## Result types and defaults

| Type | Default error type | Use |
| --- | --- | --- |
| `Result<T, E>` | `never` | Class type with an explicitly chosen error type when needed. |
| `ResultType<T, E>` | `AppError` | Alias used by Xeno.JS application contracts. |

`ResultType<T, E>` is an alias, not a value extractor or a separate runtime
class. `ResultType<User>` means `Result<User, AppError>`.
`Result.ok<T>()` returns `Result<T>` with the class's `never` error default;
`Result.fail<T, E>()` returns `Result<T, E>`.

## Results, exceptions, and HTTP

Returning `Result.fail(error)` makes failure part of the return value.
Throwing interrupts execution until a caller catches the exception. `Result`
does not catch exceptions from the function that creates it.

With CQRS configured, the [Exception Pipeline](/docs/application/pipelines/exception)
converts exceptions thrown by the next behavior or handler into failed
results. It preserves an existing `AppError` and wraps other thrown values.
This conversion applies within that pipeline, not to arbitrary code.

A result is not an HTTP response. A controller can inspect a failed
`ResultType` and pass its error to `this.fail()`. A successful value can be
passed to `this.ok()` after deciding whether `undefined` is valid for the
operation. See [API Responses](/docs/http/api-response) for the mapping and
[AppError](/docs/application/app-error) for error creation.

## Troubleshooting

### A successful result has no value

`Result.ok()` and `Result.ok<T>()` can both succeed without a value. Check
for `undefined` when the operation requires data; for `void` or optional
data, handle the absence according to the contract.

### Reading the result throws

Call `isOk()` first. Use `getValueOrThrow()` on success and
`getErrorOrThrow()` on failure. The accessor errors indicate that the wrong
side was read; they are not the error stored in the result.

### An exception did not become a failed result

`Result` does not wrap exceptions automatically. Check whether the throwing
code ran inside the configured CQRS exception behavior. See
[Exception Pipeline](/docs/application/pipelines/exception) for its scope.

---
