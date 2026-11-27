# Source coverage and exclusions

Inventory of every Markdown file found recursively under
`website/src/content/docs/` when this skill was assembled: **101 `.md` files; no
`.mdx` files**. The ten technical references include all body content from the
99 technical documentation pages, preserving documented code examples, imports,
and API spellings. Each source section is labeled with its original relative
path.

The transformation removes each page's YAML frontmatter (website/SEO metadata
such as canonical URLs, author, sidebar, breadcrumbs, FAQ/keyword fields) and
the repeated terminal `## Support Us` sponsorship block. No technical body
section was intentionally dropped. The docs contained no standalone Astro/MDX
component markup in their Markdown bodies that needed removal; code examples are
retained as documented.

## Included pages by reference

### `application-cqrs-pipelines.md`

- `docs/application/cqrs/command.md`
- `docs/application/cqrs/handler.md`
- `docs/application/cqrs/overview.md`
- `docs/application/cqrs/query.md`
- `docs/application/cqrs/zod-schema.md`
- `docs/application/app-error.md`
- `docs/application/overview.md`
- `docs/application/result.md`
- `docs/application/pipelines/enrichment-identity.md`
- `docs/application/pipelines/exception.md`
- `docs/application/pipelines/validation.md`

### `logging-observability.md`

- `docs/application/logging/custom-logger.md`
- `docs/application/logging/overview.md`
- `docs/application/logging/pino-logger.md`
- `docs/application/logging/sentry-node.md`
- `docs/application/logging/sentry-vue.md`
- `docs/application/pipelines/logging.md`

### `caching-resilience.md`

- `docs/application/pipelines/cache-invalidation.md`
- `docs/application/pipelines/caching.md`
- `docs/application/pipelines/concurrency.md`
- `docs/application/pipelines/idempotency.md`
- `docs/application/pipelines/performance.md`
- `docs/cache/overview.md`
- `docs/cache/redis.md`

### `architecture-and-project-workflow.md`

- `docs/cli/generate-cqrs.md`
- `docs/cli/generate-http.md`
- `docs/cli/new-project.md`
- `docs/cli/overview.md`
- `docs/getting_started/create-project.md`
- `docs/getting_started/installation.md`
- `docs/getting_started/project-structure.md`
- `docs/introduction/architecture-overview.md`
- `docs/introduction/core-principles.md`
- `docs/introduction/why-xeno-js.md`
- `docs/introduction.md`

### `data-and-persistence.md`

- `docs/data/datasources/datasource.md`
- `docs/data/db-schema.md`
- `docs/data/entities/aggregate-root.md`
- `docs/data/entities/entity.md`
- `docs/data/entities/events.md`
- `docs/data/entities/unique-id.md`
- `docs/data/entities/value-object.md`
- `docs/data/node-postgresql.md`
- `docs/data/outbox.md`
- `docs/data/overview.md`
- `docs/data/repositories/read-dao.md`
- `docs/data/repositories/repository.md`
- `docs/data/sqllite.md`
- `docs/data/unit-of-work.md`

### `runtime-di-context.md`

- `docs/dependency-injection/captive-dependencies.md`
- `docs/dependency-injection/dependency-graph.md`
- `docs/dependency-injection/lifetimes/scoped.md`
- `docs/dependency-injection/lifetimes/singleton.md`
- `docs/dependency-injection/lifetimes/transient.md`
- `docs/dependency-injection/registration.md`
- `docs/dependency-injection/resolution.md`
- `docs/dependency-injection/service-container.md`
- `docs/fundamentals/app-builder.md`
- `docs/fundamentals/context-scopes.md`
- `docs/fundamentals/dependency-injection.md`
- `docs/fundamentals/overview.md`
- `docs/fundamentals/xeno-application.md`
- `docs/fundamentals/xeno-registry.md`

### `http-and-transport.md`

- `docs/http/api-response.md`
- `docs/http/client.md`
- `docs/http/controller.md`
- `docs/http/helper-http.md`
- `docs/http/plugins/fastify.md`
- `docs/http/plugins/node.md`
- `docs/http/plugins/vercel.md`
- `docs/http/routes.md`
- `docs/http/server.md`

### `security-authentication-authorization.md`

- `docs/security/authentication/custom.md`
- `docs/security/authentication/overview.md`
- `docs/security/authentication/supabase.md`
- `docs/security/authorization/overview.md`
- `docs/security/middlewares/allow-method.md`
- `docs/security/middlewares/allow-origin.md`
- `docs/security/middlewares/authentication-middleware.md`
- `docs/security/middlewares/cookie-middleware.md`
- `docs/security/middlewares/cors-middleware.md`
- `docs/security/middlewares/csrf-middleware.md`
- `docs/security/middlewares/options-middleware.md`
- `docs/security/middlewares/overview.md`
- `docs/security/middlewares/rate-limit.md`
- `docs/security/middlewares/request-context-midleware.md`
- `docs/security/overview.md`

### `shared-utilities.md`

- `docs/utils/date.md`
- `docs/utils/enumerable.md`
- `docs/utils/guards.md`
- `docs/utils/guid.md`
- `docs/utils/http.md`
- `docs/utils/math.md`
- `docs/utils/overview.md`
- `docs/utils/promise.md`
- `docs/utils/route.md`
- `docs/utils/sanitize.md`
- `docs/utils/string.md`

### `vue-integration.md`

- `docs/vue/plugin-vue.md`

## Excluded root-level pages

- `contributing-guide.md` - repository contribution workflow, rather than
  framework/API documentation. Its relevant guardrails (focused changes,
  regression tests where practical, and keeping documentation aligned with
  public behavior) are summarized in `SKILL.md`; issue/PR process details are
  omitted. Package responsibilities are covered by the architecture references.
- `support-us.md` - sponsorship and project-support information, not
  framework/API documentation.

These two root-level pages were read and inventoried but are not reproduced in
technical references. Repeated `## Support Us` sections at the ends of technical
pages are likewise omitted as site promotion; all preceding technical content
remains included.
