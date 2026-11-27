# Architecture and project workflow

Technical reference assembled from the website documentation. Each section
identifies its original Markdown source. Website frontmatter and the repeated
sponsorship footer are omitted; body content and fenced examples are retained.

---

## Contents

- [`docs/cli/generate-cqrs.md`](#source-1)
- [`docs/cli/generate-http.md`](#source-2)
- [`docs/cli/new-project.md`](#source-3)
- [`docs/cli/overview.md`](#source-4)
- [`docs/getting_started/create-project.md`](#source-5)
- [`docs/getting_started/installation.md`](#source-6)
- [`docs/getting_started/project-structure.md`](#source-7)
- [`docs/introduction/architecture-overview.md`](#source-8)
- [`docs/introduction/core-principles.md`](#source-9)
- [`docs/introduction/why-xeno-js.md`](#source-10)
- [`docs/introduction.md`](#source-11)

<a id="source-1"></a>

# Source: `docs/cli/generate-cqrs.md`

## Introduction

The CLI organizes generated commands and queries inside feature modules. It
creates the application files and updates the project's registry and service
registrations.

Run these commands from the root of a project created by the current CLI.

---

## Create a Feature

```bash
xeno-js g feature Products
```

The CLI creates `src/features/products/products.module.ts`, then registers the
module in the Core composition root (`src/bootstrap.ts`) or the Vue composition
root (`src/main.ts`).

---

## Generate a Command or Query

```bash
xeno-js g command CreateProduct --feature Products
xeno-js g query GetProducts --feature Products
```

The full command name, `generate`, also works in place of `g`. If the feature
does not exist, the CLI asks whether to create it. Declining stops generation.

`--feature` is required. Names must start with a letter and may contain letters,
digits, hyphens, and underscores. The CLI normalizes names for files, classes,
and service tokens. Handler tokens must be unique across the project.

The project type is detected from `xeno.json`. You can pass `--core` or `--vue`
to check that the project matches the intended target. A mismatch stops
generation. Custom output paths, including `--output` and `-o`, are not
supported.

---

## Generated Application Files

For the examples above, a Core project contains:

```text
src/
├── bootstrap.ts
├── registry.ts
└── features/
    └── products/
        ├── products.module.ts
        ├── application/
        │   ├── commands/
        │   │   └── create-product.command.ts
        │   ├── queries/
        │   │   └── get-products.query.ts
        │   └── handlers/
        │       ├── create-product.handler.ts
        │       └── get-products.handler.ts
        └── infrastructure/
            └── schemas/                  # When Zod is enabled
                ├── create-product.schema.ts
                └── get-products.schema.ts
```

The CLI registers each handler as scoped in the feature module and adds its
typed entry to `src/registry.ts`. When Zod is enabled, it also registers the
generated schema in the composition root.

An operation does **not** generate an entity, database table, HTTP controller,
or route. See [Generate HTTP Routes](./generate-http) to expose a Core
operation.

---

## Complete the Use Case

Generated commands and queries contain an empty payload interface and use `void`
as the result type. Handlers return a `NOT_IMPLEMENTED` failure with status
`501`.

To implement an operation:

1. Define the payload and result types required by the use case.
2. Implement the handler and register its application-specific dependencies.
3. If Zod was enabled, update the schema to match the payload. The generated
   schema initially accepts an empty, strict object.
4. Update affected registry, controller, or composable types when changing the
   result type. Editing types inside managed sections will prevent subsequent
   CLI updates to those sections.
5. Test the behavior and any authorization or validation rules.

Generated queries bypass caching by default. Configure their query options
deliberately if the application needs caching.

---

## Vue Composables

Vue projects generate the same application request and handler structure, plus
`presentation/composables/use-<operation>.ts`. The composable exposes `loading`,
`error`, and `execute`, and resolves the mediator through
`ServicesUtils.useApp()`.

`execute` accepts an optional abort signal. The generated composable does not
create or cancel an `AbortController` for you.

---

## Managed Sections

The CLI tracks generated features, operations, and hashes of managed sections in
`xeno.json`. Managed sections are marked with comments such as
`// xeno:module-services:start` and `// xeno:module-services:end`.

Do not edit the contents or remove the markers of a managed section. The CLI
stops if a section it needs is missing or changed. You can edit application code
outside those sections.

The generator also rejects existing generated files and duplicate handler
tokens. A project lock prevents concurrent CLI generation. If a stale
`.xeno-cli.lock` remains after an interrupted run, remove it only after
confirming that no generator is running.

---

## Where to Go Next

See [Generate HTTP Routes](./generate-http) to add a controller and route for an
existing Core command or query.

---

---

<a id="source-2"></a>

# Source: `docs/cli/generate-http.md`

## Introduction

HTTP generation adds a presentation layer to an existing command or query. It is
available in projects created by the current CLI with a Core Node, Fastify, or
Vercel target. Console and Vue projects do not support this command.

---

## Generate a Route

First generate the application operation:

```bash
xeno-js g feature Products
xeno-js g query GetProducts --feature Products
```

From the project root, start the interactive HTTP generator:

```bash
xeno-js g http
```

The prompts ask you to:

1. Select an operation that does not already have an HTTP controller.
2. Select `GET`, `POST`, `PUT`, `PATCH`, or `DELETE`.
3. Enter a route path, for example `/api/products`.
4. Choose whether the route requires authentication.

The CLI rejects duplicate method and path pairs. Each generated operation can
have one generated HTTP controller.

---

## Generated Files and Registrations

For `GetProducts`, the CLI creates
`src/features/products/presentation/controllers/get-products.controller.ts`. It
also:

- registers the controller as transient in the feature module;
- adds the controller type to `src/registry.ts`;
- adds the route to `src/routes.ts`;
- adds an authorization policy when the route is protected.

The route extracts the request payload and calls the controller, which
dispatches the operation through the mediator. Implement the handler and
complete the payload and schema before relying on the endpoint. Until then, the
handler returns `NOT_IMPLEMENTED` with status `501`.

---

## Authentication and Public Routes

The authentication prompt defaults to **protected**. A protected route requires
Supabase authentication to have been selected during project scaffolding;
configure its environment variables before starting the project.

The generated policy requires a user ID on the handler intent. It also applies
when the same handler is called internally, not only through HTTP. Review and
extend the policy for the application's authorization requirements.

To create a public route, answer no to the authentication prompt and explicitly
confirm that the route is public. Do not expose an operation publicly unless its
behavior and data are intended for unauthenticated callers.

---

## Vercel Paths

For Vercel projects, the generated `vercel.json` sends `/api/*` requests to
`api/index.ts` and other paths to `public/index.html`. Choose an `/api` path for
generated endpoints that should reach the serverless handler.

---

---

<a id="source-3"></a>

# Source: `docs/cli/new-project.md`

## Introduction

The `new` command creates a project, writes its composition and configuration
files, and installs the selected dependencies.

```bash
xeno-js new <project-name> [--core | --vue]
```

Core is the default. Use `--vue` for a Vue project:

```bash
xeno-js new my-api --core
xeno-js new my-web --vue
```

The CLI asks which integrations to configure and whether to initialize Git. It
then runs `npm install` in the project directory.

---

## Core Projects

During Core setup, select one target:

| Target  | Generated entry point                                            |
| ------- | ---------------------------------------------------------------- |
| Console | `src/main.ts`, passing command-line arguments to `main(...args)` |
| Node    | `src/main.ts` and the Node HTTP adapter                          |
| Fastify | `src/main.ts` and the Fastify HTTP adapter                       |
| Vercel  | `api/index.ts`, `vercel.json`, and `public/index.html`           |

Select Console in the prompt rather than passing `--console`. HTTP route
generation is available only for Node, Fastify, and Vercel.

Core options include PostgreSQL, SQLite or no database, Axios, Cockatiel,
Supabase Auth, Redis, Pino, Sentry, and Zod. Configure the environment variables
for the integrations you select before starting the application.

For Vercel, the generated configuration rewrites `/api/*` to the serverless
entry point and other paths to the static index page.

Start a Console, Node, or Fastify project with:

```bash
cd my-api
npm run start
```

For a Vercel project, `npm run start` runs `vercel dev`.

---

## Vue Projects

The Vue scaffold includes Pinia for request context. Its optional integrations
include Vue Router, Tailwind CSS, Axios, Cockatiel, Supabase Auth, Sentry, and
Zod.

```bash
cd my-web
npm run dev
```

The scaffold currently declares `@xeno-js/core` with a `^5.1.0` version range.
The published `5.1.0` package imports Node's `async_hooks`, which Vite cannot
bundle for browsers. A Vue production build therefore requires a compatible Core
release; check the resolved version when diagnosing the build.

---

## Generated Project Metadata

The scaffold creates `xeno.json` alongside the application files. The CLI uses
it to detect the project type, selected integrations, and managed sections when
generating features and operations.

Keep this file and the managed section markers intact. The generator is designed
for projects created by the current CLI, not arbitrary existing projects.

---

## Where to Go Next

See [Generate Application Components](./generate-cqrs) to add your first
feature, command, or query.

---

---

<a id="source-4"></a>

# Source: `docs/cli/overview.md`

## Introduction

The Xeno.JS CLI (`@xeno-js/cli`) creates projects and generates application
components using explicit composition.

The CLI creates the starting structure. Your application defines the business
behavior, payloads, dependencies, and access rules.

---

## Installation

Node.js 20 or later is required. Install the CLI globally:

```bash
npm install -g @xeno-js/cli
```

Or run it without a global installation:

```bash
npx @xeno-js/cli new my-app --core
```

The examples below use `xeno-js` from a global installation. With `npx`, replace
`xeno-js` with `npx @xeno-js/cli`. Use a CLI release that includes the managed
generators described here.

---

## Commands

| Command                                        | Purpose                                      |
| ---------------------------------------------- | -------------------------------------------- |
| `xeno-js new <name> [--core \| --vue]`         | Create a Core or Vue project.                |
| `xeno-js g feature <Name>`                     | Create and register a feature module.        |
| `xeno-js g command <Name> --feature <Feature>` | Generate and register a command and handler. |
| `xeno-js g query <Name> --feature <Feature>`   | Generate and register a query and handler.   |
| `xeno-js g http`                               | Interactively expose an operation over HTTP. |
| `xeno-js --help`                               | Show the CLI usage guide.                    |

`generate` is the full command name; `g` is its alias. Run generators from the
root of a project created by the current CLI. The project type is detected from
`xeno.json`; an optional `--core` or `--vue` flag verifies that type.

---

## How Generation Fits Together

```text
Project
  └── Feature module
       ├── Command or query
       ├── Handler
       ├── Optional Zod schema
       └── Presentation
            ├── Vue composable, for Vue projects
            └── HTTP controller and route, when explicitly generated
```

The CLI updates the registry and the relevant module and composition root. HTTP
exposure is a separate step, available for Core projects targeting Node,
Fastify, or Vercel.

Generated handlers return `NOT_IMPLEMENTED` until you implement them. When Zod
is enabled, update the generated schema alongside the payload type.

---

## Where to Go Next

1. [Create a Project](./new-project)
2. [Generate Application Components](./generate-cqrs)
3. [Generate HTTP Routes](./generate-http)
4. [Xeno Registry](../fundamentals/xeno-registry)

---

---

<a id="source-5"></a>

# Source: `docs/getting_started/create-project.md`

## Introduction

The Xeno.JS CLI can scaffold a new application with the initial Xeno.JS
architecture.

## 1. Create the project

Run:

```bash id="1b6v3u"
npx @xeno-js/cli
```

Follow the CLI prompts to choose the project name and configuration.

Alternatively, if you installed the CLI globally:

```bash id="5h0q7e"
xeno-js new my-xeno-app
```

The CLI generates the project in the selected directory.

Replace `my-xeno-app` with the name you selected during project creation.

See more on [CLI](../cli/new-project)

## 2. Enter the project

```bash id="6x3m1p"
cd my-xeno-app
```

## 3. Install dependencies

If dependencies have not already been installed:

```bash id="m1k9z4"
npm install
```

## 4. Start development

Use the development command generated by the project:

```bash id="7c4r2n"
npm run start
```

Your Xeno.JS application is now ready for development.

---

## What happens next?

The CLI has created the initial application structure.

Before adding features, take a look at how the generated project is organized:

Next: [Project Structure](./project-structure)

---

---

<a id="source-6"></a>

# Source: `docs/getting_started/installation.md`

## Introduction

Before creating a Xeno.JS project, make sure you have:

- [Node.js](https://nodejs.org/) installed;
- npm or another compatible package manager.

## Install the Xeno.JS CLI

The recommended way to start a new project is through the official CLI.

You can install it globally:

```bash
npm install -g @xeno-js/cli
```

Or use it directly with `npx`:

```bash
npx @xeno-js/cli
```

The CLI is used to create and scaffold Xeno.JS applications.

Continue with [Create Project](./create-project).

## Manual Installation

If you are setting up an existing TypeScript application instead, install the
core package directly:

```bash
npm install @xeno-js/core
```

For a TypeScript Node.js application, install TypeScript and Node.js types as
development dependencies when they are not already present:

```bash
npm install --save-dev typescript @types/node
```

The core package can then be configured and bootstrapped through
`XenoAppBuilder`.

## Optional Dependencies

Xeno.JS keeps several integrations optional.

For example, if you enable HTTP Core, you install its required dependencies
separately:

```bash
npm install axios cockatiel
```

Other integrations may have their own dependencies. Install them only when the
corresponding feature is enabled.

---

## CLI

See more on [CLI](../cli/overview)

---

---

<a id="source-7"></a>

# Source: `docs/getting_started/project-structure.md`

## Introduction

The Xeno CLI creates a Core backend or Vue frontend project from the options you
select. It writes the application entry points, registry, configuration, and
`xeno.json` scaffold metadata. Optional integrations add their own dependencies
and files.

Create a Core project:

```bash
npx @xeno-js/cli new my-app --core
```

Create a Vue project:

```bash
npx @xeno-js/cli new my-app --vue
```

The CLI asks which target and integrations to use, generates the matching files,
then optionally initializes Git and runs `npm install`.

## A new Core project

With the Console target and no database selected, the generated project
contains:

```text
my-app/
├── src/
│   ├── bootstrap.ts
│   ├── main.ts
│   └── registry.ts
├── .env
├── .env.example
├── .gitignore
├── package.json
├── readme
├── tsconfig.json
└── xeno.json
```

The Node.js and Fastify targets also generate `src/routes.ts` and an HTTP entry
point in `src/main.ts`. The Vercel target adds `src/routes.ts`, `api/index.ts`,
`public/index.html`, and `vercel.json`; it uses `api/index.ts` as its serverless
entry point.

Selecting a database adds `drizzle.config.ts` and `src/schema.ts`. The CLI
supports PostgreSQL and SQLite through Drizzle.

The Core scaffold does not create feature or domain folders by default. Generate
them when the application needs its first feature.

## `src/bootstrap.ts`

`bootstrap.ts` is the Core project's composition root. The CLI generates an
`XenoAppBuilder`, configures the selected integrations, and exports `XenoApp`
for the application entry point.

The builder starts with request context, cache, logging, pipeline, and service
registration. The selected options add database, authentication, HTTP server,
routes, HTTP client, validation, and observability configuration.

Generated sections use `xeno` markers. Later CLI generators update those
sections and `xeno.json`; keep generated sections intact so managed generation
can verify and update them.

For example, the default Console scaffold builds this chain:

```typescript
const builder = new XenoAppBuilder<MyRegistry>()
  .addContext()
  .addCache(undefined)
  .addLogger([withConsoleLogger(() => {})])
  .addPipeline()
  .addServices((services) => {
    // xeno:services:start
    // xeno:services:end
  })
```

## `src/registry.ts`

`registry.ts` defines the application's Xeno registry type. The Core registry
extends `XenoRegistry`; database options add the database context and
transaction types.

Add application tokens in the registry as the project grows. The CLI inserts
generated handler and controller types into managed registry sections.

```ts
export interface MyRegistry extends XenoRegistry {
  // xeno:registry:start
  // xeno:registry:end
}
```

## `src/main.ts`

The generated entry point depends on the selected Core target:

- Console loads `dotenv`, builds the Xeno application, and passes command-line
  arguments to `main`.
- Node.js and Fastify start the HTTP server and read the port from
  configuration.
- Vercel uses `api/index.ts` to pass requests to the Xeno Vercel adapter.

Application-specific behavior belongs in the features and handlers you add to
the scaffold.

## Database files

Selecting a database during project creation adds Drizzle configuration and a
starting schema.

### PostgreSQL

The PostgreSQL option generates:

```text
drizzle.config.ts
src/schema.ts
```

It adds the PostgreSQL adapter and reads the connection string from
`DATABASE_URL`.

### SQLite

The SQLite option also generates:

```text
drizzle.config.ts
src/schema.ts
```

It uses the SQLite Drizzle primitives and reads the database URL from
`SQLITE_DATABASE_URL`.

Both generated schemas are starting points. Replace or extend them with the
tables required by the application.

## Environment files

The CLI creates `.env` and `.env.example`. Their variables depend on the
selected integrations.

For example, PostgreSQL uses `DATABASE_URL`, SQLite uses `SQLITE_DATABASE_URL`,
Redis adds its connection settings, and Supabase adds authentication settings.
The generated `.gitignore` excludes `.env`; keep secrets there and use
`.env.example` to document required variables.

## `package.json`

The Core `package.json` includes `@xeno-js/core`, `@xeno-js/shared`, `dotenv`,
TypeScript, and `tsx`. The CLI adds packages for selected features, such as a
Drizzle adapter, Axios, Cockatiel, Pino, Sentry, Redis, Supabase, or Zod.

The generated scripts include `start` and `build`. Database options add Drizzle
generation, push, and migration scripts. The Vercel target uses `vercel dev` for
local startup.

## What happens after generation

The CLI creates the selected files, writes `xeno.json` to record scaffold
metadata, and then optionally initializes a Git repository. It runs
`npm install` after generation and prints the project path and next steps.

## Adding application code

Start by generating a feature, then generate commands or queries inside it:

```bash
xeno-js g feature Users
xeno-js g command CreateUser --feature Users
xeno-js g query FindUser --feature Users
```

Run these commands from the project root. The CLI reads `xeno.json` to detect
the project type. Use `--core` or `--vue` only to verify that target.

The feature generator creates `src/features/users/users.module.ts`. A command or
query adds its request and handler under that feature's `application` folder,
then registers them in the module and `src/registry.ts`. When Zod is enabled,
the generator also creates a payload schema. Vue projects receive a composable
for each operation.

Generated handlers return `NOT_IMPLEMENTED` until you add the application
behavior. If you enabled Zod, update the generated payload schema when you
change the payload type.

For Node.js, Fastify, or Vercel Core projects, run `xeno-js g http` to expose an
existing operation as a route. Routes are protected by default. Creating a
public route requires explicit confirmation.

## A Vue project

The Vue scaffold includes Vue, Vite, the Xeno Vue integration, a Pinia-backed
request context, and its own registry. A basic project starts with:

```text
my-app/
├── src/
│   ├── App.vue
│   ├── main.ts
│   └── registry.ts
├── .env
├── .env.example
├── .gitignore
├── index.html
├── package.json
├── README.md
├── tsconfig.json
├── vite.config.ts
└── xeno.json
```

Selecting Vue Router adds `src/router/index.ts` and `src/views/HomeView.vue`.
Selecting Tailwind adds `src/assets/style.css` and the Vite integration. Axios,
Cockatiel, Supabase, Sentry, and Zod are optional.

The Vue entry point configures the selected integrations, provides the Xeno
service container to the app, and mounts `App.vue`. Use the same feature and
operation commands from the project root to add application code.

## CLI documentation

For CLI commands, project options, and generators, see:

[CLI Overview](../cli/overview)

---

---

<a id="source-8"></a>

# Source: `docs/introduction/architecture-overview.md`

## Introduction

Xeno.JS is built around an **application-centered architecture**.

The application is not defined by its HTTP server, UI framework, database, or
other infrastructure.

Instead, these concerns connect to an application layer that coordinates use
cases and their dependencies.

The dependency direction is not a single stack in which infrastructure depends
on the domain. Presentation calls application use cases; application logic may
use domain concepts; infrastructure implements contracts required by the
application. The composition root connects those implementations:

```text
Presentation ───────► Application ───────► Domain
                          ▲
                          │ implements contracts used by
                    Infrastructure

             XenoAppBuilder wires the components
```

This separation is the architectural foundation of Xeno.JS.

---

## The Xeno.JS architecture

A typical Xeno.JS application can be understood through four main concerns:

```text
┌─────────────────────────────────────────────┐
│                 Presentation                │
│          HTTP / Worker / Vue UI             │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│                 Application                 │
│                                             │
│ Commands / Queries / Handlers / Pipelines   │
│ Executed by the runtime with DI, scopes,    │
│ context, and configured modules             │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│                    Domain                   │
│                                             │
│ Entities / Aggregates / Value Objects       │
│ Domain Events / Rules / Contracts           │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│                Infrastructure               │
│                                             │
│ DB / Repositories / Data Sources / Cache    │
│ HTTP Clients / Auth / Logging / Resilience  │
└─────────────────────────────────────────────┘

Application logic uses domain concepts and contracts.
Infrastructure implements contracts and is wired at the composition root.
```

These are architectural responsibilities, not necessarily four physical folders
that every application must reproduce exactly.

The important property is that **the boundaries are explicit**.

---

## 1. Presentation

The presentation layer is where an application receives an external interaction.

Examples include:

- HTTP requests
- CLI commands
- background workers
- other application entry points
- Vue UI interactions

Presentation translates an external interaction into an application operation.

For example:

```text
HTTP request
     │
     ▼
Presentation
     │
     ▼
Command / Query
     │
     ▼
Application
```

The presentation layer should not become the place where the application's
business rules are implemented.

Its responsibility is to connect the outside world to the application.

---

### Transport independence

HTTP is one possible entry point, not the boundary that defines a use case. When
HTTP services are configured, the runtime application can dispatch an incoming
request through its registered adapter and dispatcher. Other application work
can run through the application runtime without an HTTP request.

Conceptually:

```text
HTTP request ──► Presentation ──► Application use case
                                      ▲
                                      │
Other execution ──► runInScope() ─────┘
```

The application use case remains separate from the transport-specific entry
point.

---

## 2. Application

The application layer is the center of execution in Xeno.JS.

It coordinates **what the application does**.

Application execution is organized around concepts such as:

- commands
- queries
- handlers
- pipelines
- dependency injection
- service scopes
- request context
- modules
- application composition

A simplified execution flow is:

```text
Command / Query
       │
       ▼
    Mediator
       │
       ▼
   Pipeline
       │
       ▼
    Handler
       │
       ▼
 Application Result
```

This is one of the most important aspects of the Xeno.JS architecture.

The framework does not treat CQRS as an isolated utility.

Commands, queries, handlers, dependency resolution, scopes, and pipelines
participate in the application's execution model.

---

## 3. Commands and Queries

Xeno.JS represents application operations through commands and queries.

A command represents an operation that changes application state.

A query represents an operation that retrieves information.

Conceptually:

```text
Command
   │
   ▼
Command Pipeline
   │
   ▼
Command Handler


Query
   │
   ▼
Query Pipeline
   │
   ▼
Query Handler
```

The mediator is responsible for finding the appropriate handler and executing
the request through the configured pipeline.

This keeps the application entry point independent from the concrete handler
implementation.

For example:

```text
HTTP Controller
      │
      ▼
CreateUserCommand
      │
      ▼
Mediator
      │
      ▼
CreateUserHandler
      │
      ▼
Domain / Repository
```

The controller does not need to know how the use case is implemented.

---

## 4. Pipelines

Application execution can require behavior that should not be implemented
repeatedly inside every handler.

Examples include:

- validation
- authorization
- logging
- performance measurement
- idempotency
- concurrency handling
- exception handling
- query behavior

Xeno.JS models these concerns through pipelines.

```text
Request
   │
   ▼
┌───────────────────┐
│ Validation        │
├───────────────────┤
│ Authorization     │
├───────────────────┤
│ Logging           │
├───────────────────┤
│ Performance       │
├───────────────────┤
│ Handler           │
└───────────────────┘
```

The exact pipeline configuration is application-dependent.

The architectural principle is that cross-cutting execution can be composed
around the use case instead of being duplicated across individual entry points.

---

## 5. Dependency Injection and Composition

Xeno.JS uses explicit dependency injection.

Dependencies are registered in the application's composition root rather than
discovered through decorators or implicit runtime scanning.

The central composition mechanism is `XenoAppBuilder`.

Conceptually:

```text
XenoAppBuilder
    │
    ├── configuration
    ├── modules
    ├── services
    ├── pipelines
    ├── infrastructure
    └── application components
              │
              ▼
       ServiceContainer
              │ retained by builder
              ▼
       XenoApplication
        (build result)
```

A simplified composition looks like:

```typescript
const app = new XenoAppBuilder().addServices((services) => {
  services.addScoped(
    'USER_REPOSITORY',
    (scope) => new UserRepository(scope.resolve('USER_DATA_SOURCE')),
  )
})

const application = await app.build()

await application.runInScope(async (services) => {
  const repository = services.USER_REPOSITORY
})
```

The important property is not the fluent API itself.

It is that the **dependency graph is explicitly composed**.

`build()` initializes queued modules and, unless a custom application factory is
configured, returns an `XenoApplication`. The builder retains the service
container; runtime code should use the returned application's APIs.

---

## 6. Service Lifetimes and Scopes

Xeno.JS distinguishes dependency lifetimes such as:

- singleton
- scoped
- transient

This allows dependencies to have an explicit lifecycle.

For example:

```text
Application
     │
     ├── Singleton
     │
     ├── Scoped ─────── Request / Execution Scope
     │
     └── Transient
```

A scoped service belongs to the current application execution scope.

This becomes particularly important when an application uses request-specific
dependencies such as:

- request context
- transaction state
- scoped repositories
- request-specific services

The service container also tracks dependency resolution and detects conditions
such as circular dependencies and captive scoped dependencies.

Singleton factories resolve from the root scope. The container rejects a scoped
dependency reached during singleton construction. Scoped services must be
resolved inside an active scope; use `runInScope()` when an operation needs to
create one.

---

## 7. Request Context

Xeno.JS connects request context with the service scope.

With the default Node configuration, asynchronous request context is propagated
through `AsyncLocalStorage`.

Conceptually:

```text
Incoming Request
       │
       ▼
Request Context
       │
       ├── Identity
       ├── Network Context
       └── Service Scope
               │
               ▼
       Application Execution
```

The context allows request-specific information to remain available throughout
asynchronous application execution without passing it manually through every
method.

The context execution disposes its service scope when the operation completes,
including when the operation fails. `XenoApplication.runInScope()` provides this
lifecycle for work that does not start from an HTTP request.

This makes request boundaries an explicit part of the application runtime.

---

## 8. Domain

The domain represents the business concepts and rules of the application.

Xeno.JS and `@xeno-js/shared` provide domain-oriented primitives including:

- entities
- aggregates
- value objects
- domain events
- domain errors
- domain contracts
- result types

The purpose of the domain is to express business concepts without depending on
application runtime, transport, or infrastructure details.

A simplified model is:

```text
Domain
├── Entities
├── Value Objects
├── Aggregates
├── Domain Events
├── Domain Errors
└── Domain Contracts
```

The domain is therefore different from the application layer.

The application coordinates a use case.

The domain expresses the business model used by that use case.

---

## 9. Infrastructure

Infrastructure contains concrete implementations and integrations required by
the application.

Examples in the Xeno.JS ecosystem include:

- databases
- repositories
- data sources
- HTTP clients
- caching
- authentication
- logging
- resilience
- configuration

Application code uses contracts; infrastructure depends on those contracts to
provide concrete implementations. The composition root wires an implementation
for runtime use:

```text
Application ──uses──► Application contract
                            ▲
                            │ implements
                    Infrastructure
                            ▲
                            │ wired by
                     XenoAppBuilder
```

For example, an application may depend on a repository contract while
infrastructure provides the concrete database-backed repository.

This keeps infrastructure details out of the application and domain model.

---

## 10. Data Sources and Repositories

Xeno.JS distinguishes between application-facing persistence abstractions and
infrastructure-level data access.

A simplified flow is:

```text
Application
     │ uses
     ▼
Repository Contract
     ▲
     │ implemented by
Repository Implementation
     │
     ▼
Data Source
     │
     ▼
Database

XenoAppBuilder wires the implementation during composition.
```

The application uses the contract for the capability it needs. Infrastructure
determines how that capability is implemented, and composition connects the
implementation to the application.

---

## 11. Modules

Xeno.JS uses modules to compose optional application capabilities.

`XenoAppBuilder` queues modules and initializes them during `build()`. It sorts
queued modules by ascending priority and configures them sequentially. With Node
configuration, the builder queues the default context module if none was
configured; with Vite configuration, the application must configure a context
plugin explicitly.

Conceptually:

```text
AppBuilder
    │
    ├── Context Module
    ├── Pipeline Module
    ├── Logger Module
    ├── Auth Module
    ├── Database Module
    ├── Cache Module
    ├── HTTP Module
    └── Custom Modules
```

This allows infrastructure and application capabilities to be enabled explicitly
rather than requiring every application to use the same runtime configuration.

The application composition therefore becomes part of the architecture itself.

---

## 12. The Xeno.JS Ecosystem

The architecture is distributed across several packages.

They have different responsibilities.

```text
                         Xeno.JS

                    @xeno-js/shared
                           │
              Domain primitives & contracts
                           │
             ┌─────────────┴─────────────┐
             │                           │
             ▼                           ▼
      @xeno-js/core                @xeno-js/vue
      Node.js runtime              Vue integration

                     @xeno-js/cli
             project and component tooling
```

### `@xeno-js/shared`

`@xeno-js/shared` contains framework-neutral concepts shared across the
ecosystem.

Its source structure includes domain primitives such as:

- aggregates
- entities
- value objects
- domain events
- errors
- results

It also defines application contracts for commands and queries.

The package therefore provides a common architectural language without owning
the Node.js runtime or Vue runtime.

### `@xeno-js/core`

`@xeno-js/core` is the main Node.js application architecture package.

It provides:

- application composition
- dependency injection
- service lifetimes
- service scopes
- CQRS
- mediator execution
- pipelines
- request context
- modules
- repositories
- data sources
- infrastructure integrations

The package contains the runtime mechanisms that execute the application
architecture.

### `@xeno-js/vue`

`@xeno-js/vue` applies the Xeno application model in Vue applications. Its
specific runtime contracts are separate from the Node.js `XenoApplication`
described here.

### `@xeno-js/cli`

`@xeno-js/cli` is the developer tooling layer.

It is responsible for:

- creating Xeno projects
- selecting project types
- generating architecture-aligned files
- generating commands
- generating queries
- scaffolding Core applications
- scaffolding Vue applications

The CLI does not execute the application architecture at runtime.

It **creates the starting structure that the other packages execute**.

---

## 13. How the Packages Relate

The relationship can be summarized as:

```text
                  ┌───────────────────┐
                  │   @xeno-js/shared │
                  │                   │
                  │ Domain primitives │
                  │ Application       │
                  │ contracts         │
                  └─────────┬─────────┘
                            │
              ┌─────────────┴─────────────┐
              │                           │
              ▼                           ▼
   ┌────────────────────┐      ┌────────────────────┐
   │   @xeno-js/core    │      │   @xeno-js/vue     │
   │   Node.js runtime  │      │  Vue integration   │
   └────────────────────┘      └────────────────────┘

                  ┌──────────────────┐
                  │   @xeno-js/cli   │
                  │ project tooling  │
                  └──────────────────┘
```

The important distinction is:

> **Shared defines common concepts. Core and Vue execute those concepts in their
> respective environments. CLI creates projects and components around them.**

---

## 14. A Complete Application Flow

Putting the architecture together, a backend request can be understood as:

```text
HTTP Request ──► Presentation ──► Command / Query ──► Mediator
                                                        │
                                                        ▼
                                                Configured Pipeline
                                                        │
                                                        ▼
                                                     Handler
                                                        │
                                     ┌──────────────────┴───────────┐
                                     ▼                              ▼
                               Domain Logic              Application Contract
                                                                    ▲
                                                                    │ implemented by
                                                         Infrastructure Adapter
                                                                    │
                                                                    ▼
                                                        Data Source / Database
```

The request runs in an application scope. When a use case needs an
infrastructure capability, it uses an application-facing contract whose
implementation was selected during composition. The response then travels back
through the application boundary to the presentation layer.

The important point is that the HTTP request is only the **entry point**.

The application architecture does not depend on HTTP being the thing that
defines the use case.

---

## 15. Frontend Application Flow

The same architectural idea can be applied in a Vue application.

A simplified flow is:

```text
                     Vue UI
                       │
                       ▼
                Application Action
                       │
                       ▼
                  Mediator / Pipe
                       │
                       ▼
                  Application
                       │
                       ▼
                 Remote Data Source
                       │
                       ▼
                    Backend
```

This allows frontend code to use application-oriented concepts instead of
placing all application behavior directly inside Vue components. The exact Vue
runtime composition is not described by the Node.js builder and application
contracts covered in this guide.

---

## 16. The Composition Root

One of the most important architectural concepts in Xeno.JS is the **composition
root**.

This is where the application decides:

- which services exist
- which implementations they use
- which lifetimes they have
- which modules are enabled
- which pipelines are configured
- which infrastructure is available

In Xeno.JS, `XenoAppBuilder` provides the primary mechanism for this
composition.

The builder queues configuration actions and modules; `build()` initializes the
modules in ascending priority order and returns the configured runtime
application. The default runtime is `XenoApplication`, which exposes
`getServices()` for root resolution and `runInScope()` for scoped work.

```text
                    Composition Root
                         │
                   XenoAppBuilder
                         │
        ┌────────────────┼─────────────────┐
        │                │                 │
        ▼                ▼                 ▼
   Dependencies       Modules          Pipelines
        │                │                 │
        └────────────────┼─────────────────┘
                         ▼
                  Service Container
                         │
              build() returns the runtime
                         ▼
                  XenoApplication
```

This is where the application's architecture becomes executable configuration.

---

## 17. Why the Architecture Is Application-Centered

The architecture can be summarized by moving from the outside inward:

```text
Transport
    ↓
Presentation
    ↓
Application ──uses──► Domain
    │
    └──uses application contracts implemented by Infrastructure
```

Infrastructure implements contracts used by the application; it is not a lower
layer that the domain depends on.

The application defines what it needs.

Infrastructure supplies implementations.

This gives the application a stable center while allowing external technologies
to change around it.

For example:

```text
                ┌── PostgreSQL
                │
Application ────┼── Redis
                │
                ├── External API
                │
                └── Another implementation
```

The composition root selects the concrete implementation. The application uses
the contract, not the provider-specific technology.

---

## 18. The Architectural Boundary

The most important boundary in Xeno.JS is not a particular folder.

It is the boundary between:

```text
             Application Intent
                     │
                     ▼
             ┌───────────────┐
             │  Application  │
             └───────┬───────┘
                     │ uses
                     ▼
             Application contract
                     ▲
                     │ implements
             ┌───────┴────────┐
             │ Infrastructure │
             └───────┬────────┘
                     ▲
                     │ configured by
                XenoAppBuilder
```

The application should express **what it needs**.

Infrastructure should determine **how that need is fulfilled**.

This is what allows Xeno.JS to remain independent from a specific transport,
database, or infrastructure implementation.

---

## 19. Architecture in One Diagram

The complete Xeno.JS model can therefore be reduced to:

```text
┌──────────────────────────────────────────────────────────────┐
│                         ENTRY POINTS                         │
│                                                              │
│        HTTP       Worker       Vue / Browser                  │
└──────────────────────────────┬───────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────┐
│                       PRESENTATION                           │
│                                                              │
│          Translate external interactions into use cases      │
└──────────────────────────────┬───────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────┐
│                       APPLICATION                            │
│                                                              │
│  Commands • Queries • Handlers • Mediator • Pipelines        │
│  Runtime scopes and request context are provided by Core     │
└──────────────────────────────┬───────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────┐
│                          DOMAIN                              │
│                                                              │
│  Entities • Aggregates • Value Objects • Events • Rules      │
└──────────────────────────────────────────────────────────────┘

     Application-facing contracts
                 ▲
                 │ implemented by
┌──────────────────────────────────────────────────────────────┐
│                      INFRASTRUCTURE                          │
│                                                              │
│  Database • Repositories • Data Sources • Cache • Auth       │
│  HTTP Clients • Logging • External Services                  │
└──────────────────────────────────────────────────────────────┘
                 ▲
                 │ wired during composition
            XenoAppBuilder


                    SHARED ARCHITECTURAL LANGUAGE

                         @xeno-js/shared
                                │
                 ┌──────────────┴──────────────┐
                 ▼                             ▼
          @xeno-js/core                  @xeno-js/vue
            Node.js runtime              Vue integration

                         @xeno-js/cli
                       project tooling
```

This diagram summarizes the runtime layers. The CLI is developer tooling, not an
application entry point or runtime layer.

This is the mental model to keep while reading the rest of the documentation.

---

---

<a id="source-9"></a>

# Source: `docs/introduction/core-principles.md`

## Introduction

Xeno.JS is built around a simple architectural idea:

> **Make application architecture explicit in code.**

As an application grows, its architecture is shaped by decisions about
dependencies, business logic, infrastructure, execution flow, state, and
application boundaries.

Xeno.JS treats those decisions as first-class architectural concerns.

This does not mean that every application must follow the same structure. It
means that when a structure is chosen, important boundaries should be visible,
understandable, and composable.

---

## 1. Application architecture should be explicit

The first principle of Xeno.JS is explicitness.

A growing application eventually needs to answer questions such as:

- Which component owns this business rule?
- Which dependencies does this use case require?
- Where is a dependency created?
- How long does that dependency live?
- Which cross-cutting behaviors run before a use case?
- Which part of the application knows about the database?
- Which part knows about HTTP?
- Which part represents the domain?

Xeno.JS makes these architectural relationships explicit through application
composition, dependency registration, scopes, commands, queries, pipelines,
contracts, and modules.

The goal is not to eliminate abstraction.

The goal is to make important abstractions visible.

---

## 2. The application is independent from its transport

HTTP is a transport mechanism.

It is not the application itself.

The same application logic may be invoked through:

- an HTTP API;
- a CLI command;
- a background worker;
- a message consumer;
- a scheduled job;
- a browser application.

Xeno.JS therefore places application logic behind explicit boundaries rather
than coupling it directly to a particular transport.

A controller, route, or other entry point should translate an external request
into an application operation.

Conceptually:

```text
External Transport
       │
       ▼
Presentation / Entry Point
       │
       ▼
Application Operation
       │
       ▼
Domain + Infrastructure
```

This allows the application model to remain stable while the surrounding
transport changes.

---

## 3. Dependencies should be explicit

Dependencies are part of architecture.

A service that requires a repository, logger, cache, authorization service, or
other component should express that dependency explicitly.

Xeno.JS uses explicit dependency registration and resolution rather than relying
primarily on automatic discovery.

This makes the composition of the application visible at its composition
boundary.

The important distinction is:

```text
Implicit architecture
    ↓
"the framework discovers and connects things"

Explicit architecture
    ↓
"the application defines how things are connected"
```

The XenoRegistry and `ServiceContainer` are part of this model. The registry
describes the application's dependency contract, while the container handles
runtime registration, resolution, lifetimes, and scopes.

---

## 4. Composition belongs at the composition root

Application components should not be responsible for constructing the entire
dependency graph themselves.

Composition should happen at a well-defined boundary.

In Xeno.JS, `XenoAppBuilder` provides the main composition mechanism for
configuring modules, services, infrastructure, pipelines, logging, database
integrations, and other application capabilities before the application is
built.

Conceptually:

```text
Application Components
        │
        │ declare dependencies
        ▼
Dependency Contracts
        │
        ▼
Composition Root
        │
        ├── implementations
        ├── lifetimes
        ├── modules
        └── infrastructure
```

This separates **using a dependency** from **deciding how that dependency is
constructed**.

---

## 5. Lifetimes are architectural decisions

Not every dependency should live for the same amount of time.

An application may contain:

- long-lived services;
- request-scoped services;
- short-lived objects.

Xeno.JS makes these lifetimes explicit through its dependency injection model
and scopes.

This becomes particularly important when application execution is asynchronous.

A request-scoped dependency should belong to the request in which it was created
rather than accidentally becoming shared application state.

Xeno.JS uses asynchronous execution context to associate request-specific state
and dependency scopes with the current execution flow.

The principle is therefore:

> **Dependency lifetime should be deliberate, not accidental.**

---

## 6. Application execution should be composable

An application operation rarely consists of only one function call.

A command or query may require:

```text
Request
  ↓
Authentication
  ↓
Authorization
  ↓
Validation
  ↓
Logging
  ↓
Caching
  ↓
Handler
  ↓
Result
```

Xeno.JS models this execution flow through commands, queries, handlers, and
pipeline behaviors.

This gives cross-cutting concerns a defined place in the application execution
model instead of requiring every handler to implement them independently.

For example, Xeno's command and query pipelines can host behaviors for
validation, authorization, logging, exception handling, caching, idempotency,
and concurrency-related concerns.

The principle is:

> **Application execution should have explicit extension points.**

---

## 7. Commands and queries represent application intent

The application layer should express what the application is being asked to do.

Commands represent operations that modify application state.

Queries represent operations that retrieve information.

This distinction is useful because the two operations often have different
requirements.

For example:

```text
CreateOrderCommand
UpdateCustomerCommand
CancelSubscriptionCommand
```

are different in nature from:

```text
GetOrderQuery
SearchCustomersQuery
GetSubscriptionQuery
```

Xeno.JS makes these operations first-class application messages handled through
its mediator and pipeline model.

The principle is not that every system must use CQRS at maximum complexity.

The principle is:

> **Application intent should be explicit.**

---

## 8. The domain should not depend on infrastructure

Business concepts should not need to know how data is persisted or how external
systems are contacted.

A domain model should be able to express concepts such as:

- entities;
- value objects;
- aggregates;
- domain events;
- domain errors;
- business invariants.

Infrastructure can then provide concrete implementations for things such as:

- databases;
- repositories;
- external APIs;
- caches;
- messaging systems;
- logging providers.

Xeno.JS's shared and domain primitives are designed around this separation. Its
repository and data-access abstractions also distinguish application/domain
concepts from concrete persistence mechanisms.

Conceptually:

```text
        Application
             │
             ▼
          Domain
             │
             │ contracts
             ▼
      Infrastructure
             │
       ┌─────┼─────┐
       ▼     ▼     ▼
      DB    API   Cache
```

The infrastructure can change without requiring the domain model to become a
database model.

---

## 9. Frameworks should be boundaries, not the architecture

Xeno.JS does not need to own every part of an application.

An application may use an HTTP framework, database library, ORM, authentication
provider, message broker, or frontend framework.

Those technologies solve specific problems.

Xeno.JS provides the application architecture around them.

This is why Xeno.JS can sit underneath transport technologies rather than
replacing them.

The distinction is important:

```text
Framework / Transport
        │
        ▼
Application Architecture
        │
        ▼
Domain
        │
        ▼
Infrastructure
```

The architecture should describe the application.

Individual frameworks should implement particular boundaries.

---

## 10. Prefer explicitness over hidden magic

Xeno.JS favors mechanisms that can be understood by reading the application
structure.

Examples include:

- explicit service registration;
- explicit dependency contracts;
- explicit scopes;
- explicit application operations;
- explicit pipeline registration;
- explicit module composition.

This does not mean that Xeno.JS contains no abstraction or automation.

The distinction is that automation should support an architecture that remains
understandable rather than making the architecture dependent on invisible
framework behavior.

---

## 11. Shared architecture should use a shared language

Xeno.JS is organized as an ecosystem rather than a single runtime package.

The packages have different responsibilities:

```text
@xeno-js/shared
        │
        ├── domain primitives
        ├── application contracts
        └── shared architectural types
             │
       ┌─────┴─────┐
       ▼           ▼
@xeno-js/core   @xeno-js/vue
       │           │
       │           │
    Node.js      Vue
    runtime     runtime
       │
       └─────┬─────┘
             ▼
        @xeno-js/cli
      scaffolding/tooling
```

The objective is not to make backend and frontend identical.

It is to provide a common architectural vocabulary where the same concepts make
sense across the ecosystem.

---

## 12. Architecture should remain understandable as the system grows

The ultimate purpose of these principles is not architectural complexity for its
own sake.

A system should remain understandable when it grows.

New features should have identifiable places for:

- application operations;
- domain rules;
- dependencies;
- infrastructure implementations;
- cross-cutting behaviors;
- composition.

That makes architectural decisions easier to reason about as the number of
features and dependencies increases.

The principle can be summarized as:

> **Growth should add functionality without making the architecture
> progressively invisible.**

---

## The Xeno.JS Mental Model

The principles above can be reduced to one model:

```text
                    ENTRY POINTS
       HTTP · CLI · Worker · Vue · Messages
                         │
                         ▼
                APPLICATION LAYER
        Commands · Queries · Handlers
        Pipelines · Dependencies · Context
                         │
                         ▼
                    DOMAIN
      Entities · Value Objects · Aggregates
             Events · Business Rules
                         │
                         ▼
                 INFRASTRUCTURE
        Database · APIs · Cache · Messaging
```

And around this model:

```text
Explicit Composition
Explicit Dependencies
Explicit Lifetimes
Explicit Execution Flow
Explicit Boundaries
```

These are the architectural foundations on which the Xeno.JS ecosystem is built.

---

## What These Principles Do Not Mean

These principles should not be interpreted as requirements to introduce maximum
architectural complexity into every project.

A small application may not need:

- multiple modules;
- complex domain models;
- elaborate CQRS;
- many pipeline behaviors;
- multiple infrastructure adapters.

The purpose of Xeno.JS is not to force every application into the same level of
abstraction.

Instead, the architecture provides explicit boundaries that can be used when the
application needs them.

The important distinction is between **architectural capability** and
**architectural complexity**.

---

## Summary

Xeno.JS is guided by a small set of architectural principles:

1. **Make application architecture explicit.**
2. **Keep application logic independent from transport.**
3. **Make dependencies explicit.**
4. **Compose the application at a defined composition root.**
5. **Treat dependency lifetimes and scopes as architectural concerns.**
6. **Make application execution composable through explicit pipelines.**
7. **Represent application intent explicitly through commands and queries.**
8. **Keep domain concepts independent from infrastructure.**
9. **Treat frameworks and transports as boundaries around the application.**
10. **Prefer understandable explicit mechanisms over hidden framework
    behavior.**
11. **Use a shared architectural language across the Xeno.JS ecosystem.**
12. **Keep the architecture understandable as the system grows.**

Together, these principles define the architectural mindset of Xeno.JS.

---

---

<a id="source-10"></a>

# Source: `docs/introduction/why-xeno-js.md`

## Introduction

A TypeScript application rarely becomes difficult because of its first few
features.

The complexity usually appears later.

As the application grows, business rules, databases, external services,
authentication, validation, caching, background jobs, and transport concerns
begin to interact.

At that point, the main challenge is no longer simply writing code.

It is **keeping the architecture understandable as the codebase grows**.

Xeno.JS exists to address that problem.

---

## The problem is architectural complexity

A small application can often work well with a straightforward structure:

```text
Request
  ↓
Controller
  ↓
Service
  ↓
Database
```

As the application grows, that structure can become harder to maintain.

A service may start handling business rules, database access, authorization,
external APIs, logging, and validation at the same time.

Controllers can become responsible for more than transport.

Infrastructure concerns can leak into domain logic.

Dependencies can become difficult to see.

The architecture still exists, but much of it is implicit.

Xeno.JS takes a different approach:

> **Make the application architecture explicit.**

Instead of relying on conventions scattered across the codebase, Xeno.JS
provides explicit application-level primitives and boundaries.

---

## Keep the application independent from the transport

One of the main reasons for separating application architecture from transport
is that the same application logic may be reached through different entry
points.

For example:

```text
                 ┌── HTTP
                 │
                 ├── CLI
                 │
                 ├── Worker
                 │
                 └── Other transport
                         │
                         ▼
                  Xeno.JS Application
                         │
                         ▼
                       Domain
```

The transport determines **how the application is reached**.

The application determines **what the system does**.

Keeping those concerns separate means application use cases do not need to be
designed around a particular HTTP framework.

This allows an application to use frameworks such as Fastify, Express, or Hono
without making them the architectural center of the application.

---

## Make dependencies explicit

As applications grow, dependency relationships become increasingly important.

A use case may depend on a repository.

A repository may depend on a database.

Another service may depend on an external API.

Authentication may depend on a token provider.

When these relationships are hidden inside framework mechanisms or constructed
throughout the application, understanding the dependency graph becomes harder.

Xeno.JS makes dependency injection part of application composition.

The goal is simple:

```text
Application
    │
    ├── Use Case
    │      ├── Repository
    │      └── External Service
    │
    └── Infrastructure
```

The dependencies are part of the architecture rather than an implementation
detail scattered across the codebase.

---

## Keep business logic separate from infrastructure

Business rules should not need to know whether data comes from PostgreSQL, an
external API, a cache, or another implementation.

A growing application benefits from a clear distinction between:

```text
Domain
  ↓
Application
  ↓
Infrastructure
```

The domain expresses business concepts and rules.

The application coordinates use cases.

Infrastructure provides concrete implementations.

Xeno.JS provides contracts and application primitives that help maintain these
boundaries.

---

## Treat application execution as an architectural concern

A real application does more than execute a function.

A use case may need validation, authorization, logging, error handling,
performance monitoring, idempotency, or other cross-cutting behavior.

Without a consistent execution model, these concerns can end up duplicated
across controllers and services.

Xeno.JS provides commands, queries, and pipelines as part of its application
model.

Conceptually:

```text
Command / Query
       ↓
    Pipeline
       ↓
     Handler
       ↓
 Application Result
```

This makes cross-cutting execution part of the application architecture instead
of something every feature needs to assemble independently.

---

## Reduce framework coupling

Frameworks are useful because they provide conventions and infrastructure.

The problem appears when application logic becomes dependent on those
conventions.

When business logic, use cases, and domain concepts depend directly on a
transport framework, changing that framework can become an architectural change
rather than a transport change.

Xeno.JS is designed to keep the application layer independent from that choice.

That means the application can use:

```text
Fastify
Express
Hono
CLI
Worker
```

as entry points while keeping the application model underneath them.

---

## One application model instead of many unrelated patterns

Large TypeScript applications often accumulate multiple architectural patterns
over time.

One module may use services.

Another may use controllers.

Another may introduce a custom command bus.

Another may use a separate dependency injection solution.

Another may implement its own middleware pipeline.

Each individual choice can be reasonable.

The problem is the resulting system: developers have to understand many
different ways of expressing application behavior.

Xeno.JS brings dependency injection, commands, queries, pipelines, context,
modules, and application composition into one architectural model.

The goal is not to eliminate every architectural choice.

It is to provide a consistent foundation for those choices.

---

## The Xeno.JS approach

The central idea can be summarized in one sentence:

> **Make application architecture explicit in code.**

Xeno.JS provides a common foundation for defining:

- where application logic lives
- how dependencies are composed
- how use cases are executed
- how cross-cutting concerns are applied
- how domain and infrastructure remain separated
- how the application stays independent from its transport

The next page explains how these pieces fit together.

---

## Next steps

- [Architecture Overview](./architecture-overview)
- [Core Principles](./core-principles)

---

---

<a id="source-11"></a>

# Source: `docs/introduction.md`

## Introduction

Xeno.JS is an **application architecture framework for TypeScript**.

It helps you structure applications around explicit boundaries between
application logic, domain logic, dependencies, and infrastructure.

Xeno.JS is not an HTTP framework and does not replace the framework you use to
expose your application.

> **Your HTTP framework handles HTTP. Xeno.JS handles the application.**

You can use Xeno.JS with the transport, UI framework, or runtime that fits your
application.

---

## The problem

As a TypeScript application grows, more responsibilities start to accumulate:

- business rules
- use cases
- database access
- external services
- authentication and authorization
- validation
- background jobs
- caching
- logging
- HTTP or other transports

Without clear architectural boundaries, these responsibilities tend to become
increasingly coupled.

Controllers start containing business logic. Domain logic starts depending on
infrastructure. Dependencies become difficult to understand. Changing one part
of the system can require changes across unrelated parts.

The problem is not TypeScript itself.

The problem is that the **application architecture is often implicit**.

Xeno.JS makes that architecture explicit in code.

---

## Where Xeno.JS fits

Xeno.JS sits at the application layer rather than at the transport layer.

A simplified view is:

```text
┌─────────────────────────────┐
│ HTTP / CLI / Worker / UI    │
├─────────────────────────────┤
│ Presentation                │
├─────────────────────────────┤
│ Xeno.JS Application Layer   │
│                             │
│ Commands / Queries          │
│ Application Services        │
│ Pipelines                   │
│ Dependency Injection        │
│ Request Context             │
├─────────────────────────────┤
│ Domain                      │
├─────────────────────────────┤
│ Infrastructure              │
└─────────────────────────────┘
```

The transport is responsible for receiving and returning data.

The application layer is responsible for executing application use cases.

The domain contains business rules.

Infrastructure provides the concrete implementations required by the
application.

Xeno.JS provides the architectural primitives that connect these layers while
keeping their boundaries explicit.

---

## What does Xeno.JS provide?

Xeno.JS brings together several application-level building blocks that are
commonly assembled separately in TypeScript projects.

These include:

- explicit dependency injection
- application scopes and dependency lifetimes
- commands and queries
- execution pipelines
- request context
- application and domain contracts
- modular application composition

The goal is not simply to provide more utilities.

The goal is to give these pieces a **coherent application architecture**.

---

## Framework-agnostic by design

Xeno.JS does not require your application to adopt a specific HTTP framework.

You can keep the transport layer that already fits your system and use Xeno.JS
underneath it.

For example:

```text
Fastify ──┐
Express ──┤
Hono ─────┼──> Xeno.JS application
CLI ──────┤
Worker ───┘
```

This separation allows the application architecture to remain independent from
the way a request enters the system.

The same principle can also be applied on the frontend through the Xeno.JS
ecosystem.

---

## A framework for the application, not just the transport

Traditional TypeScript frameworks often start from the transport:

```text
Request
   ↓
Controller
   ↓
Service
   ↓
Database
```

Xeno.JS starts from the application instead:

```text
Transport
   ↓
Application Use Case
   ↓
Domain
   ↓
Infrastructure
```

The distinction is important.

An HTTP framework answers questions such as:

> How do I receive an HTTP request?

Xeno.JS focuses on questions such as:

> How is this application use case executed? Which dependencies does it require?
> Which boundaries should it cross? Which cross-cutting behaviors should run
> around it? How can the application remain independent from its transport?

---

## When does Xeno.JS make sense?

Xeno.JS is particularly relevant when an application has meaningful business
logic and its architecture needs to remain understandable as the system grows.

It can be useful when you want:

- explicit application boundaries
- explicit dependencies instead of hidden framework behavior
- a consistent model for commands and queries
- clear separation between domain and infrastructure
- reusable application logic across different transports
- an architecture that can evolve without coupling everything to a single
  framework

For a small application with very little business logic, introducing an
application architecture framework may not be necessary.

Xeno.JS is designed for applications where **architecture itself becomes an
important part of the problem**.

---

## The core idea

Xeno.JS is built around a simple idea:

> **Make application architecture explicit in code.**

Instead of allowing application boundaries to emerge implicitly as the codebase
grows, Xeno.JS provides explicit primitives for defining those boundaries from
the beginning.

The next pages explain the reasoning behind this approach and how the
architecture is structured.

## Next steps

- [Why Xeno.JS?](./introduction/why-xeno-js)
- [Architecture Overview](./introduction/architecture-overview)
- [Core Principles](./introduction/core-principles)
- [Getting Started](./getting-started/installation)

---
