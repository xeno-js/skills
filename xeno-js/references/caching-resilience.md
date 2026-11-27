# Caching and resilience

Technical reference assembled from the website documentation. Each section
identifies its original Markdown source. Website frontmatter and the repeated
sponsorship footer are omitted; body content and fenced examples are retained.

---

## Contents

- [`docs/application/pipelines/cache-invalidation.md`](#source-1)
- [`docs/application/pipelines/caching.md`](#source-2)
- [`docs/application/pipelines/concurrency.md`](#source-3)
- [`docs/application/pipelines/idempotency.md`](#source-4)
- [`docs/application/pipelines/performance.md`](#source-5)
- [`docs/cache/overview.md`](#source-6)
- [`docs/cache/redis.md`](#source-7)

<a id="source-1"></a>

# Source: `docs/application/pipelines/cache-invalidation.md`

## What is command cache invalidation?

The command cache invalidation pipeline removes cached entries after a
successful command. A command declares exact `keys`, `patterns`, or both in
`cacheInvalidation`. This lets a write invalidate query results that depend on
the changed data.

The command result and the cache update are separate operations. Invalidation is
best effort, so it does not guarantee that every later read is fresh.

## Enable the pipeline

Pass `withQueryCaching()` to `addPipeline()`. The CQRS module adds both query
caching and command invalidation; there is no separate invalidation plugin.

```ts
import { XenoAppBuilder, withQueryCaching } from '@xeno-js/core'

const application = await new XenoAppBuilder()
  .addPipeline(withQueryCaching())
  .build()
```

If no cache is registered, Xeno.JS configures the default in-memory cache. Use
`addCache()` to select or configure a provider, such as
[Redis](/docs/cache/redis). Calling `addCache()` alone does not enable these
pipeline behaviors.

## Invalidate a query by key

Query caching transforms the logical `cacheKey` into a contextual storage key.
Build the command's exact invalidation key with the same `ICacheKeyBuilder` and
identity context used by the query:

```ts
import { Command, Query } from '@xeno-js/core'
import type { ICacheKeyBuilder } from '@xeno-js/core'

interface User {
  id: string
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

Register handlers under `GetUserQuery` and `UpdateUserCommand`, and execute them
through the mediator. Resolve the key builder while the relevant identity is
active. This example creates a new scope with the default guest identity:

```ts
import { TOKENS } from '@xeno-js/core'

// application is built with both handlers registered.
await application.runInScope(async (services) => {
  const keyBuilder = services[TOKENS.CACHE_KEY_BUILDER]
  const mediator = services[TOKENS.MEDIATOR]
  const command = new UpdateUserCommand('42', 'Ada', keyBuilder)
  const result = await mediator.send(command, new AbortController().signal)

  if (!result.isOk()) {
    throw result.getErrorOrThrow()
  }
})
```

The example invalidates the public entry for the default guest. `runInScope()`
creates a new context and uses the guest identity unless an explicit context is
passed as its second argument. Inside an existing request, resolve the builder
from the active scope instead. The identity used to build the command key must
match the identity used to cache the query, including any changes made by
identity enrichment. The command handler must complete the write before
returning a successful result. For handler registration and scopes, see
[Creating Commands](/docs/application/cqrs/command).

## Match the stored cache key

`QueryCachingPipeline` applies the configured key builder to
`cacheOptions.cacheKey`. `CommandCacheInvalidationBehavior` does not apply the
key builder: it passes each declared string directly to the cache.

With the built-in key builder, the logical key `user:42` produces:

| Query scope | Identity                | Stored key                   |
| ----------- | ----------------------- | ---------------------------- |
| Contextual  | No tenant               | `user:42:public`             |
| Contextual  | Tenant `acme`           | `tenant:acme:user:42`        |
| User-scoped | User `7`, no tenant     | `user:7:user:42:public`      |
| User-scoped | User `7`, tenant `acme` | `tenant:acme:user:7:user:42` |

Use `buildContextualKey()` for a query with `isUserScoped: false` and
`buildUserScopedKey()` for a query with `isUserScoped: true`. If the identity
has no user ID, `buildUserScopedKey()` falls back to the contextual key. A
custom key builder can use a different format.

An exact key for the current identity does not invalidate entries for other
users or tenants. Identify every affected cached view, including lists and other
user-scoped entries, and choose a strategy that respects tenant boundaries. Do
not accept arbitrary invalidation keys or patterns from external callers.

## Invalidate matching keys

`patterns` are passed unchanged to the cache provider's `removePattern()`
method:

```ts
super('UpdateUserCommand', {
  keys: ['tenant:acme:user:42'],
  patterns: ['tenant:acme:users:page:*'],
})
```

This example targets the `acme` tenant and assumes that paginated list entries
actually use the shown stored-key prefix. Adapt the pattern to the key builder
and provider in use.

The built-in in-memory cache supports `*` as a wildcard and treats other regex
metacharacters literally. Redis uses its own glob matching and the current Redis
provider calls `KEYS` followed by `DEL`; broad patterns can be expensive on
large databases. Check custom provider semantics and restrict patterns to the
intended namespace. Avoid building patterns from untrusted input.

Although `ICache` declares `removePattern()`, the pipeline checks for a callable
method at runtime and silently skips patterns if it is absent. Exact keys are
still processed.

## Execution and failures

For a command that reaches this behavior:

1. The behavior awaits the next command pipeline step.
2. A failed result or missing `cacheInvalidation` skips removal. If the
   downstream step throws, removal is not attempted.
3. On success, exact keys are removed sequentially, followed by patterns.
4. A removal error stops the remaining operations and logs
   `[Cache Invalidation ERROR]` at warning level. Earlier removals remain
   applied, and the original successful result is returned.

Successful removals are logged at debug level. Check the configured logger's
level if those messages are not visible.

The behavior is appended after the other sorted command behaviors. A preceding
behavior can short-circuit without calling it. For example, an idempotency
replay can return a stored result without running the handler or invalidation
again.

## Consistency and common problems

- **A successful result does not prove that invalidation completed.** Monitor
  cache warnings and use an appropriate TTL or a separate recovery mechanism
  where stale data matters.
- **Invalidation is not atomic with the database write.** Ensure the write has
  committed before returning success. A concurrent query can also repopulate
  stale data after a removal; use a stronger application-specific strategy when
  required.
- **Logical keys are not necessarily storage keys.** Match the tenant, user, and
  public components shown above, or use the configured key builder.
- **A direct handler call bypasses the mediator pipeline.** Send the command
  through the mediator and enable `withQueryCaching()`.
- **In-memory caches are local to the cache instance.** Separate application
  instances need a shared cache or another coordination mechanism if they must
  invalidate each other's entries.

For cache reads, TTLs, and bypass options, see
[Caching Pipeline](/docs/application/pipelines/caching).

---

---

<a id="source-2"></a>

# Source: `docs/application/pipelines/caching.md`

## Introduction

Use the Caching Pipeline to cache the results of **Queries** in Xeno.JS.

The built-in caching behavior:

1. checks whether the query has cache options;
2. checks the cache when caching is not bypassed;
3. returns the cached value when one is available;
4. executes the query when there is no cached value;
5. stores successful query results using the configured TTL.

Query result caching runs in the **Query pipeline**. When `withQueryCaching()`
is enabled, Xeno.JS also adds a command pipeline behavior that invalidates cache
entries declared by successful commands. The invalidation keys must match the
stored cache keys, including their context. See
[Command Cache Invalidation](/docs/application/pipelines/cache-invalidation) for
key construction, patterns, and failure handling.

If you first need to create a Query, see [Create a Query](../cqrs/query).

## Before you start

You need:

- an `XenoAppBuilder` instance;
- a cache, which Xeno.JS configures in-memory by default when query caching is
  enabled;
- a Query;
- a Query Handler;
- the Query executed through the Xeno.JS mediator.

The cache can use the built-in in-memory implementation or Redis. Configure a
cache explicitly only when you need to choose or customize its provider.

For the complete Query creation workflow, including how to extend `Query` and
define its parameters, see [Create a Query](../cqrs/query).

## Enable the cache

To use the default in-memory cache, configure it through
`XenoAppBuilder.addCache()`:

For example, to use the built-in in-memory cache:

```ts
import { XenoAppBuilder } from '@xeno-js/core'

const app = new XenoAppBuilder().addCache().build()
```

The default `XenoAppBuilder` cache configuration uses the in-memory cache.

You can also explicitly set its in-memory option:

```ts
import { XenoAppBuilder } from '@xeno-js/core'

const app = new XenoAppBuilder()
  .addCache((options) => {
    options.inMemory = true
  })
  .build()
```

To use Redis, pass the `withRedis()` plugin to `addCache()`:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withRedis } from '@xeno-js/redis'

const app = new XenoAppBuilder().addCache(withRedis(() => {})).build()
```

See [Redis Cache](../../cache/redis) for Redis configuration options.

## Enable query caching

Enable query caching by passing `withQueryCaching()` to `addPipeline()`:

```ts
import { XenoAppBuilder, withQueryCaching } from '@xeno-js/core'

const app = new XenoAppBuilder().addPipeline(withQueryCaching()).build()
```

This plugin enables the query-specific caching behavior. If no cache has already
been registered, Xeno.JS automatically configures the default in-memory cache.

The two configuration steps have different responsibilities when you configure
both:

- `addCache()` configures or selects the cache implementation;
- `addPipeline(withQueryCaching())` enables query caching.

Calling `addCache()` is optional when the default in-memory cache is sufficient.

## Create a cacheable Query

The caching pipeline reads caching information from the Query's `cacheOptions`.

A Query can be created like this:

```ts
import { Query } from '@xeno-js/shared'

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

The Query itself is covered in detail in [Create a Query](../cqrs/query).

That page should be your starting point when you need to:

- create a Query class;
- define Query parameters;
- create a Query Handler;
- connect the Query to its handler;
- add validation to the Query.

This page focuses only on the caching options used by the Query.

## Configure the cache key

Use `cacheKey` to identify the cached result.

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

The cache key should represent the input that determines the query result.

For example, a Query that retrieves a user by ID can use:

```ts
cacheKey: `user:${userId}`
```

A Query that retrieves a paginated list should include the parameters that
affect the result:

```ts
cacheKey: `users:${page}:${pageSize}`
```

If the `cacheKey` is empty or missing, the caching behavior is skipped and the
Query executes normally.

## Configure the TTL

Use `ttl` to specify the lifetime of the cached result:

```ts
cacheOptions: {
  cacheKey: `user:${userId}`,
  ttl: 60,
  bypassCache: false,
  consistentRead: false,
  isUserScoped: false,
}
```

The value is a TTL in seconds and is passed to the configured cache when the
successful Query result is stored.

Choose a TTL according to how long the cached result can safely remain available
for your application.

## Use user-scoped caching

Set `isUserScoped` to `true` when the cached result must be isolated per user.
The active identity must provide a `userId`; otherwise, the key builder falls
back to the contextual key:

```ts
super('GetCurrentUserQuery', {
  cacheKey: 'current-user',
  ttl: 60,
  bypassCache: false,
  consistentRead: false,
  isUserScoped: true,
})
```

This is useful for Queries whose result depends on the current authenticated
user.

For example:

```ts
export class GetCurrentUserQuery extends Query<User> {
  constructor() {
    super('GetCurrentUserQuery', {
      cacheKey: 'current-user',
      ttl: 60,
      bypassCache: false,
      consistentRead: false,
      isUserScoped: true,
    })
  }
}
```

Use user-scoped caching when the same logical Query can return different data
for different users. The key builder includes the current user ID when available
and applies the tenant context as well.

## Bypass the cache

Set `bypassCache` to `true` when you want to skip reading an existing cached
value:

```ts
super('GetUserQuery', {
  cacheKey: `user:${userId}`,
  ttl: 60,
  bypassCache: true,
  consistentRead: false,
  isUserScoped: false,
})
```

When `bypassCache` is enabled:

1. Xeno.JS does not read the cached value;
2. the Query executes normally;
3. if the Query succeeds, its result is still stored in the cache.

For example:

```text
Query
  │
  ▼
bypassCache = true
  │
  ├── skip cache read
  │
  ▼
Query Handler
  │
  ▼
successful result
  │
  ▼
store result in cache
```

This skips the cached value and executes the Query handler; it does not change
the consistency guarantees of the underlying data source. A successful result is
still written to the cache.

## Use consistent reads

Set `consistentRead` to `true` to skip the cached value and execute the Query
handler. This option bypasses the cache; it does not change the consistency
guarantees of the underlying data source:

```ts
super('GetUserQuery', {
  cacheKey: `user:${userId}`,
  ttl: 60,
  bypassCache: false,
  consistentRead: true,
  isUserScoped: false,
})
```

When `consistentRead` is `true`, Xeno.JS skips the cache read.

As with `bypassCache`, a successful result is still written to the cache.

Therefore:

```text
consistentRead = true
        │
        ▼
   skip cache read
        │
        ▼
   execute Query
        │
        ▼
   successful result
        │
        ▼
   update cache
```

## What happens on a cache hit?

When caching is enabled and a cached value exists for the Query's cache key,
Xeno.JS returns that value immediately.

The Query Handler is not executed.

For example:

```text
Query
  │
  ▼
Cache lookup
  │
  ├── HIT ──► return cached result
  │
  └── MISS
        │
        ▼
   Query Handler
```

This means the handler is not called again when a valid cached result is
available.

## What happens on a cache miss?

When there is no cached value, Xeno.JS executes the Query normally.

If the Query succeeds, the result is stored in the cache using the configured
`cacheKey` and `ttl`.

For example:

```text
Query
  │
  ▼
Cache lookup
  │
  └── MISS
       │
       ▼
  Query Handler
       │
       ▼
  successful result
       │
       ▼
  store in cache
       │
       ▼
  return result
```

The same Query can then use the cached result on subsequent executions.

## Are failed Query results cached?

No.

The caching pipeline stores the result only when the Query returns a successful
`Result`.

If the Query fails:

```ts
return Result.fail(error)
```

the result is returned to the caller but is not written to the cache.

This prevents a failed execution from becoming the cached response for
subsequent requests.

## What happens when the cache read fails?

A cache read failure does not fail the Query.

For example, if the cache service is temporarily unavailable:

```text
Query
  │
  ▼
Cache read
  │
  └── error
       │
       ▼
   log warning
       │
       ▼
   execute Query
```

Xeno.JS logs a warning and continues with normal Query execution.

This allows the application to continue serving the Query even when the cache is
unavailable.

## What happens when the cache write fails?

A cache write failure also does not fail an otherwise successful Query.

The Query result is returned normally and Xeno.JS logs a warning about the cache
error.

For example:

```text
Query Handler
     │
     ▼
successful result
     │
     ├── cache write succeeds ──► return result
     │
     └── cache write fails
              │
              ▼
          log warning
              │
              ▼
          return result
```

The cache is therefore an optimization for Query execution rather than a
requirement for returning a successful Query result.

## Complete example

The following example shows the relevant application configuration and a
cacheable Query.

### Application configuration

```ts
import { XenoAppBuilder, withQueryCaching } from '@xeno-js/core'

const app = new XenoAppBuilder()
  .addCache()
  .addPipeline(withQueryCaching())
  .build()
```

### Query

```ts
import { Query } from '@xeno-js/shared'

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

### Execute the Query

Execute the Query through the mediator as you normally would:

```ts
const result = await mediator.query(new GetUserQuery(userId), signal)
```

The first successful execution populates the cache.

Subsequent executions using the same cache key can return the cached result
without executing the Query Handler.

For the complete Query implementation and Handler registration workflow, see
[Create a Query](../cqrs/query) and [Create a Query Handler](../cqrs/handler).

## Troubleshooting

### Query caching is not running

Check:

1. `withQueryCaching()` is passed to `addPipeline()`;
2. the Query defines a non-empty `cacheKey`;
3. the Query is executed through the Xeno.JS mediator.

For example:

```ts
import { XenoAppBuilder, withQueryCaching } from '@xeno-js/core'

const app = new XenoAppBuilder().addPipeline(withQueryCaching()).build()
```

### The Query Handler still runs every time

Check:

1. the Query has a `cacheKey`;
2. the cache key is stable for the same Query parameters;
3. `bypassCache` is not `true`;
4. `consistentRead` is not `true`;
5. the configured TTL has not expired.

Also make sure that the cache key includes every parameter that can change the
Query result.

### Different users receive the same cached result

If the Query result depends on the current user, use:

```ts
isUserScoped: true
```

The current identity must contain a distinct `userId` for each user. Without it,
the key builder falls back to the contextual key.

For example:

```ts
super('GetCurrentUserQuery', {
  cacheKey: 'current-user',
  ttl: 60,
  bypassCache: false,
  consistentRead: false,
  isUserScoped: true,
})
```

### The application fails while enabling query caching

Make sure `withQueryCaching()` is passed to `addPipeline()`. If no cache is
already registered, Xeno.JS configures the default in-memory cache
automatically.

For example:

```ts
import { XenoAppBuilder, withQueryCaching } from '@xeno-js/core'

const app = new XenoAppBuilder().addPipeline(withQueryCaching()).build()
```

Use `addCache()` only when you need to select or customize a cache provider.

### Cache errors appear in the logs

Cache read and write failures are logged as warnings.

The Query itself should continue executing when a cache operation fails.

Check the cache configuration if these warnings persist, especially when using
Redis.

## Related documentation

- [Application Overview](../overview)
- [Create a Query](../cqrs/query)
- [Create a Query Handler](../cqrs/handler)
- [Enable Validation](./validation)
- [Enable Performance Tracking](./performance)
- [Configure Idempotency](./idempotency)
- [Configure Concurrency](./concurrency)

---

---

<a id="source-3"></a>

# Source: `docs/application/pipelines/concurrency.md`

## Introduction

Use the Concurrency Pipeline to automatically retry a **Command** when its
execution returns a concurrency conflict.

The pipeline is useful with optimistic concurrency, where two operations can
attempt to update the same resource and one operation may fail because the
resource version has changed.

The concurrency retry configuration belongs to the **command bus**. It does not
apply to queries.

## Before you start

You need:

- an `XenoAppBuilder` instance;
- a Command and its handler;
- concurrency conflicts represented as a failed `Result` with a conflict error;
- the command executed through the Xeno.JS mediator.

If you are creating the Command itself, see [Create a Command](../cqrs/command).

## Enable concurrency retries

Pass `withConcurrency()` to `addPipeline()` and configure its options in the
callback:

```ts
import { XenoAppBuilder, withConcurrency } from '@xeno-js/core'

const app = new XenoAppBuilder()
  .addPipeline(
    withConcurrency((options) => {
      options.maxRetries = 3
      options.delayConfig = {
        baseDelayMs: 100,
        maxJitterMs: 50,
      }
    }),
  )
  .build()
```

Once configured, Xeno.JS adds the concurrency retry behavior to the **Command
pipeline**.

You do not need to register `ConcurrencyRetryPipeline` manually.

## Configure the retry limit

Use `maxRetries` to control the maximum number of command pipeline attempts,
including the first execution.

For example:

```ts
withConcurrency((options) => {
  options.maxRetries = 3
  options.delayConfig = {
    baseDelayMs: 100,
    maxJitterMs: 50,
  }
})
```

With `maxRetries: 3`, a command can be executed up to three times when it keeps
returning a concurrency conflict:

```text
Attempt 1
   │
   ├── success ───────────────► return result
   │
   └── conflict
         │
         ▼
       delay
         │
         ▼
Attempt 2
   │
   ├── success ───────────────► return result
   │
   └── conflict
         │
         ▼
       delay
         │
         ▼
Attempt 3
   │
   ├── success ───────────────► return result
   │
   └── conflict ───────────────► return failed Result
```

Use a positive integer for `maxRetries`. The retry pipeline does not validate
this value.

## Configure retry delays

Use `delayConfig` to configure the delay applied between concurrency-conflict
attempts. The pipeline waits for `baseDelayMs` plus a random jitter from zero up
to (but not including) `maxJitterMs`; it does not increase the base delay on
later attempts.

```ts
withConcurrency((options) => {
  options.maxRetries = 5
  options.delayConfig = {
    baseDelayMs: 100,
    maxJitterMs: 50,
  }
})
```

### `baseDelayMs`

`baseDelayMs` defines the fixed base delay in milliseconds before another
attempt. Use a non-negative integer; the retry pipeline does not validate this
value.

```ts
baseDelayMs: 100
```

### `maxJitterMs`

`maxJitterMs` defines the upper bound, in milliseconds, for the random jitter
added to the base delay. Use a non-negative integer; the retry pipeline does not
validate this value.

```ts
maxJitterMs: 50
```

Using jitter helps avoid multiple concurrent operations retrying at exactly the
same time.

## Configure only what you need

The concurrency configuration is optional.

You can enable it with the defaults:

```ts
const app = new XenoAppBuilder().addPipeline(withConcurrency()).build()
```

When a value is omitted, the pipeline defaults are used: `maxRetries` is `3`,
`baseDelayMs` is `20`, and `maxJitterMs` is `30`.

You can override only the retry count:

```ts
const app = new XenoAppBuilder()
  .addPipeline(
    withConcurrency((options) => {
      options.maxRetries = 5
    }),
  )
  .build()
```

Or provide the complete configuration:

```ts
const app = new XenoAppBuilder()
  .addPipeline(
    withConcurrency((options) => {
      options.maxRetries = 5
      options.delayConfig = {
        baseDelayMs: 100,
        maxJitterMs: 50,
      }
    }),
  )
  .build()
```

## Configure retries per intent

Use `intentRegistry` to override retry settings for specific command intents.
The registry key must match the command's `intent`. Settings defined for an
intent take precedence over the corresponding global options; any omitted fields
fall back to the global configuration and then to the pipeline defaults.

For example, set global options and give selected commands their own retry
limits and delays:

```ts
const app = new XenoAppBuilder()
  .addPipeline(
    withConcurrency((options) => {
      options.maxRetries = 3
      options.delayConfig = {
        baseDelayMs: 100,
        maxJitterMs: 50,
      }
      options.intentRegistry = {
        UpdateUserCommand: {
          maxRetries: 5,
        },
        SubmitPaymentCommand: {
          maxRetries: 2,
          delayConfig: {
            baseDelayMs: 250,
            maxJitterMs: 100,
          },
        },
      }
    }),
  )
  .build()
```

In this example, `UpdateUserCommand` uses five attempts with the global delay
settings. `SubmitPaymentCommand` uses two attempts and its own delay settings.
Commands not listed in `intentRegistry` use the global options.

## What counts as a concurrency conflict?

The retry pipeline does not retry every failed command.

A result is retried only when its error is a conflict with:

- the `CONFLICT` error code;
- HTTP status `409`.

For example, a command handler can return a conflict result when an optimistic
concurrency check detects that the entity was modified by another operation.

```ts
return Result.fail(
  AppError.conflict(
    request.intent,
    'The user was modified by another request.',
  ),
)
```

That failed result is eligible for the concurrency retry pipeline.

Other failures are returned immediately.

For example:

```text
Command
   │
   ▼
Handler
   │
   ├── success ───────────────► return result
   │
   ├── validation error ──────► return error
   │
   ├── authorization error ───► return error
   │
   └── conflict (409) ────────► retry
```

## What happens during a retry?

When a command returns a concurrency conflict:

1. Xeno.JS checks whether the error is a conflict;
2. if the retry limit has not been reached, it waits using the configured delay
   and jitter;
3. the command pipeline is executed again;
4. the new result is evaluated again.

A successful retry immediately returns the successful result.

For example:

```text
Command
   │
   ▼
Conflict
   │
   ▼
wait
   │
   ▼
Command again
   │
   ▼
Success
   │
   ▼
Return success
```

A non-concurrency error is not retried.

## What happens when all attempts fail?

If every allowed attempt returns a concurrency conflict, the pipeline stops
retrying and returns a failed conflict `Result`.

The returned error indicates that the maximum retry attempts were exceeded.

For example:

```text
Maximum retry attempts (3) exceeded due to concurrency conflicts.
```

The command is not executed again after the limit has been reached.

## Complete example

A typical application can configure the command bus as follows:

```ts
import { XenoAppBuilder, withConcurrency } from '@xeno-js/core'

const app = new XenoAppBuilder()
  .addPipeline(
    withConcurrency((options) => {
      options.maxRetries = 3
      options.delayConfig = {
        baseDelayMs: 100,
        maxJitterMs: 50,
      }
    }),
  )
  .build()
```

Your command handler is responsible for returning a conflict when an optimistic
concurrency check fails:

```ts
import { AppError, Result } from '@xeno-js/shared'

export class UpdateUserHandler extends BaseHandler<
  UpdateUserCommand,
  UpdateUserResult
> {
  protected async executeAsync(
    request: UpdateUserCommand,
  ): Promise<Result<UpdateUserResult>> {
    const user = await this.userRepository.findById(request.userId)

    if (!user) {
      return Result.fail(AppError.notFound(request.intent, 'User not found.'))
    }

    if (user.version !== request.version) {
      return Result.fail(
        AppError.conflict(
          request.intent,
          'The user was modified by another request.',
        ),
      )
    }

    // Update the entity and persist it.

    return Result.ok({
      userId: user.id,
    })
  }
}
```

With concurrency retries enabled, a conflict can cause the command to be
executed again according to the configured retry policy.

## Concurrency applies to Commands

The retry options are provided to:

```ts
withConcurrency((options) => {
  options.maxRetries = 3
})
```

The plugin adds the retry behavior to the Command pipeline; it does not add
retry behavior to the Query pipeline.

The application pipeline is conceptually:

```text
Command
   │
   ▼
Common pipelines
   │
   ▼
Command-specific pipelines
   │
   └── Concurrency Retry
          │
          ▼
       Handler
```

If you need query result caching or other query-specific behavior, see the
corresponding query pipeline documentation.

## Other errors are not retried

The retry pipeline retries only failed results whose error code is `CONFLICT`
and whose HTTP status is `409`. Other errors are returned without retrying.

For example:

```text
Non-conflict error
      │
      └──► return immediately

Conflict / 409
      │
      └──► retry
```

Increasing `maxRetries` does not cause errors that fail either conflict check to
be retried.

## Troubleshooting

### The command is not being retried

Check that `withConcurrency()` is passed to `addPipeline()`:

```ts
import { withConcurrency } from '@xeno-js/core'

app.addPipeline(
  withConcurrency((options) => {
    options.maxRetries = 3
  }),
)
```

Then check that the handler returns a conflict error.

The retry pipeline only recognizes errors with:

- conflict error code;
- HTTP status `409`.

A generic failed `Result` will not trigger a retry.

### A failed command is not retried

Check the error returned by the handler.

For example, this is a concurrency conflict:

```ts
return Result.fail(
  AppError.conflict(request.intent, 'Concurrent update detected.'),
)
```

Whereas returning another error type will not activate the retry behavior.

### `maxRetries` has an unexpected value

Use a positive integer for `maxRetries`. The retry pipeline does not validate
this value.

Not recommended:

```ts
maxRetries: 0
```

Not recommended:

```ts
maxRetries: -1
```

Recommended:

```ts
maxRetries: 3
```

### `baseDelayMs` or `maxJitterMs` has an unexpected value

Use non-negative integers for `baseDelayMs` and `maxJitterMs`. The retry
pipeline does not validate these values.

Not recommended:

```ts
delayConfig: {
  baseDelayMs: -100,
  maxJitterMs: 50,
}
```

Recommended:

```ts
delayConfig: {
  baseDelayMs: 100,
  maxJitterMs: 50,
}
```

## Related docs

- [Application Overview](../overview)
- [Create a Command](../cqrs/command)
- [Create a Handler](../cqrs/handler)
- [Exception Pipeline](./exception)
- [Validation Pipeline](./validation)
- [Idempotency Pipeline](./idempotency)
- [Caching](./caching)

---

---

<a id="source-4"></a>

# Source: `docs/application/pipelines/idempotency.md`

## Introduction

Use the Idempotency Pipeline to deduplicate repeated **Command** executions that
use the same request ID while the corresponding cache entry is available.

When idempotency is enabled, Xeno.JS:

1. checks whether the request has already been processed;
2. returns the stored payload in a successful `Result` when one is available;
3. otherwise acquires an idempotency lock;
4. executes the command;
5. stores the successful result payload for subsequent requests;
6. releases the lock when the command fails or throws an exception.

Idempotency is available for **Commands only**. It is not added to the Query
pipeline.

## Before you start

You need:

- an `XenoAppBuilder` instance;
- a Command and its handler;
- a cache implementation; the CQRS module configures the default in-memory cache
  if needed;
- a network request context that provides a `requestId`;
- the command executed through the Xeno.JS mediator.

If you are creating the Command itself, see [Create a Command](../cqrs/command).

If you need to configure the cache used by Xeno.JS, see the cache documentation.

## Enable idempotency

Pass `withIdempotency()` to `addPipeline()` and configure its options:

```ts
import { XenoAppBuilder, withIdempotency } from '@xeno-js/core'

const app = new XenoAppBuilder().addPipeline(
  withIdempotency((options) => {
    options.lockTtlSeconds = 30
    options.processedTtlSeconds = 60
  }),
)

const application = await app.build()
```

You do not need to instantiate or register `IdempotencyPipeline` manually.

Once configured, Xeno.JS adds the idempotency behavior to the Command pipeline.

## Configure the lock TTL

Use `lockTtlSeconds` to control how long an idempotency lock remains in the
cache.

```ts
import { XenoAppBuilder, withIdempotency } from '@xeno-js/core'

const app = new XenoAppBuilder().addPipeline(
  withIdempotency((options) => {
    options.lockTtlSeconds = 30
    options.processedTtlSeconds = 60
  }),
)

const application = await app.build()
```

The value is expressed in **seconds**.

Use a positive integer value.

For example:

```ts
lockTtlSeconds: 30
```

means that the lock can remain in the cache for up to 30 seconds if it is not
explicitly released earlier.

The lock is released by the pipeline when:

- the command returns a failed `Result`;
- the next behavior or command handler throws.

A successful command is marked as processed instead of explicitly releasing the
lock.

## Configure the processed-result TTL

Use `processedTtlSeconds` to control how long the result of a successfully
processed command remains available.

```ts
import { XenoAppBuilder, withIdempotency } from '@xeno-js/core'

const app = new XenoAppBuilder().addPipeline(
  withIdempotency((options) => {
    options.lockTtlSeconds = 30
    options.processedTtlSeconds = 300
  }),
)

const application = await app.build()
```

The value is expressed in **seconds**. Use a positive integer.

While the processed payload is still available, another request with the same
`requestId` receives that payload in a successful `Result` instead of executing
the command again.

## Configure idempotency per intent

Use `intentRegistry` to override TTL settings for specific command intents. A
registry key must match the command's `intent`. Each per-intent TTL overrides
the corresponding global setting; omitted values fall back to the global setting
and then to the pipeline default.

```ts
import { XenoAppBuilder, withIdempotency } from '@xeno-js/core'

const app = new XenoAppBuilder().addPipeline(
  withIdempotency((options) => {
    options.lockTtlSeconds = 30
    options.processedTtlSeconds = 300
    options.intentRegistry = {
      CreateUserCommand: {
        processedTtlSeconds: 900,
      },
      SubmitPaymentCommand: {
        lockTtlSeconds: 60,
        processedTtlSeconds: 3600,
      },
    }
  }),
)

const application = await app.build()
```

In this example, `CreateUserCommand` uses the global lock TTL and a processed
payload TTL of 900 seconds. `SubmitPaymentCommand` uses both of its configured
TTLs. Other commands use the global values; any global value not specified falls
back to the pipeline default.

The intent selects TTL configuration only. The idempotency store is keyed by the
network context's `requestId`, not by the command intent. Use request IDs that
are unique across commands, and reuse the same ID only when retrying the same
operation.

## Use the request ID

Idempotency is based on the request's `requestId`.

The same request ID identifies the same idempotent operation.

For example, if a command is executed with:

```text
requestId = "request-123"
```

and the command completes successfully, a subsequent execution using the same
request ID can return the stored result without invoking the command handler
again.

The request ID must therefore remain stable when a client intentionally retries
the same operation.

Do not generate a new request ID for every retry if the intention is to retrieve
the result of the original operation.

## What happens when a request was already processed?

When Xeno.JS finds a processed payload for the request ID, it returns
`Result.ok(payload)` immediately. The payload must be defined in the store.

The command pipeline does not execute the command again.

If the store reports that the request was processed but has no payload for its
ID, the pipeline returns a failed conflict `Result` instead.

Conceptually:

```text
Request
   │
   ▼
Has this request already been processed?
   │
   ├── Yes ──► Return Result.ok(stored payload)
   │
   └── No
         │
         ▼
     Acquire lock
         │
         ▼
     Execute command
         │
         ▼
     Store successful payload
```

This is the main behavior provided by the Idempotency Pipeline.

## What happens when another request is processing the same ID?

If the request has not been processed yet but another execution already holds
the idempotency lock, the second execution cannot acquire the lock.

Xeno.JS returns a failed `Result` containing a conflict error.

The command handler is not executed by the second request.

This prevents two concurrent executions with the same request ID from processing
the command simultaneously.

## What happens when the command succeeds?

When the command returns a successful `Result`, Xeno.JS stores its value
(`result.getValueOrThrow()`) using the configured `processedTtlSeconds`.

For example:

```ts
const result = Result.ok({
  id: user.id,
})
```

Only the successful payload is stored. On replay, the pipeline creates a new
`Result.ok(payload)`; it does not persist and replay the original `Result`
object or its error metadata.

A subsequent request with the same request ID can receive:

```ts
{
  id: user.id,
}
```

without executing the command again.

## What happens when the command returns a failure?

A failed command result is **not** stored as the processed result.

For example:

```ts
return Result.fail(
  AppError.create({
    code: 'USER_NOT_FOUND',
    message: 'User not found',
    status: 404,
    name: 'CreateUserCommand',
  }),
)
```

The idempotency lock is released and the failed result is returned.

A later request with the same request ID can therefore attempt the command
again.

This is different from a successful execution, where the payload remains
available for the configured processed-result TTL.

## What happens when the command throws?

If the next pipeline behavior or command handler throws, Xeno.JS releases the
idempotency lock and rethrows the value. The CQRS exception behavior handles
that thrown value separately.

The exception is not stored as a processed result.

This means that a later request using the same request ID can attempt the
command again.

For example:

```ts
const result = await mediator.send(command)
```

If the handler throws:

```text
Command execution
      │
      ▼
   thrown value
      │
      ▼
 release lock
      │
      ▼
 rethrow value
```

## Configure both TTLs

A typical configuration explicitly sets both TTL values:

```ts
import { XenoAppBuilder, withIdempotency } from '@xeno-js/core'

const app = new XenoAppBuilder().addPipeline(
  withIdempotency((options) => {
    options.lockTtlSeconds = 30
    options.processedTtlSeconds = 300
  }),
)

const application = await app.build()
```

Use:

- `lockTtlSeconds` for the lifetime of an in-progress idempotency lock;
- `processedTtlSeconds` for the lifetime of a successful command payload.

Use positive integers for both values.

If they are omitted, Xeno.JS uses the built-in defaults.

## Complete example

The following example shows the relevant application configuration:

```ts
import { XenoAppBuilder, withIdempotency } from '@xeno-js/core'

const app = new XenoAppBuilder()
  .addPipeline(
    withIdempotency((options) => {
      options.lockTtlSeconds = 30
      options.processedTtlSeconds = 300
    }),
  )
  .addServices((services) => {
    services.addTransient('CREATE_USER_HANDLER', (scope) => {
      return new CreateUserHandler(scope.resolve('USER_REPOSITORY'))
    })
  })

const application = await app.build()
```

Your command can then be executed normally through the mediator:

```ts
const result = await mediator.send(
  new CreateUserCommand({
    email: 'user@example.com',
    name: 'John Doe',
  }),
)
```

The idempotency pipeline operates around the command execution without requiring
the handler to implement locking or result storage itself.

## Idempotency applies to Commands

`withIdempotency()` adds idempotency to the Command pipeline. The CQRS module
does not add this behavior to the Query pipeline.

If you need query result caching, see [Caching](./caching).

## Idempotency uses a cache

The built-in idempotency store uses the Xeno.JS cache to store:

- active idempotency locks;
- successfully processed command payloads.

When idempotency is enabled and no cache has been registered, the CQRS module
configures the default in-memory cache. Register a cache explicitly if the
application needs a different provider or cache configuration.

## Troubleshooting

### The command is executed more than once

Check:

1. `withIdempotency()` is passed to `addPipeline()`;
2. the command is executed through the Xeno.JS mediator;
3. the requests use the same `requestId`;
4. application building succeeds, so the CQRS module can configure the behavior
   and cache;
5. the processed-result TTL has not expired.

Remember that using a different request ID represents a different idempotent
operation.

### The second request returns a conflict

This can happen when the first request is still processing the command.

Check:

1. whether the first request is still running;
2. whether `lockTtlSeconds` is appropriate for the command duration;
3. whether the same request ID is being used concurrently.

A request that cannot acquire the existing lock does not execute the command.

### The command can be executed again after a failure

This is expected behavior.

Failed `Result` values are not stored as processed results. The lock is released
so that a later request can retry the operation.

### The same request is processed again after some time

Check `processedTtlSeconds`.

Once the stored successful payload expires, a later request with the same
request ID can be processed again.

Choose a processed-result TTL that matches how long your application needs to
recognize repeated requests as duplicates.

### Choose valid TTL values

Use positive integer values for:

```ts
lockTtlSeconds
processedTtlSeconds
```

The pipeline does not validate TTL values, so avoid zero, negative, or
fractional values.

## Related documentation

- [Application Overview](../overview)
- [Create a Command](../cqrs/command)
- [Create a Command Handler](../cqrs/handler)
- [Configure Concurrency](./concurrency)
- [Enable Caching](./caching)
- [Handle Exceptions](./exception)

---

---

<a id="source-5"></a>

# Source: `docs/application/pipelines/performance.md`

## Introduction

The Performance Pipeline measures how long commands and queries take to execute
and logs a warning when their execution time exceeds the configured threshold.

You enable it by passing `withPerformance()` to `XenoAppBuilder.addPipeline()`.

The pipeline supports two levels of configuration:

- `thresholdMs`: global threshold used for all intents.
- `intentThresholdMs`: optional thresholds for specific intents.

When an intent has a specific threshold configured, that value takes precedence
over the global threshold.

## Enable the Performance Pipeline

Add the pipeline when configuring your application:

```ts
import { XenoAppBuilder, withPerformance } from '@xeno-js/core'

const app = new XenoAppBuilder().addPipeline(withPerformance()).build()
```

`withPerformance()` adds the Performance Pipeline to the common pipeline
behaviors used by commands and queries.

## Configure the Global Threshold

Use the `thresholdMs` option to define the default threshold for commands and
queries.

```ts
import { XenoAppBuilder, withPerformance } from '@xeno-js/core'

const app = new XenoAppBuilder()
  .addPipeline(
    withPerformance((options) => {
      options.thresholdMs = 500
    }),
  )
  .build()
```

The default value is:

```ts
500
```

The value is expressed in milliseconds.

The global threshold is used whenever the current request intent does not have a
specific threshold configured.

## Configure Thresholds by Intent

Use `intentThresholdMs` as a registry of per-intent threshold overrides. Each
key must match the request's `intent`; each value is an object containing a
`thresholdMs` property.

An intent-specific threshold takes precedence over the global `thresholdMs`
option.

```ts
import { XenoAppBuilder, withPerformance } from '@xeno-js/core'

const app = new XenoAppBuilder()
  .addPipeline(
    withPerformance((options) => {
      options.thresholdMs = 500
      options.intentThresholdMs = {
        CREATE_USER_HANDLER: { thresholdMs: 300 },
      }
    }),
  )
  .build()
```

In this example:

- all requests use `500ms` by default;
- requests with intent `CREATE_USER_HANDLER` use `300ms`;
- the specific `CREATE_USER_HANDLER` threshold overrides the global threshold.

This is useful when different application operations have different expected
execution times.

## Global and Intent-Registry Thresholds

You can configure multiple intents independently in the `intentThresholdMs`
registry:

```ts
import { XenoAppBuilder, withPerformance } from '@xeno-js/core'

const app = new XenoAppBuilder()
  .addPipeline(
    withPerformance((options) => {
      options.thresholdMs = 500
      options.intentThresholdMs = {
        CREATE_USER_HANDLER: { thresholdMs: 300 },
        UPDATE_USER_HANDLER: { thresholdMs: 400 },
        DELETE_USER_HANDLER: { thresholdMs: 250 },
        GET_USER_HANDLER: { thresholdMs: 100 },
      }
    }),
  )
  .build()
```

The effective threshold for a request is selected in this order:

1. `options.intentThresholdMs[request.intent].thresholdMs`
2. `options.thresholdMs`
3. the default global threshold of `500ms`

For example, with:

```ts
options.thresholdMs = 500
options.intentThresholdMs = {
  CREATE_USER_HANDLER: { thresholdMs: 300 },
}
```

the behavior is:

| Intent                  | Effective threshold |
| ----------------------- | ------------------: |
| `CREATE_USER_HANDLER`   |             `300ms` |
| `UPDATE_USER_HANDLER`   |             `500ms` |
| `DELETE_USER_HANDLER`   |             `500ms` |
| Any unconfigured intent |             `500ms` |

An intent-specific threshold is therefore an override, not an additional
threshold.

## Configure Only Specific Intents

You can keep the global threshold and override only the operations that require
different limits:

```ts
import { XenoAppBuilder, withPerformance } from '@xeno-js/core'

const app = new XenoAppBuilder()
  .addPipeline(
    withPerformance((options) => {
      options.thresholdMs = 1000
      options.intentThresholdMs = {
        CREATE_USER_HANDLER: { thresholdMs: 300 },
        CHECKOUT_HANDLER: { thresholdMs: 800 },
      }
    }),
  )
  .build()
```

Here:

- `CREATE_USER_HANDLER` uses `300ms`;
- `CHECKOUT_HANDLER` uses `800ms`;
- every other intent uses `1000ms`.

## Performance Warnings

When execution exceeds the effective threshold, the pipeline logs a warning:

```text
Performance warning: CREATE_USER_HANDLER took 342.17ms
```

The warning contains:

- the request intent;
- the measured execution time in milliseconds.

A warning is logged only when the execution time is **greater than** the
effective threshold.

For example, with a threshold of `300ms`:

- `299ms` → no warning;
- `300ms` → no warning;
- `301ms` → warning.

## Threshold Validation

The global `thresholdMs` must be greater than zero. Intent-specific
`thresholdMs` values must be positive integers.

For example, this configuration is invalid:

```ts
import { XenoAppBuilder, withPerformance } from '@xeno-js/core'

const app = new XenoAppBuilder()
  .addPipeline(
    withPerformance((options) => {
      options.thresholdMs = 0
      options.intentThresholdMs = {
        CREATE_USER_HANDLER: { thresholdMs: -100 },
      }
    }),
  )
  .build()
```

The global threshold rejects negative values and zero, but is not checked for
integer precision. The pipeline validates each intent-specific threshold as a
positive integer.

Each configured intent threshold is validated independently.

## Complete Example

The following example configures a global threshold together with different
thresholds for selected handlers:

```ts
import { XenoAppBuilder, withPerformance } from '@xeno-js/core'

const app = new XenoAppBuilder()
  .addPipeline(
    withPerformance((options) => {
      // Fallback threshold for all intents.
      options.thresholdMs = 500

      // Overrides for selected intents.
      options.intentThresholdMs = {
        CREATE_USER_HANDLER: { thresholdMs: 300 },
        UPDATE_USER_HANDLER: { thresholdMs: 400 },
        CHECKOUT_HANDLER: { thresholdMs: 1000 },
      }
    }),
  )
  .build()
```

The resulting behavior is:

```text
CREATE_USER_HANDLER → 300ms
UPDATE_USER_HANDLER  → 400ms
CHECKOUT_HANDLER     → 1000ms
Everything else      → 500ms
```

## Commands and Queries

The Performance Pipeline applies to both commands and queries executed through
the configured CQRS pipeline.

The intent-specific configuration works the same way for both:

```ts
withPerformance((options) => {
  options.intentThresholdMs = {
    CREATE_USER_HANDLER: { thresholdMs: 300 },
    GET_USER_HANDLER: { thresholdMs: 100 },
  }
})
```

The intent identifies the operation being executed, regardless of whether it is
a command or query.

## Troubleshooting

### My intent-specific threshold is ignored

Check that the registry key in `intentThresholdMs` exactly matches the request
`intent`, and that its value contains `thresholdMs`:

```ts
options.intentThresholdMs = {
  CREATE_USER_HANDLER: { thresholdMs: 300 },
}
```

The pipeline compares the key directly with `request.intent`.

If the intent does not match, Xeno.JS falls back to:

```ts
options.thresholdMs
```

### I expected a warning at exactly the threshold

The pipeline logs a warning only when execution time is greater than the
threshold.

With:

```ts
options.thresholdMs = 500
```

an execution time of exactly `500ms` does not generate a warning.

### My threshold value is rejected

Check that the global value is greater than zero. Intent-specific values must be
positive integers.

Valid:

```ts
300
500
1000
```

Invalid global values:

```ts
0 - 100
```

An intent-specific value such as `250.5` is also invalid because intent
thresholds must be integers.

## Related Documentation

- [Application](../overview)
- [CQRS](../cqrs/overview)
- [Commands](../cqrs/command)
- [Queries](../cqrs/query)
- [Handlers](../cqrs/handler)
- [Exception Pipeline](./exception)
- [Logging Pipeline](./logging)

---

---

<a id="source-6"></a>

# Source: `docs/cache/overview.md`

## Introduction

Xeno.JS provides a common cache API that can be backed by different cache
implementations.

The cache is configured through `XenoAppBuilder` and resolved through
`TOKENS.CACHE`.

## Configure the Cache

Add the cache module to your application:

```ts
import { XenoAppBuilder } from '@xeno-js/core'

const app = new XenoAppBuilder()

app.addCache()
```

The default configuration enables the **in-memory cache**.

You can also configure the cache explicitly:

```ts
app.addCache((options) => {
  options.inMemory = true
})
```

For Redis configuration, see [Redis Cache](./redis).

## Cache Types

Xeno.JS currently provides:

- **In-memory cache** — stores values in the current application process.
- **Redis cache** — stores values in Redis and can be used when cache state must
  be shared outside the process.

Both implementations expose the same cache contract, so application code can
resolve and use the cache without depending on the underlying storage.

## Resolve the Cache

After building the application, resolve `TOKENS.CACHE` through the builder:

```ts
import { XenoAppBuilder, TOKENS } from '@xeno-js/core'

const app = new XenoAppBuilder()

app.addCache()

await app.build()

const { cache } = app.getServices()
```

The resolved value is the configured cache implementation.

## Use the Cache

### Set and Get a Value

```ts
await cache.set('user:123', { id: '123', name: 'Alice' }, 60)

const user = await cache.get<{ id: string; name: string }>('user:123')
```

The third argument of `set()` is the TTL in seconds.

### Check a Key

```ts
const exists = await cache.has('user:123')
```

### Remove a Value

```ts
await cache.remove('user:123')
```

### Set Only If the Key Does Not Exist

```ts
const created = await cache.setIfAbsent('lock:123', { locked: true }, 30)
```

The method returns `true` when the value is stored and `false` when the key
already exists.

### Clear the Cache

```ts
await cache.clear()
```

### Increment a Value

The configured cache also exposes the atomic `increment()` operation:

```ts
const count = await cache.increment('counter', 60)
```

If the key does not exist, the counter starts from `1`.

## Where Xeno.JS Uses the Cache

The configured cache is used by Xeno.JS itself for several application features.

### Query Caching

The query pipeline can cache query results when caching is configured on a
query.

For example, a query can provide:

```ts
cacheOptions: {
  cacheKey: 'users:list',
  ttl: 60,
  bypassCache: false,
  consistentRead: false,
  isUserScoped: false,
}
```

The query caching pipeline uses `TOKENS.CACHE` to read and store the result.

For details about query caching, see the CQRS documentation.

### Command Idempotency

Command idempotency uses the same cache abstraction to store locks and
previously processed command results.

This means changing the configured cache implementation does not require
changing the idempotency feature.

### Rate Limiting

HTTP rate limiting also uses the configured cache.

This allows rate-limit state to be stored through the same cache abstraction
used by the rest of the application.

## Cache Configuration and Pipelines

When you call `addPipeline()`, Xeno.JS queues the cache module if query caching
or command idempotency is enabled and a cache has not already been configured.

When you call `addMiddlewares()`, Xeno.JS queues the cache module if rate
limiting is enabled and a cache has not already been configured.

Therefore, call `addCache()` explicitly when you want to configure the cache
yourself, for example to select Redis.

## Choose a Cache

Use the in-memory cache when:

- the cache only needs to live inside one application process;
- you are developing locally;
- shared cache state is not required.

Use Redis when:

- multiple application instances need to share cached values;
- cache state must live outside the application process;
- you need a distributed cache.

For Redis configuration, see [Redis Cache](./redis).

## Related Documentation

- [Redis Cache](./redis)
- [CQRS Query](../application/cqrs/query)
- [Caching Pipeline](../application/pipelines/caching)
- [Command Idempotency](../application/pipelines/idempotency)
- [Dependency Injection Resolution](../dependency-injection/resolution)

---

---

<a id="source-7"></a>

# Source: `docs/cache/redis.md`

## Introduction

Xeno.JS can use Redis as the application's cache backend through the `withRedis`
cache plugin.

Pass the plugin to `XenoAppBuilder.addCache()`. The plugin supplies a Redis
cache provider to the cache module.

## Install Redis Support

Install the Redis integration package:

```bash
npm install @xeno-js/redis
```

## Configure Redis

Pass `withRedis` to `addCache()`. Its setup callback receives Redis
configuration; the plugin defaults to enabled, with host `localhost` and port
`6379`.

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withRedis } from '@xeno-js/redis'

const app = new XenoAppBuilder()

app.addCache(
  withRedis((options, config) => {
    options.host = config.get('REDIS_HOST') ?? 'localhost'
    options.port = config.getNumber('REDIS_PORT') ?? 6379
    options.options = {
      username: config.get('REDIS_USERNAME'),
      password: config.get('REDIS_PASSWORD'),
    }
  }),
)
```

The Redis configuration fields shown by the plugin are:

| Field       | Description                                                                   |
| ----------- | ----------------------------------------------------------------------------- |
| `isEnabled` | Enables the Redis provider when `true`; defaults to `true`.                   |
| `host`      | Redis server hostname; defaults to `localhost`.                               |
| `port`      | Redis server port; defaults to `6379`.                                        |
| `options`   | Options object initialized with `username` and `password` from configuration. |

## Using Environment Variables

A typical configuration can be kept entirely in environment variables:

```env
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_USERNAME=
REDIS_PASSWORD=
```

Then configure the application:

```ts
import { withRedis } from '@xeno-js/redis'

app.addCache(
  withRedis((options, config) => {
    options.host = config.get('REDIS_HOST') ?? 'localhost'
    options.port = config.getNumber('REDIS_PORT') ?? 6379
    options.options = {
      username: config.get('REDIS_USERNAME'),
      password: config.get('REDIS_PASSWORD'),
    }
  }),
)
```

## Resolve the Redis Cache

The application does not resolve Redis directly.

Resolve the common Xeno.JS cache token:

```ts
import { XenoAppBuilder, TOKENS } from '@xeno-js/core'
import { withRedis } from '@xeno-js/redis'

const app = new XenoAppBuilder()

app.addCache(withRedis(() => {}))

await app.build()

const cache = app.resolve(TOKENS.CACHE)
```

`TOKENS.CACHE` returns the configured Redis-backed cache.

Your application therefore remains independent from the Redis client:

```ts
await cache.set('user:123', { id: '123' }, 60)

const user = await cache.get<{ id: string }>('user:123')
```

## Redis Cache Operations

The Redis-backed cache supports the same cache API described in the
[Cache Overview](./overview):

```ts
await cache.set('key', value, 60)

const value = await cache.get<MyValue>('key')

await cache.remove('key')

const exists = await cache.has('key')

await cache.clear()
```

It also supports atomic operations used by Xeno.JS features:

```ts
await cache.setIfAbsent('lock:key', true, 30)

await cache.increment('counter', 60)
```

## Complete Example

```ts
import { XenoAppBuilder, TOKENS } from '@xeno-js/core'
import { withRedis } from '@xeno-js/redis'

const app = new XenoAppBuilder()

app.addCache(
  withRedis((options, config) => {
    options.host = config.get('REDIS_HOST') ?? 'localhost'
    options.port = config.getNumber('REDIS_PORT') ?? 6379
    options.options = {
      username: config.get('REDIS_USERNAME'),
      password: config.get('REDIS_PASSWORD'),
    }
  }),
)

await app.build()

const cache = app.resolve(TOKENS.CACHE)

await cache.set('example', { value: 'hello' }, 60)

const value = await cache.get<{ value: string }>('example')

console.log(value)
```

## Troubleshooting

### Redis Is Not Being Used

Pass the Redis cache plugin to `addCache()`:

```ts
app.addCache(withRedis(() => {}))
```

The plugin selects its Redis provider by default. Set `isEnabled` to `false` in
its setup callback to use the in-memory cache instead.

### Redis Connection Fails

Check:

- `REDIS_HOST`
- `REDIS_PORT`
- `REDIS_USERNAME`
- `REDIS_PASSWORD`
- Redis availability

### The Cache Is Not Registered

If the cache module receives neither an enabled in-memory configuration nor a
cache provider, its configuration fails.

## Related Documentation

- [Cache Overview](./overview)
- [Dependency Injection Resolution](../dependency-injection/resolution)
- [Caching Pipeline](../application/pipelines/caching)
- [Idempotency](../application/pipelines/idempotency)

---
