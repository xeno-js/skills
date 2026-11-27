# Logging and observability

Technical reference assembled from the website documentation. Each section
identifies its original Markdown source. Website frontmatter and the repeated
sponsorship footer are omitted; body content and fenced examples are retained.

---

## Contents

- [`docs/application/logging/custom-logger.md`](#source-1)
- [`docs/application/logging/overview.md`](#source-2)
- [`docs/application/logging/pino-logger.md`](#source-3)
- [`docs/application/logging/sentry-node.md`](#source-4)
- [`docs/application/logging/sentry-vue.md`](#source-5)
- [`docs/application/pipelines/logging.md`](#source-6)

<a id="source-1"></a>

# Source: `docs/application/logging/custom-logger.md`

## Introduction

Xeno.JS lets you add your own logging provider by implementing `ILoggerClient`.

## Create a Custom Logger

Implement `ILoggerClient` and its `track()` method:

```ts
import type { ILoggerClient, LogLevel } from '@xeno-js/shared'

class MyLogger implements ILoggerClient {
  track<T>(
    level: LogLevel,
    message: string,
    context: T | undefined,
    error: unknown | undefined,
  ): void {
    console.log({
      level,
      message,
      context,
      error,
    })
  }
}
```

The `track()` method receives the log level, message, optional logging context,
and optional error.

## Register the Logger

Pass a factory through `customLoggers`:

```ts
import { XenoAppBuilder } from '@xeno-js/core'

const app = new XenoAppBuilder()

app.addLogger((options) => {
  options.customLoggers = [() => new MyLogger()]
})
```

Provide a factory that returns an `ILoggerClient`:

```ts
options.customLoggers = [() => new MyLogger()]
```

The logger module processes the registered factory during application build. The
shown contract does not pass a service scope or container to the factory.

## Resolve and Use the Logger

After building the application, resolve `TOKENS.LOGGER`:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { TOKENS } from '@xeno-js/shared'

const app = new XenoAppBuilder()

app.addLogger((options) => {
  options.customLoggers = [() => new MyLogger()]
})

await app.build()

const { logger } = app.getServices()

logger.info('Application started')
logger.error('Something went wrong', new Error('Example error'))
```

`TOKENS.LOGGER` resolves the Xeno.JS application logger. Your custom logger is
registered as one of its logging clients, so it receives the log events together
with any other configured providers.

## Complete Example

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { TOKENS } from '@xeno-js/shared'
import type { ILoggerClient, LogLevel } from '@xeno-js/shared'

class MyLogger implements ILoggerClient {
  track<T>(
    level: LogLevel,
    message: string,
    context: T | undefined,
    error: unknown | undefined,
  ): void {
    console.log({
      level,
      message,
      context,
      error,
    })
  }
}

const app = new XenoAppBuilder()

app.addLogger((options) => {
  options.customLoggers = [() => new MyLogger()]
})

await app.build()

const { logger } = app.getServices()

logger.info('Hello from Xeno.JS')
```

---

Related

- [Observability Overview](./overview)
- [Pino Logger](./pino-logger)
- [Sentry Node Logger](./sentry-node).

---

---

<a id="source-2"></a>

# Source: `docs/application/logging/overview.md`

## Introduction

Xeno.JS provides an application-level logger that can send the same log event to
one or more logging providers.

You can use:

- the built-in Console logger;
- Pino;
- Sentry Node;
- Sentry Vue;
- one or more custom logger clients;
- any combination of the providers above.

The application code uses the Xeno.JS `ILogger` abstraction rather than
depending directly on a provider.

This lets you configure logging without changing the code that writes log
messages.

## Add the Default Console Logger

The simplest configuration is:

```ts
import { XenoAppBuilder } from '@xeno-js/core'

const app = new XenoAppBuilder()

app.addLogger()

await app.build()
```

Calling `addLogger()` enables the Console provider with the default `DEBUG`
minimum level.

The logger is registered in the application container and can then be resolved
through `TOKENS.LOGGER`.

## Configure the Log Level

Xeno.JS supports four log levels:

| Level   | Value | Purpose                                      |
| ------- | ----: | -------------------------------------------- |
| `DEBUG` |   `0` | Detailed diagnostic information              |
| `INFO`  |   `1` | Normal application activity                  |
| `WARN`  |   `2` | Potentially harmful or unexpected situations |
| `ERROR` |   `3` | Errors and failures                          |

The configured level is a minimum level.

For example, with `INFO`, `DEBUG` messages are ignored while `INFO`, `WARN`, and
`ERROR` messages are accepted.

Configure it through the `addLogger()` setup callback:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { LOG_LEVEL } from '@xeno-js/shared'

const app = new XenoAppBuilder()

app.addLogger((options) => {
  options.level = LOG_LEVEL.INFO
})

await app.build()
```

The default level is `LOG_LEVEL.DEBUG`.

The minimum level is applied by the Xeno.JS logger before the event is sent to
the configured providers.

## Disable the Console Logger

Console logging is enabled by default.

To disable it:

```ts
app.addLogger((options) => {
  options.console = false
})
```

This does not disable the Xeno.JS logger itself. It only removes the Console
provider.

You can therefore disable Console while keeping another provider enabled:

```ts
import { XenoAppBuilder, withConsoleLogger } from '@xeno-js/core'
import { withPinoLogger } from '@xeno-js/pino'

const app = new XenoAppBuilder()

app.addLogger([
  withConsoleLogger((options) => {
    options.isEnabled = false
  }),
  withPinoLogger((options) => {
    options.destination = 'stdout'
  }),
])
```

## Use Multiple Providers

The logger configuration is not limited to one provider.

For example, an application can configure Console, Pino, and Sentry together:

```ts
import { XenoAppBuilder, withConsoleLogger } from '@xeno-js/core'
import { withPinoLogger } from '@xeno-js/pino'
import { withSentryNodeLogger } from '@xeno-js/sentry-node'
import { LOG_LEVEL } from '@xeno-js/shared'

const app = new XenoAppBuilder()

app.addLogger([
  withConsoleLogger((options) => {
    options.level = LOG_LEVEL.INFO
  }),
  withPinoLogger((options) => {
    options.destination = 'stdout'
  }),
  withSentryNodeLogger((options) => {
    options.dsn = process.env.SENTRY_DSN ?? ''
    options.environment = 'production'
  }),
])

await app.build()
```

Each event accepted by the application-level minimum is passed to every
configured provider. A provider can apply an additional minimum level before
writing or reporting it.

This means the same application log can be:

- printed to the Console;
- written as a structured Pino log;
- reported to Sentry when its level is eligible.

You can also combine built-in providers with custom logger clients.

## Add a Custom Logger

Xeno.JS accepts custom logger clients through `customLoggers`.

A custom logger is provided as a factory receiving the application's service
container through the `IServiceScope` contract:

```ts
app.addLogger((options) => {
  options.customLoggers = [() => customLogger]
})
```

The custom logger must implement the Xeno.JS `ILoggerClient` contract:

```ts
interface ILoggerClient {
  track<T>(
    level: LogLevel,
    message: string,
    context: T | undefined,
    error: unknown | undefined,
  ): void
}
```

For example:

```ts
import type { ILoggerClient } from '@xeno-js/shared'

const customLogger: ILoggerClient = {
  track(level, message, context, error) {
    // Send the event to your logging system.
    console.log({
      level,
      message,
      context,
      error,
    })
  },
}
```

Register it through `addLogger()`:

```ts
app.addLogger((options) => {
  options.customLoggers = [() => customLogger]
})
```

A custom logger can be used by itself:

```ts
app.addLogger((options) => {
  options.console = false

  options.customLoggers = [() => customLogger]
})
```

or together with the built-in Console provider:

```ts
app.addLogger((options) => {
  options.customLoggers = [() => customLogger]
})
```

The custom logger receives the same Xeno.JS log event as the other configured
logger clients. Pino and Sentry are registered separately through their logger
plugins.

## Resolve the Logger

Xeno.JS registers the application logger under `TOKENS.LOGGER`.

After building the application, resolve it through the builder:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { TOKENS } from '@xeno-js/shared'

const app = new XenoAppBuilder()

app.addLogger()

await app.build()

const { logger } = app.getServices()
```

The resolved value implements `ILogger`.

The public logger API is intentionally small:

```ts
interface ILogger {
  info(message: string): void
  warn(message: string): void
  debug(message: string): void
  error(message: string, error: unknown): void
}
```

## Write Log Messages

Use the resolved logger directly:

```ts
logger.debug('Loading configuration')
logger.info('Application started')
logger.warn('Cache entry was not available')
logger.error('Unable to process request', error)
```

`error()` accepts the message followed by the associated error value.

Context from the current Xeno.JS request is added automatically. Application
code therefore does not need to pass the request context to every logging call.

## What Does Xeno.JS Log?

Xeno.JS creates a log event containing:

1. the log level;
2. the log message;
3. the current safe context;
4. the optional error.

The logger converts the current request context into a restricted
`LoggerContext` before passing it to providers.

The safe context contains:

```text
identity
├── userId
└── tenantId

network
├── requestId
├── path
├── origin
└── masked clientIp

tracing
├── correlationId
├── spanId
├── startTime
└── parentSpanId

messaging
└── optional messaging context
```

The original request context contains additional information such as email,
name, roles, permissions, user-agent, CSRF values, and cookies. These fields are
not copied into the safe logging context.

The client IP is passed through Xeno.JS IP masking before it is included.

### Why Safe Context Matters

The safe context gives log entries useful information for correlating and
diagnosing requests without serializing the complete request context.

For example, a log can identify:

- which request produced the event;
- which user or tenant was involved;
- which correlation/span identifiers belong to the execution;
- which request path was involved.

It does not automatically serialize the entire authenticated identity or network
request.

Safe context does not mean that arbitrary application data is safe to log.

If application code explicitly puts secrets, tokens, passwords, request bodies,
or other sensitive information into a log message, that data can still reach the
configured providers.

Treat application-provided log messages and error objects as potentially
sensitive.

## Console Output

The Console provider maps Xeno.JS levels to the corresponding console method:

| Xeno.JS level | Console method    |
| ------------- | ----------------- |
| `DEBUG`       | `console.debug()` |
| `INFO`        | `console.info()`  |
| `WARN`        | `console.warn()`  |
| `ERROR`       | `console.error()` |

The Xeno.JS logger adds the textual level name to the message before passing it
to the provider.

For example:

```text
[INFO] Application started
```

The Console provider then prefixes the numeric Xeno.JS level.

A representative Console call therefore looks like:

```text
[1] [INFO] Application started
{
  identity: {
    userId: "...",
    tenantId: "..."
  },
  network: {
    requestId: "...",
    path: "/users",
    origin: "...",
    clientIp: "..."
  },
  tracing: {
    correlationId: "...",
    spanId: "...",
    startTime: 1234567890,
    parentSpanId: "..."
  }
}
```

For an error, the provider adds the error to the payload:

```text
[3] [ERROR] Unable to process request
{
  ...safeContext,
  error: Error(...)
}
```

The exact values depend on the active request context.

## Pino Output

The Pino provider receives the safe context as a structured object and the
Xeno.JS message as the Pino message.

Xeno.JS configures Pino with its standard error serializers and redaction for
`password`, `token`, `authorization`, `accessToken`, `refreshToken`, `secret`,
`cookie`, `creditCard`, `*.password`, `*.token`, `headers.authorization`, and
`headers.cookie`. Additional redaction paths and Pino options can be supplied
through the plugin.

The default Pino destination is `stdout`.

Pino pretty printing is disabled by default. Enable it explicitly with the
plugin's `prettyPrint` option. Xeno.JS does not set an ISO timestamp by default;
Pino's default timestamp behavior is used unless overridden through
`pinoOptions`.

A structured Pino event is therefore conceptually represented as:

```json
{
  "level": 30,
  "time": 1791043200000,
  "msg": "[INFO] Application started",
  "identity": {
    "userId": "...",
    "tenantId": "..."
  },
  "network": {
    "requestId": "...",
    "path": "/users",
    "origin": "...",
    "clientIp": "..."
  },
  "tracing": {
    "correlationId": "...",
    "spanId": "...",
    "startTime": 1234567890,
    "parentSpanId": "..."
  }
}
```

The exact serialized representation is produced by Pino.

See [Pino Logger](./pino-logger) for Pino-specific configuration.

## Sentry Behavior

The Xeno.JS Sentry Node provider is intended for warning and error events.

It applies the configured minimum level first.

`DEBUG` and `INFO` events are not sent to Sentry.

For `WARN` and `ERROR`:

- when an error object is supplied, Xeno.JS calls Sentry `captureException()`;
- when no error object is supplied, Xeno.JS calls Sentry `captureMessage()` with
  the Sentry severity `"warning"`;
- the safe context is attached to the Sentry scope as extras.

This means an `ERROR` log without an error object is still captured as a Sentry
message with warning severity.

Register and configure Sentry through its logger plugin:

```ts
import { withSentryNodeLogger } from '@xeno-js/sentry-node'

app.addLogger(
  withSentryNodeLogger((options, config) => {
    options.dsn = config.getOrThrow('SENTRY_DSN')
    options.environment = config.getEnvironment()
  }),
)
```

The Sentry configuration requires a DSN.

See [Sentry Node](./sentry-node) for the complete Sentry configuration and
warning/error behavior. For browser applications, see
[Sentry Vue](/docs/application/logging/sentry-vue), which supports configurable
provider log levels and browser tracing.

## Where Xeno.JS Uses the Logger

The logger is used by Xeno.JS infrastructure and application pipelines.

One important automatic use is the CQRS logging pipeline.

For each command or query handled by that pipeline, Xeno.JS logs:

```text
Handling <request.type> <request.intent>
```

When the request succeeds:

```text
Successfully handled <request.type> <request.intent>
```

When a result contains an error:

```text
Failed to handle <request.type> <request.intent>: <error.message>
```

When handling throws an exception:

```text
Exception while handling <request.type> <request.intent>
```

The pipeline uses:

- `INFO` for start and successful completion;
- `ERROR` for failed results and thrown exceptions.

Other Xeno.JS infrastructure components also use `ILogger` for warnings, errors,
and diagnostic messages. For example, the performance pipeline emits a warning
when an operation exceeds its configured threshold.

## Complete Example

The following example enables Console and Pino while keeping Sentry disabled:

```ts
import { XenoAppBuilder, withConsoleLogger } from '@xeno-js/core'
import { withPinoLogger } from '@xeno-js/pino'
import { LOG_LEVEL, TOKENS } from '@xeno-js/shared'

const app = new XenoAppBuilder()

app.addLogger([
  withConsoleLogger((options) => {
    options.level = LOG_LEVEL.INFO
  }),
  withPinoLogger((options) => {
    options.destination = 'stdout'
    options.prettyPrint = true
  }),
])

await app.build()

const { logger } = app.getServices()

logger.info('Application started')
logger.warn('Example warning')

try {
  throw new Error('Example failure')
} catch (error) {
  logger.error('Application operation failed', error)
}
```

With this configuration:

- `DEBUG` events are filtered out;
- `INFO`, `WARN`, and `ERROR` events are accepted;
- Console and Pino receive all events accepted by the application-level minimum;
- the same safe context is supplied to both providers.

## Common Problems

### No Logs Are Printed

Check that the logger module was added:

```ts
app.addLogger()
```

Then make sure the message is at or above the configured level.

For example, this configuration suppresses `DEBUG`:

```ts
app.addLogger((options) => {
  options.level = LOG_LEVEL.INFO
})
```

### Console Logs Are Missing

Check whether Console was disabled:

```ts
options.console = false
```

If Console is disabled, another configured provider must be used to receive the
log.

### Pino Is Configured but Nothing Is Written

Check:

- the `@xeno-js/pino` package is installed;
- `withPinoLogger()` is registered with `app.addLogger()`;
- the configured destination is valid;
- the Pino and application-level minimum levels allow the event.

See [Pino Logger](./pino-logger).

### Sentry Does Not Receive an INFO Log

This is expected.

The Xeno.JS Sentry provider only processes `WARN` and `ERROR` levels.

If you need an event in Sentry, use a warning or error that satisfies the
configured minimum level.

### Sensitive Data Appears in a Log

Safe context only controls the request context that Xeno.JS automatically adds.

It does not sanitize arbitrary application messages or errors.

Avoid logging:

- passwords;
- authentication tokens;
- secrets;
- credentials;
- complete request bodies;
- cookies;
- authorization headers;
- other sensitive application data.

Pino also applies provider-level redaction for specific fields, but applications
should not rely on redaction as a replacement for careful logging.

## Related Docs

- [Pino Logger](./pino-logger)
- [Sentry Node](./sentry-node)
- [Dependency Injection — Resolution](../dependency-injection/resolution)
- [Context & Scopes](../fundamentals/context-scopes)
- [CQRS — Handler](../application/cqrs/handler)
- [Pipelines — Logging](../application/pipelines/logging)

---

---

<a id="source-3"></a>

# Source: `docs/application/logging/pino-logger.md`

## Introduction

Xeno.JS provides a Pino integration through its application logger.

You do not replace the Xeno.JS logger with a Pino-specific API. Instead, you
enable Pino as a logging provider and continue to use the Xeno.JS `ILogger`
abstraction.

The basic configuration is:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withPinoLogger } from '@xeno-js/pino'

const app = new XenoAppBuilder()

app.addLogger(
  withPinoLogger((options) => {
    options.destination = 'stdout'
  }),
)

await app.build()
```

`build()` returns the application instance. After building, resolve the
application logger through its service proxy:

```ts
const application = await app.build()
const { logger } = application.getServices()
```

---

## Prerequisites

Install Xeno.JS and its Pino integration:

```bash
npm install @xeno-js/core @xeno-js/shared @xeno-js/pino
```

`@xeno-js/pino` provides the Pino adapter. Install any external dependencies
required by that adapter according to its package metadata; `pino` is not an
optional peer dependency of `@xeno-js/core`.

If you enable pretty printing, Xeno.JS configures Pino to use the `pino-pretty`
transport. Install it as well:

```bash
npm install pino-pretty
```

You only need `pino-pretty` when pretty printing is enabled.

---

## Enable Pino

Register `withPinoLogger()` with `addLogger()` to enable the Pino provider. The
plugin's setup callback receives Pino adapter options directly.

For example:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withPinoLogger } from '@xeno-js/pino'

const app = new XenoAppBuilder()

app.addLogger(withPinoLogger(() => {}))

const application = await app.build()
```

The plugin is enabled by default. Set `isEnabled` to `false` in its setup
callback to skip registering the Pino logger.

Installing the adapter alone does not register it with the application.

---

## Configure the Log Level

The minimum log level is configured through the Xeno.JS logger configuration:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { LOG_LEVEL } from '@xeno-js/shared'

const app = new XenoAppBuilder()

app.addLogger((options) => {
  options.level = LOG_LEVEL.INFO
})
```

Xeno.JS supports:

- `LOG_LEVEL.DEBUG`
- `LOG_LEVEL.INFO`
- `LOG_LEVEL.WARN`
- `LOG_LEVEL.ERROR`

The configured level is the minimum level accepted by the Xeno.JS logger.

For example:

```ts
options.level = LOG_LEVEL.WARN
```

allows:

```text
WARN
ERROR
```

and filters:

```text
DEBUG
INFO
```

The Xeno.JS logger applies its configured minimum level before sending an event
to the configured providers. `addLogger()` accepts either a setup callback or
logger plugin(s); it is not additive across multiple calls. To register Pino
alongside other providers, pass the plugins together in one array. The Pino
plugin callback receives Pino adapter options, not the `LoggerConfig` passed to
the setup callback.

---

## Configure the Destination

Xeno.JS supports two Pino destinations:

```ts
type Destination = 'stdout' | 'file'
```

### Write to stdout

This is the default:

```ts
app.addLogger(
  withPinoLogger((options) => {
    options.destination = 'stdout'
  }),
)
```

If `destination` is omitted, Xeno.JS uses `stdout`.

### Write to a file

Configure:

```ts
app.addLogger(
  withPinoLogger((options) => {
    options.destination = 'file'
    options.filePath = 'logs/app.log'
  }),
)
```

If `destination` is `file` and `filePath` is omitted, Xeno.JS uses:

```text
logs/app.log
```

The file path is passed to Pino's destination stream.

Make sure the application process has permission to create and write to the
configured path.

---

## Configure the Environment

Pino has an optional `env` configuration:

```ts
app.addLogger(
  withPinoLogger((options) => {
    options.env = 'production'
    options.destination = 'stdout'
  }),
)
```

If `env` is not specified, Xeno.JS uses:

1. `process.env.NODE_ENV`, when defined;
2. otherwise `development`.

The environment affects the default value of `prettyPrint`.

---

## Pretty Printing

Xeno.JS enables pretty printing by default when the configured environment is
`development`.

For example:

```ts
app.addLogger(
  withPinoLogger((options) => {
    options.env = 'development'
  }),
)
```

is equivalent to using:

```ts
app.addLogger(
  withPinoLogger((options) => {
    options.env = 'development'
    options.prettyPrint = true
  }),
)
```

To explicitly disable it:

```ts
app.addLogger(
  withPinoLogger((options) => {
    options.env = 'development'
    options.prettyPrint = false
  }),
)
```

To explicitly enable it:

```ts
app.addLogger(
  withPinoLogger((options) => {
    options.env = 'production'
    options.prettyPrint = true
  }),
)
```

When pretty printing is enabled, Xeno.JS configures the Pino `pino-pretty`
transport with:

- colorized output;
- standard translated timestamps;
- `pid` and `hostname` omitted from the pretty output.

If you enable pretty printing, make sure `pino-pretty` is installed.

---

## Resolve and Use the Logger

Pino is not exposed as a separate DI service token.

Xeno.JS registers the application logger under `TOKENS.LOGGER`.

Resolve it after building the application:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withPinoLogger } from '@xeno-js/pino'

const app = new XenoAppBuilder()

app.addLogger(
  withPinoLogger((options) => {
    options.destination = 'stdout'
  }),
)

const application = await app.build()

const { logger } = application.getServices()
```

The returned value implements the Xeno.JS `ILogger` interface.

Use the normal Xeno.JS logging methods:

```ts
logger.debug('Loading configuration')
logger.info('Application started')
logger.warn('Cache entry was not available')
logger.error('Unable to process request', error)
```

You do not need to call Pino's `debug()`, `info()`, `warn()`, or `error()`
methods directly in application code.

Xeno.JS forwards each accepted log event to the configured Pino provider.

---

## Use Pino Together With Other Providers

Pino does not have to be the only provider.

For example, you can enable Console and Pino together:

```ts
import { XenoAppBuilder, withConsoleLogger } from '@xeno-js/core'
import { withPinoLogger } from '@xeno-js/pino'
const app = new XenoAppBuilder()

app.addLogger([
  withConsoleLogger((options) => {
    options.isEnabled = true
  }),
  withPinoLogger((options) => {
    options.destination = 'stdout'
  }),
])
```

You can also configure Pino and Sentry:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withPinoLogger } from '@xeno-js/pino'
import { withSentryNodeLogger } from '@xeno-js/sentry-node'

const app = new XenoAppBuilder()

app.addLogger([
  withPinoLogger((options) => {
    options.destination = 'stdout'
  }),
  withSentryNodeLogger((options) => {
    options.dsn = process.env.SENTRY_DSN ?? ''
    options.environment = process.env.NODE_ENV
  }),
])
```

The Xeno.JS logger sends accepted events to all configured logger clients.

This allows the same application logging API to feed different observability
systems.

---

## Structured Pino Output

Xeno.JS passes the log context to Pino as structured data.

The Pino adapter maps Xeno.JS levels to Pino methods:

| Xeno.JS level | Pino method |
| ------------- | ----------- |
| `DEBUG`       | `debug()`   |
| `INFO`        | `info()`    |
| `WARN`        | `warn()`    |
| `ERROR`       | `error()`   |

For example, an informational event is passed to Pino conceptually as:

```ts
pinoLogger.info(
  {
    ...context,
  },
  message,
)
```

An error event includes the error in the structured payload:

```ts
pinoLogger.error(
  {
    ...context,
    error,
  },
  message,
)
```

Pino is configured by Xeno.JS with ISO timestamps and its standard error
serializer.

A structured log therefore contains information such as:

```json
{
  "level": 30,
  "time": "2026-10-03T16:00:00.000Z",
  "msg": "[INFO] Application started",
  "requestId": "...",
  "correlationId": "..."
}
```

The exact JSON representation is produced by Pino.

---

## Safe Context

Xeno.JS does not pass the complete request context directly to Pino.

The Xeno.JS logger builds a safe logging context before sending it to logger
clients.

Depending on the active request context, this can contain information such as:

```text
identity
├── userId
└── tenantId

network
├── requestId
├── path
├── origin
└── masked clientIp

tracing
├── correlationId
├── spanId
├── startTime
└── parentSpanId
```

The purpose is to provide useful correlation and diagnostic information without
serializing the complete request context.

For example, request-specific information such as authentication state, tracing
identifiers, and the request path can be available to Pino without automatically
copying arbitrary request data.

The client IP is masked before being included in the safe logging context.

### Application Data Is Still Your Responsibility

Safe context does not make arbitrary application logging safe.

If application code explicitly logs:

```ts
logger.info(`Password: ${password}`)
```

the password is part of the message.

Likewise, application-provided structured data or error objects can contain
sensitive information.

Avoid logging:

- passwords;
- authentication tokens;
- secrets;
- credentials;
- cookies;
- authorization headers;
- complete request bodies;
- other sensitive application data.

---

## Pino Redaction

Xeno.JS configures Pino redaction for the following paths:

```text
password
token
secret
authorization
headers.authorization
```

These values are censored as:

```text
***
```

For example, a payload containing:

```ts
{
  token: 'secret-token',
  userId: 'user-123',
}
```

is redacted by Pino before output.

Redaction is an additional protection layer. It does not replace careful logging
practices.

Do not deliberately include sensitive information in log messages simply because
Pino has redaction configured.

---

## Error Logging

Use the Xeno.JS error API:

```ts
try {
  await processOrder()
} catch (error) {
  logger.error('Unable to process order', error)
}
```

Xeno.JS passes the error to the Pino adapter.

The adapter adds the error to the structured payload and calls Pino's `error()`
method.

Pino is configured with its standard error serializer, so error information is
serialized using Pino's error serialization behavior.

---

## Complete Example

A complete application configuration can look like this:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withPinoLogger } from '@xeno-js/pino'

const app = new XenoAppBuilder()

app.addLogger(
  withPinoLogger((options, config) => {
    options.destination = 'file'
    options.filePath = config.get('LOG_FILE_PATH', 'logs/app.log')
    options.env = 'production'
    options.prettyPrint = false
  }),
)

const application = await app.build()

const { logger } = application.getServices()

logger.info('Application started')

logger.warn('Example warning')

try {
  throw new Error('Example failure')
} catch (error) {
  logger.error('Application operation failed', error)
}
```

This configuration:

- registers the Pino provider without enabling Console;
- writes Pino logs to `logs/app.log`;
- uses structured JSON output;
- uses the Xeno.JS safe context;
- applies Pino redaction;
- resolves the application logger through the built application's service proxy.

---

## Development Configuration

For local development, pretty printing can make Pino output easier to read:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withPinoLogger } from '@xeno-js/pino'

const app = new XenoAppBuilder()

app.addLogger(
  withPinoLogger((options) => {
    options.env = 'development'
    options.destination = 'stdout'
    options.prettyPrint = true
  }),
)

await app.build()
```

Make sure `pino-pretty` is installed when using this configuration.

---

## Production Configuration

For structured logs, disable pretty printing:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withPinoLogger } from '@xeno-js/pino'

const app = new XenoAppBuilder()

app.addLogger(
  withPinoLogger((options) => {
    options.env = 'production'
    options.destination = 'stdout'
    options.prettyPrint = false
  }),
)

await app.build()
```

This keeps the output in Pino's structured format, which can then be consumed by
the application's log collection infrastructure.

---

## Common Problems

### Pino Does Not Produce Any Logs

Make sure you register the Pino plugin:

```ts
app.addLogger(withPinoLogger(() => {}))
```

The plugin is enabled by default; installing its package alone does not register
it with the application.

Also check whether the configured Xeno.JS logger minimum level filters out the
event before it reaches the provider.

### Pino Cannot Be Imported

If `withPinoLogger` cannot be imported, make sure the provider package is
installed:

```bash
npm install @xeno-js/pino
```

Install any external dependencies required by the adapter according to its
package metadata. They are not optional peer dependencies of `@xeno-js/core`.

### Pretty Printing Fails

If `prettyPrint` is enabled, install:

```bash
npm install pino-pretty
```

Xeno.JS configures the Pino transport with `pino-pretty` when pretty printing is
enabled.

### Logs Are Not Written to the Expected File

Check:

```ts
app.addLogger(
  withPinoLogger((options) => {
    options.destination = 'file'
    options.filePath = 'logs/app.log'
  }),
)
```

If `filePath` is omitted, Xeno.JS uses:

```text
logs/app.log
```

Also make sure the Node.js process has permission to write to the selected
directory.

### I Cannot Resolve a Pino Logger Directly

This is expected.

Xeno.JS registers the application `ILogger`, not the Pino instance, under the DI
token:

```ts
TOKENS.LOGGER
```

Resolve the logger from the built application:

```ts
const { logger } = application.getServices()
```

and use the Xeno.JS logging API.

Pino remains an implementation behind the Xeno.JS logger abstraction.

---

## Related Docs

- [Logging Overview](./overview)
- [Sentry Node](./sentry-node)
- [Dependency Injection — Resolution](../dependency-injection/resolution)
- [Context & Scopes](../fundamentals/context-scopes)
- [Pipelines — Logging](../application/pipelines/logging)

---

---

<a id="source-4"></a>

# Source: `docs/application/logging/sentry-node.md`

## Introduction

Use Sentry Node when you want Xeno.JS logging to report warnings and errors to
Sentry while keeping the application code independent from the Sentry SDK.

Xeno.JS integrates `@sentry/node` as a logging provider. You configure it
through `AppBuilder.addLogger()`, while application code continues to use the
Xeno.JS `ILogger` abstraction.

## Prerequisites

Install the Xeno.JS packages used by your application and the Sentry Node SDK:

```bash
npm install @xeno-js/core @xeno-js/sentry-node
```

Register the `@xeno-js/sentry-node` plugin to add Sentry as a logging provider.

Configure the integration through `withSentryNodeLogger()` rather than calling
Sentry APIs from application code.

## Configure Sentry

Configure Sentry through `AppBuilder.addLogger()`:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withSentryNodeLogger } from '@xeno-js/sentry-node'

const app = new XenoAppBuilder()

app.addLogger(
  withSentryNodeLogger((options, config) => {
    options.dsn = config.get('SENTRY_DSN') ?? ''
    options.release = config.get('SENTRY_RELEASE') ?? ''
    options.environment = config.get('SENTRY_ENVIRONMENT') ?? ''
  }),
)
```

The Sentry configuration exposes:

| Option        | Description                                                                   |
| ------------- | ----------------------------------------------------------------------------- |
| `dsn`         | Sentry Data Source Name used to send events to the configured Sentry project. |
| `release`     | Release associated with the Sentry events.                                    |
| `environment` | Environment associated with the Sentry events.                                |

The plugin initializes these options from `SENTRY_DSN`, `SENTRY_RELEASE`, and
`SENTRY_ENVIRONMENT` via the configuration service. Its setup callback can
override them.

### Configure the log level

The application log level is configured through the same `LoggerConfig` used by
the other Xeno.JS providers:

```ts
import { LOG_LEVEL } from '@xeno-js/shared'
import { XenoAppBuilder, withConsoleLogger } from '@xeno-js/core'
import { withSentryNodeLogger } from '@xeno-js/sentry-node'

const app = new XenoAppBuilder()

app.addLogger([
  withConsoleLogger((options) => {
    options.level = LOG_LEVEL.INFO
  }),
  withSentryNodeLogger((options, config) => {
    options.dsn = config.get('SENTRY_DSN') ?? ''
    options.release = config.get('SENTRY_RELEASE') ?? ''
    options.environment = config.get('SENTRY_ENVIRONMENT') ?? ''
  }),
])
```

Xeno.JS supports these log levels:

- `DEBUG`
- `INFO`
- `WARN`
- `ERROR`

The configured level is the minimum level accepted by the application logger.

Sentry has an additional provider-specific rule: **only `WARN` and `ERROR`
messages are sent to Sentry**. `DEBUG` and `INFO` messages are not converted
into Sentry events.

For example, with:

```ts
options.level = LOG_LEVEL.ERROR
```

only errors can reach Sentry.

With:

```ts
options.level = LOG_LEVEL.INFO
```

the application logger accepts `INFO`, `WARN`, and `ERROR`, but Sentry still
receives only `WARN` and `ERROR`.

## Build the application

The logger is registered during application bootstrap:

```ts
const application = await app.build()
```

Configure Sentry before calling `build()`.

A complete minimal setup is:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withSentryNodeLogger } from '@xeno-js/sentry-node'

const app = new XenoAppBuilder()

app.addLogger(
  withSentryNodeLogger((options, config) => {
    options.dsn = config.get('SENTRY_DSN') ?? ''
    options.release = config.get('SENTRY_RELEASE') ?? ''
    options.environment = config.get('SENTRY_ENVIRONMENT') ?? ''
  }),
)

const application = await app.build()
```

## Resolve the application logger

Xeno.JS does not expose a separate DI token for the Sentry logger.

Resolve the application logger through the built application's service proxy:

```ts
const { logger } = application.getServices()
```

The resolved object is the Xeno.JS `ILogger` abstraction.

This is important when multiple providers are configured. For example:

```text
ILogger
  ├── Console
  ├── Pino
  ├── Sentry
  └── Custom logger
```

Your application calls the same logger regardless of which providers are
enabled.

## Write logs

Use the Xeno.JS logger methods:

```ts
logger.debug('Debug information')
logger.info('User created')
logger.warn('User profile is incomplete')
logger.error('Failed to create user', error)
```

The public error signature accepts the message first and the error as the second
argument:

```ts
logger.error('Failed to create user', error)
```

You do not need to import or call the Sentry SDK from application code.

### Example in an application service

```ts
import type { ILogger } from '@xeno-js/shared'

export class UserService {
  constructor(private readonly logger: ILogger) {}

  async createUser(): Promise<void> {
    try {
      // Application work...
      this.logger.info('User created')
    } catch (error) {
      this.logger.error('Failed to create user', error)
      throw error
    }
  }
}
```

When Sentry is enabled, the same `logger.error()` call is forwarded to Sentry
according to the Xeno.JS Sentry provider rules.

## Warning behavior

Sentry receives `WARN` messages.

When a warning does not include an error object:

```ts
logger.warn('User profile is incomplete')
```

Xeno.JS calls Sentry's message capture mechanism with warning severity.

Conceptually, the resulting Sentry event is a warning message rather than an
exception.

If a warning is associated with an error object, Xeno.JS captures the error as
an exception:

```ts
logger.warn('User profile validation failed', error)
```

The provider therefore distinguishes between:

```text
WARN + no error
    ↓
Sentry message with warning severity

WARN + error
    ↓
Sentry exception
```

## Error behavior

Errors are sent to Sentry only when the configured application log level allows
them.

When an error includes an error object:

```ts
logger.error('Database operation failed', error)
```

Xeno.JS uses Sentry's exception capture mechanism.

When no error object is supplied:

```ts
logger.error('Database operation failed')
```

the Sentry provider uses Sentry's message capture path with warning severity.

Therefore, when reporting an actual exception, pass the `Error` object:

```ts
try {
  await repository.save(user)
} catch (error) {
  logger.error('Failed to save user', error)
  throw error
}
```

This preserves the exception for Sentry rather than reducing the event to a
message.

## Zod errors

Xeno.JS filters out events whose original exception is a `ZodError`.

This is done before the event is sent to Sentry.

As a result, validation failures represented by `ZodError` are not reported as
Sentry events through this integration.

This behavior is specific to the Xeno.JS Sentry integration and should not be
interpreted as a general Sentry SDK behavior.

## Safe context

Xeno.JS does not send the complete request context to logging providers.

The application logger builds a restricted safe context for observability. This
context is intended to provide useful diagnostic information without
automatically serializing arbitrary request/application state.

The safe context can include information such as:

- `userId`
- `tenantId`
- `requestId`
- request `path`
- request `origin`
- masked client IP
- `correlationId`
- `spanId`
- `startTime`
- `parentSpanId`
- applicable messaging context

The full request context can contain additional information that is not
appropriate for automatic logging, such as authentication details, cookies, CSRF
values, user profile information, or transport-specific objects.

For Sentry, Xeno.JS attaches the available safe context to the Sentry scope as
event extras.

### Safe context is not a security boundary for application messages

Safe context does not prevent application code from explicitly sending sensitive
information to a logger.

Avoid code such as:

```ts
logger.info(`User password: ${password}`)
```

or:

```ts
logger.error('Authentication failed', {
  token,
  authorizationHeader,
})
```

Do not explicitly log:

- passwords;
- access tokens;
- refresh tokens;
- API keys;
- authorization credentials;
- session cookies;
- secrets;
- sensitive personal data that is not required for diagnostics.

The safe-context mechanism limits automatically propagated context. It cannot
remove sensitive information that application code explicitly places in a log
message or error.

## What reaches Sentry?

For a typical Xeno.JS log:

```text
Application code
      │
      ▼
  ILogger
      │
      ├── minimum log-level check
      │
      ▼
  Sentry provider
      │
      ├── DEBUG → ignored
      ├── INFO  → ignored
      ├── WARN  → Sentry message or exception
      └── ERROR → Sentry exception/message
```

The Sentry provider adds the available safe context as event extras.

Sentry itself then stores and presents the resulting event according to the
Sentry project configuration.

## Configure Sentry together with other providers

Sentry does not have to be the only logging provider.

For example, you can enable Console, Pino, and Sentry together:

```ts
import { XenoAppBuilder, withConsoleLogger } from '@xeno-js/core'
import { withPinoLogger } from '@xeno-js/pino'
import { withSentryNodeLogger } from '@xeno-js/sentry-node'

const app = new XenoAppBuilder()

app.addLogger([
  withConsoleLogger(),
  withPinoLogger((options) => {
    options.destination = 'stdout'
  }),
  withSentryNodeLogger((options, config) => {
    options.dsn = config.get('SENTRY_DSN') ?? ''
    options.release = config.get('SENTRY_RELEASE') ?? ''
    options.environment = config.get('SENTRY_ENVIRONMENT') ?? ''
  }),
])

await app.build()
```

The same application log can then be processed by the configured providers.

For example:

```text
logger.error('Failed to process payment', error)
        │
        ├── Console
        ├── Pino
        └── Sentry
```

Each provider handles the log according to its own integration rules.

## Sentry initialization performed by Xeno.JS

When the plugin is enabled, it creates the Sentry logger through
`SentryLoggerFactory.create(options)`, passing the configured DSN, release, and
environment.

The exact `@sentry/node` initialization behavior, including integrations,
sampling rates, and event filters, cannot be verified from the current context;
checking the factory implementation is required.

## Complete example

A typical application can configure Sentry and keep application logging
independent from Sentry:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { LOG_LEVEL } from '@xeno-js/shared'
import { withSentryNodeLogger } from '@xeno-js/sentry-node'

async function bootstrap() {
  const app = new XenoAppBuilder()

  app.addLogger(
    withSentryNodeLogger((options, config) => {
      options.dsn = config.get('SENTRY_DSN') ?? ''
      options.release = config.get('SENTRY_RELEASE') ?? ''
      options.environment = config.get('SENTRY_ENVIRONMENT') ?? ''
    }),
  )

  const application = await app.build()

  const { logger } = application.getServices()

  logger.info('Application started')

  try {
    throw new Error('Example failure')
  } catch (error) {
    logger.error('Application operation failed', error)
  }
}

await bootstrap()
```

The application knows only about `ILogger`. It does not need to depend on the
Sentry API to report the error.

## Troubleshooting

### Sentry does not receive any events

Check the following:

1. `app.addLogger()` is called.
2. `withSentryNodeLogger()` is passed to `addLogger()`.
3. A valid `dsn` is provided.
4. `app.build()` is called after the logger configuration.
5. The log level allows the event.
6. The event is `WARN` or `ERROR`.
7. The exception is not a `ZodError`.

For example:

```ts
app.addLogger([
  withConsoleLogger((options) => {
    options.level = LOG_LEVEL.WARN
  }),
  withSentryNodeLogger((options, config) => {
    options.dsn = config.get('SENTRY_DSN') ?? ''
    options.release = config.get('SENTRY_RELEASE') ?? ''
    options.environment = config.get('SENTRY_ENVIRONMENT') ?? ''
  }),
])
```

### Bootstrap fails with a Sentry configuration error

Check that `withSentryNodeLogger()` is registered and that the DSN is
configured:

```ts
app.addLogger(
  withSentryNodeLogger((options, config) => {
    options.dsn = config.get('SENTRY_DSN') ?? ''
  }),
)
```

and verify that `SENTRY_DSN` is actually defined.

### `INFO` logs do not appear in Sentry

This is expected.

The Sentry provider only forwards `WARN` and `ERROR` levels.

For example:

```ts
logger.info('User created')
```

is not sent to Sentry.

Use an appropriate warning or error when the event should be reported to Sentry:

```ts
logger.warn('User profile requires attention')
```

or:

```ts
logger.error('User creation failed', error)
```

### An error appears as a message instead of an exception

Pass the actual error object to `logger.error()`:

```ts
logger.error('Failed to process request', error)
```

instead of only logging a string:

```ts
logger.error('Failed to process request')
```

The Sentry integration uses `captureException()` when an error object is
provided.

### Sensitive information appears in Sentry

The safe context only controls automatically propagated contextual information.

Inspect the message, error, and application data passed to the logger directly.

Remove secrets, credentials, tokens, passwords, and unnecessary sensitive
information from application-level log calls.

## Related documentation

- [Observability Overview](./overview)
- [Pino Logger](./pino-logger)
- [Dependency Injection — Resolution](../dependency-injection/resolution)
- [Fundamentals — Context & Scopes](../fundamentals/context-scopes)
- [Application — Pipelines](../application/pipelines/overview)

---

---

<a id="source-5"></a>

# Source: `docs/application/logging/sentry-vue.md`

## Sentry for Vue frontends

`@xeno-js/sentry-vue` connects a Vue application to Sentry through the Xeno.JS
logger. Register `withSentryVueLogger()` during bootstrap, then use the
application's `ILogger` instead of resolving a separate Sentry logger.

The plugin initializes `@sentry/vue` with browser tracing and session replay
integrations. It can attach tracing to Vue Router when a router is supplied. For
server applications, use the separate
[Sentry Node logger](/docs/application/logging/sentry-node).

## Install

In an existing Vue 3 application using Vite, install the Xeno.JS packages and
Pinia integration:

```bash
npm install @xeno-js/core @xeno-js/vue @xeno-js/sentry-vue pinia
```

The Sentry Vue package depends on `@sentry/vue`. Install `vue-router` when your
application uses routing. The plugin works without a router.

## Configure Sentry during Vue bootstrap

This example enables Sentry in production browser builds, attaches the Vue app
and router, and disables replay sampling until your application has established
an appropriate privacy and consent policy.

Set `VITE_SENTRY_DSN` to the public DSN for the frontend Sentry project. Vite
exposes `VITE_` variables in client bundles, so do not put secrets in them.

```ts
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import { LOG_LEVEL, withConsoleLogger, XenoAppBuilder } from '@xeno-js/core'
import { withSentryVueLogger } from '@xeno-js/sentry-vue'
import { ServicesUtils, withPiniaContext } from '@xeno-js/vue'

import App from './App.vue'
import { router } from './router'

const app = createApp(App)
app.use(createPinia())

const xenoApp = await new XenoAppBuilder(undefined, 'vite')
  .addContext(withPiniaContext())
  .addLogger([
    withConsoleLogger((options) => {
      options.isEnabled = config.getEnvironment() === 'development'
      options.level = LOG_LEVEL.DEBUG
    }),
    withSentryVueLogger((options, config) => {
      options.isEnabled =
        !import.meta.env.SSR && config.getEnvironment() === 'production'
      if (!options.isEnabled) return

      const dsn = config.getOrThrow('VITE_SENTRY_DSN')

      options.dsn = dsn
      options.environment = config.getEnvironment()
      options.app = app
      options.router = router
      options.level = LOG_LEVEL.WARN
      options.tracesSampleRate = 0.2
      options.replaysSessionSampleRate = 0
      options.replaysOnErrorSampleRate = 0
    }),
  ])
  .build()

ServicesUtils.provide(app, xenoApp)
app.use(router)
app.mount('#app')
```

`App.vue` and `router` belong to the application. Omit `options.router` and
`app.use(router)` when the frontend does not use Vue Router. Supplying the
router before installing it lets Sentry configure router tracing during
bootstrap.

`withPiniaContext()` provides the browser context used by Xeno.JS services.
Install Pinia before building and call `ServicesUtils.provide()` before using
the logger from Vue components. See [Vue Plugin](/docs/vue/plugin-vue) for the
context and service lifecycle.

### Options

| Option                     | Purpose                                                                           |
| -------------------------- | --------------------------------------------------------------------------------- |
| `isEnabled`                | Controls whether the Xeno.JS plugin registers the provider. Defaults to `true`.   |
| `dsn`                      | Identifies the Sentry project receiving browser events.                           |
| `app`                      | Supplies the Vue app to the Sentry Vue SDK.                                       |
| `router`                   | Supplies Vue Router to the browser tracing integration. Optional.                 |
| `environment`, `release`   | Label events with the deployment environment and release.                         |
| `level`                    | Sets the Sentry logger's minimum Xeno.JS log level. Defaults to `LOG_LEVEL.WARN`. |
| `tracesSampleRate`         | Sampling rate for tracing. Defaults to `0.2`.                                     |
| `replaysSessionSampleRate` | Sampling rate for session replay. Defaults to `0.1`.                              |
| `replaysOnErrorSampleRate` | Error-triggered replay sampling rate. Defaults to `1.0`.                          |

Rates range from `0` to `1`. The factory includes browser tracing and replay
integrations even when a sample rate is zero. Supplying a custom `integrations`
option replaces the factory's integration array; include any integrations you
still need.

`isEnabled` is the Xeno.JS plugin switch. When it is `false`, the plugin skips
the provider factory and does not initialize Sentry. It is distinct from Sentry
SDK options passed through to `Sentry.init()`.

The setup callback receives the Xeno.JS configuration service. In Vite mode, its
keys use the `VITE_` prefix and `getEnvironment()` reads the Vite mode. The
current Vite configuration service converts a missing value to the string
`"undefined"`, so `getOrThrow()` alone is not sufficient to validate a missing
DSN. The example checks the Vite environment variable directly.

## Use the application logger

After building and providing the application, use the combined Xeno.JS logger in
a Vue component or composable:

```ts
import { ServicesUtils } from '@xeno-js/vue'

const { logger } = ServicesUtils.useApp()

logger.warn('Unable to load the latest catalog')
logger.error('Checkout request failed', new Error('Request failed'))
```

Outside Vue presentation code, use `xenoApp.getServices().logger` or inject
`ILogger` into an application service. `logger.error()` takes the message first
and an error value second; `logger.warn()`, `logger.info()`, and
`logger.debug()` take a message.

Use the logger after `ServicesUtils.provide()` when the Vue context plugin is
active. The logger reads the current context when it emits an event.

### Log levels and event mapping

The application logger applies its minimum level before the Sentry Vue provider
applies `options.level`. Both thresholds must allow an event. In the bootstrap
example, the console plugin sets the application minimum to `DEBUG`, and the
Sentry provider accepts `WARN` and above.

Sentry Vue maps `DEBUG`, `INFO`, `WARN`, and `ERROR` to Sentry severities
`debug`, `info`, `warning`, and `error`. Unlike the Node provider, the Vue
provider can receive `DEBUG` and `INFO` when its level and the application
minimum allow them.

The provider calls `captureException()` when it receives an error value;
otherwise it calls `captureMessage()` with the mapped severity. The core
application logger currently normalizes every error argument into an object,
including an omitted error on `warn()`, `info()`, and `debug()`. Consequently,
normal calls through `ILogger` reach the Vue provider's exception path even for
a warning without an explicit error. The original `Error` is also normalized to
an object before reaching the provider. Check event grouping and stack
presentation in your Sentry project rather than assuming the original exception
object is passed unchanged.

The provider attaches the log message as `log_message` on the Sentry scope when
it captures an exception. It adds the available safe logging context as event
extras.

## Tracing, replay, and privacy

The plugin creates a browser tracing integration with the configured router, or
without a router when none is supplied. `tracesSampleRate` controls the
configured trace sampling rate; it does not change the Sentry logger's severity
threshold.

Session replay can contain information from a user's browser session. Review
Sentry's replay masking and blocking settings, your application's consent
requirements, and your data-retention policy before collecting sessions. The
bootstrap example sets both replay sampling rates to zero. Raising either rate
should be a deliberate production decision.

Xeno.JS restricts the request fields copied into the logging context, but the
result can still contain identifiers, paths, origins, tracing information, and
messaging metadata. Error messages, normalized error properties, browser
instrumentation, and replay can contain additional data. Do not log credentials,
tokens, personal data, or sensitive payloads. Configure Sentry-side filtering
and SDK hooks to suit your application; the Vue plugin does not configure a Zod
error filter.

## Vite SSG and server-side rendering

The browser SDK should only initialize in the browser. In a Vite SSG callback,
use the callback's `isClient` flag when configuring the provider:

```ts
withSentryVueLogger((options, config) => {
  options.isEnabled = isClient && config.getEnvironment() === 'production'
  if (!options.isEnabled) return

  const dsn = config.getOrThrow('VITE_SENTRY_DSN')

  options.dsn = dsn
  options.app = app
  options.router = router
  options.environment = config.getEnvironment()
  options.replaysSessionSampleRate = 0
  options.replaysOnErrorSampleRate = 0
})
```

Here `isClient`, `app`, and `router` are supplied by the application's SSG
bootstrap. Configure the plugin in the same `addLogger()` call as other logger
providers. Server-side error reporting requires a separate server integration
and lifecycle.

## Common problems

### No events appear in Sentry

Check that `isEnabled` is true in the running browser, that the DSN points to
the intended project, and that the application was built with the plugin. Verify
both log-level thresholds, network requests, SDK filtering, and Sentry project
settings. An enabled plugin with a missing or invalid DSN is not a working
transport.

### Debug and info logs do not appear

The Vue provider defaults to `WARN`. Set `options.level` to `LOG_LEVEL.DEBUG` or
`LOG_LEVEL.INFO` and ensure the application logger's minimum also permits that
level. Review the volume and privacy impact before sending lower-level logs.

### Router traces are missing

Pass the same router instance used by the app in `options.router`, build the
Xeno application before installing the router, and choose a nonzero trace sample
rate. Sampling means not every navigation is expected to appear.

### Sentry initializes during static generation

Set `isEnabled` from `isClient` in Vite SSG or guard with the runtime's
server-side flag. Do not read a required browser DSN or initialize the browser
provider on the server path.

### The console or Sentry provider is missing

Pass all logger plugins in one `addLogger([...])` call. The builder ignores
later `addLogger()` calls. The plugin-array form does not enable Console by
itself; add `withConsoleLogger()` when you need it.

### A warning appears as an exception

The core logger currently passes a normalized error object even for
`logger.warn()`. The Sentry Vue provider therefore calls `captureException()`
for that call. See
[Log levels and event mapping](#log-levels-and-event-mapping).

## Related documentation

- [Logging Overview](/docs/application/logging/overview)
- [Sentry Node Logger](/docs/application/logging/sentry-node)
- [Vue Plugin](/docs/vue/plugin-vue)
- [Context Scopes](/docs/fundamentals/context-scopes)

---

---

<a id="source-6"></a>

# Source: `docs/application/pipelines/logging.md`

## Introduction

The Logging Pipeline logs the execution of commands and queries handled through
the Xeno.JS CQRS pipeline.

It records:

- when a request starts;
- when a request completes successfully;
- when a request returns a failed `Result`;
- when request execution throws an exception.

The pipeline uses the application logger, so you can configure where logs are
written without changing your commands or handlers.

## Before you start

You need:

- an `XenoAppBuilder` instance;
- a command or query handled through the Xeno.JS mediator;
- the CQRS pipeline enabled with `addPipeline()`.

You do **not** need to register the `LoggingPipeline` manually.

## Enable logging

Enable the CQRS pipeline through `XenoAppBuilder`:

```ts
import { XenoAppBuilder } from '@xeno-js/core'

const app = new XenoAppBuilder().addPipeline()

const application = await app.build()
```

When `addPipeline()` is configured, the CQRS module adds the Logging Pipeline as
a common behavior. It also adds a logger if none has been registered.

Both commands and queries use the logging pipeline.

## Configure the logger

Use `addLogger()` to configure the logger used by the application.

For example, to configure the minimum log level:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { LOG_LEVEL } from '@xeno-js/shared'

const app = new XenoAppBuilder()
  .addLogger((config) => {
    config.level = LOG_LEVEL.INFO
  })
  .addPipeline()

const application = await app.build()
```

The `addLogger()` setup callback configures core logger settings. Provider
options are configured through the provider's logger plugin:

| Option          | Description                                                    |
| --------------- | -------------------------------------------------------------- |
| `level`         | Minimum log level forwarded to the configured logging clients. |
| `console`       | Enables or disables console logging.                           |
| `customLoggers` | Registers custom logging clients.                              |

`addLogger()` also accepts a logger plugin or an array of plugins. If the CQRS
module needs a logger and none has been registered, it adds one during
application build. See the [logger overview](../logging/overview) for logger
defaults and provider setup.

## Choose the log level

Xeno.JS filters messages below the configured minimum level.

For example:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { LOG_LEVEL } from '@xeno-js/shared'

const app = new XenoAppBuilder()
  .addLogger((config) => {
    config.level = LOG_LEVEL.WARN
  })
  .addPipeline()

const application = await app.build()
```

With `LOG_LEVEL.WARN`, informational messages from the Logging Pipeline are
filtered out, while warnings and errors remain available to logging clients.

The Logging Pipeline itself uses:

- `info` for request start;
- `info` for successful completion;
- `error` for failed `Result` values;
- `error` for thrown exceptions.

## Configure console logging

The default Console provider is enabled by default. The CQRS module adds a
logger when needed, so logging works without manually registering one.

You can configure it explicitly:

```ts
const app = new XenoAppBuilder().addLogger((config) => {
  config.console = true
})

app.addPipeline()

const application = await app.build()
```

To disable the console logger:

```ts
const app = new XenoAppBuilder().addLogger((config) => {
  config.console = false
})

app.addPipeline()

const application = await app.build()
```

Disabling the Console provider does not disable the Logging Pipeline itself.
Logs can still be sent to other configured logger clients.

## Configure Pino

Register the Pino logger plugin from `@xeno-js/pino`. The plugin callback
receives Pino-specific options; these are separate from the core logger setup
callback:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withPinoLogger } from '@xeno-js/pino'

const app = new XenoAppBuilder().addLogger(
  withPinoLogger((options) => {
    options.destination = 'stdout'
  }),
)

app.addPipeline()

const application = await app.build()
```

### Write Pino logs to a file

Set `destination` to `file` and provide a file path through the Pino plugin:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withPinoLogger } from '@xeno-js/pino'

const app = new XenoAppBuilder().addLogger(
  withPinoLogger((options) => {
    options.destination = 'file'
    options.filePath = 'logs/app.log'
  }),
)
```

Supported destinations are:

- `stdout`;
- `file`.

If `filePath` is omitted when using the `file` destination, Xeno.JS uses
`logs/app.log`.

For Pino behavior, including formatting, defaults, and redaction, see
[Pino Logger](../logging/pino-logger). Those options belong to the Pino
integration, not to the core logger configuration.

## Configure Sentry

Register the Sentry Node logger plugin from `@xeno-js/sentry-node`. Sentry
provider options belong to that plugin; the core logger setup callback does not
configure them.

Configure the Sentry DSN:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withSentryNodeLogger } from '@xeno-js/sentry-node'

const app = new XenoAppBuilder().addLogger(
  withSentryNodeLogger((options) => {
    options.dsn = process.env.SENTRY_DSN ?? ''
    options.environment = process.env.NODE_ENV
  }),
)

app.addPipeline()

const application = await app.build()
```

The Sentry Node integration requires a DSN when enabled. Its environment and
other provider-specific behavior are described in
[Sentry Node Logger](../logging/sentry-node).

## Use multiple logging clients

You can configure more than one logging destination.

For example, console and Pino can be enabled together:

```ts
import { XenoAppBuilder, withConsoleLogger } from '@xeno-js/core'
import { withPinoLogger } from '@xeno-js/pino'
import { LOG_LEVEL } from '@xeno-js/shared'

const app = new XenoAppBuilder()
  .addLogger([
    withConsoleLogger((options) => {
      options.level = LOG_LEVEL.INFO
    }),
    withPinoLogger((options) => {
      options.destination = 'stdout'
    }),
  ])
  .addPipeline()

const application = await app.build()
```

Pass provider plugins together in one `addLogger()` call. The application logger
forwards each accepted message to the configured logging clients.

## Add a custom logger

Use `customLoggers` when you need to integrate a logging destination that is not
provided by Xeno.JS.

A custom logger must implement the `ILoggerClient` contract from
`@xeno-js/shared`.

For example:

```ts
import type { ILoggerClient } from '@xeno-js/shared'

const customLogger = (): ILoggerClient => ({
  track(level, message, context, error) {
    console.log({
      level,
      message,
      context,
      error,
    })
  },
})
```

Register it with `addLogger()`:

```ts
const app = new XenoAppBuilder().addLogger((config) => {
  config.customLoggers = [() => customLogger()]
})

app.addPipeline()

const application = await app.build()
```

Custom logger factories use the scope contract described in the
[logger overview](../logging/overview) when they need service resolution.

## What does the Logging Pipeline log?

For a request such as:

```ts
{
  type: 'COMMAND',
  intent: 'CREATE_USER'
}
```

the pipeline logs the start of the request:

```text
Handling COMMAND CREATE_USER
```

When the request succeeds:

```text
Successfully handled COMMAND CREATE_USER
```

When the handler returns a failed `Result`:

```text
Failed to handle COMMAND CREATE_USER: User already exists
```

When request execution throws:

```text
Exception while handling COMMAND CREATE_USER
```

The failed `Result` is returned unchanged.

The Logging Pipeline logs a thrown value and rethrows it. When the CQRS
Exception Pipeline is also active, it can convert that thrown value into a
failed `Result`; see [Exception Pipeline](./exception).

## Logging failed Results

A command or query can complete without throwing but still return a failed
`Result`.

For example:

```ts
const result = await mediator.send(command)

if (!result.isOk()) {
  // The Logging Pipeline has already logged the failure.
}
```

The Logging Pipeline does not replace or transform the `Result`.

It logs the error and returns the same result to the caller.

## Logging thrown exceptions

If execution throws an exception, the Logging Pipeline logs it and rethrows it
to the surrounding CQRS behavior. With the Exception Pipeline enabled, the
mediator returns a failed `Result`:

```ts
const result = await mediator.send(command)

if (!result.isOk()) {
  // The failure is available as a Result.
}
```

The Logging Pipeline does not convert exceptions itself; the CQRS Exception
Pipeline handles that conversion.

## Complete example

A typical application can configure logging and the CQRS pipeline together:

```ts
import { XenoAppBuilder, withConsoleLogger } from '@xeno-js/core'
import { withPinoLogger } from '@xeno-js/pino'
import { LOG_LEVEL } from '@xeno-js/shared'

const app = new XenoAppBuilder()
  .addLogger([
    withConsoleLogger((options) => {
      options.level = LOG_LEVEL.INFO
    }),
    withPinoLogger((options) => {
      options.destination = 'stdout'
    }),
  ])
  .addPipeline()

const application = await app.build()
```

After the application is built, commands and queries executed through the
configured mediator are logged automatically.

## Logging is not running

If commands or queries are not producing logs, check:

1. `addPipeline()` is called;
2. the command or query is executed through the Xeno.JS mediator;
3. console logging has not been disabled with `config.console = false`;
4. the configured minimum `config.level` does not filter the messages you
   expect;
5. at least one logging client is configured.

For example, if you configure:

```ts
config.level = LOG_LEVEL.WARN
```

the normal `info` messages generated by the Logging Pipeline will not be
forwarded.

## Pino logs are not written to the expected destination

Check that the Pino plugin is passed to `addLogger()` and configure its
destination in the plugin callback:

```ts
app.addLogger(
  withPinoLogger((options) => {
    options.destination = 'file'
    options.filePath = 'logs/app.log'
  }),
)
```

For a file destination:

- `destination` must be `file`;
- `filePath` should point to the desired output file.

For standard output:

```ts
app.addLogger(
  withPinoLogger((options) => {
    options.destination = 'stdout'
  }),
)
```

**[Pino Logger Documentation](../logging/pino-logger)**

## Sentry events are not appearing

Check that the Sentry Node plugin is registered and that it receives a DSN:

```ts
import { withSentryNodeLogger } from '@xeno-js/sentry-node'

app.addLogger(
  withSentryNodeLogger((options, config) => {
    options.dsn = config.get('SENTRY_DSN') ?? ''
  }),
)
```

For the Sentry plugin's configuration and environment-variable behavior, see
[Sentry Node Logger](../logging/sentry-node).

---

## See Loggers Documentation

- [Loggers Documentation](../logging/overview)

---

## Related docs

- [Application overview](../overview)
- [Exception Pipeline](./exception)
- [Performance Pipeline](./performance)
- [Creating Commands](../cqrs/command)
- [Creating Queries](../cqrs/query)
- [Creating Handlers](../cqrs/handler)
- [Loggers Documentation](../logging/overview)

---
