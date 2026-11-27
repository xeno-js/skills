---
name: xeno-js
description:
  Use whenever a task involves Xeno.JS or xeno-js TypeScript applications,
  @xeno-js/core, @xeno-js/shared, @xeno-js/vue, XenoAppBuilder, DDD, CQRS
  commands, queries, handlers or the mediator, pipelines, dependency injection
  and Singleton/Scoped/Transient lifetimes, RequestContext or AsyncLocalStorage,
  repositories, data sources, database schemas, Unit of Work, outbox, HTTP
  controllers/routes/adapters, authentication, authorization, security
  middleware, logging, caching, Vue integration, or the Xeno.JS CLI. Consult
  these references when designing, implementing, debugging, reviewing, or
  documenting Xeno.JS features—even if the request is phrased as a general
  TypeScript architecture question. Verify exact package APIs and signatures
  instead of inferring them from names. Do not use for unrelated TypeScript
  work.
---

# Xeno.JS engineering skill

Use the bundled references as the documentation corpus for Xeno.JS. The
reference pages preserve the technical Markdown from the website docs; consult
the closest domain page before proposing code or describing lifecycle behavior.
If a requested API or behavior is not established there, inspect the current
public exports/implementation when available or state that it cannot be
verified. Do not guess, silently rename, or normalize an API.

## Non-negotiable architecture

- Keep the **Domain** independent: domain entities, value objects, aggregates,
  events, and rules do not depend on Xeno.JS, transport, persistence, or
  infrastructure.
- Let the **Application** coordinate use cases. Commands, queries, handlers, and
  pipeline behaviors orchestrate domain work and depend on application-facing
  contracts.
- Let **Infrastructure** implement those contracts (repositories, data sources,
  database/cache clients, logging and external services). Wire implementations
  at the composition root.
- Keep **Presentation** adapters (HTTP, worker, Vue) focused on translating
  input/output and dispatching application work. Do not move business rules into
  controllers or transport middleware.
- Compose applications with `XenoAppBuilder`: queue services, modules, context,
  middleware, pipelines and integrations, then call `build()` to initialize the
  queued configuration. Use the documented registration methods and package
  imports; do not bypass the composition root with invented APIs.
- Respect the service lifetimes. `Singleton` shares an instance at container
  lifetime; `Scoped` shares within a service scope; `Transient` creates an
  instance per resolution. Never capture a scoped dependency in a singleton (a
  captive dependency), and do not resolve scoped services from the root
  container.
- Treat execution context and service scope as related but distinct concerns. In
  the default Node context, Xeno.JS carries `RequestContext` through
  asynchronous work with `AsyncLocalStorage`; this is not a universal mechanism
  for every runtime. The Vue integration uses its documented Vue/Pinia context
  rather than Node's `AsyncLocalStorage`.
- CQRS and explicit DI add types, registrations, handlers, and wiring. Explain
  that cost honestly: the separation supports maintainable boundaries, scoped
  execution, and testable use cases; it is not zero-boilerplate.

## Feature workflow

1. **Classify the change.** Identify the package and layer: domain model,
   application use case, infrastructure adapter, presentation integration, or
   composition/configuration. Read the matching reference below.
2. **Define the boundary first.** Keep domain types framework-independent. Put
   use-case orchestration and application-facing contracts in Application; put
   database, network, cache, logging, and provider implementations in
   Infrastructure.
3. **Choose the application mechanism.** Use the documented
   command/query/handler and mediator path for CQRS work; use documented
   interfaces and registration methods for other features. Preserve the exact
   public names, imports, signatures, and package boundaries from the
   references.
4. **Implement and register.** Add services/modules/integrations through
   `XenoAppBuilder`; select lifetimes deliberately. For a scoped service, ensure
   execution enters the documented scope. Call `build()` after queuing
   configuration.
5. **Account for execution context.** Use `RequestContext` only where the
   documented runtime supplies it. Do not claim `AsyncLocalStorage` applies to
   Vue or all environments, or retain request-scoped identity/auth/cookie
   services in a singleton.
6. **Check behavior and explain it.** Keep the change focused, add or update
   tests (including a regression test when practical), and verify contracts and
   runtime lifecycle from docs/source. When public behavior changes, update the
   relevant guide/API reference/example so docs and implementation stay aligned.
   Distinguish compile-time contract checking from runtime guarantees; surface
   unverified behavior and upstream-library options separately.

## Canonical composition example

This example is copied from the `Create an Application` section of
`docs/fundamentals/app-builder.md`. Preserve its imports and API spelling when
using it as a starter:

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

The builder is the composition root: declare the registry, register the service,
then build before using the configured application. Keep this sample intact when
quoting it; adapt only after checking the relevant API reference and application
requirements.

## Common anti-patterns

- **Inventing APIs from naming conventions.** Do not create a class, method,
  option, or export because its name sounds plausible. This documentation corpus
  does not establish a `ClientMediator` API; do not recommend it unless current
  public exports or implementation verify it. The docs describe the
  mediator/CQRS APIs by their actual documented names.
- **Crossing layer boundaries.** Do not import infrastructure or framework code
  into Domain, put persistence/provider details in handlers, or make a
  controller own business rules.
- **Skipping builder/module registration.** Do not present an integration as
  available merely by instantiating an undocumented class. Show the documented
  `XenoAppBuilder` queue/build lifecycle and exact package setup.
- **Misusing lifetimes.** Do not store request-specific state in a singleton,
  inject scoped services into singletons, or resolve scoped registrations from
  the root container.
- **Treating `AsyncLocalStorage` as universal.** Qualify it as the default Node
  context mechanism; follow the Vue/Pinia context documentation for Vue.
- **Conflating runtimes.** Do not assume Node `XenoApplication`, Vue
  integration, serverless adapters, and external framework packages expose
  identical contracts.
- **Overstating guarantees or hiding trade-offs.** Name what the types,
  pipeline, or runtime actually checks; do not claim that architecture or DI
  prevents every runtime failure.
- **Blurring Xeno options with upstream options.** For integrations such as
  Pino, Redis, Supabase, Axios, or Sentry, label which configuration belongs to
  Xeno.JS and which belongs to the upstream library.

## Reference map

Read only the domain pages needed for the task. Each reference records its
source Markdown paths; `source-coverage.md` lists all 101 inputs and the
exclusions.

| Reference                                                                                         | Use for                                                                                                           | Source coverage                                                                                                            |
| ------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| [Architecture and project workflow](references/architecture-and-project-workflow.md)              | Layer boundaries, project setup, core principles, CLI and scaffolding                                             | `docs/introduction*.md`, `docs/introduction/`, `docs/getting_started/`, `docs/cli/`                                        |
| [Runtime, DI and context](references/runtime-di-context.md)                                       | `XenoAppBuilder`, modules, runtime, registry, context/scopes, service container and lifetimes                     | `docs/fundamentals/`, `docs/dependency-injection/`                                                                         |
| [Application, CQRS and pipelines](references/application-cqrs-pipelines.md)                       | Commands, queries, handlers, schemas, `Result`, `AppError`, mediator, validation, identity enrichment, exception and pipeline behavior | `docs/application/`, `docs/application/cqrs/`, pipeline pages other than logging and the caching/resilience set |
| [Data and persistence](references/data-and-persistence.md)                                        | Domain data primitives, entities, repositories, data sources, database schemas, adapters, Unit of Work and outbox | `docs/data/`                                                                                                               |
| [HTTP and transport](references/http-and-transport.md)                                            | Controllers, routes, response mapping, clients, server and Fastify/Node/Vercel adapters                           | `docs/http/`                                                                                                               |
| [Security, authentication and authorization](references/security-authentication-authorization.md) | Authentication providers, authorization, request security and middleware                                          | `docs/security/`                                                                                                           |
| [Logging and observability](references/logging-observability.md)                                  | Logger contracts, console/Pino/Sentry integrations and logging pipeline                                           | `docs/application/logging/`, `docs/application/pipelines/logging.md`                                                       |
| [Caching and resilience](references/caching-resilience.md)                                        | Cache providers, query caching/invalidation, concurrency, idempotency and performance pipeline                    | `docs/cache/`, selected `docs/application/pipelines/`                                                                      |
| [Vue integration](references/vue-integration.md)                                                  | Vue plugin, Pinia context and frontend-specific integration                                                       | `docs/vue/`                                                                                                                |
| [Shared utilities](references/shared-utilities.md)                                                | Utility APIs for dates, strings, HTTP, guards, identifiers, promises, routing, sanitization, math and enumerables | `docs/utils/`                                                                                                              |
| [Source coverage and exclusions](references/source-coverage.md)                                   | Full input inventory and why non-technical/site-only content was omitted                                          | All 101 Markdown files                                                                                                      |
