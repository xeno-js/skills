# Data and persistence

Technical reference assembled from the website documentation. Each section
identifies its original Markdown source. Website frontmatter and the repeated
sponsorship footer are omitted; body content and fenced examples are retained.

---

## Contents

- [`docs/data/datasources/datasource.md`](#source-1)
- [`docs/data/db-schema.md`](#source-2)
- [`docs/data/entities/aggregate-root.md`](#source-3)
- [`docs/data/entities/entity.md`](#source-4)
- [`docs/data/entities/events.md`](#source-5)
- [`docs/data/entities/unique-id.md`](#source-6)
- [`docs/data/entities/value-object.md`](#source-7)
- [`docs/data/node-postgresql.md`](#source-8)
- [`docs/data/outbox.md`](#source-9)
- [`docs/data/overview.md`](#source-10)
- [`docs/data/repositories/read-dao.md`](#source-11)
- [`docs/data/repositories/repository.md`](#source-12)
- [`docs/data/sqllite.md`](#source-13)
- [`docs/data/unit-of-work.md`](#source-14)

<a id="source-1"></a>

# Source: `docs/data/datasources/datasource.md`

## Introduction

A data source is the infrastructure-facing implementation of an application's
data-access contract. It translates operations on DTOs into queries or commands
for a database or another provider.

The application depends on the contract, not on the database library:

```text
Application use case
        │
        ▼
IReadDataSource<TDto> or IWriteDataSource<TDto>
        │
        ▼
Concrete data source
        │
        ▼
Database or external provider
```

This boundary keeps provider-specific query code out of application logic. The
trade-off is an explicit contract and implementation for each data source; that
separation makes provider access replaceable and testable.

## Choose a data-source contract

Import the contracts from `@xeno-js/core`:

```ts
import type { IReadDataSource, IWriteDataSource } from '@xeno-js/core'
```

Both interfaces are generic over the DTO used at the data boundary, and both
extend `IDisposable`.

### Read operations

Use `IReadDataSource<TDto>` when the implementation only needs to retrieve data:

```ts
import type { Optional, UserContext } from '@xeno-js/core'

interface IReadDataSource<TDto> {
  findAll(
    ctx?: Optional<UserContext>,
    signal?: Optional<AbortSignal>,
  ): Promise<TDto[]>

  findById(
    id: string | number,
    ctx?: Optional<UserContext>,
    signal?: Optional<AbortSignal>,
  ): Promise<Optional<TDto>>
}
```

`Optional<T>` means `T | undefined`. Consequently, `findById()` represents a
missing row as `undefined`; the contract does not throw a not-found error or
wrap the result in a Xeno.JS `ResultType`.

### Read and write operations

`IWriteDataSource<TDto>` contains the same read operations and adds:

```ts
import type { Optional, UserContext } from '@xeno-js/core'

interface IWriteDataSource<TDto> {
  findAll(
    ctx?: Optional<UserContext>,
    signal?: Optional<AbortSignal>,
  ): Promise<TDto[]>

  findById(
    id: string | number,
    ctx?: Optional<UserContext>,
    signal?: Optional<AbortSignal>,
  ): Promise<Optional<TDto>>

  insert(dto: TDto, signal?: Optional<AbortSignal>): Promise<void>

  delete(
    dto: TDto,
    ctx?: Optional<UserContext>,
    signal?: Optional<AbortSignal>,
  ): Promise<void>

  update(
    id: string | number,
    dto: Partial<TDto>,
    ctx?: Optional<UserContext>,
    signal?: Optional<AbortSignal>,
  ): Promise<void>
}
```

The full interface also declares `findAll()` and `findById()`. The current
contracts declare `IWriteDataSource` and `IReadDataSource` independently;
`IWriteDataSource` does not extend `IReadDataSource`, even though both expose
the read methods.

The methods intentionally return `void` for writes. They do not specify
generated identifiers, affected-row counts, or database-specific return values.
Add a more specific application contract if a use case needs those results.

## Implement a read data source

The data source owns the provider query and returns DTOs. For example, a
PostgreSQL-backed implementation can use the typed `DbContext` and Drizzle query
builder:

```ts
// user.schema.ts
import { pgTable, text, varchar } from 'drizzle-orm/pg-core'

export const users = pgTable('users', {
  id: text('id').primaryKey(),
  name: varchar('name', { length: 255 }).notNull(),
})
```

```ts
// schema.ts
import type { users } from './user.schema'

export type DbSchema = { users: typeof users }
```

The implementation can then use that schema and the inferred row type:

```ts
import { AppError } from '@xeno-js/core'
import type { IReadDataSource, Optional, UserContext } from '@xeno-js/core'
import type { DbContext } from '@xeno-js/postgresql'
import { eq } from 'drizzle-orm'

import type { DbSchema } from './schema'
import { users } from './user.schema'
type UserDto = typeof users.$inferSelect

export class UserDataSource implements IReadDataSource<UserDto> {
  constructor(private readonly db: DbContext<DbSchema>) {}

  async findAll(
    ctx?: Optional<UserContext>,
    signal?: Optional<AbortSignal>,
  ): Promise<UserDto[]> {
    AppError.throwIfAborted(signal, 'UserDataSource.findAll')

    return this.db.select().from(users).execute()
  }

  async findById(
    id: string | number,
    ctx?: Optional<UserContext>,
    signal?: Optional<AbortSignal>,
  ): Promise<Optional<UserDto>> {
    AppError.throwIfAborted(signal, 'UserDataSource.findById')

    const [user] = await this.db
      .select()
      .from(users)
      .where(eq(users.id, String(id)))
      .limit(1)
      .execute()

    return user
  }

  async dispose(): Promise<void> {}
}
```

The schema type binds the database context to the table definition. The
implementation checks the optional `AbortSignal` before starting each query. A
provider operation that supports cancellation can also receive the signal; the
data-source contract does not automatically cancel an operation.

This example does not use `ctx`, so it performs no user- or tenant-specific
filtering. If the operation requires that boundary, apply the relevant
conditions in the data source rather than assuming the contract enforces them.
An empty `dispose()` is appropriate only when the data source owns no resources;
dispose resources that the implementation itself acquires.

## Add write operations

Implement all five query and command methods, plus `dispose()`, when the
application needs `IWriteDataSource<TDto>`. The signatures are shown above; the
implementation maps each operation to provider-specific calls.

The method contracts distinguish the update key from the partial DTO, while
`delete()` receives the DTO itself. Implementations decide how these inputs map
to provider operations. For example, a database implementation may soft-delete a
row rather than physically remove it, if that matches the application's
persistence rules.

The shared contracts do not define filtering, sorting, pagination, transactions,
or authorization behavior. Add explicit methods to an application-specific
contract for those requirements:

```ts
import type {
  IPaginatedResult,
  IWriteDataSource,
  Optional,
  UserContext,
} from '@xeno-js/core'

type OrderDto = { id: string; userId: string }

interface IOrderDataSource extends IWriteDataSource<OrderDto> {
  findByUserId(
    userId: string,
    ctx: UserContext,
    signal: Optional<AbortSignal>,
  ): Promise<OrderDto[]>

  findPaginatedByUserId(
    userId: string,
    page: number,
    pageSize: number,
    ctx: UserContext,
    signal: Optional<AbortSignal>,
  ): Promise<IPaginatedResult<OrderDto>>
}
```

These methods are application-specific extensions; the generic
`IWriteDataSource` does not supply them.

## Register and resolve the implementation

Keep the application token and its implementation type in the registry. Register
the data source while composing the application with `XenoAppBuilder`:

```ts
import { XenoAppBuilder, TOKENS } from '@xeno-js/core'
import type { XenoRegistry } from '@xeno-js/core'
import type { DbContext, DbTransaction } from '@xeno-js/postgresql'

import type { DbSchema } from './schema'
import { UserDataSource } from './user.datasource'

interface AppRegistry extends XenoRegistry<DbContext<DbSchema>, DbTransaction> {
  USER_DATA_SOURCE: UserDataSource
}

const app = new XenoAppBuilder<AppRegistry>()

// Configure the database provider with app.addDb(...) before resolving DB_CONTEXT.
app.addServices((services) => {
  services.addScoped(
    'USER_DATA_SOURCE',
    (container) => new UserDataSource(container.resolve(TOKENS.DB_CONTEXT)),
  )
})

const application = await app.build()

await application.runInScope(async (services) => {
  const users = await services.USER_DATA_SOURCE.findAll()
  console.log(users)
})
```

`addServices()` queues the service registration as part of builder composition.
`build()` configures the queued modules; `runInScope()` creates an execution
scope in which the scoped data source can be resolved. The database provider
must register `DB_CONTEXT` before the data source factory runs.

Choose the lifetime according to the dependencies:

- Use `addScoped()` when the data source depends on a scoped database context,
  transaction, or request-specific state.
- Use `addSingleton()` only when all dependencies are safe to share for the
  lifetime of the container.
- Use `addTransient()` when each resolution should create a separate instance.

Do not capture a scoped dependency in a singleton data source. See
[Xeno Registry](../../fundamentals/xeno-registry),
[App Builder](../../fundamentals/app-builder), and
[XenoApplication](../../fundamentals/xeno-application) for registry typing,
composition, and scoped resolution.

## Data source, repository, and mapper

A data source deals with DTOs and provider operations. It does not own the
domain entity mapping. A repository or DAO can coordinate the data source with
an `IMapper`:

```text
Application
    │
    ▼
Repository or DAO
    ├── IMapper<TEntity, TDto>
    └── IWriteDataSource<TDto>
              │
              ▼
          Data provider
```

Use a data source directly when the application contract is intentionally
DTO-oriented. Use a repository or DAO when the application needs entity-oriented
operations and mapping. See [Repository](../repositories/repository) and
[ReadDao](../repositories/read-dao) for those abstractions.

---

---

<a id="source-2"></a>

# Source: `docs/data/db-schema.md`

## Introduction

A Xeno.JS database schema defines the tables used by your application and gives
the database context a concrete TypeScript type.

The basic flow is:

```text
Individual table definitions
        ↓
      DbSchema
        ↓
XenoRegistry<DbContext<DbSchema>, DbTransaction, ApplicationServices>
        ↓
 dbContext (`TOKENS.DB_CONTEXT`)
        ↓
   Data Sources
```

For example, an application can define `users` and `profiles` as separate tables
and then group them into one `DbSchema`:

```ts
import { users } from './data/schema/user.schema'
import { profiles } from './data/schema/profile.schema'

type DbSchema = {
  users: typeof users
  profiles: typeof profiles
}
```

That type can then be supplied to `XenoRegistry`:

```ts
import type { DbSchema } from './data/schema'
import type { XenoRegistry } from '@xeno-js/core'
import type { DbContext, DbTransaction } from '@xeno-js/postgresql'

interface IUserRepository {
  findById(id: string): Promise<void>
}

interface IProfileRepository {
  findByUserId(userId: string): Promise<void>
}

interface MyAppRegistry extends XenoRegistry<
  DbContext<DbSchema>,
  DbTransaction
> {
  USER_REPOSITORY: IUserRepository
  PROFILE_REPOSITORY: IProfileRepository
}
```

The result is a typed application registry where the `dbContext` service,
registered under `TOKENS.DB_CONTEXT`, uses the application's database schema.

## Prerequisites

You need:

- Xeno.JS Core
- Drizzle ORM
- a database provider such as PostgreSQL or SQLite/libSQL
- TypeScript

For PostgreSQL, the table definitions use Drizzle's PostgreSQL builders.

```bash
npm install @xeno-js/core @xeno-js/postgresql drizzle-orm pg
```

For SQLite/libSQL, install `@xeno-js/sqlite` and use the corresponding
Drizzle/libSQL packages.

The database connection itself is configured separately from the schema.

## Create a Table Schema

Define each table in its own file when the application contains multiple tables.

For example:

```text
src/
├── data/
│   └── schema/
│       ├── user.schema.ts
│       ├── profile.schema.ts
│       └── index.ts
├── registry.ts
└── bootstrap.ts
```

A PostgreSQL `users` table can be defined with Drizzle:

```ts
import { integer, pgTable, varchar } from 'drizzle-orm/pg-core'

export const users = pgTable('users', {
  id: integer('id').primaryKey().generatedAlwaysAsIdentity(),
  email: varchar('email', { length: 255 }).notNull(),
})
```

A `profiles` table can be defined separately:

```ts
import { integer, pgTable, varchar } from 'drizzle-orm/pg-core'

export const profiles = pgTable('profiles', {
  id: integer('id').primaryKey().generatedAlwaysAsIdentity(),
  userId: integer('user_id').notNull(),
  displayName: varchar('display_name', { length: 255 }).notNull(),
})
```

These are normal Drizzle table definitions. Xeno.JS does not replace the
database provider's schema-definition API.

## Create the DbSchema Type

Once the individual tables exist, group them into one schema type.

Create `src/data/schema/index.ts`:

```ts
import { profiles } from './profile.schema'
import { users } from './user.schema'

export type DbSchema = {
  profiles: typeof profiles
  users: typeof users
}
```

The keys of `DbSchema` should correspond to the tables that you want to expose
through the typed database context.

The values are the actual Drizzle table definitions.

This is the important distinction:

```ts
import { users } from './user.schema'
import { profiles } from './profile.schema'

type DbSchema = {
  users: typeof users
  profiles: typeof profiles
}
```

`DbSchema` is the **TypeScript type**.

The actual table objects remain:

```ts
users
profiles
```

## Use DbSchema with XenoRegistry

Define a registry using the core type and the database provider's context and
transaction types:

```ts
import type { XenoRegistry } from '@xeno-js/core'
import type { DbContext, DbTransaction } from '@xeno-js/postgresql'
```

The first parameter is the typed database context, the second is the provider's
transaction type, and the optional third parameter can add application-specific
service keys.

An application registry can extend it with `DbSchema` and its own service keys.

For example:

```ts
import type { XenoRegistry } from '@xeno-js/core'
import type { DbContext, DbTransaction } from '@xeno-js/postgresql'

import type { DbSchema } from './data/schema'
import type { IProfileRepository } from './data/profile.repository'
import type { IUserRepository } from './data/user.repository'

export interface MyAppRegistry extends XenoRegistry<
  DbContext<DbSchema>,
  DbTransaction
> {
  USER_REPOSITORY: IUserRepository
  PROFILE_REPOSITORY: IProfileRepository
}
```

The structure is:

```text
XenoRegistry<DbContext<DbSchema>, DbTransaction>
  + application-specific DI tokens
```

The `DbContext<DbSchema>` parameter describes the schema. `DbTransaction`
describes the provider transaction type; define application-specific tokens by
extending the registry interface.

For example:

```ts
interface MyAppRegistry extends XenoRegistry<
  DbContext<DbSchema>,
  DbTransaction
> {
  USER_REPOSITORY: IUserRepository
  PROFILE_REPOSITORY: IProfileRepository
}
```

This keeps database schema typing and application dependency typing in the same
registry.

## Understand the dbContext Type

The shared token constant is `TOKENS.DB_CONTEXT` (whose service key is
`dbContext`); the typed registry property uses the `dbContext` name.

`ApplicationRegistry` defines the typed database service property as:

```ts
readonly dbContext: T
```

`XenoRegistry<DbContext<DbSchema>, DbTransaction>` specializes
`ApplicationRegistry` with:

```ts
ApplicationRegistry<DbContext<DbSchema>, DbTransaction>
```

Therefore, when your registry extends:

```ts
XenoRegistry<DbContext<DbSchema>, DbTransaction>
```

the `dbContext` service is typed using:

```ts
DbContext<DbSchema>
```

This is what connects your application-defined schema to Xeno.JS's database
context.

The relationship is:

```text
DbSchema
   │
   ▼
XenoRegistry<DbContext<DbSchema>, DbTransaction>
   │
   ▼
ApplicationRegistry<DbContext<DbSchema>>
   │
   ▼
dbContext
```

You do not manually register the `DbSchema` as a service.

It is a compile-time type parameter.

## Configure the Database Separately

Defining `DbSchema` does not configure the database connection.

Database connectivity is configured separately by passing a provider plugin to
`XenoAppBuilder.addDb()`.

For PostgreSQL:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withPostgresql } from '@xeno-js/postgresql'

const app = new XenoAppBuilder()

app.addDb(
  withPostgresql((options, config) => {
    options.connectionString = config.get('DATABASE_URL')!
  }),
)
```

The two concerns are therefore separate:

```text
DbSchema
    │
    └── describes tables and their types

XenoAppBuilder.addDb(withPostgresql(...))
    │
    └── configures database connectivity
```

You need both when building a database-backed application.

## Resolve the Typed Database Context

After the application is built, the database context is available as the scoped
`dbContext` service.

Resolve it from the runtime application's active scope:

```ts
const application = await app.build()

await application.runInScope(async (services) => {
  const db = services.dbContext
  // db is typed using the registry's DbSchema.
})
```

If your application uses a typed `MyAppRegistry`, the registry connects the
`dbContext` service to:

```ts
DbContext<DbSchema>
```

This allows the database API to retain the schema information defined by your
application.

## Use DbSchema in a Data Source

Implement the shared data-source contract required by your application and
inject the provider's `DbContext<DbSchema>`. Xeno.JS does not provide a
PostgreSQL base data-source class.

```ts
import { AppError, Enumerable } from '@xeno-js/core'
import type { IReadDataSource, Optional, UserContext } from '@xeno-js/core'
import type { DbContext } from '@xeno-js/postgresql'
import { eq } from 'drizzle-orm'

import type { DbSchema } from './schema'
import { users } from './schema/user.schema'

type UserDto = typeof users.$inferSelect

export class UserDataSource implements IReadDataSource<UserDto> {
  constructor(private readonly _db: DbContext<DbSchema>) {}

  async findAll(
    _ctx?: Optional<UserContext>,
    signal?: Optional<AbortSignal>,
  ): Promise<UserDto[]> {
    AppError.throwIfAborted(signal, 'UserDataSource.findAll')
    return this._db.select().from(users).execute()
  }

  async findById(
    id: string | number,
    _ctx?: Optional<UserContext>,
    signal?: Optional<AbortSignal>,
  ): Promise<Optional<UserDto>> {
    AppError.throwIfAborted(signal, 'UserDataSource.findById')
    const result = await this._db
      .select()
      .from(users)
      .where(eq(users.id, Number(id)))
      .limit(1)
      .execute()
    return Enumerable.firstOrDefault(result)
  }

  async dispose(): Promise<void> {}
}
```

The same `DbSchema` used by the registry is supplied to the data source's
constructor.

## Register a Data Source

Application-specific data sources are normal DI services.

For example:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withPostgresql } from '@xeno-js/postgresql'
import type { MyAppRegistry } from './registry'
import { UserDataSource } from './data/user.data-source'

const app = new XenoAppBuilder<MyAppRegistry>()

app.addDb(
  withPostgresql<MyAppRegistry>((options, config) => {
    options.connectionString = config.get('DATABASE_URL')!
  }),
)

app.addServices((services) => {
  services.addScoped('USER_DATA_SOURCE', (scope) => {
    const db = scope.resolve('dbContext')

    return new UserDataSource(db)
  })
})
```

The exact registration token must also exist in the application's registry.

For example:

```ts
interface MyAppRegistry extends XenoRegistry<
  DbContext<DbSchema>,
  DbTransaction
> {
  USER_DATA_SOURCE: UserDataSource
}
```

This gives the application a consistent type flow:

```text
DbSchema
   ↓
XenoRegistry<DbContext<DbSchema>, DbTransaction>
   ↓
dbContext: DbContext<DbSchema>
   ↓
UserDataSource<DbSchema>
```

## Organize a Larger Schema

For a larger application, keep each table definition separate and expose the
complete schema from one module.

For example:

```text
src/
└── data/
    └── schema/
        ├── user.schema.ts
        ├── profile.schema.ts
        ├── order.schema.ts
        ├── order-item.schema.ts
        └── index.ts
```

Each file defines one table:

```ts
// user.schema.ts
export const users = ...
```

```ts
// profile.schema.ts
export const profiles = ...
```

```ts
// order.schema.ts
export const orders = ...
```

The schema entry point groups them:

```ts
import { orderItems } from './order-item.schema'
import { orders } from './order.schema'
import { profiles } from './profile.schema'
import { users } from './user.schema'

export type DbSchema = {
  orderItems: typeof orderItems
  orders: typeof orders
  profiles: typeof profiles
  users: typeof users
}
```

This gives the rest of the application one stable type to import:

```ts
import type { DbSchema } from './data/schema'
```

Instead of importing every table definition whenever a type parameter is
required.

## Keep Table Definitions and DbSchema Together

A useful convention is:

```text
data/
└── schema/
    ├── user.schema.ts
    ├── profile.schema.ts
    ├── order.schema.ts
    └── index.ts
```

The individual files own the table definitions.

`index.ts` owns the application-level schema type.

For example:

```ts
// data/schema/user.schema.ts
export const users = ...
```

```ts
// data/schema/profile.schema.ts
export const profiles = ...
```

```ts
// data/schema/index.ts
import { profiles } from './profile.schema'
import { users } from './user.schema'

export type DbSchema = {
  profiles: typeof profiles
  users: typeof users
}
```

This makes the schema boundary easy to find when adding a new table.

## PostgreSQL and SQLite

The `DbSchema` concept is independent from the Xeno.JS application registry, but
the actual table definitions are provided by Drizzle's database-specific APIs.

For PostgreSQL:

```ts
import { pgTable, integer, varchar } from 'drizzle-orm/pg-core'
```

For SQLite/libSQL, use the corresponding SQLite Drizzle builders.

The registry must use the context and transaction types from the provider
selected at bootstrap. For SQLite/libSQL, import those types from
`@xeno-js/sqlite`:

```ts
import type { XenoRegistry } from '@xeno-js/core'
import type { DbContext, DbTransaction } from '@xeno-js/sqlite'

interface SqliteAppRegistry extends XenoRegistry<
  DbContext<DbSchema>,
  DbTransaction
> {}
```

The application-level pattern remains:

```ts
type DbSchema = {
  users: typeof users
  profiles: typeof profiles
}
```

However, do not assume that a PostgreSQL table definition can be reused
unchanged for SQLite. The table builders and supported column types are
provider-specific.

Choose provider-specific schema definitions and registry types that match the
database plugin passed to `XenoAppBuilder.addDb()`.

## Complete Example

A small PostgreSQL application can use this structure:

```text
src/
├── data/
│   └── schema/
│       ├── user.schema.ts
│       ├── profile.schema.ts
│       └── index.ts
├── registry.ts
└── bootstrap.ts
```

### User table

```ts
// src/data/schema/user.schema.ts

import { integer, pgTable, varchar } from 'drizzle-orm/pg-core'

export const users = pgTable('users', {
  id: integer('id').primaryKey().generatedAlwaysAsIdentity(),
  email: varchar('email', { length: 255 }).notNull(),
})
```

### Profile table

```ts
// src/data/schema/profile.schema.ts

import { integer, pgTable, varchar } from 'drizzle-orm/pg-core'

export const profiles = pgTable('profiles', {
  id: integer('id').primaryKey().generatedAlwaysAsIdentity(),
  userId: integer('user_id').notNull(),
  displayName: varchar('display_name', { length: 255 }).notNull(),
})
```

### DbSchema

```ts
// src/data/schema/index.ts

import { profiles } from './profile.schema'
import { users } from './user.schema'

export type DbSchema = {
  profiles: typeof profiles
  users: typeof users
}
```

### Application registry

```ts
// src/registry.ts

import type { XenoRegistry } from '@xeno-js/core'
import type { DbContext, DbTransaction } from '@xeno-js/postgresql'

import type { DbSchema } from './data/schema'
import type { UserDataSource } from './data/user.data-source'

export interface MyAppRegistry extends XenoRegistry<
  DbContext<DbSchema>,
  DbTransaction
> {
  USER_DATA_SOURCE: UserDataSource
}
```

The application-specific token types can be expanded as the application grows:

```ts
export interface MyAppRegistry extends XenoRegistry<
  DbContext<DbSchema>,
  DbTransaction
> {
  USER_DATA_SOURCE: UserDataSource
  PROFILE_DATA_SOURCE: ProfileDataSource
  USER_REPOSITORY: IUserRepository
  PROFILE_REPOSITORY: IProfileRepository
}
```

The database schema remains a separate concern:

```ts
type DbSchema = {
  profiles: typeof profiles
  users: typeof users
}
```

### Database bootstrap

```ts
// src/bootstrap.ts

import { XenoAppBuilder } from '@xeno-js/core'
import { withPostgresql } from '@xeno-js/postgresql'
import type { MyAppRegistry } from './registry'

const app = new XenoAppBuilder<MyAppRegistry>()

app.addDb(
  withPostgresql<MyAppRegistry>((options, config) => {
    options.connectionString = config.get('DATABASE_URL')!
  }),
)

export { app }
```

The important relationship is now explicit:

```text
user.schema.ts
profile.schema.ts
       │
       ▼
    DbSchema
       │
       ▼
MyAppRegistry extends XenoRegistry<DbContext<DbSchema>, DbTransaction>
       │
       ▼
 dbContext: DbContext<DbSchema>
       │
       ▼
 application data sources
```

## Common Problems

### `DbSchema` is defined but `dbContext` is not typed

Make sure the application registry specializes `XenoRegistry` with the
provider's context type:

```ts
interface MyAppRegistry extends XenoRegistry<
  DbContext<DbSchema>,
  DbTransaction
> {}
```

Not:

```ts
interface MyAppRegistry extends XenoRegistry {}
```

The latter leaves the database context at the default untyped `unknown` type.

### A table is missing from the database type

Add the table to `DbSchema`:

```ts
type DbSchema = {
  users: typeof users
  profiles: typeof profiles
  orders: typeof orders
}
```

Defining a table file alone does not add it to the application schema type.

### The database connection does not work

Check the database configuration independently from `DbSchema`.

For PostgreSQL:

```ts
app.addDb(
  withPostgresql((options, config) => {
    options.connectionString = config.get('DATABASE_URL')!
  }),
)
```

`DbSchema` describes the database structure at the TypeScript level. It does not
establish the database connection.

### A scoped database service cannot be resolved

Database services registered by Xeno.JS are scoped.

Resolve them through the runtime application's active scope:

```ts
const application = await app.build()

await application.runInScope(async (services) => {
  const db = services.dbContext
})
```

For request-driven code, use the application's active request scope rather than
creating an unrelated scope for each operation.

### The PostgreSQL data source rejects the schema type

Make sure the registry and data source use the same provider context and schema
type:

```ts
import type { XenoRegistry } from '@xeno-js/core'
import type { DbContext, DbTransaction } from '@xeno-js/postgresql'

type DbSchema = {
  users: typeof users
  profiles: typeof profiles
}

interface MyAppRegistry extends XenoRegistry<
  DbContext<DbSchema>,
  DbTransaction
> {}

class UserDataSource {
  constructor(private readonly db: DbContext<DbSchema>) {}
  // ...
}
```

Using one shared `DbSchema` type prevents the registry and data sources from
describing different database structures.

## DbSchema Is Not a Migration

`DbSchema` describes the database structure to TypeScript and Drizzle.

It does not, by itself, create or migrate database tables.

Keep these concerns separate:

```text
DbSchema
    ↓
Type-safe database access

Database migrations
    ↓
Actual database structure
```

Use the database tooling appropriate to your Drizzle/provider setup to create
and migrate the physical database.

## DbSchema and the Xeno CLI

Xeno CLI provides project scaffolding and generators for the Xeno application
architecture.

The audited CLI does not expose a dedicated command that generates a `DbSchema`
from database tables.

Therefore, create and maintain the schema explicitly in the application:

```text
src/data/schema/
├── user.schema.ts
├── profile.schema.ts
└── index.ts
```

The CLI can provide the application structure around this code, but the database
model remains application-specific.

## Related Docs

- [Data Overview](./overview)
- [Node PostgreSQL](./node-postgresql)
- [SQLite](./sqllite)
- [Unit of Work](./unit-of-work)
- [Service Registration](../dependency-injection/registration)
- [Service Resolution](../dependency-injection/resolution)
- [Dependency Graph](../dependency-injection/dependency-graph)

---

---

<a id="source-3"></a>

# Source: `docs/data/entities/aggregate-root.md`

## Introduction

An aggregate groups domain objects that must remain consistent under one
boundary. Its aggregate root is the entity through which application code
changes that group. In Xeno.JS, extend `AggregateRoot` when the root also needs
versioned domain events and event replay.

## The Problem

Changing an aggregate's state directly does not record what happened, and
independently updating objects inside an aggregate can bypass its invariants.
Persistence and other parts of the system may also need to know which domain
changes have not yet been committed.

## The Principle

Use the aggregate root as the boundary for domain operations. Keep its state
changes behind methods that enforce the aggregate's rules, and represent
meaningful changes as domain events.

The aggregate root belongs in the domain layer. Application services coordinate
commands and persistence; infrastructure implements storage and event delivery.
`AggregateRoot` does not persist or publish events itself.

## The Mechanism

`AggregateRoot<TProps, TEvent, TId>` extends `Entity<TProps, TId>` and provides:

- the stable `id` and property access inherited from `Entity`;
- a `version` that starts at `0` and advances when the root raises an event;
- `getUncommittedEvents()`, which returns a new array containing the pending
  events;
- `clearUncommittedEvents()`, which empties the pending event list without
  resetting state or version;
- `loadFromHistory()`, which applies stored events with `isNew` set to `false`
  and sets the version to each event's version;
- the protected `raise()` method, which assigns the aggregate ID, next version,
  and occurrence time before applying and recording an event;
- the protected abstract `apply()` method, which the concrete aggregate
  implements to update its state.

The event type `IDomainEvent<TEvent, TId>` includes `aggregateId`,
`aggregateType`, `eventType`, `version`, `occurredAt`, and `payload`. It can
also include an optional `destination`.

The root's `apply()` method owns the state transition. The base class calls it
with `isNew: true` for a newly raised event and `isNew: false` when replaying
history. It does not validate history ordering, version gaps, or aggregate
identity, so the caller must provide history belonging to that aggregate in the
intended order.

## Implement an Aggregate Root

The `Order` aggregate keeps its confirmation rule in the domain and applies the
same state transition both for a new event and for replay:

```ts
import { AggregateRoot } from '@xeno-js/shared'
import type { IDomainEvent } from '@xeno-js/shared'

interface OrderProps {
  status: 'draft' | 'confirmed'
}

interface OrderEventPayload {
  status: 'confirmed'
}

export class Order extends AggregateRoot<
  OrderProps,
  OrderEventPayload,
  'Order'
> {
  public constructor(props: OrderProps, id?: string) {
    super(props, id)
  }

  public confirm(): void {
    if (this.getProps().status !== 'draft') {
      throw new Error('Only draft orders can be confirmed.')
    }

    this.raise({
      aggregateType: 'Order',
      eventType: 'OrderConfirmed',
      payload: { status: 'confirmed' },
    })
  }

  protected apply(
    event: IDomainEvent<OrderEventPayload, 'Order'>,
    _isNew: boolean,
  ): void {
    if (event.eventType === 'OrderConfirmed') {
      this.setProps({ ...this.getProps(), status: event.payload.status })
    }
  }
}

const order = new Order({ status: 'draft' })
order.confirm()

console.log(order.getProps().status) // 'confirmed'
console.log(order.version) // 1
console.log(order.getUncommittedEvents()[0].eventType) // 'OrderConfirmed'
```

`raise()` is protected so callers cannot bypass the aggregate's domain methods.
The example's `confirm()` method checks the current state before raising the
event; `apply()` performs the actual state transition.

## Replay Event History

Create an aggregate with its persisted ID, then apply its ordered history to
reconstruct state and version:

```ts
import { AggregateRoot, UniqueId } from '@xeno-js/shared'
import type { IDomainEvent } from '@xeno-js/shared'

interface OrderProps {
  status: 'draft' | 'confirmed'
}

interface OrderEventPayload {
  status: 'confirmed'
}

class Order extends AggregateRoot<OrderProps, OrderEventPayload, 'Order'> {
  public constructor(props: OrderProps, id?: string) {
    super(props, id)
  }

  public confirm(): void {
    if (this.getProps().status !== 'draft') {
      throw new Error('Only draft orders can be confirmed.')
    }

    this.raise({
      aggregateType: 'Order',
      eventType: 'OrderConfirmed',
      payload: { status: 'confirmed' },
    })
  }

  protected apply(
    event: IDomainEvent<OrderEventPayload, 'Order'>,
    _isNew: boolean,
  ): void {
    if (event.eventType === 'OrderConfirmed') {
      this.setProps({ ...this.getProps(), status: event.payload.status })
    }
  }
}

const orderId = 'c0a80123-4567-489a-8abc-1234567890ab'
const history: IDomainEvent<OrderEventPayload, 'Order'>[] = [
  {
    aggregateId: UniqueId.create('Order', orderId),
    aggregateType: 'Order',
    eventType: 'OrderConfirmed',
    version: 1,
    occurredAt: new Date('2025-01-15T10:00:00.000Z'),
    payload: { status: 'confirmed' },
  },
]

const restoredOrder = new Order({ status: 'draft' }, orderId)
restoredOrder.loadFromHistory(history)

console.log(restoredOrder.getProps().status) // 'confirmed'
console.log(restoredOrder.version) // 1
console.log(restoredOrder.getUncommittedEvents().length) // 0
```

The history event's aggregate ID must match the restored root.
`loadFromHistory()` does not add replayed events to the uncommitted list. It
also does not clear events already pending on the instance, so replay history
into a newly constructed root.

## Handling Uncommitted Events

After an application operation changes an aggregate, it can read the pending
events and pass them to the persistence workflow. Clear the list only when that
workflow has successfully handled them:

```ts
const pendingEvents = order.getUncommittedEvents()

// Persist the aggregate and pendingEvents in the application's persistence workflow.
order.clearUncommittedEvents()
```

`getUncommittedEvents()` copies the array, but not the event objects inside it.
Treat returned events as read-only. `clearUncommittedEvents()` only clears the
in-memory list; it does not persist events, publish them, or roll back the
aggregate's state.

## Behavior

- New aggregates start at version `0`.
- `raise()` increments the version and assigns the current time before applying
  the event; it adds the event to the uncommitted list only after `apply()`
  returns.
- If `apply()` throws, `raise()` does not add the event to the uncommitted list,
  but the version has already advanced. Validate domain operations before
  calling `raise()`.
- `loadFromHistory()` applies events without adding them to the uncommitted list
  and sets the version to the version of each event as it iterates.
- `clearUncommittedEvents()` clears pending events but leaves state and version
  unchanged.
- The base class does not validate event order, version continuity, aggregate
  identity, or domain rules. The concrete aggregate and calling workflow own
  those responsibilities.

## Related Documentation

- [Entities](./entity)
- [Unique ID](./unique-id)
- [Value Objects](./value-object)

---

---

<a id="source-4"></a>

# Source: `docs/data/entities/entity.md`

## Introduction

Use an entity to model a domain object whose identity remains stable while its
state changes. Two orders can have the same properties and still be different
orders because they have different identifiers.

## The Problem

Comparing domain objects only by their current properties does not capture their
lifecycle. For example, an order's status or total can change without turning it
into a different order.

## The Principle

An entity has a stable identity and encapsulates changeable state. Use its
identifier to refer to the same domain object over time; use its behavior to
change state while preserving domain rules.

Entities belong in the domain layer. Application services coordinate domain
operations, while infrastructure handles persistence and other external
concerns.

## The Mechanism

`Entity<T, TId>` is an abstract implementation of `IEntity<T, TId>`. It
provides:

- a readonly `id` of type `UniqueId<TId>`;
- `getId()`, which returns the entity's identifier;
- `getProps()`, which returns a structured clone of the current properties;
- `setProps()`, which replaces the properties with a frozen shallow copy.

The constructor is protected, so create concrete entity classes by extending
`Entity`. It accepts initial properties and an optional UUID string. If the ID
is omitted or empty, the base class generates one. If a non-empty ID is invalid,
construction throws an error. The base class does not define domain validation
or entity-specific behavior; the concrete class owns those rules.

## Implement an Entity

This order keeps the same identity while changing its status through a domain
operation:

```ts
import { Entity } from '@xeno-js/shared'

interface OrderProps {
  status: 'draft' | 'confirmed'
  total: number
}

export class Order extends Entity<OrderProps, 'Order'> {
  public constructor(props: OrderProps, id?: string) {
    super(props, id)
  }

  public confirm(): void {
    const props = this.getProps()

    if (props.status !== 'draft') {
      throw new Error('Only draft orders can be confirmed.')
    }

    this.setProps({ ...props, status: 'confirmed' })
  }
}

const order = new Order({ status: 'draft', total: 125 })
const originalId = order.id

order.confirm()

console.log(order.getProps().status) // 'confirmed'
console.log(order.id === originalId) // true
```

The type marker `'Order'` keeps this entity's identifier distinct from other
`UniqueId` kinds at compile time. Keep it aligned with the concrete class name:
the base class uses `this.constructor.name` when creating an ID.

## Identity and Existing IDs

The entity's `id` is readonly, and the base class does not provide an operation
to replace it. Pass a valid UUID when reconstructing an entity from persisted
data:

```ts
const restoredOrder = new Order(
  { status: 'confirmed', total: 125 },
  'c0a80123-4567-489a-8abc-1234567890ab',
)

console.log(restoredOrder.id.toString())
```

An omitted or empty ID causes a new UUID to be generated. A non-empty invalid
UUID throws during construction rather than silently creating a different
identity. Validate external input separately when the calling workflow needs a
more specific error.

## State and Copies

`setProps()` replaces the entity's entire property set. The base class freezes a
shallow copy of the supplied object, but it does not recursively freeze nested
values. `getProps()` returns a `structuredClone` of the stored properties, so
changes to the returned clone do not update the entity:

```ts
const order = new Order({ status: 'draft', total: 125 })
const props = order.getProps()

props.total = 200

console.log(order.getProps().total) // 125
```

These methods do not add domain validation. Keep state changes behind methods on
the concrete entity so those methods can enforce its invariants. For nested
properties, use immutable values or manage their copying and mutation rules in
the concrete type.

## Entities and Aggregate Roots

Use `Entity` as the base for domain entities. When the entity is the root of an
aggregate and needs aggregate versioning and uncommitted domain-event handling,
extend `AggregateRoot`, which builds on `Entity`.

## Related Documentation

- [Aggregate Roots](./aggregate-root)
- [Unique ID](./unique-id)
- [Value Objects](./value-object)

---

---

<a id="source-5"></a>

# Source: `docs/data/entities/events.md`

## Introduction

A domain event records a meaningful change in a domain, such as an order being
confirmed. Xeno.JS describes its structure with `IDomainEvent<TPayload, TId>`, a
typed contract for the event data and the aggregate metadata associated with it.

## The Problem

Passing only an event name or an untyped payload makes it difficult for
consumers to know which aggregate produced the event, which revision it
represents, and what data it carries. A shared event shape makes those parts
explicit while allowing each event payload to remain specific to its domain.

## The Principle

Describe an event as a fact that has occurred, and include only the data its
consumers need to understand that fact. Keep the payload type aligned with the
event's meaning.

Events belong to the domain model. The event interface does not define
persistence, dispatch, or delivery guarantees. Aggregate behavior creates domain
events; application and infrastructure workflows decide how to store or deliver
them.

## The Mechanism

`IDomainEvent<TPayload, TId>` is an interface with readonly fields:

- `aggregateId: UniqueId<TId>` identifies the aggregate associated with the
  event;
- `aggregateType: string` identifies the aggregate type;
- `eventType: string` names the event;
- `version: number` identifies the event's position in the aggregate's version
  sequence;
- `occurredAt: Date` records when the event occurred;
- `payload: TPayload` contains event-specific data;
- `destination?: string` optionally identifies a destination for a consuming
  workflow.

`TPayload` defaults to `unknown`. Specify a payload type to retain type safety
when reading event data. `TId` defaults to `string` and types the aggregate
identifier; use a literal kind such as `'Order'` to distinguish aggregate ID
types in TypeScript.

The `readonly` modifiers prevent assigning to these properties through the
interface in TypeScript. They do not freeze event objects at runtime.

## Type and Consume an Event

Define the payload for an event and specialize `IDomainEvent` with the aggregate
kind. Consumers can then use the event's metadata and payload without treating
the payload as `unknown`:

```ts
import type { IDomainEvent } from '@xeno-js/shared'

interface OrderConfirmedPayload {
  orderNumber: string
  customerId: string
}

type OrderConfirmedEvent = IDomainEvent<OrderConfirmedPayload, 'Order'>

function describeOrderConfirmation(event: OrderConfirmedEvent): string {
  return `${event.payload.orderNumber} was confirmed at ${event.occurredAt.toISOString()}`
}
```

In normal aggregate workflows, the aggregate root creates the complete event.
Its `raise()` method supplies `aggregateId`, increments and assigns `version`,
sets `occurredAt`, applies the event to the aggregate, and records it as
uncommitted. The aggregate's `apply()` implementation handles the state
transition. See [Aggregate Roots](./aggregate-root) for a complete example.

## Event Data and Delivery

An event describes a domain fact; it does not itself publish that fact.
`IDomainEvent` has no methods and does not specify a transport, persistence
format, retry policy, or delivery guarantee. The application and infrastructure
layers handle those concerns through their own workflows.

`destination` is optional metadata in the event contract. The interface does not
define its format or enforce how a consumer uses it. Treat it according to the
conventions of the workflow that reads it.

## Behavior

- The interface provides a typed shape; it does not create an event or perform
  runtime validation.
- `payload` is generic and defaults to `unknown`.
- `aggregateId` is a `UniqueId<TId>`, with `TId` serving as a compile-time kind
  marker.
- `version` and `occurredAt` are part of the event contract;
  `AggregateRoot.raise()` assigns them when it creates a new event.
- `destination` is optional.
- The interface does not define publishing, persistence, or delivery behavior.

## Related Documentation

- [Aggregate Roots](./aggregate-root)
- [Entities](./entity)
- [Unique ID](./unique-id)

---

---

<a id="source-6"></a>

# Source: `docs/data/entities/unique-id.md`

## Introduction

Entities need identifiers that remain stable while their other data changes. A
plain string carries no indication of which kind of entity it identifies, so
code can accidentally mix identifiers for different domain concepts.

`UniqueId<TKind>` wraps a UUID value and uses `TKind` as a compile-time marker.
Create identifiers through its static factory; the class has a private
constructor and freezes each instance.

## Create an Identifier

Call `UniqueId.create()` with a kind name. The factory generates a UUID v4 when
no usable identifier is provided:

```ts
import { UniqueId } from '@xeno-js/shared'

const orderId = UniqueId.create('Order')

console.log(orderId.toString())
```

The generated value is available as a string through `toString()`, or as the
underlying `Guid` through `getValue()`.

## Use Typed Identifiers

Use a different kind for each domain identifier. TypeScript then prevents
passing one kind where another is expected:

```ts
import { UniqueId } from '@xeno-js/shared'

type OrderId = UniqueId<'Order'>
type CustomerId = UniqueId<'Customer'>

function loadOrder(id: OrderId): void {
  console.log(`Loading order ${id.toString()}`)
}

const orderId: OrderId = UniqueId.create('Order')
const customerId: CustomerId = UniqueId.create('Customer')

loadOrder(orderId)
// loadOrder(customerId) // Type error: a Customer ID is not an Order ID.
```

The kind argument exists for typing; it does not change the UUID or add a
runtime prefix. Keep the kind string consistent wherever the same identifier
type is used.

## Supply an Existing Identifier

Pass an existing UUID string as the optional second argument when rebuilding an
identifier, for example when mapping a value read from storage:

```ts
import { UniqueId } from '@xeno-js/shared'

const storedOrderId = 'c0a80123-4567-489a-8abc-1234567890ab'
const orderId = UniqueId.create('Order', storedOrderId)

console.log(orderId.toString())
```

`UniqueId.create()` keeps the supplied string only when it is non-empty and a
valid GUID. If it is empty or invalid, the factory generates a new UUID instead.
Validate external input separately if silently replacing an invalid value is not
appropriate for the calling workflow.

## Compare Identifiers

Use `equals()` to compare identifiers of the same kind:

```ts
import { UniqueId } from '@xeno-js/shared'

const originalId = UniqueId.create(
  'Order',
  'c0a80123-4567-489a-8abc-1234567890ab',
)
const rehydratedId = UniqueId.create(
  'Order',
  'c0a80123-4567-489a-8abc-1234567890ab',
)
const anotherOrderId = UniqueId.create('Order')

console.log(originalId.equals(rehydratedId)) // true
console.log(originalId.equals(anotherOrderId)) // false
```

Equality compares the underlying GUID values. `equals()` also returns `false`
when the other value is not a `UniqueId` instance.

## Behavior

- `create(kind)` generates a UUID v4.
- `create(kind, id)` uses `id` only when it is non-empty and a valid GUID;
  otherwise, it generates a UUID v4.
- `toString()` returns the identifier's string representation, and `getValue()`
  returns its underlying `Guid`.
- `equals()` compares the underlying values.
- Instances are frozen after construction.
- The generic kind distinguishes identifier types at compile time; it does not
  alter their runtime representation.

---

---

<a id="source-7"></a>

# Source: `docs/data/entities/value-object.md`

## Introduction

Use a value object to represent a domain concept whose meaning comes from its
values rather than from a unique identity.

For example, two email addresses with the same value represent the same address
even if the application created them as separate instances.

## The Problem

Object identity does not express value equality:

```ts
const first = { value: 'orders@example.com' }
const second = { value: 'orders@example.com' }

console.log(first === second) // false
```

In domain code, comparing instances by identity can therefore treat equivalent
values as different.

## The Principle

A value object encapsulates the properties that define a domain value. It has no
separate identity, and its equality is based on those properties.

Keep the value object's responsibilities in the domain layer. It can enforce
domain-specific rules in its own constructor or factory, while application
services coordinate its use and infrastructure provides external
implementations.

## The Mechanism

`ValueObject<T>` is an abstract implementation of `IValueObject<T>`. It
provides:

- a protected constructor that stores a frozen copy of the supplied properties
  and freezes the instance;
- `getValue()`, which returns a shallow copy of the stored properties;
- `equals()`, which compares the serialized properties of two value objects;
- `toString()`, which serializes the stored properties.

The base class does not define domain validation. A concrete value object must
add any rules its domain requires.

## Implement a Value Object

For example, an email address can expose its value while delegating storage and
equality to `ValueObject`:

```ts
import { ValueObject } from '@xeno-js/shared'

interface EmailAddressProps {
  value: string
}

export class EmailAddress extends ValueObject<EmailAddressProps> {
  public constructor(value: string) {
    super({ value })
  }

  public get value(): string {
    return this.getValue().value
  }
}
```

This example stores the supplied string but does not validate its format. Add
domain validation in `EmailAddress` if the application requires it.

## Compare Values

Use `equals()` when the domain needs to compare value objects:

```ts
const primaryEmail = new EmailAddress('orders@example.com')
const secondaryEmail = new EmailAddress('orders@example.com')
const supportEmail = new EmailAddress('support@example.com')

console.log(primaryEmail.equals(secondaryEmail)) // true
console.log(primaryEmail.equals(supportEmail)) // false
```

`equals()` compares serialized property values; it does not compare the
instances by reference.

## Immutability and Copies

The constructor freezes a shallow copy of the properties and then freezes the
value object instance. `getValue()` returns a new, shallow copy, so updating a
top-level property on the returned object does not change the stored properties:

```ts
const email = new EmailAddress('orders@example.com')
const properties = email.getValue()

properties.value = 'changed@example.com'

console.log(email.value) // 'orders@example.com'
```

These are shallow operations. If a property contains a nested object, the base
class does not recursively clone or freeze that nested value. Prefer immutable
property values or handle nested data explicitly in the concrete type.

## String Representation

`toString()` returns a serialized representation of the stored properties:

```ts
const email = new EmailAddress('orders@example.com')

console.log(email.toString())
```

The exact serialization format is provided by `StringHelper.safeStringify`; use
`toString()` as a representation, not as a substitute for a domain-specific
display format.

## The Key Idea

Use `ValueObject` when the domain identifies a concept by its values rather than
by a persistent identity. The base class supplies immutable top-level state and
value-based equality; the concrete type supplies the domain meaning and any
required validation.

## Related Documentation

- [Entities](./unique-id)

---

---

<a id="source-8"></a>

# Source: `docs/data/node-postgresql.md`

## Introduction

If your application uses PostgreSQL, Xeno.JS lets you configure the database
during application bootstrap, resolve the scoped database context, and implement
application-specific data sources using the shared data-source contracts.

The typical flow is:

```text
PostgreSQL configuration
        ↓
XenoAppBuilder.addDb(withPostgresql())
        ↓
await app.build()
        ↓
resolve scoped DB_CONTEXT
        ↓
application Data Source implementing a shared interface
        ↓
repository / application service / handler
```

For transactional operations, add Unit of Work:

```text
application service / handler
        ↓
UNIT_OF_WORK
        ↓
runInTransaction(...)
        ↓
PostgreSQL Data Source
        ↓
PostgreSQL transaction
```

## Prerequisites

Before configuring PostgreSQL, make sure your application has:

- Node.js 20 or later.
- `@xeno-js/core`.
- `@xeno-js/postgresql`.
- `pg`.
- `drizzle-orm`.
- a reachable PostgreSQL database.
- a PostgreSQL connection string.

Install the PostgreSQL integration and its runtime dependencies:

For example:

```bash
npm install @xeno-js/core @xeno-js/postgresql pg drizzle-orm
```

## Configure PostgreSQL

Configure PostgreSQL with `withPostgresql()` from `@xeno-js/postgresql` and pass
the plugin to `XenoAppBuilder.addDb()`.

A minimal configuration is:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withPostgresql } from '@xeno-js/postgresql'

const app = new XenoAppBuilder()

app.addDb(
  withPostgresql((options) => {
    options.connectionString = process.env.DATABASE_URL!
  }),
)
```

The connection string is passed to the PostgreSQL provider configured by
Xeno.JS.

### Configure the connection from application configuration

`addDb()` receives both the database options and Xeno.JS configuration service.

This allows the connection string to come from your application's configuration
source:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withPostgresql } from '@xeno-js/postgresql'

const app = new XenoAppBuilder()

app.addDb(
  withPostgresql((options, config) => {
    options.connectionString =
      config.get('DATABASE_URL') ?? process.env.DATABASE_URL!
  }),
)
```

The configuration key is application-defined. Xeno.JS does not require a
specific environment variable name.

## Register PostgreSQL in XenoAppBuilder

The public registration API is:

```ts
app.addDb(withPostgresql(...))
```

Register the database before building the application:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withPostgresql } from '@xeno-js/postgresql'

const app = new XenoAppBuilder()

app.addDb(
  withPostgresql((options) => {
    options.connectionString = process.env.DATABASE_URL!
  }),
)

const application = await app.build()
```

After `build()` completes, the database services registered by the database
module are available through dependency injection.

Do not create a PostgreSQL `Pool` manually when using the Xeno.JS database
registration.

The application should configure the provider through `withPostgresql()` and
consume the services exposed by Xeno.JS.

## Resolve the PostgreSQL database context

Xeno.JS registers the database context under the `DB_CONTEXT` token.

`DB_CONTEXT` is scoped. Resolve it from the runtime application's active scope:

```ts
const application = await app.build()

await application.runInScope(async (services) => {
  const db = services.DB_CONTEXT
  // Use the scoped database context here.
})
```

Do not resolve this scoped service from the builder's root container.

For more information about service resolution and scopes, see:

- [Service Resolution](../dependency-injection/resolution)
- [Scoped Lifetime](../dependency-injection/lifetimes/scoped)
- [Service Container](../dependency-injection/service-container)

## Create a PostgreSQL Data Source

Implement the shared data-source interface required by your application and
inject the PostgreSQL `DbContext`. Xeno.JS does not provide a PostgreSQL base
data-source class.

## Use a PostgreSQL Data Source

A concrete data source normally receives `DB_CONTEXT` through dependency
injection.

For example:

```ts
import { AppError, Enumerable } from '@xeno-js/core'
import type { IReadDataSource, Optional, UserContext } from '@xeno-js/core'
import type { DbContext } from '@xeno-js/postgresql'
import { eq } from 'drizzle-orm'

import { users } from './schema'

type UserDto = typeof users.$inferSelect

export class UserDataSource implements IReadDataSource<UserDto> {
  constructor(private readonly _db: DbContext) {}

  async findAll(
    _ctx?: Optional<UserContext>,
    signal?: Optional<AbortSignal>,
  ): Promise<UserDto[]> {
    AppError.throwIfAborted(signal, 'UserDataSource.findAll')
    return this._db.select().from(users).execute()
  }

  async findById(
    id: string | number,
    _ctx?: Optional<UserContext>,
    signal?: Optional<AbortSignal>,
  ): Promise<Optional<UserDto>> {
    AppError.throwIfAborted(signal, 'UserDataSource.findById')
    const result = await this._db
      .select()
      .from(users)
      .where(eq(users.id, Number(id)))
      .limit(1)
      .execute()
    return Enumerable.firstOrDefault(result)
  }

  async dispose(): Promise<void> {}
}
```

The `users` table and its schema are application-defined. Xeno.JS does not
define your application's database schema.

For example, with Drizzle:

```ts
import { pgTable, serial, varchar } from 'drizzle-orm/pg-core'

export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  email: varchar('email', { length: 255 }).notNull(),
})
```

The query can then use standard Drizzle operators:

```ts
import { eq } from 'drizzle-orm'
```

The important Xeno.JS boundary is:

```text
DB_CONTEXT
    ↓
IReadDataSource<UserDto>
    ↓
UserDataSource
    ↓
application service / repository
```

Your application owns the concrete data-access operations.

## Register a Data Source in Dependency Injection

If the data source is consumed by handlers or application services, register it
in the service container.

Because `DB_CONTEXT` is scoped, the concrete data source should normally be
scoped as well.

For example:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withPostgresql } from '@xeno-js/postgresql'

const app = new XenoAppBuilder()

app.addDb(
  withPostgresql((options) => {
    options.connectionString = process.env.DATABASE_URL!
  }),
)

app.addServices((container) => {
  container.addScoped('USER_DATA_SOURCE', (scope) => {
    return new UserDataSource(scope.resolve('DB_CONTEXT'))
  })
})

await app.build()
```

Your application's registry type should include custom tokens when you want
those services to be strongly typed through the runtime application's
`getServices()`.

The important lifetime relationship is:

```text
DB_CONTEXT          → scoped
USER_DATA_SOURCE    → scoped
```

The data source receives the database context belonging to the same scope.

For general service registration rules, see
[Registration](../dependency-injection/registration) and
[Scoped Lifetime](../dependency-injection/lifetimes/scoped).

## Use Unit of Work with PostgreSQL

Use Unit of Work when multiple database operations must succeed or fail
together.

Xeno.JS registers Unit of Work under:

```ts
UNIT_OF_WORK
```

Resolve it from an active application scope:

```ts
const application = await app.build()

await application.runInScope(async (services) => {
  const unitOfWork = services.UNIT_OF_WORK
  await unitOfWork.runInTransaction(async (tx) => {
    // Database operations use this transaction context.
  })
})
```

The public API does not require manual:

```ts
begin()
commit()
rollback()
```

Instead, `runInTransaction()` defines the transaction boundary.

### Transactional application operation

A realistic operation can combine multiple data changes:

```ts
await application.runInScope(async (services) => {
  await services.UNIT_OF_WORK.runInTransaction(async (tx) => {
    await tx.insert(users).values({
      email: 'alice@example.com',
    })
  })
})
```

If the callback completes successfully, the transactional operation completes
successfully.

If the callback throws, the error propagates and the transaction does not
complete successfully.

Keep the related database operations inside the same `runInTransaction()`
callback when they must share one transaction.

## Run the PostgreSQL Outbox worker

When aggregates are passed to `runInTransaction()`, the Unit of Work writes
their uncommitted events to the Outbox in that PostgreSQL transaction. Enable
this integration by setting `outbox.enable` and providing one or more aggregate
names. The plugin creates a `<name-lowercase>_outbox_messages` table for each
name; a name is required when the Outbox is enabled.

Set `outbox.worker` to `true` to register the background worker and optionally
configure event handlers. Registering the worker does not start its polling
loop: the application must start it explicitly.

```ts
app.addDb(
  withPostgresql((options) => {
    options.connectionString = process.env.DATABASE_URL!
    options.outbox = {
      enable: true,
      name: 'Order',
      worker: true,
    }
  }),
)
```

After building the application, start the worker from the runtime services and
stop it during shutdown:

```ts
const application = await app.build()
const { worker } = application.getServices()

worker.start(5000)

// During application shutdown:
worker.stop()
```

The interval defaults to 5 seconds. Each polling pass handles at most 50 pending
messages per configured Outbox table. Failed dispatches are retried; after the
third failed attempt, the worker marks a message as `FAILED`. The worker
dispatches events in a synthetic request context.

Use the configured `outbox.handlers` factories to provide handlers for events
read from the Outbox. The plugin invokes each factory with the service container
and subscribes the returned handler to the event dispatcher when it registers
the worker.

## Complete example

A minimal application can be structured like this:

```text
src/
├── app.ts
├── data/
│   ├── schema.ts
│   └── user.data-source.ts
└── users/
    └── user.service.ts
```

### `src/data/schema.ts`

```ts
import { pgTable, serial, varchar } from 'drizzle-orm/pg-core'

export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  email: varchar('email', { length: 255 }).notNull(),
})
```

### `src/data/user.data-source.ts`

```ts
import { eq } from 'drizzle-orm'
import { AppError, Enumerable } from '@xeno-js/core'
import type { IReadDataSource, Optional, UserContext } from '@xeno-js/core'
import type { DbContext } from '@xeno-js/postgresql'

import { users } from './schema'

type UserDto = typeof users.$inferSelect

export class UserDataSource implements IReadDataSource<UserDto> {
  constructor(private readonly _db: DbContext) {}

  async findAll(
    _ctx?: Optional<UserContext>,
    signal?: Optional<AbortSignal>,
  ): Promise<UserDto[]> {
    AppError.throwIfAborted(signal, 'UserDataSource.findAll')
    return this._db.select().from(users).execute()
  }

  async findById(
    id: string | number,
    _ctx?: Optional<UserContext>,
    signal?: Optional<AbortSignal>,
  ): Promise<Optional<UserDto>> {
    AppError.throwIfAborted(signal, 'UserDataSource.findById')
    const result = await this._db
      .select()
      .from(users)
      .where(eq(users.id, Number(id)))
      .limit(1)
      .execute()
    return Enumerable.firstOrDefault(result)
  }

  async dispose(): Promise<void> {}
}
```

### `src/app.ts`

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withPostgresql } from '@xeno-js/postgresql'

import { UserDataSource } from './data/user.data-source'

const app = new XenoAppBuilder()

app.addDb(
  withPostgresql((options, config) => {
    options.connectionString =
      config.get('DATABASE_URL') ?? process.env.DATABASE_URL!
  }),
)

app.addServices((container) => {
  container.addScoped('USER_DATA_SOURCE', (scope) => {
    return new UserDataSource(scope.resolve('DB_CONTEXT'))
  })
})

await app.build()
```

At this point:

1. PostgreSQL is configured.
2. `addDb()` registers the database infrastructure.
3. `DB_CONTEXT` is available through DI.
4. `UserDataSource` receives `DB_CONTEXT`.
5. Application code can consume `USER_DATA_SOURCE`.

For production applications, keep the connection string outside source code and
provide it through your application's configuration environment.

## Troubleshooting

### PostgreSQL is not configured

If `addDb()` is not called, the database module is not registered.

Check that bootstrap contains:

```ts
app.addDb(
  withPostgresql((options) => {
    options.connectionString = process.env.DATABASE_URL!
  }),
)
```

and that the application is built afterward:

```ts
await app.build()
```

### `DB_CONTEXT` cannot be resolved

Check that:

1. `addDb()` is registered.
2. `await app.build()` has completed before resolving the service.
3. resolution occurs inside an active application scope using the registered
   token:

```ts
await application.runInScope(async (services) => {
  const db = services.DB_CONTEXT
})
```

If the problem is related to service lifetimes or resolution, see
[Service Resolution](../dependency-injection/resolution).

### The data source does not satisfy its contract

Implement the shared data-source interface used by the application, and inject
`DbContext` into the implementation. Keep provider-specific Drizzle queries
inside the data source rather than exposing the database context to callers.

### Transactional work is not behaving as one unit

Make sure all operations that must succeed or fail together execute inside the
same:

```ts
unitOfWork.runInTransaction(...)
```

For example:

```ts
await unitOfWork.runInTransaction(async (tx) => {
  // Execute all related PostgreSQL operations with tx.
})
```

Do not split operations that must be atomic across separate transaction
callbacks.

### Unit of Work cannot be resolved

`UNIT_OF_WORK` is registered by the database module.

Check that PostgreSQL was registered with `addDb()` and that the application has
been built:

```ts
app.addDb(withPostgresql(...))
await app.build()
```

For general DI problems, see [Resolution](../dependency-injection/resolution).

## Related docs

- [Data Overview](./overview)
- [SQLite](./sqlite)
- [Unit of Work](./unit-of-work)
- [Service Container](../dependency-injection/service-container)
- [Registration](../dependency-injection/registration)
- [Service Resolution](../dependency-injection/resolution)
- [Scoped Lifetime](../dependency-injection/lifetimes/scoped)
- [Dependency Graph](../dependency-injection/dependency-graph)
- [CQRS Commands](../application/cqrs/command)
- [CQRS Queries](../application/cqrs/query)

---

---

<a id="source-9"></a>

# Source: `docs/data/outbox.md`

## What is the transactional outbox pattern?

The transactional outbox pattern stores a message in the same database
transaction as the business change that produced it. If the transaction rolls
back, neither change is committed. A separate worker can dispatch the stored
message after the transaction completes.

In Xeno.JS, the PostgreSQL and SQLite plugins connect the outbox to
[Unit of Work](/docs/data/unit-of-work). When outbox support is enabled,
`runInTransaction()` saves the uncommitted events of the supplied aggregate
roots after the callback succeeds, using the same database transaction.

This solves the database side of the dual-write problem: an order update and its
outbox row succeed or fail together. Dispatch to a handler or external service
happens later and has its own failure modes.

## How the outbox and Unit of Work interact

1. A domain operation raises an event on an aggregate root.
2. The application passes the aggregate to `runInTransaction()` and saves its
   state in the callback.
3. After the callback succeeds, the provider-specific Unit of Work inserts the
   aggregate's uncommitted events into its outbox table.
4. The database commits the state changes and outbox rows together.
5. If started, the outbox worker polls pending rows and sends events to the
   event dispatcher.

If the callback or outbox insert fails, the transaction rejects and the database
changes roll back. Returning a failure value from the callback does not itself
cause rollback: throw or propagate an error instead.

The Unit of Work does not save aggregate state automatically. Use the
transaction argument or scoped data-access services for every database operation
that belongs to the transaction.

## Configure the outbox

Choose the database provider used by the application. Enable the outbox and set
`name` to the lowercase runtime class name of each aggregate. For a class named
`Order`, use `order`; the plugin creates `order_outbox_messages`.

The lowercase spelling also matters to the built-in worker: table creation
lowercases configured names, while the worker uses the configured names when
constructing its queries. Keep aggregate class names stable across builds
because the Unit of Work derives table names from `aggregate.constructor.name`,
not from the event's `aggregateType`.

### PostgreSQL

```ts
import type { XenoRegistry } from '@xeno-js/core'
import { XenoAppBuilder } from '@xeno-js/core'
import type { DbContext, DbTransaction } from '@xeno-js/postgresql'
import { withPostgresql } from '@xeno-js/postgresql'

export interface MyRegistry extends XenoRegistry<DbContext, DbTransaction> {}

const app = new XenoAppBuilder<MyRegistry>()

app.addDb(
  withPostgresql((options, config) => {
    options.connectionString = config.get('DATABASE_URL')!
    options.outbox = {
      enable: true,
      name: 'order',
      worker: true,
    }
  }),
)
```

### SQLite

```ts
import type { XenoRegistry } from '@xeno-js/core'
import { XenoAppBuilder } from '@xeno-js/core'
import type { DbContext, DbTransaction } from '@xeno-js/sqlite'
import { withSqlite } from '@xeno-js/sqlite'

export interface MyRegistry extends XenoRegistry<DbContext, DbTransaction> {}

const app = new XenoAppBuilder<MyRegistry>()

app.addDb(
  withSqlite((options, config) => {
    options.url = config.get('DATABASE_URL', 'file:./data/app.db')!
    options.outbox = {
      enable: true,
      name: 'order',
      worker: true,
    }
  }),
)
```

For multiple aggregate classes, use an array such as
`name: ['order', 'product']`. Each name creates a separate outbox table. An
enabled outbox without a name causes configuration to throw. Use fixed, trusted
names in configuration.

Set `worker: true` only when this application instance should register the
built-in worker. Outbox persistence works without it, but another process must
handle dispatch.

## Raise a domain event

The following aggregate records an `OrderConfirmed` event when its state
changes. `raise()` assigns the aggregate ID, event version and occurrence time,
calls `apply()`, and adds the event to the uncommitted list.

```ts
import { AggregateRoot } from '@xeno-js/shared'
import type { IDomainEvent } from '@xeno-js/shared'

interface OrderProps {
  status: 'draft' | 'confirmed'
}

interface OrderConfirmedPayload {
  status: 'confirmed'
}

class Order extends AggregateRoot<OrderProps, OrderConfirmedPayload, 'Order'> {
  public constructor(props: OrderProps, id?: string) {
    super(props, id)
  }

  public confirm(): void {
    if (this.getProps().status !== 'draft') {
      throw new Error('Only draft orders can be confirmed.')
    }

    this.raise({
      aggregateType: 'Order',
      eventType: 'OrderConfirmed',
      payload: { status: 'confirmed' },
    })
  }

  protected apply(event: IDomainEvent<OrderConfirmedPayload, 'Order'>): void {
    if (event.eventType === 'OrderConfirmed') {
      this.setProps({ ...this.getProps(), status: event.payload.status })
    }
  }
}
```

See [Aggregate Roots](/docs/data/entities/aggregate-root) and
[Domain Events](/docs/data/entities/events) for the domain contracts.

## Save aggregate state and events together

Build the configured application and resolve `unitOfWork` inside a service
scope. In this example, `orders` is an application-defined Drizzle table and
`order` is an `Order` instance loaded or created by the application.

```ts
const application = await app.build()

await application.runInScope(async (services) => {
  order.confirm()

  await services.unitOfWork.runInTransaction(async (tx) => {
    await tx
      .insert(orders)
      .values({
        id: order.id.toString(),
        status: order.getProps().status,
      })
      .execute()
  }, order)
})
```

Use an insert for a new order or an update for an existing one. The example uses
an insert to show the transaction boundary; the application owns its schema and
persistence rules.

Pass an array as the second argument when several aggregates have events to
store:

```ts
await unitOfWork.runInTransaction(
  async (tx) => {
    // Persist both aggregates using tx.
  },
  [firstOrder, secondOrder],
)
```

An optional `AbortSignal` can be passed as the third argument. The provider
checks whether it is already aborted before beginning the transaction; this
check does not guarantee cancellation of work already running.

### Event clearing and rollback

Each provider clears an aggregate's in-memory uncommitted events after inserting
its rows, before the database transaction has finished committing. If a later
insert or the final commit fails, database changes roll back, but events already
cleared from earlier aggregates are not restored in memory. An outbox insert
that fails before clearing leaves that aggregate's events in memory.

After any transaction failure, discard affected in-memory aggregate instances
and reload or reconstruct them before retrying. Do not manually clear events
before `runInTransaction()`.

## Dispatch stored events

With `worker: true`, the plugin registers the worker but does not start it.
Start it after building the application and stop it during shutdown:

```ts
const application = await app.build()
const { worker } = application.getServices()

worker.start() // Default polling interval: 5000 ms.

// During application shutdown:
worker.stop()
```

Each poll reads up to 50 `PENDING` messages per configured table, ordered by
creation time. A successful dispatcher result marks the row `PUBLISHED`. A
failed dispatcher result increments `retry_count`, stores `last_error`, and
returns the row to `PENDING`; the third failed attempt marks it `FAILED`.
`stop()` prevents future polls; it does not wait for a dispatch already in
progress.

The `outbox.handlers` option accepts factories returning application event
handlers. With `worker: true`, the plugin calls each factory with the service
container and subscribes the returned handler using its `eventType`. Register a
handler for each event type that needs processing. The dispatcher returns
success when no handler is subscribed, so an unmatched event can be marked
`PUBLISHED` without any handler running.

The worker publishes an object containing `eventType`, `aggregateId` and
`payload`. It does not pass the outbox row ID, event version, destination, or
stored tracing and identity fields to the handler. It also creates a new
synthetic request context. Include any stable application-level idempotency key
and required business context in the payload; do not assume the original request
context is restored.

Publishing here means dispatching to the Xeno.JS event dispatcher. Sending to a
message broker, email service, or external API requires an application handler
or a separate publisher.

## Delivery limits and operations

The outbox makes the database write atomic; it does not guarantee exactly-once
delivery. A handler can complete before the worker records `PUBLISHED`, so a
later attempt can repeat its side effect. Make handlers idempotent using an
application-defined key carried in the payload and a durable deduplication
strategy appropriate to the destination.

The built-in workers do not provide an unconditional at-least-once guarantee:

- PostgreSQL selects pending rows with `FOR UPDATE SKIP LOCKED`, but releases
  the transaction before dispatch. Multiple workers can select the same row
  after that transaction ends.
- SQLite marks selected rows `PROCESSING` before dispatch. An interruption can
  leave rows in that state; the built-in worker only selects `PENDING` rows and
  has no automatic recovery for stranded `PROCESSING` rows.
- Dispatcher failures stop retrying after three attempts. Unexpected worker
  errors are logged and do not necessarily follow the same retry path.

Monitor pending and failed rows, retry counts, worker logs and, for SQLite,
stranded `PROCESSING` rows. Define an operational recovery procedure before
manually changing a row's status: first determine whether its handler already
performed an external side effect. Neither provider automatically cleans up
published rows, so plan retention according to application requirements.

## Common problems

### No outbox rows appear

Enable the provider's outbox, configure a name matching the aggregate's
lowercase runtime class name, and pass the aggregate as the second argument to
`runInTransaction()`. The aggregate must have uncommitted events; raising an
event alone does not persist it. The core Unit of Work used without the
provider's outbox integration does not insert outbox rows.

### Business data changed but no event was stored

Check that the aggregate was passed to the same `runInTransaction()` call as the
database write. A write performed before the callback is outside that
transaction. Verify that errors are propagated instead of caught and ignored.

### Messages remain pending

Check that the worker is registered with `worker: true`, that `start()` was
called, and that the configured name is lowercase and matches the table. Inspect
worker logs and `last_error`. A row in `FAILED` needs an application-specific
recovery decision; a SQLite row in `PROCESSING` is not retried automatically.

### A message is published without the expected side effect

Confirm that the handler factory is configured, its `eventType` matches the
stored event type exactly, and it is subscribed when the worker is registered.
An event with no subscribed handlers is treated as a successful dispatch.

### Duplicate effects occur

Do not treat `PUBLISHED` as an exactly-once guarantee. A retry or concurrent
worker may dispatch the same event again. Make the handler's side effect
idempotent and include a stable deduplication key in the payload.

## Related documentation

- [Unit of Work](/docs/data/unit-of-work)
- [Node PostgreSQL](/docs/data/node-postgresql)
- [SQLite](/docs/data/sqllite)
- [Aggregate Roots](/docs/data/entities/aggregate-root)
- [Domain Events](/docs/data/entities/events)

---

---

<a id="source-10"></a>

# Source: `docs/data/overview.md`

## Introduction

Xeno.JS provides the application-facing infrastructure you need to access
relational data and execute database operations without making the database
provider the center of your application architecture.

The workflow is:

```text
Choose a database
      ↓
Configure it with XenoAppBuilder and a provider plugin
      ↓
Build the application
      ↓
Resolve the database services
      ↓
Use a provider-specific Data Source
      ↓
Use Unit of Work when operations must run in a transaction
```

Xeno.JS currently supports:

- PostgreSQL through the Node PostgreSQL integration.
- SQLite through the libSQL client.

For provider-specific configuration and usage, see:

- [Node PostgreSQL](./node-postgresql)
- [SQLite](./sqlite)
- [Unit of Work](./unit-of-work)

## Before you start

You need:

- a Xeno.JS application;
- the database provider dependencies required by your selected database;
- a database connection URL or provider-specific connection options;
- an `XenoAppBuilder` instance.

The database is configured during application bootstrap. Application code should
consume the registered database services rather than creating database clients
directly.

## Configure the database

Use `XenoAppBuilder.addDb()` with a plugin from the selected database
integration.

For PostgreSQL, pass `withPostgresql()` and configure its connection options:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withPostgresql } from '@xeno-js/postgresql'

const app = new XenoAppBuilder()

app.addDb(
  withPostgresql((options, config) => {
    options.connectionString =
      config.get('DATABASE_URL') ??
      'postgres://user:password@localhost:5432/myapp'
  }),
)

await app.build()
```

For SQLite/libSQL, pass `withSqlite()` and configure its URL:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withSqlite } from '@xeno-js/sqlite'

const app = new XenoAppBuilder()

app.addDb(
  withSqlite((options) => {
    options.url = 'file:./data/app.db'
  }),
)

await app.build()
```

The option names are provider-specific:

| Provider      | Plugin           | Connection option  |
| ------------- | ---------------- | ------------------ |
| PostgreSQL    | `withPostgresql` | `connectionString` |
| SQLite/libSQL | `withSqlite`     | `url`              |

## Register the database in XenoAppBuilder

Database registration belongs in the application bootstrap.

A typical application setup looks like:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withPostgresql } from '@xeno-js/postgresql'

const app = new XenoAppBuilder()

app.addDb(
  withPostgresql((options, config) => {
    options.connectionString =
      config.get('DATABASE_URL') ??
      'postgres://user:password@localhost:5432/myapp'
  }),
)

const application = await app.build()
```

`addDb()` invokes the plugin to queue the provider-specific database module.
`build()` initializes the configured modules and returns the runtime
application, not the service container.

After the build completes, the database services are available through
dependency injection.

## Resolve the database

The database context is registered under the `DB_CONTEXT` token.

`DB_CONTEXT` is scoped, so resolve it from an active service scope. Use
`runInScope()` on the runtime application:

```ts
await application.runInScope(async (services) => {
  const db = services.DB_CONTEXT
  // Use the scoped database context here.
})
```

The exact type of `DB_CONTEXT` depends on the application's Xeno registry.

For most application code, however, you should not pass the database context
throughout the application manually. Instead, create a provider-specific Data
Source and inject it where data access is required.

## Create a Data Source

### PostgreSQL

```ts
import type { IReadDataSource } from '@xeno-js/core'

export class UserDataSource implements IReadDataSource {
  async findUserById(id: string) {
    return this._db.query.users.findFirst({
      where: (users, { eq }) => eq(users.id, id),
    })
  }
}
```

### SQLite

```ts
import type { IReadDataSource } from '@xeno-js/core'
import type { DbContext } from '@xeno-js/sqlite'
import type { DbSchema } from './schema'

export class UserDataSource implements IReadDataSource {
  constructor(private readonly _db: DbContext<DbSchema>)
  async findUserById(id: string) {
    return this._db.query.users.findFirst({
      where: (users, { eq }) => eq(users.id, id),
    })
  }
}
```

The provider-specific pages contain the complete configuration and Data Source
examples:

- [Node PostgreSQL](./node-postgresql)
- [SQLite](./sqlite)

## Register your Data Source

Your application Data Source is a normal dependency-injection service. Declare
its token and type in the application's Xeno registry before registering it.

For example:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withPostgresql } from '@xeno-js/postgresql'
import type { DbContext, DbTransaction } from '@xeno-js/postgresql'
import type { XenoRegistry } from '@xeno-js/shared'
import type { DbSchema } from './schema'

interface AppRegistry extends XenoRegistry<DbContext<DbSchema>, DbTransaction> {
  USER_DATA_SOURCE: UserDataSource
}

const app = new XenoAppBuilder<AppRegistry>()

app.addDb(
  withPostgresql<AppRegistry>((options, config) => {
    options.connectionString =
      config.get('DATABASE_URL') ??
      'postgres://user:password@localhost:5432/myapp'
  }),
)

app.addServices((services) => {
  services.addScoped('USER_DATA_SOURCE', (scope) => {
    return new UserDataSource(scope.resolve('DB_CONTEXT'))
  })
})

const application = await app.build()
```

Because this Data Source is scoped, resolve it from scoped services within
`runInScope()` or inject it into another service with a compatible lifetime.

For more information about registration and lifetimes, see:

- [Registration](../dependency-injection/registration)
- [Resolution](../dependency-injection/resolution)
- [Scoped Services](../dependency-injection/lifetimes/scoped)

## Use Unit of Work for transactions

When an operation requires multiple database changes to succeed as one unit, use
the registered Unit of Work.

`UNIT_OF_WORK` is scoped. Resolve it from the active scope and perform the work
within that scope:

```ts
await application.runInScope(async (services) => {
  const unitOfWork = services.UNIT_OF_WORK
  const userDataSource = services.USER_DATA_SOURCE

  await unitOfWork.runInTransaction(async () => {
    await userDataSource.createUser(user)
    await userDataSource.createProfile(profile)
  })
})
```

The transaction API is callback-based.

You do not manually call:

```ts
begin()
commit()
rollback()
```

Instead, the work performed inside `runInTransaction()` is executed as one
transaction.

For the complete transaction workflow, see [Unit of Work](./unit-of-work).

## Typical application flow

A typical application that uses Xeno.JS Data infrastructure can be organized
like this:

```text
HTTP / transport
       ↓
Command or Query
       ↓
Handler
       ↓
Application service / Repository
       ↓
Data Source
       ↓
Database
```

For transactional operations:

```text
Command Handler
       ↓
Unit of Work
       ↓
Data Source(s)
       ↓
Database transaction
```

The database provider remains infrastructure. Your application services,
handlers, and repositories consume the data access services through explicit
dependencies.

## Complete example

The following example shows the basic composition of a PostgreSQL application.

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withPostgresql } from '@xeno-js/postgresql'
import type { IReadDataSource } from '@xeno-js/core'
import type { DbContext, DbTransaction } from '@xeno-js/postgresql'
import type { XenoRegistry } from '@xeno-js/shared'
import type { DbSchema } from './schema'

class UserDataSource implements IReadDataSource {
  constructor(private readonly _db: DbContext<DbSchema>)
  async findUserById(id: string) {
    return this._db.query.users.findFirst({
      where: (users, { eq }) => eq(users.id, id),
    })
  }
}

interface AppRegistry extends XenoRegistry<DbContext<DbSchema>, DbTransaction> {
  USER_DATA_SOURCE: UserDataSource
}

const app = new XenoAppBuilder<AppRegistry>()

app.addDb(withPostgresql<AppRegistry>((options, config) => {
  options.connectionString =
    config.get('DATABASE_URL') ??
    'postgres://user:password@localhost:5432/myapp'
}))

app.addServices((services) => {
  services.addScoped('USER_DATA_SOURCE', (scope) => {
    return new UserDataSource(scope.resolve('DB_CONTEXT'))
  })
})

const application = await app.build()

await application.runInScope(async (services) => {
  const dataSource = services.USER_DATA_SOURCE
  const user = await dataSource.findUserById('user-123')
})
```

This example demonstrates the complete Data workflow:

1. configure the database;
2. register the database with `addDb()`;
3. register an application Data Source;
4. build the application;
5. resolve the scoped Data Source;
6. perform data access within its scope.

The provider-specific pages show the complete implementation for each database.

## Choose the right Data page

### I need PostgreSQL

Go to [Node PostgreSQL](./node-postgresql).

Use this page when you need to:

- configure a PostgreSQL connection;
- register PostgreSQL in `XenoAppBuilder`;
- create a PostgreSQL Data Source;
- access the PostgreSQL database from application infrastructure.

### I need SQLite

Go to [SQLite](./sqlite).

Use this page when you need to:

- configure SQLite;
- register SQLite in `XenoAppBuilder`;
- create a SQLite Data Source;
- access the SQLite database from application infrastructure.

### I need a transaction

Go to [Unit of Work](./unit-of-work).

Use this page when multiple database operations must execute as one
transactional operation.

## Troubleshooting

### `DB_CONTEXT` cannot be resolved

Make sure the provider plugin has been registered:

```ts
app.addDb(
  withPostgresql((options) => {
    options.connectionString = 'postgres://user:password@localhost:5432/myapp'
  }),
)
```

Then build the application and resolve the scoped service inside `runInScope()`:

```ts
const application = await app.build()

await application.runInScope(async (services) => {
  const db = services.DB_CONTEXT
})
```

### The application fails while bootstrapping the database

Check:

1. the provider-specific connection option (`connectionString` for PostgreSQL or
   `url` for SQLite/libSQL);
2. the selected provider;
3. the provider dependency installation;
4. whether the configured database is reachable.

The error may be wrapped by `XenoAppBuilder` as a bootstrap failure for the
database module.

### The wrong database provider is being used

Choose the provider by passing its plugin to `addDb()`.

PostgreSQL:

```ts
app.addDb(
  withPostgresql((options) => {
    options.connectionString = 'postgres://user:password@localhost:5432/myapp'
  }),
)
```

SQLite/libSQL:

```ts
app.addDb(
  withSqlite((options) => {
    options.url = 'file:./data/app.db'
  }),
)
```

### Unit of Work is not available

Make sure a database plugin has been configured and the application has been
built. `UNIT_OF_WORK` is scoped, so resolve it inside an active scope:

```ts
const application = await app.build()

await application.runInScope(async (services) => {
  const unitOfWork = services.UNIT_OF_WORK
})
```

## Data checklist

Before implementing database access, verify:

- [ ] I selected PostgreSQL or SQLite.
- [ ] The required provider dependency is installed.
- [ ] I configured the selected provider's connection option (`connectionString`
      or `url`).
- [ ] I registered the provider plugin with `XenoAppBuilder.addDb()`.
- [ ] I called `build()` before resolving database services.
- [ ] I use `DB_CONTEXT` through dependency injection.
- [ ] My application Data Source extends the correct provider-specific base
      class.
- [ ] I register my Data Source with the appropriate lifetime.
- [ ] I use `UNIT_OF_WORK` for transactional operations.
- [ ] I keep database infrastructure outside the domain layer.

## Related docs

### Database providers

- [Node PostgreSQL](./node-postgresql)
- [SQLite](./sqlite)
- [Unit of Work](./unit-of-work)

### Dependency Injection

- [Service Container](../dependency-injection/service-container)
- [Registration](../dependency-injection/registration)
- [Resolution](../dependency-injection/resolution)
- [Scoped Services](../dependency-injection/lifetimes/scoped)
- [Dependency Graph](../dependency-injection/dependency-graph)

### Application

- [CQRS Overview](../application/cqrs/overview)
- [Commands](../application/cqrs/command)
- [Queries](../application/cqrs/query)
- [Handlers](../application/cqrs/handler)

---

---

<a id="source-11"></a>

# Source: `docs/data/repositories/read-dao.md`

## Introduction

Use `ReadDao<T, TDto>` when you need a **read-only data access class** that
converts data-source DTOs into application entities.

The usual implementation path is:

```text
IReadDataSource<TDto>
        │
        ▼
Concrete ReadDao<T, TDto>
        │
        ├── findById()
        └── findAll()
        │
        ▼
IMapper<T, TDto>
        │
        ▼
Domain entity T
```

You normally do not instantiate `ReadDao` directly. It is an abstract base class
that you extend with your application's concrete DAO.

The base class is exported publicly by `@xeno-js/core`:

```ts
import { ReadDao } from '@xeno-js/core'
```

## Prerequisites

A concrete `ReadDao` needs four application-level pieces:

- an entity type `T`;
- a DTO type `TDto`;
- an implementation of `IReadDataSource<TDto>`;
- an implementation of `IMapper<T, TDto>`.

The relevant contracts are public Xeno.JS APIs:

```ts
interface IReadDataSource<TDto> {
  findAll(
    ctx?: Optional<UserContext>,
    signal?: Optional<AbortSignal>,
  ): Promise<TDto[]>

  findById(
    id: string | number,
    ctx?: Optional<UserContext>,
    signal?: Optional<AbortSignal>,
  ): Promise<Optional<TDto>>
}
```

`IReadDataSource<TDto>` also extends `IDisposable`.

The mapper contract is:

```ts
interface IMapper<T, TDto> {
  toDto(entity: T): TDto
  toEntity(dto: TDto): T
  toPartialDto(entity: Partial<T>): Partial<TDto>
}
```

For `ReadDao`, the relevant mapper operation is `toEntity()`.

## 1. Define the entity and DTO

The following types are application-specific examples:

```ts
export interface User {
  id: string
  name: string
}

export interface UserDto {
  id: string
  fullName: string
}
```

The DTO represents the shape returned by the data source, while `User`
represents the entity used by the application.

For example:

```text
Data source
    │
    │ UserDto
    ▼
ReadDao
    │
    │ mapper.toEntity()
    ▼
User
```

## 2. Implement the mapper

Implement `IMapper<User, UserDto>`:

```ts
import type { IMapper } from '@xeno-js/core'

import type { User, UserDto } from './types'

export class UserMapper implements IMapper<User, UserDto> {
  toDto(entity: User): UserDto {
    return {
      id: entity.id,
      fullName: entity.name,
    }
  }

  toEntity(dto: UserDto): User {
    return {
      id: dto.id,
      name: dto.fullName,
    }
  }

  toPartialDto(entity: Partial<User>): Partial<UserDto> {
    return {
      ...(entity.id !== undefined ? { id: entity.id } : {}),
      ...(entity.name !== undefined ? { fullName: entity.name } : {}),
    }
  }
}
```

`ReadDao` only needs `toEntity()` for its read operations, but the public
`IMapper` contract also requires `toDto()` and `toPartialDto()`.

## 3. Provide the read data source

The data source is responsible for obtaining DTOs.

For example, an application-specific data source can implement:

```ts
import type { IReadDataSource, Optional, UserContext } from '@xeno-js/core'

import type { UserDto } from './types'

export class UserDataSource implements IReadDataSource<UserDto> {
  async findById(
    id: string | number,
    _ctx?: Optional<UserContext>,
    _signal?: Optional<AbortSignal>,
  ): Promise<Optional<UserDto>> {
    throw new Error(
      'Implement the application-specific query for the requested user.',
    )
  }

  async findAll(
    _ctx?: Optional<UserContext>,
    _signal?: Optional<AbortSignal>,
  ): Promise<UserDto[]> {
    throw new Error('Implement the application-specific user query.')
  }

  async dispose(): Promise<void> {
    // Release resources here if this data source owns any.
  }
}
```

Replace the explicit placeholder errors with the application's database/query
implementation. They avoid returning an empty result that could look like a
successful query.

The database/query implementation belongs to the `IReadDataSource`
implementation, not to `ReadDao`.

`ReadDao` coordinates the data source and mapper.

## 4. Implement the concrete ReadDao

Extend the Xeno.JS base class:

```ts
import { ReadDao } from '@xeno-js/core'
import type { IMapper, IReadDataSource } from '@xeno-js/core'

import type { User, UserDto } from './types'

export class UserReadDao extends ReadDao<User, UserDto> {
  constructor(
    dataSource: IReadDataSource<UserDto>,
    mapper: IMapper<User, UserDto>,
  ) {
    super(dataSource, mapper)
  }
}
```

There is normally no need to override `findById()` or `findAll()`.

The base implementation already:

1. checks the `AbortSignal`;
2. calls the data source;
3. maps DTOs to entities;
4. wraps the result in `Result`.

The corresponding source implementation is equivalent to:

```ts
const result = await this._dataSource.findById(id, ctx, signal)

if (Guards.isNullOrEmpty(result)) {
  return Result.ok()
}

const entity = this._mapper.toEntity(result)

return Result.ok(entity)
```

For `findAll()`:

```ts
const results = await this._dataSource.findAll(ctx, signal)

const entities = results.map((result) => this._mapper.toEntity(result))

return Result.ok(entities)
```

## 5. Understand the ReadDao contract

The public `IReadDao<T>` interface exposes two operations.

### Find by ID

```ts
findById(
  id: string | number,
  ctx: UserContext,
  signal: Optional<AbortSignal>,
): Promise<ResultType<Optional<T>>>
```

The identifier can be either a string or a number.

The `UserContext` is passed through to the data source.

The `signal` parameter accepts `Optional<AbortSignal>`; pass `undefined` when
cancellation is not needed.

If the data source returns `undefined` or `null`, `ReadDao` returns a successful
`Result` whose value is `undefined`.

It does not call the mapper when there is no DTO.

### Find all

```ts
findAll(
  ctx: UserContext,
  signal: Optional<AbortSignal>,
): Promise<ResultType<T[]>>
```

The data source returns DTOs:

```ts
UserDto[]
```

and `ReadDao` maps every DTO:

```ts
UserDto[] → User[]
```

An empty data-source result produces a successful result containing an empty
array.

## 6. Register the concrete ReadDao

After implementing the class, register it in the application's DI container.

The important point is that **you register `UserReadDao`, not the abstract
`ReadDao`**.

A typical registration is:

```ts
app.addServices((services) => {
  services.addScoped('USER_READ_DAO', (container) => {
    return new UserReadDao(
      container.resolve('USER_DATA_SOURCE'),
      container.resolve('USER_MAPPER'),
    )
  })
})
```

The exact token names in this example are application-defined.

For this registration to work, the application must also register the
dependencies:

```text
USER_DATA_SOURCE
        │
        ▼
USER_READ_DAO
        ▲
        │
USER_MAPPER
```

For example:

```ts
app.addServices((services) => {
  services.addScoped('USER_DATA_SOURCE', () => {
    return new UserDataSource()
  })

  services.addScoped('USER_MAPPER', () => {
    return new UserMapper()
  })

  services.addScoped('USER_READ_DAO', (container) => {
    return new UserReadDao(
      container.resolve('USER_DATA_SOURCE'),
      container.resolve('USER_MAPPER'),
    )
  })
})
```

The tokens shown here (`USER_DATA_SOURCE`, `USER_MAPPER`, and `USER_READ_DAO`)
are application-defined registry entries. They are not predefined Xeno.JS
tokens.

## 7. Add the tokens to the application registry

When using a typed `XenoAppBuilder`, add the application-specific tokens to the
`XenoRegistry`.

For example:

```ts
import type { XenoRegistry } from '@xeno-js/core'
import type { IReadDataSource, IMapper } from '@xeno-js/core'

interface ApplicationRegistry extends XenoRegistry {
  USER_DATA_SOURCE: IReadDataSource<UserDto>
  USER_MAPPER: IMapper<User, UserDto>
  USER_READ_DAO: UserReadDao
}
```

The important part for the DAO is that the registry associates each token with
its resolved type:

```ts
{
  USER_DATA_SOURCE: IReadDataSource<UserDto>
  USER_MAPPER: IMapper<User, UserDto>
  USER_READ_DAO: UserReadDao
}
```

This gives `XenoAppBuilder` and the service container the corresponding
TypeScript types.

## 8. Build the application

Register the services before building the application:

```ts
const app = new XenoAppBuilder<ApplicationRegistry>()

app.addServices((services) => {
  services.addScoped('USER_DATA_SOURCE', () => {
    return new UserDataSource()
  })

  services.addScoped('USER_MAPPER', () => {
    return new UserMapper()
  })

  services.addScoped('USER_READ_DAO', (container) => {
    return new UserReadDao(
      container.resolve('USER_DATA_SOURCE'),
      container.resolve('USER_MAPPER'),
    )
  })
})

const application = await app.build()
```

`build()` initializes the configured application modules and returns the runtime
`XenoApplication`, not the service container.

## 9. Resolve the ReadDao

`ReadDao` is commonly registered as a scoped service.

A scoped service must be resolved through a scope.

```ts
await application.runInScope(async (services) => {
  const readDao = services.USER_READ_DAO
  // Use the DAO within this scope.
})
```

The resolved value has the registry type:

```ts
UserReadDao
```

Use it while the scope is active:

```ts
const userContext: UserContext = { userId: 'user-123', tenantId: 'tenant-456' }

const result = await application.runInScope(
  async ({ USER_READ_DAO: readDao }) => {
    return readDao.findById('user-123', userContext, undefined)
  },
)
```

`runInScope()` manages and disposes the scope when the callback completes. The
DAO must not be used after the callback returns.

### Resolving from `XenoAppBuilder`

`XenoAppBuilder.resolve()` resolves services from the root container, so it
cannot retrieve a scoped `ReadDao`.

Resolve the DAO through the runtime application's active scope instead:

```ts
const application = await app.build()
await application.runInScope(async (services) => {
  const readDao = services.USER_READ_DAO
  // Use the DAO within this scope.
})
```

## 10. Use ReadDao from an application service

A typical application service receives the DAO through dependency injection
rather than constructing it itself.

Conceptually:

```ts
export class GetUserService {
  constructor(private readonly users: UserReadDao) {}

  async execute(id: string, ctx: UserContext, signal: Optional<AbortSignal>) {
    return this.users.findById(id, ctx, signal)
  }
}
```

The service does not need to know:

- which database is used;
- which SQL/query implementation is used;
- how the DTO is represented;
- how the DTO is mapped to the entity.

Those responsibilities remain behind the data source and mapper.

## 11. Find an entity by ID

Call:

```ts
const result = await readDao.findById('user-123', ctx, signal)
```

The flow is:

```text
UserReadDao.findById()
        │
        ▼
IReadDataSource.findById()
        │
        ▼
UserDto | undefined
        │
        ▼
IMapper.toEntity()
        │
        ▼
ResultType<Optional<User>>
```

If no record is found, the base implementation returns an `ok` result with an
undefined value.

If a DTO is returned, it is converted through:

```ts
mapper.toEntity(dto)
```

## 12. Find all entities

Use:

```ts
const result = await readDao.findAll(ctx, signal)
```

The flow is:

```text
UserReadDao.findAll()
        │
        ▼
IReadDataSource.findAll()
        │
        ▼
UserDto[]
        │
        ▼
IMapper.toEntity() for each DTO
        │
        ▼
User[]
```

The mapper is called once for every returned DTO.

An empty result remains an empty entity array.

## 13. Cancellation

`ReadDao` checks the supplied `AbortSignal` before invoking the data source.

For example:

```ts
const controller = new AbortController()

controller.abort()

await readDao.findById('user-123', ctx, controller.signal)
```

An already-aborted signal causes the operation to throw before the data source
is called.

The Xeno.JS tests explicitly verify this behavior for both:

```ts
findById()
```

and:

```ts
findAll()
```

Pass the signal through to the DAO when the calling operation supports
cancellation:

```ts
await readDao.findAll(ctx, signal)
```

## ReadDao vs Repository

`ReadDao` and `Repository` both provide read operations, but they represent
different base abstractions.

### ReadDao

`ReadDao<T, TDto>` is read-only:

```ts
findById()
findAll()
```

It depends on:

```ts
IReadDataSource<TDto>
IMapper<T, TDto>
```

Use it when the abstraction you need is specifically for reading data.

### Repository

`Repository<T, TDto>` implements the complete `IRepository<T>` contract:

```ts
findById()
findAll()
save()
update()
delete()
```

Its data-source dependency is:

```ts
IWriteDataSource<TDto>
```

Use `Repository` when the abstraction needs both reads and writes.

The distinction is therefore:

```text
ReadDao
  └── IReadDataSource
       └── findById
       └── findAll

Repository
  └── IWriteDataSource
       ├── findById
       ├── findAll
       ├── insert
       ├── update
       └── delete
```

`IWriteDataSource` itself includes the read methods, so a write-capable
repository can also perform reads.

## ReadDao vs DataSource

The data source and DAO have different responsibilities.

### DataSource

`IReadDataSource<TDto>` works with DTO/data-source representations:

```ts
findById(
  id: string | number,
  ctx?: Optional<UserContext>,
  signal?: Optional<AbortSignal>,
): Promise<Optional<TDto>>

findAll(
  ctx?: Optional<UserContext>,
  signal?: Optional<AbortSignal>,
): Promise<TDto[]>
```

It is responsible for obtaining data.

### Base ReadDao

`ReadDao<T, TDto>` exposes entities:

```ts
findById(...): Promise<ResultType<Optional<T>>>

findAll(...): Promise<ResultType<T[]>>
```

It delegates retrieval to the data source and maps the DTOs:

```text
DataSource
    │
    │ DTO
    ▼
ReadDao
    │
    │ Mapper
    ▼
Entity
```

## Complete example

The following shows the complete application-side structure.

```text
src/
├── users/
│   ├── user.ts
│   ├── user.dto.ts
│   ├── user.mapper.ts
│   ├── user.data-source.ts
│   └── user.read-dao.ts
└── bootstrap.ts
```

### `user.ts`

```ts
export interface User {
  id: string
  name: string
}
```

### `user.dto.ts`

```ts
export interface UserDto {
  id: string
  fullName: string
}
```

### `user.mapper.ts`

```ts
import type { IMapper } from '@xeno-js/core'

import type { User } from './user'
import type { UserDto } from './user.dto'

export class UserMapper implements IMapper<User, UserDto> {
  toDto(entity: User): UserDto {
    return {
      id: entity.id,
      fullName: entity.name,
    }
  }

  toEntity(dto: UserDto): User {
    return {
      id: dto.id,
      name: dto.fullName,
    }
  }

  toPartialDto(entity: Partial<User>): Partial<UserDto> {
    return {
      ...(entity.id !== undefined ? { id: entity.id } : {}),
      ...(entity.name !== undefined ? { fullName: entity.name } : {}),
    }
  }
}
```

### `user.data-source.ts`

```ts
import type { IReadDataSource, Optional, UserContext } from '@xeno-js/core'

import type { UserDto } from './user.dto'

export class UserDataSource implements IReadDataSource<UserDto> {
  async findById(
    id: string | number,
    _ctx?: Optional<UserContext>,
    _signal?: Optional<AbortSignal>,
  ): Promise<Optional<UserDto>> {
    throw new Error(
      'Implement the application-specific query for the requested user.',
    )
  }

  async findAll(
    _ctx?: Optional<UserContext>,
    _signal?: Optional<AbortSignal>,
  ): Promise<UserDto[]> {
    throw new Error('Implement the application-specific user query.')
  }

  async dispose(): Promise<void> {
    // Release resources here if this data source owns any.
  }
}
```

### `user.read-dao.ts`

```ts
import { ReadDao } from '@xeno-js/core'
import type {
  IMapper,
  IReadDataSource,
  Optional,
  UserContext,
} from '@xeno-js/core'

import type { User } from './user'
import type { UserDto } from './user.dto'

export class UserReadDao extends ReadDao<User, UserDto> {
  constructor(
    dataSource: IReadDataSource<UserDto>,
    mapper: IMapper<User, UserDto>,
  ) {
    super(dataSource, mapper)
  }
}
```

### `bootstrap.ts`

The registration follows the normal Xeno.JS DI pattern:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import type { XenoRegistry } from '@xeno-js/core'
import type { IMapper, IReadDataSource } from '@xeno-js/core'

import type { User } from './users/user'
import type { UserDto } from './users/user.dto'
import { UserMapper } from './users/user.mapper'
import { UserDataSource } from './users/user.data-source'
import { UserReadDao } from './users/user.read-dao'

interface ApplicationRegistry extends XenoRegistry {
  USER_DATA_SOURCE: IReadDataSource<UserDto>
  USER_MAPPER: IMapper<User, UserDto>
  USER_READ_DAO: UserReadDao
}

const app = new XenoAppBuilder<ApplicationRegistry>()

app.addServices((services) => {
  services.addScoped('USER_DATA_SOURCE', () => {
    return new UserDataSource()
  })

  services.addScoped('USER_MAPPER', () => {
    return new UserMapper()
  })

  services.addScoped('USER_READ_DAO', (container) => {
    return new UserReadDao(
      container.resolve('USER_DATA_SOURCE'),
      container.resolve('USER_MAPPER'),
    )
  })
})

const application = await app.build()

const userContext: UserContext = { userId: 'user-123', tenantId: 'tenant-456' }
const signal: Optional<AbortSignal> = undefined

const result = await application.runInScope(async (services) => {
  return services.USER_READ_DAO.findAll(userContext, signal)
})
```

Replace the placeholder query errors in `UserDataSource` with the application's
database access logic. `runInScope()` manages the scoped services' lifetime.

## What ReadDao does for you

Once the concrete DAO is implemented and registered, the base class handles the
common read workflow:

```text
findById(id, ctx, signal)
        │
        ├── cancellation check
        │
        ├── dataSource.findById(...)
        │
        ├── null/undefined → Result.ok()
        │
        └── mapper.toEntity(dto)
                 │
                 ▼
             Result.ok(entity)
```

and:

```text
findAll(ctx, signal)
        │
        ├── cancellation check
        │
        ├── dataSource.findAll(...)
        │
        └── mapper.toEntity() for every DTO
                 │
                 ▼
             Result.ok(entities)
```

You therefore only need to provide the application-specific pieces:

```text
Entity
   +
DTO
   +
IReadDataSource implementation
   +
IMapper implementation
   ↓
Concrete ReadDao
```

## Troubleshooting

### `ReadDao` cannot be instantiated

`ReadDao` is abstract.

Do this:

```ts
class UserReadDao extends ReadDao<User, UserDto> {
  // ...
}
```

not:

```ts
const dao = new ReadDao<User, UserDto>(...)
```

### The constructor dependencies do not match

The base constructor requires:

```ts
constructor(
  dataSource: IReadDataSource<TDto>,
  mapper: IMapper<T, TDto>,
)
```

Make sure the DTO type is the same in both dependencies:

```ts
IReadDataSource<UserDto>
IMapper<User, UserDto>
```

### `resolve()` reports that the registration does not exist

Make sure the DAO was registered before:

```ts
await app.build()
```

and that the token used for registration is exactly the token used for
resolution:

```ts
services.addScoped('USER_READ_DAO', ...)
```

then:

```ts
await application.runInScope(async (services) => {
  const readDao = services.USER_READ_DAO
})
```

### A scoped DAO cannot be resolved from the root container

This is expected for a scoped registration.

Root resolution cannot retrieve scoped services. Use
`XenoApplication.runInScope()`:

```ts
await application.runInScope(async (services) => {
  const dao = services.USER_READ_DAO
})
```

### `findById()` returns no entity

The base `ReadDao` treats `null` and `undefined` from the data source as "not
found" and returns a successful result with an undefined value.

Check the data source:

```ts
await dataSource.findById(id, ctx, signal)
```

and verify that it returns the expected DTO when a record exists.

### The entity has the wrong shape

`ReadDao` does not transform DTO fields itself.

The transformation is performed by:

```ts
mapper.toEntity(dto)
```

Check the concrete `IMapper<T, TDto>` implementation.

### A cancelled operation still reaches the data source

Check that the same `AbortSignal` is passed to the DAO:

```ts
await readDao.findById(id, ctx, signal)
```

`ReadDao` checks an already-aborted signal before invoking its data source.

## Checklist

To implement a read-only DAO:

- [ ] Define the entity type.
- [ ] Define the data-source DTO type.
- [ ] Implement `IReadDataSource<TDto>`.
- [ ] Implement `IMapper<T, TDto>`.
- [ ] Create a concrete class extending `ReadDao<T, TDto>`.
- [ ] Pass the data source and mapper to `super(...)`.
- [ ] Add the concrete DAO to the application's registry.
- [ ] Register the data source.
- [ ] Register the mapper.
- [ ] Register the DAO.
- [ ] Build the `XenoAppBuilder`.
- [ ] Resolve scoped DAO work with `XenoApplication.runInScope()`.
- [ ] Call `findById()` or `findAll()`.
- [ ] Pass the `UserContext`.
- [ ] Pass an `AbortSignal` when cancellation is needed.

---

## Related Docs

- [Data Overview](../overview)
- [Node PostgreSQL](../node-postgresql)
- [SQLite](../sqllite)
- [Unit of Work](../unit-of-work)
- [Service Registration](../../dependency-injection/registration)
- [Service Resolution](../../dependency-injection/resolution)
- [Dependency Graph](../../dependency-injection/dependency-graph)
- [Repository](./repository)

---

---

<a id="source-12"></a>

# Source: `docs/data/repositories/repository.md`

## Introduction

A repository in Xeno.JS is the application-facing abstraction used to work with
persisted entities.

You normally create a repository by extending the abstract `Repository<T, TDto>`
class:

```ts
import { Repository } from '@xeno-js/core'
import type { IMapper, IWriteDataSource } from '@xeno-js/core'

import type { User } from './user'
import type { UserDto } from './user.dto'

export class UserRepository extends Repository<User, UserDto> {
  constructor(
    dataSource: IWriteDataSource<UserDto>,
    mapper: IMapper<User, UserDto>,
  ) {
    super(dataSource, mapper)
  }
}
```

The base `Repository` already implements the standard operations:

- `findById()`
- `findAll()`
- `save()`
- `update()`
- `delete()`

Your concrete repository therefore normally contains the **entity/DTO types and
constructor dependencies**, rather than reimplementing those CRUD methods.

The complete flow is:

```text
Entity
   │
   ▼
Concrete Repository
   │
   ├── IMapper
   │
   └── IWriteDataSource
           │
           ▼
        Database

AppBuilder
   │
   └── registers Repository
            │
            ▼
        DI Container
            │
            ▼
        resolve(...)
```

## Prerequisites

Install Xeno.JS in your application:

```bash
npm install @xeno-js/core
```

You also need implementations of:

- the DTO used by the data source;
- `IWriteDataSource<TDto>`;
- `IMapper<TEntity, TDto>`.

`Repository` is exported from `@xeno-js/core`.

```ts
import { XenoAppBuilder, Repository } from '@xeno-js/core'
```

The shared contracts are also exported through the Xeno.JS package:

```ts
import type { IMapper, IWriteDataSource } from '@xeno-js/core'
```

## 1. Define the Entity

Start with the application entity.

For example:

```ts
export interface User {
  id: string
  name: string
  email: string
}
```

The entity is the type exposed by the repository to the application.

The repository does not expose the DTO used internally by the data source.

## 2. Define the DTO

Define the representation expected by the data source.

For example:

```ts
export interface UserDto {
  id: string
  fullName: string
  emailAddress: string
}
```

The entity and DTO can have different property names or structures.

That is the reason the repository receives an `IMapper`.

## 3. Implement the Mapper

`Repository<T, TDto>` requires an `IMapper<T, TDto>`.

The public mapper contract contains three methods:

```ts
interface IMapper<TE, TDto> {
  toDto(entity: TE): TDto
  toEntity(dto: TDto): TE
  toPartialDto(entity: Partial<TE>): Partial<TDto>
}
```

A concrete mapper can therefore be implemented as:

```ts
import type { IMapper } from '@xeno-js/core'

import type { User } from './user'
import type { UserDto } from './user.dto'

export class UserMapper implements IMapper<User, UserDto> {
  toDto(entity: User): UserDto {
    return {
      id: entity.id,
      fullName: entity.name,
      emailAddress: entity.email,
    }
  }

  toEntity(dto: UserDto): User {
    return {
      id: dto.id,
      name: dto.fullName,
      email: dto.emailAddress,
    }
  }

  toPartialDto(entity: Partial<User>): Partial<UserDto> {
    return {
      ...(entity.id !== undefined && { id: entity.id }),
      ...(entity.name !== undefined && { fullName: entity.name }),
      ...(entity.email !== undefined && { emailAddress: entity.email }),
    }
  }
}
```

The mapper is used by the base repository as follows:

```text
findById / findAll
        │
        ▼
IWriteDataSource<UserDto>
        │
        ▼
     UserDto
        │
        ▼
IMapper.toEntity()
        │
        ▼
      User
```

For write operations:

```text
User
 │
 ▼
IMapper.toDto()
 │
 ▼
UserDto
 │
 ▼
IWriteDataSource
```

For partial updates:

```text
Partial<User>
 │
 ▼
IMapper.toPartialDto()
 │
 ▼
Partial<UserDto>
 │
 ▼
IWriteDataSource.update()
```

## 4. Implement the Data Source

The repository depends on `IWriteDataSource<TDto>`.

The contract extends `IDisposable` and contains both read and write operations:

```ts
import type { IDisposable, Optional, UserContext } from '@xeno-js/core'

interface IWriteDataSource<TDto> extends IDisposable {
  findAll(
    ctx?: Optional<UserContext>,
    signal?: Optional<AbortSignal>,
  ): Promise<TDto[]>

  findById(
    id: string | number,
    ctx?: Optional<UserContext>,
    signal?: Optional<AbortSignal>,
  ): Promise<Optional<TDto>>

  insert(dto: TDto, signal?: Optional<AbortSignal>): Promise<void>

  delete(
    dto: TDto,
    ctx?: Optional<UserContext>,
    signal?: Optional<AbortSignal>,
  ): Promise<void>

  update(
    id: string | number,
    dto: Partial<TDto>,
    ctx?: Optional<UserContext>,
    signal?: Optional<AbortSignal>,
  ): Promise<void>

  dispose(): Promise<void>
}
```

The concrete data source must implement these operations and `dispose()`. It is
responsible for the actual persistence operation and releasing any resources it
owns.

The important separation is:

```text
UserRepository
    │
    ▼
IWriteDataSource<UserDto>
    │
    ▼
database implementation
```

The repository does not construct SQL itself when using this abstraction. The
data source owns the persistence-specific implementation.

## 5. Create the Concrete Repository

Now create the repository itself.

The Xeno.JS base class is:

```ts
abstract class Repository<T, TDto>
```

Its constructor requires:

```ts
constructor(
  dataSource: IWriteDataSource<TDto>,
  mapper: IMapper<T, TDto>,
)
```

Therefore the concrete repository can be:

```ts
import { Repository } from '@xeno-js/core'

import type { IMapper, IWriteDataSource } from '@xeno-js/core'

export class UserRepository extends Repository<User, UserDto> {
  constructor(
    dataSource: IWriteDataSource<UserDto>,
    mapper: IMapper<User, UserDto>,
  ) {
    super(dataSource, mapper)
  }
}
```

This is enough to obtain the standard repository operations.

You do **not** need to implement:

```ts
findById()
findAll()
save()
update()
delete()
```

again.

They are already implemented by `Repository`.

### Adding application-specific methods

If your repository needs operations beyond the base CRUD contract, you can add
methods to the concrete class.

For example:

```ts
import type {
  IMapper,
  IWriteDataSource,
  Optional,
  ResultType,
  UserContext,
} from '@xeno-js/core'
import { Repository } from '@xeno-js/core'

import type { User } from './user'
import type { UserDto } from './user.dto'

export class UserRepository extends Repository<User, UserDto> {
  constructor(
    dataSource: IWriteDataSource<UserDto>,
    mapper: IMapper<User, UserDto>,
  ) {
    super(dataSource, mapper)
  }

  async findByEmail(
    email: string,
    ctx: UserContext,
    signal: Optional<AbortSignal>,
  ): Promise<ResultType<Optional<User>>> {
    throw new Error('Implement the application-specific email lookup.')
  }
}
```

Only methods that are not provided by the base `Repository` need to be added by
the concrete repository.

## 6. Register the Repository in `XenoAppBuilder`

Xeno.JS uses explicit dependency injection registration.

The application composition root is `XenoAppBuilder`.

The registration examples assume `app` uses the typed `Registry` shown in the
next section.

Use:

```ts
app.addServices(...)
```

to register application services.

The README contains the same pattern for a repository:

```ts
services.addScoped('USER_REPOSITORY', (container) => {
  return new UserRepository(
    container.resolve('USER_DATA_SOURCE'),
    container.resolve('USER_MAPPER'),
  )
})
```

A complete registration therefore looks like:

```ts
const app = new XenoAppBuilder()

app.addServices((services) => {
  services.addScoped('USER_DATA_SOURCE', (container) => {
    return new UserDataSource(container.resolve('DB_CONTEXT'))
  })

  services.addScoped('USER_MAPPER', () => {
    return new UserMapper()
  })

  services.addScoped('USER_REPOSITORY', (container) => {
    return new UserRepository(
      container.resolve('USER_DATA_SOURCE'),
      container.resolve('USER_MAPPER'),
    )
  })
})
```

The important point is that **the repository is not automatically discovered**.

You explicitly register:

```text
USER_DATA_SOURCE
       │
       ▼
USER_MAPPER
       │
       ▼
USER_REPOSITORY
```

and the container resolves the dependencies when it creates the repository.

## 7. Register the Repository With the Correct Lifetime

Xeno.JS supports three service lifetimes:

```ts
services.addSingleton(...)
services.addScoped(...)
services.addTransient(...)
```

### Scoped

A scoped registration creates one instance per service scope:

```ts
services.addScoped('USER_REPOSITORY', (container) => {
  return new UserRepository(
    container.resolve('USER_DATA_SOURCE'),
    container.resolve('USER_MAPPER'),
  )
})
```

A scoped service must be resolved from an active scope. `XenoAppBuilder.build()`
returns the runtime application, so use `XenoApplication.runInScope()`:

```ts
import type { UserContext } from '@xeno-js/core'

const application = await app.build()
const userContext: UserContext = {
  userId: 'd108b2d2-1df8-45ca-8a27-686f9f443269',
  tenantId: '6fdeff88-9f53-4c58-b24f-2e2db77d95ce',
}

const result = await application.runInScope(async (services) => {
  const repository = services.USER_REPOSITORY
  return repository.findAll(userContext, undefined)
})
```

The application manages the scope lifetime and disposes it when the callback
completes.

### Singleton

A singleton is shared by the container:

Register a repository as singleton only when its data source and mapper are also
root-resolvable and safe to share. Do not use the scoped `USER_DATA_SOURCE` and
`USER_MAPPER` registrations shown above for this lifetime.

For a repository registered as singleton (not the scoped registration shown
above), `XenoAppBuilder.resolve()` resolves it from the root container:

```ts
const repository = app.resolve('USER_REPOSITORY')
```

A singleton should only be used when its dependencies and application semantics
are compatible with singleton lifetime. In particular, do not capture scoped
dependencies in a singleton.

### Transient

A transient creates a new instance on each resolution:

```ts
services.addTransient('USER_REPOSITORY', (container) => {
  return new UserRepository(
    container.resolve('USER_DATA_SOURCE'),
    container.resolve('USER_MAPPER'),
  )
})
```

Because the dependencies registered above are scoped, resolve this transient
repository inside `application.runInScope()`. Root resolution is only valid when
the transient repository's dependency graph is also root-resolvable. A
repository that depends on scoped services must not be registered as a
singleton.

## 8. Extend the XenoRegistry for Application Services

Xeno.JS types the dependency injection container through `XenoRegistry`.

The registry can be extended with application-specific services.

For example:

```ts
import type { IMapper, IWriteDataSource } from '@xeno-js/core'

import type { XenoRegistry } from '@xeno-js/core'
import type { User } from './user'
import type { UserDto } from './user.dto'
import type { UserRepository } from './user.repository'

interface Registry extends XenoRegistry {
  USER_DATA_SOURCE: IWriteDataSource<UserDto>
  USER_MAPPER: IMapper<User, UserDto>
  USER_REPOSITORY: UserRepository
}
```

The builder can then use that registry:

```ts
const app = new XenoAppBuilder<Registry>()
```

This makes the custom tokens part of the compile-time registry used by
`addServices()` and `resolve()`.

The pattern is:

```text
XenoRegistry
    │
    └── application extensions
            │
            ├── USER_DATA_SOURCE
            ├── USER_MAPPER
            └── USER_REPOSITORY
```

This is preferable to leaving application dependencies untyped.

## 9. Complete Registration Example

A complete composition root can therefore look like this:

```ts
import type { IMapper, IWriteDataSource } from '@xeno-js/core'
import type { User } from './domain/users/user'
import type { UserDto } from './infrastructure/users/user.dto'
import { UserMapper } from './infrastructure/users/user.mapper'
import { UserDataSource } from './infrastructure/users/user.data-source'
import { UserRepository } from './infrastructure/users/user.repository'

import { XenoAppBuilder } from '@xeno-js/core'

import type { XenoRegistry } from '@xeno-js/core'

interface Registry extends XenoRegistry {
  USER_DATA_SOURCE: IWriteDataSource<UserDto>
  USER_MAPPER: IMapper<User, UserDto>
  USER_REPOSITORY: UserRepository
}

const app = new XenoAppBuilder<Registry>()

app.addServices((services) => {
  services.addScoped('USER_DATA_SOURCE', (container) => {
    return new UserDataSource(container.resolve('DB_CONTEXT'))
  })

  services.addScoped('USER_MAPPER', () => {
    return new UserMapper()
  })

  services.addScoped('USER_REPOSITORY', (container) => {
    return new UserRepository(
      container.resolve('USER_DATA_SOURCE'),
      container.resolve('USER_MAPPER'),
    )
  })
})

const application = await app.build()
```

At this point the repository has been registered. `build()` returns the runtime
application, not the service container.

## 10. Resolve a Scoped Repository

Resolve the repository within the scope managed by the runtime application:

```ts
const result = await application.runInScope(async (services) => {
  const repository = services.USER_REPOSITORY
  return repository.findById(userId, userContext, undefined)
})
```

The registry types `services.USER_REPOSITORY` as `UserRepository`. The scope is
disposed when the callback completes.

## 11. Resolve the Repository from `XenoAppBuilder`

`XenoAppBuilder` also exposes:

```ts
app.resolve(token)
```

This resolves from the root service container; it cannot resolve a scoped
repository. Resolve scoped repositories through the runtime application's
`runInScope()` method, as shown above.

## 12. Use the Repository in an Application Service

A common pattern is to inject or resolve the repository at the application
composition boundary and then use it from an application service or handler.

For example:

```ts
export class CreateUserHandler {
  constructor(private readonly repository: UserRepository) {}

  async execute(user: User): Promise<void> {
    await this.repository.save(user, undefined)
  }
}
```

The application service should receive the repository rather than constructing
it itself.

Avoid:

```ts
const repository = new UserRepository(dataSource, mapper)
```

inside every handler.

The DI container should own the construction:

```text
AppBuilder
    │
    ▼
ServiceContainer
    │
    ├── UserDataSource
    ├── UserMapper
    └── UserRepository
             │
             ▼
       Application code
```

This keeps the repository's dependencies explicit and replaceable.

## 13. Repository Operations

The base `Repository` implements the `IRepository<T>` contract.

### Find by ID

```ts
const result = await repository.findById(id, ctx, signal)
```

Signature:

```ts
findById(
  id: string | number,
  ctx: UserContext,
  signal: Optional<AbortSignal>,
): Promise<ResultType<Optional<T>>>
```

The repository:

1. checks the abort signal;
2. asks the data source for the DTO;
3. returns an empty successful result when the data source returns `null` or
   `undefined`;
4. maps the DTO to the entity;
5. returns the entity inside the `Result`.

### Find all

```ts
const result = await repository.findAll(ctx, signal)
```

Signature:

```ts
findAll(
  ctx: UserContext,
  signal: Optional<AbortSignal>,
): Promise<ResultType<T[]>>
```

Every DTO returned by the data source is converted with:

```ts
mapper.toEntity(dto)
```

### Save

```ts
const result = await repository.save(user, signal)
```

Signature:

```ts
save(
  entity: T,
  signal: Optional<AbortSignal>,
): Promise<ResultType<void>>
```

The repository converts the entity with:

```ts
mapper.toDto(entity)
```

and passes the DTO to:

```ts
dataSource.insert(dto, signal)
```

### Update

```ts
const result = await repository.update(userId, { name: 'Alice' }, ctx, signal)
```

Signature:

```ts
update(
  id: string | number,
  entity: Partial<T>,
  ctx: UserContext,
  signal: Optional<AbortSignal>,
): Promise<ResultType<void>>
```

The partial entity is converted with:

```ts
mapper.toPartialDto(entity)
```

before being passed to the data source.

### Delete

```ts
const result = await repository.delete(user, ctx, signal)
```

Signature:

```ts
delete(
  entity: T,
  ctx: UserContext,
  signal: Optional<AbortSignal>,
): Promise<ResultType<void>>
```

The entity is first converted to its DTO representation and then passed to the
data source.

## 14. Cancellation

The base repository accepts an optional `AbortSignal`.

For example:

```ts
const controller = new AbortController()

const result = await repository.findById(userId, ctx, controller.signal)
```

The repository checks whether the operation has already been aborted before
invoking the corresponding data-source operation.

The repository tests explicitly verify this behavior for:

- `findById`;
- `findAll`;
- `save`;
- `delete`;
- `update`.

One implementation detail that matters when reasoning about this behavior is
that `save`, `delete`, and `update` perform their mapper operation before the
abort check. The data-source operation is not invoked after the signal has
already been aborted.

## 15. Result Handling

Repository methods return `ResultType`.

For example:

```ts
const result = await repository.findById(userId, ctx, undefined)
```

The repository itself wraps successful operations with `Result.ok(...)`.

For a missing entity, `findById()` returns a successful result whose value is
`undefined`. If a data-source or mapper operation throws, the repository does
not catch that exception or convert it into a `Result`; the returned promise
rejects.

Therefore a missing entity should not be confused with a repository failure.

Conceptually:

```text
findById()
    │
    ├── entity found
    │      └── Result.ok(entity)
    │
    ├── entity not found
    │      └── Result.ok(undefined)
    │
    └── underlying operation throws
           └── rejected promise; the base class does not convert it to Result
```

## 16. Repository vs Data Source

The two abstractions have different responsibilities.

### Repository

The repository works with application entities:

```ts
User
```

and coordinates:

```text
Entity
  ↕
Mapper
  ↕
DTO
  ↕
DataSource
```

### Data Source

The data source works with persistence DTOs:

```ts
UserDto
```

and performs the actual data access.

This separation allows the application-facing repository to remain independent
from the persistence representation.

## 17. Repository vs ReadDao

Xeno.JS also exposes an abstract `ReadDao<T, TDto>`.

`ReadDao` implements the read-only `IReadDao<T>` contract:

```ts
findById(...)
findAll(...)
```

`Repository` implements the broader `IRepository<T>` contract:

```ts
findById(...)
findAll(...)
save(...)
update(...)
delete(...)
```

Use `Repository` when the abstraction needs the complete CRUD contract.

Use `ReadDao` when the abstraction is intentionally read-only.

The important distinction is therefore the public contract:

```text
ReadDao
  └── findById
  └── findAll

Repository
  ├── findById
  ├── findAll
  ├── save
  ├── update
  └── delete
```

## 18. Complete Example

A typical application can be organized as:

```text
src/
├── application/
│   └── users/
│       └── create-user.handler.ts
├── domain/
│   └── users/
│       └── user.ts
├── infrastructure/
│   └── users/
│       ├── user.data-source.ts
│       ├── user.mapper.ts
│       └── user.repository.ts
└── bootstrap.ts
```

### `user.repository.ts`

```ts
import type { User } from '../../domain/users/user'
import type { UserDto } from './user.dto'

import { Repository } from '@xeno-js/core'

import type { IMapper, IWriteDataSource } from '@xeno-js/core'

export class UserRepository extends Repository<User, UserDto> {
  constructor(
    dataSource: IWriteDataSource<UserDto>,
    mapper: IMapper<User, UserDto>,
  ) {
    super(dataSource, mapper)
  }
}
```

### `bootstrap.ts`

```ts
import type {
  IMapper,
  Optional,
  UserContext,
  IWriteDataSource,
} from '@xeno-js/core'
import type { User } from './domain/users/user'
import type { UserDto } from './infrastructure/users/user.dto'
import { UserMapper } from './infrastructure/users/user.mapper'
import { UserDataSource } from './infrastructure/users/user.data-source'
import { UserRepository } from './infrastructure/users/user.repository'

import { XenoAppBuilder } from '@xeno-js/core'

import type { XenoRegistry } from '@xeno-js/core'

interface Registry extends XenoRegistry {
  USER_DATA_SOURCE: IWriteDataSource<UserDto>
  USER_MAPPER: IMapper<User, UserDto>
  USER_REPOSITORY: UserRepository
}

const app = new XenoAppBuilder<Registry>()

app.addServices((services) => {
  services.addScoped('USER_DATA_SOURCE', (container) => {
    return new UserDataSource(container.resolve('DB_CONTEXT'))
  })

  services.addScoped('USER_MAPPER', () => {
    return new UserMapper()
  })

  services.addScoped('USER_REPOSITORY', (container) => {
    return new UserRepository(
      container.resolve('USER_DATA_SOURCE'),
      container.resolve('USER_MAPPER'),
    )
  })
})

const application = await app.build()
const userId = 'user-123'
const userContext: UserContext = {
  userId: 'd108b2d2-1df8-45ca-8a27-686f9f443269',
  tenantId: '6fdeff88-9f53-4c58-b24f-2e2db77d95ce',
}
const signal: Optional<AbortSignal> = undefined

const result = await application.runInScope(async (services) => {
  const repository = services.USER_REPOSITORY
  return repository.findById(userId, userContext, signal)
})
```

The important application workflow is:

```text
1. Define Entity
       ↓
2. Define DTO
       ↓
3. Implement IMapper
       ↓
4. Implement IWriteDataSource
       ↓
5. Extend Repository<Entity, DTO>
       ↓
6. Register dependencies with addServices()
       ↓
7. Register concrete Repository
       ↓
8. build()
       ↓
9. runInScope() for scoped services
       ↓
10. Resolve USER_REPOSITORY inside the scope
       ↓
11. repository.findById(...)
    repository.findAll(...)
    repository.save(...)
    repository.update(...)
    repository.delete(...)
```

## Troubleshooting

### `Repository` cannot be instantiated

`Repository` is abstract.

Do not do:

```ts
const repository = new Repository<User, UserDto>(dataSource, mapper)
```

Create a concrete subclass:

```ts
class UserRepository extends Repository<User, UserDto> {
  constructor(
    dataSource: IWriteDataSource<UserDto>,
    mapper: IMapper<User, UserDto>,
  ) {
    super(dataSource, mapper)
  }
}
```

### The repository cannot be resolved

Check that it was registered:

```ts
app.addServices((services) => {
  services.addScoped('USER_REPOSITORY', ...)
})
```

Also check that `build()` has completed before resolving services. For a scoped
repository, resolve it inside `application.runInScope()`.

### A scoped repository cannot be resolved from `XenoAppBuilder`

This is expected.

A scoped service requires an active scope.

Use the runtime application's scoped execution API:

```ts
const application = await app.build()
await application.runInScope(async (services) => {
  const repository = services.USER_REPOSITORY
  // Use the repository within this scope.
})
```

### A repository dependency cannot be resolved

Check every dependency used by its factory:

```ts
services.addScoped('USER_REPOSITORY', (container) => {
  return new UserRepository(
    container.resolve('USER_DATA_SOURCE'),
    container.resolve('USER_MAPPER'),
  )
})
```

Both:

```text
USER_DATA_SOURCE
USER_MAPPER
```

must be registered before the repository can be constructed.

### The repository token is not type-safe

Extend the application's `XenoRegistry`:

```ts
interface Registry extends XenoRegistry {
  USER_REPOSITORY: UserRepository
}
```

and construct:

```ts
const app = new XenoAppBuilder<Registry>()
```

This makes the custom token part of the registry used by the DI APIs.

### The mapper is not converting data correctly

Check all three mapper methods:

```ts
toDto(...)
toEntity(...)
toPartialDto(...)
```

The base repository uses them for different operations:

| Repository operation | Mapper method  |
| -------------------- | -------------- |
| `findById`           | `toEntity`     |
| `findAll`            | `toEntity`     |
| `save`               | `toDto`        |
| `update`             | `toPartialDto` |
| `delete`             | `toDto`        |

## Checklist

Before using a repository in an application, verify:

- [ ] The entity type is defined.
- [ ] The persistence DTO is defined.
- [ ] `IMapper<Entity, Dto>` is implemented.
- [ ] `IWriteDataSource<Dto>` is implemented.
- [ ] The concrete repository extends `Repository<Entity, Dto>`.
- [ ] The mapper is registered in the DI container.
- [ ] The data source is registered in the DI container.
- [ ] The repository is registered with `addServices()`.
- [ ] The repository token is included in the application's registry when
      type-safe custom tokens are desired.
- [ ] The application is built before resolving services.
- [ ] Scoped repository work runs inside `XenoApplication.runInScope()`.
- [ ] The repository is resolved using the same token used during registration.

---

## Related Docs

- [Data Overview](../overview)
- [Node PostgreSQL](../node-postgresql)
- [SQLite](../sqllite)
- [Unit of Work](../unit-of-work)
- [Service Registration](../../dependency-injection/registration)
- [Service Resolution](../../dependency-injection/resolution)
- [Dependency Graph](../../dependency-injection/dependency-graph)
- [Read Dao](./read-dao)

---

---

<a id="source-13"></a>

# Source: `docs/data/sqllite.md`

## Introduction

Use the SQLite/libSQL integration when your application needs relational data
access through Xeno.JS, whether the database runs locally or through a libSQL
service.

The workflow is:

1. Configure the SQLite connection.
2. Register the SQLite plugin with `XenoAppBuilder.addDb()`.
3. Build the application.
4. Resolve `DB_CONTEXT` inside an active service scope.
5. Use `UNIT_OF_WORK` when multiple database operations must execute
   transactionally.

Xeno.JS provides the application-facing database integration. Your repositories
and application services decide how that database access is used by the
application.

## Prerequisites

Before configuring SQLite, make sure that:

- your application uses `@xeno-js/core`;
- the SQLite/libSQL database dependencies required by your application are
  installed;
- you have a SQLite connection URL accepted by the configured SQLite/libSQL
  client;
- your application bootstraps through `XenoAppBuilder`.

For example:

```bash
npm install @xeno-js/core @xeno-js/sqlite drizzle-orm @libsql/client
```

The exact dependency versions should follow the versions used by your Xeno.JS
installation.

## Configure SQLite

Configure SQLite with `withSqlite()` from `@xeno-js/sqlite`. By default, the
plugin reads `DATABASE_URL` and `DB_AUTH_TOKEN` from the configuration service
and sets the libSQL concurrency option to `50`. Override these values in the
setup callback when required.

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withSqlite } from '@xeno-js/sqlite'

const app = new XenoAppBuilder()

app.addDb(
  withSqlite((options) => {
    options.url = 'file:./data/app.db'
  }),
)

const application = await app.build()
```

The `url` option is passed to the libSQL client. Use a URL supported by that
client, such as a local `file:` URL.

## Register SQLite in XenoAppBuilder

Register `withSqlite()` during application bootstrap with `addDb()`:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withSqlite } from '@xeno-js/sqlite'

const app = new XenoAppBuilder()

app.addDb(
  withSqlite((options) => {
    options.url = 'file:./data/app.db'
  }),
)

const application = await app.build()
```

Call `build()` before resolving application services.

`addDb()` accepts a database plugin. The plugin configures the client and queues
the database module; you do not need to instantiate a SQLite client manually.

## Resolve the SQLite database context

After `addDb()` has been configured and the application has been built, Xeno.JS
exposes the database context through the `DB_CONTEXT` service.

`DB_CONTEXT` is scoped, so it must be resolved from an active service scope.

Resolve the scoped service inside an application scope:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withSqlite } from '@xeno-js/sqlite'

const app = new XenoAppBuilder()

app.addDb(
  withSqlite((options) => {
    options.url = 'file:./data/app.db'
  }),
)

const application = await app.build()

await application.runInScope(async (services) => {
  const db = services.DB_CONTEXT
  // Use the scoped database context here.
})
```

`DB_CONTEXT` is scoped. Do not resolve it from the builder's root container; use
`runInScope()` or an already active scope.

For more information about scoped services and resolution, see:

- [Scoped](../dependency-injection/lifetimes/scoped)
- [Resolution](../dependency-injection/resolution)
- [Service Container](../dependency-injection/service-container)

## Create a SQLite Data Source

Xeno.JS exposes `IReadDataSource` and `IWriteDataSource` as contracts for data
access.

Create a concrete data source in your application:

```ts
import type { IReadDataSource } from '@xeno-js/core'
import type { DbSchema } from './schema'
import type { DbContext } from '@xeno-js/sqlite'

export class UserDataSource implements IReadDataSource {
  constructor(private readonly _db: DbContext<DbSchema>)
  async findUserById(id: string) {
    // Use this.db for SQLite queries.
  }
}
```

In normal application code, the database context is resolved from the active
scope and passed to the data source.

## Use the Data Source from Application Code

A typical application keeps SQLite access behind a data source or repository
instead of resolving the database context throughout the application.

For example:

```ts
import type { IWriteDataSource } from '@xeno-js/core'
import type { DbSchema } from './schema'
import type { DbContext } from '@xeno-js/sqlite'

export class UserDataSource implements IWriteDataSource {
  constructor(private readonly _db: DbContext<DbSchema>)
  async findUserById(id: string) {
    // Use this.db to execute the query for your application's schema.
  }

  async createUser(user: {
    id: string
    email: string
  }) {
    // Use this._db to insert the user.
  }
}
```

The application service can then depend on `UserDataSource` rather than directly
depending on the SQLite client.

A typical application flow is:

```text
Command / Query
      ↓
Handler
      ↓
Application Service / Repository
      ↓
UserDataSource
      ↓
SQLite
```

This keeps the database provider as application infrastructure rather than
making the database client part of the application boundary.

## Register a Data Source in DI

If your data source is an application service, register it through the normal
Xeno.JS dependency injection APIs.

For example:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withSqlite } from '@xeno-js/sqlite'
import type { DbContext, DbTransaction } from '@xeno-js/sqlite'
import type { XenoRegistry } from '@xeno-js/shared'

interface AppRegistry extends XenoRegistry<DbContext, DbTransaction> {
  USER_DATA_SOURCE: UserDataSource
}

const app = new XenoAppBuilder<AppRegistry>()

app.addDb(
  withSqlite<AppRegistry>((options) => {
    options.url = 'file:./data/app.db'
  }),
)

app.addServices((container) => {
  container.addScoped('USER_DATA_SOURCE', (scope) => {
    const db = scope.resolve('DB_CONTEXT')

    return new UserDataSource(db)
  })
})

const application = await app.build()
```

The exact token used for an application-specific service depends on your
application's registry.

The important relationship is:

```text
SQLite registration
      ↓
DB_CONTEXT
      ↓
UserDataSource
      ↓
application service / repository
```

Because both the database context and the data source are scoped, resolve them
inside the active scope in which the application operation executes.

For example:

```ts
await application.runInScope(async (services) => {
  const userDataSource = services.USER_DATA_SOURCE
  await userDataSource.findUserById('user-1')
})
```

See [Registration](../dependency-injection/registration) and
[Scoped](../dependency-injection/lifetimes/scoped) for the general DI rules.

## Use SQLite with Unit of Work

Use Unit of Work when several database operations must succeed or fail as one
transactional operation.

For example, related data changes can be grouped in one transaction:

```ts
await application.runInScope(async (services) => {
  const unitOfWork = services.UNIT_OF_WORK
  const userDataSource = services.USER_DATA_SOURCE

  await unitOfWork.runInTransaction(async () => {
    await userDataSource.createUser({
      id: 'user-1',
      email: 'user@example.com',
    })
  })
})
```

The public API is:

```ts
runInTransaction(callback, aggregates?, signal?)
```

The callback receives the transaction database context. The optional
`aggregates` argument lets the SQLite Unit of Work persist uncommitted domain
events to the outbox table when outbox support is configured. The optional
`signal` can abort the operation.

You do not manually call:

```ts
begin()
commit()
rollback()
```

The callback defines the transactional work.

If the callback completes successfully, the transaction completes successfully.
If the callback fails, the transactional operation does not complete
successfully and the error is propagated to the caller.

## Configure the Outbox

The SQLite plugin can create outbox tables for named aggregates. Enable the
outbox and provide at least one aggregate name:

```ts
app.addDb(
  withSqlite((options) => {
    options.url = 'file:./data/app.db'
    options.outbox = {
      enable: true,
      name: 'Order',
      worker: true,
    }
  }),
)
```

The plugin creates a table named `order_outbox_messages`. For multiple
aggregates, provide an array of names. Setting `outbox.enable` to `true` without
`outbox.name` causes configuration to throw.

When outbox support is enabled, `runInTransaction()` can accept an aggregate
root or an array of aggregate roots as its second argument. The SQLite Unit of
Work writes their uncommitted events to the outbox table in the transaction,
then clears those events after insertion.

When `outbox.worker` is `true`, the plugin registers the worker under
`TOKENS.BACKGROUND_WORKER`; it does not start it during plugin configuration.
The worker's `start()` method polls every 5000 ms by default, processes up to 50
pending messages per table per poll, and marks a message `FAILED` after its
third unsuccessful dispatch attempt.

## Complete Example

A minimal application can be structured like this:

```text
src/
├── app.ts
└── users/
    └── user.data-source.ts
```

### `src/users/user.data-source.ts`

```ts
import { BaseSqliteSqlDataSource } from '@xeno-js/core/db'

export class UserDataSource extends BaseSqliteSqlDataSource {
  async createUser(user: { id: string; email: string }) {
    // Use this.db with the application's Drizzle schema.
    // Example query omitted because the schema is application-specific.
  }
}
```

### `src/app.ts`

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withSqlite } from '@xeno-js/sqlite'
import type { DbContext, DbTransaction } from '@xeno-js/sqlite'
import type { XenoRegistry } from '@xeno-js/shared'

import { UserDataSource } from './users/user.data-source'

interface AppRegistry extends XenoRegistry<DbContext, DbTransaction> {
  USER_DATA_SOURCE: UserDataSource
}

const app = new XenoAppBuilder<AppRegistry>()

app.addDb(
  withSqlite<AppRegistry>((options) => {
    options.url = 'file:./data/app.db'
  }),
)

app.addServices((container) => {
  container.addScoped('USER_DATA_SOURCE', (scope) => {
    return new UserDataSource(scope.resolve('DB_CONTEXT'))
  })
})

const application = await app.build()

await application.runInScope(async (services) => {
  const userDataSource = services.USER_DATA_SOURCE
  const unitOfWork = services.UNIT_OF_WORK

  await unitOfWork.runInTransaction(async () => {
    await userDataSource.createUser({
      id: 'user-1',
      email: 'user@example.com',
    })

    // Additional related database operations can be performed here.
  })
})
```

The query itself is intentionally application-specific: Xeno.JS provides the
SQLite data source base class and database context, while your application
defines its Drizzle schema and data-access operations.

## Troubleshooting

### SQLite is not being used

Check that `withSqlite()` is passed to `addDb()`:

```ts
app.addDb(
  withSqlite((options) => {
    options.url = 'file:./data/app.db'
  }),
)
```

The SQLite plugin selects the libSQL client; its configuration does not use an
`enableSqlLite` switch.

### The database connection is not configured

Check that:

```ts
options.url = '...'
```

is assigned in the `withSqlite()` setup callback, or that `DATABASE_URL` is
configured.

Also verify that the connection URL is valid for the SQLite/libSQL client used
by your application.

### `DB_CONTEXT` cannot be resolved

Make sure that:

1. `addDb()` was called;
2. `await app.build()` completed successfully;
3. the resolution happens inside an active service scope.

For scoped resolution, use:

```ts
await application.runInScope(async (services) => {
  const db = services.DB_CONTEXT
})
```

Resolving a scoped service without an active scope throws an error.

See [Resolution](../dependency-injection/resolution) for the general
scoped-resolution rules.

### The SQLite data source cannot access `db`

`BaseSqliteSqlDataSource` exposes `db` as a `protected` property.

Use it from a class that extends `BaseSqliteSqlDataSource`:

```ts
class UserDataSource extends BaseSqliteSqlDataSource {
  async findUser() {
    return this.db
  }
}
```

Do not expect callers outside the data source to access `db` directly.

### The provider-specific data source does not work

Make sure the provider matches the base data source:

```text
SQLite configuration
        ↓
BaseSqliteSqlDataSource
```

For PostgreSQL, use `BasePostgresSqlDataSource` instead.

Do not mix the provider configuration and provider-specific data source.

### Transactional work is failing

Check that:

- `UNIT_OF_WORK` is resolved from an active scope;
- all operations that must belong to the transaction execute inside the
  `runInTransaction()` callback;
- errors from the transactional operation are not being silently discarded.

Example:

```ts
await unitOfWork.runInTransaction(async () => {
  await userDataSource.createUser(user)
})
```

Do not manually manage `begin`, `commit`, or `rollback`.

## Related Docs

- [Data Overview](./overview)
- [Node PostgreSQL](./node-postgresql)
- [Unit of Work](./unit-of-work)
- [Service Container](../dependency-injection/service-container)
- [Registration](../dependency-injection/registration)
- [Resolution](../dependency-injection/resolution)
- [Scoped](../dependency-injection/lifetimes/scoped)
- [Dependency Graph](../dependency-injection/dependency-graph)

---

---

<a id="source-14"></a>

# Source: `docs/data/unit-of-work.md`

## Introduction

Use Unit of Work when multiple database operations must succeed or fail
together.

A typical application operation looks like this:

```text
Application service / handler
        ↓
Unit of Work
        ↓
Repository / Data Source
        ↓
Database
```

The shared contract accepts a callback, optional aggregate root(s), and an
optional cancellation signal:

```text
await unitOfWork.runInTransaction(callback, aggregates?, signal?)
```

You do not manually call `begin()`, `commit()`, or `rollback()`.

## Prerequisites

Before using Unit of Work:

1. Configure a database plugin with `XenoAppBuilder.addDb()`.
2. Build the application.
3. Execute the transactional operation inside an active service scope.
4. Resolve `unitOfWork` from that scope.

For provider-specific configuration, see:

- [Node PostgreSQL](./node-postgresql)
- [SQLite](./sqlite)

For the DI concepts used by this page, see:

- [Registration](../dependency-injection/registration)
- [Resolution](../dependency-injection/resolution)
- [Scoped Lifetime](../dependency-injection/lifetimes/scoped)

## Configure the Database

Unit of Work is registered by the database module, so the database must be
configured during application bootstrap.

For example:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withPostgresql } from '@xeno-js/postgresql'

const app = new XenoAppBuilder()

app.addDb(
  withPostgresql((options, config) => {
    options.connectionString = config.get('DATABASE_URL')!
  }),
)
```

For SQLite, enable the SQLite provider:

```ts
import { withSqlite } from '@xeno-js/sqlite'

app.addDb(
  withSqlite((options, config) => {
    options.url = config.get('DATABASE_URL', 'file:./data/app.db')!
  }),
)
```

The provider configuration belongs in application bootstrap. Unit of Work does
not require a separate transaction configuration.

## Build the Application

Build the application before resolving `unitOfWork`:

```ts
const application = await app.build()
```

`build()` returns the runtime application. Resolve scoped services through its
active scope.

`unitOfWork` is a scoped service, so it must be resolved from an active scope.

## Resolve Unit of Work

Use `runInScope()` to resolve `unitOfWork` for an operation:

```ts
const application = await app.build()

await application.runInScope(async (services) => {
  const unitOfWork = services.unitOfWork
  // transactional work
})
```

The scope determines the lifetime of the Unit of Work instance and is disposed
when the callback completes.

In a request-driven application, the framework integration may already provide
an active service scope. In that case, resolve the service from the current
application scope instead of creating a second scope unnecessarily.

## Start Transactional Work

Call `runInTransaction()` with an asynchronous callback:

```ts
await unitOfWork.runInTransaction(async (tx) => {
  // Execute database operations with tx.
})
```

The callback contains the complete set of operations that must execute as one
transaction.

The callback receives the transaction context. The second argument is for
aggregate root(s); the third is an optional `AbortSignal`. When no aggregates
are supplied, pass `undefined` before the signal.

## Base and Provider-Specific Implementations

The shared `IUnitOfWork<TTransaction>` contract is provider-neutral. The core
`UnitOfWork` implementation runs the callback through the configured
transactional database and supplies its active transaction as `tx`. It also
exposes that transaction to scoped `DB_CONTEXT` consumers for the duration of
the callback, then clears the transaction state.

SQLite and PostgreSQL use the same callback contract, with `tx` typed as their
Drizzle database context. When their Outbox option is enabled, the database
plugin supplies a provider-specific Unit of Work: after the callback succeeds,
it writes any uncommitted events from the supplied aggregate root(s) to the
Outbox table in that transaction. Configure the Outbox and a name for each
aggregate type in the provider plugin; passing aggregates alone to the core
implementation does not persist events.

For example, the provider-specific call shape is:

```ts
await unitOfWork.runInTransaction(
  async (tx) => {
    await tx.insert(orders).values(orderDto).execute()
  },
  order,
  signal,
)
```

The Outbox argument is optional. Without an aggregate argument,
`runInTransaction(callback)` executes the ordinary database transaction. See
[Transactional Outbox](/docs/data/outbox) for configuration, delivery limits,
and recovery considerations.

## Perform Multiple Data Operations

Unit of Work is useful when one application operation changes multiple pieces of
data.

For example, creating a user may require both a user record and a related
profile:

```ts
class UserService {
  constructor(
    private readonly unitOfWork: IUnitOfWork,
    private readonly users: UserRepository,
    private readonly profiles: ProfileRepository,
  ) {}

  async createUser(input: CreateUserInput): Promise<User> {
    return this.unitOfWork.runInTransaction(async (_tx) => {
      const user = await this.users.create({
        email: input.email,
        name: input.name,
      })

      await this.profiles.create({
        userId: user.id,
        displayName: input.displayName,
      })

      return user
    })
  }
}
```

The repositories in this example are application-defined. The important Xeno.JS
contract is the transaction boundary around the application operation.

Resolve the Unit of Work and the repositories from the same active scope. The
scoped `DB_CONTEXT` exposes the transaction opened by `runInTransaction()` to
data-access services used by the callback.

If `users.create()` and `profiles.create()` both succeed, the callback completes
successfully.

If either operation fails, the callback rejects and the transaction does not
complete successfully.

## Commit and Rollback Behavior

Xeno.JS exposes transaction boundaries through `runInTransaction()` rather than
separate commit and rollback methods.

### Successful callback

When the callback completes successfully:

```ts
await unitOfWork.runInTransaction(async (_tx) => {
  await users.create(user)
  await profiles.create(profile)
})
```

the transaction completes successfully.

There is no explicit `commit()` call.

### Failed callback

When an operation throws an error:

```ts
await unitOfWork.runInTransaction(async (_tx) => {
  await users.create(user)

  throw new Error('Profile creation failed')
})
```

the error is propagated to the caller.

There is no explicit `rollback()` call.

Handle the application error at the appropriate application boundary:

```ts
try {
  await unitOfWork.runInTransaction(async (_tx) => {
    await users.create(user)
    await profiles.create(profile)
  })
} catch (error) {
  // Handle or propagate the application failure.
  throw error
}
```

Do not catch an error only to continue execution as if the transactional
operation had succeeded.

## Use an AbortSignal

`runInTransaction()` accepts an optional `AbortSignal`:

```ts
const controller = new AbortController()

await unitOfWork.runInTransaction(
  async (_tx) => {
    await users.create(user)
  },
  undefined,
  controller.signal,
)
```

If the signal has already been aborted when `runInTransaction()` is called,
Xeno.JS rejects the operation before starting the transaction.

Use the signal when the surrounding application operation already has
cancellation semantics.

## Complete PostgreSQL Example

The following example shows the complete application-level flow:

```ts
import { XenoAppBuilder } from '@xeno-js/core'
import { withPostgresql } from '@xeno-js/postgresql'
import { sql } from 'drizzle-orm'

const app = new XenoAppBuilder()

app.addDb(
  withPostgresql((options) => {
    options.connectionString = process.env.DATABASE_URL!
  }),
)

const application = await app.build()

await application.runInScope(async (services) => {
  return services.unitOfWork.runInTransaction(async (tx) => {
    return tx.execute(sql`select 1`)
  })
})
```

The important flow is:

```text
addDb(withPostgresql(...))
  ↓
app.build()
  ↓
application.runInScope(...)
  ↓
services.UNIT_OF_WORK
  ↓
runInTransaction(...)
```

Replace the example query with the operations required by your application. For
business workflows, keep application services and scoped repositories inside the
same `runInScope()` callback.

## Use Unit of Work from a Handler

Unit of Work normally belongs below the transport boundary.

For a CQRS command, the flow can be:

```text
HTTP / transport
      ↓
Command
      ↓
Command Handler
      ↓
Application Service
      ↓
Unit of Work
      ↓
Data Source / Repository
      ↓
Database
```

For example:

```ts
class CreateUserHandler {
  constructor(private readonly users: UserService) {}

  async execute(command: CreateUserCommand): Promise<User> {
    return this.users.createUser({
      email: command.email,
      name: command.name,
      displayName: command.displayName,
    })
  }
}
```

The handler does not need to manage transaction boundaries directly when the
application service owns the transactional operation.

This keeps transaction management in the application layer rather than in the
HTTP or transport layer.

## Common Problems

### `UNIT_OF_WORK` cannot be resolved

Make sure that:

1. `addDb()` was called during bootstrap.
2. The application was built with `await app.build()`.
3. You are resolving `UNIT_OF_WORK` from an active service scope.
4. The database module was not omitted from the application bootstrap.

Example:

```ts
const application = await app.build()

await application.runInScope(async (services) => {
  const unitOfWork = services.unitOfWork
})
```

### Resolving a scoped service from the root container

`UNIT_OF_WORK` is scoped.

Do not do this:

```ts
const application = await app.build()

// Do not resolve UNIT_OF_WORK from the root application services.
application.getServices().unitOfWork
```

Resolve it inside a scoped operation:

```ts
await application.runInScope(async (services) => {
  const unitOfWork = services.unitOfWork
})
```

See [Scoped Lifetime](../dependency-injection/lifetimes/scoped) for the general
DI lifetime rules.

### Transaction work is not grouped together

Make sure every operation that must succeed or fail together is inside the same
callback:

```ts
await unitOfWork.runInTransaction(async (_tx) => {
  await users.create(user)
  await profiles.create(profile)
  await auditLog.create(entry)
})
```

Operations performed outside the callback are not part of that transaction
boundary.

### Manually calling `commit()` or `rollback()`

Do not add manual transaction control around `runInTransaction()`.

The public API is:

```text
await unitOfWork.runInTransaction(callback, aggregates?, signal?)
```

The callback defines the transactional operation.

### The transaction callback throws

A failure from the callback is propagated to the caller.

Handle the error at the application boundary where the operation should be
reported, retried, or converted into an application error.

Do not assume that returning a failure value automatically represents a
transaction failure. If the operation must fail transactionally, let the
callback reject.

## Unit of Work and Data Sources

Unit of Work defines the transaction boundary; the provider-specific Data Source
performs the actual data access.

For example:

```text
Unit of Work
    ↓
transactional application operation
    ↓
PostgreSQL Data Source
    ↓
PostgreSQL
```

or:

```text
Unit of Work
    ↓
transactional application operation
    ↓
SQLite Data Source
    ↓
SQLite
```

The same Unit of Work API is used with the database plugin configured through
`XenoAppBuilder.addDb()`.

Provider-specific setup belongs in the provider documentation:

- [Node PostgreSQL](./node-postgresql)
- [SQLite](./sqlite)

## Related Docs

### Data

- [Data Overview](./overview)
- [Node PostgreSQL](./node-postgresql)
- [SQLite](./sqlite)

### Dependency Injection

- [Service Container](../dependency-injection/service-container)
- [Registration](../dependency-injection/registration)
- [Resolution](../dependency-injection/resolution)
- [Scoped Lifetime](../dependency-injection/lifetimes/scoped)
- [Dependency Graph](../dependency-injection/dependency-graph)

### Application

- [Commands](../application/cqrs/command)
- [Command Handlers](../application/cqrs/handler)
- [Queries](../application/cqrs/query)

---
