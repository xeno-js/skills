<div align="center">
  <img src="logo/logo.png" alt="Xeno.JS logo" width="140" />

  <h1>Xeno.JS Skills</h1>
  <p><strong>Manage Xeno.JS skills efficiently.</strong></p>

  <p>
    <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue?style=flat-square" alt="MIT license" /></a>
    <a href="https://buymeacoffee.com/xenojs">
      <img src="https://img.shields.io/badge/Buy%20Me%20A%20Coffee-Support-FFdd00?style=flat-square&logo=buy-me-a-coffee&logoColor=black" alt="Buy Me A Coffee" />
    </a>
  </p>
</div>

---

## What it contains

This folder contains the skills and reference documentation for working with the Xeno.JS ecosystem.

| Path | Content |
| --- | --- |
| [xeno-js/SKILL.md](https://www.google.com/search?q=xeno-js/SKILL.md) | Main skill: architecture, development workflow, anti-patterns, and reference map |
| [xeno-js/references/](https://www.google.com/search?q=xeno-js/references/) | 12 technical documents, one per domain (runtime, CQRS, data, HTTP, security, logging, cache, Vue, utilities) |
| [logo/](https://www.google.com/search?q=logo/) | Graphic assets |

## Scope

The skill covers the entire Xeno.JS ecosystem:

- `@xeno-js/core`: `XenoAppBuilder`, modules, context, DI and lifetimes, CQRS, pipelines
- `@xeno-js/shared`: shared types and utilities
- `@xeno-js/vue`: Vue integration and Pinia context
- Data layer, repositories, Unit of Work, and outbox
- Controllers, routes, and HTTP adapters (Fastify, Node, Vercel)
- Authentication, authorization, and security middleware
- Logging, observability, and caching
- Xeno.JS CLI and scaffolding

## Agent specialized in Xeno.JS

The reference for the agent is the `xeno-js` skill. There is no separate agent file: anyone configuring an agent (Copilot, Claude, or others) can load `xeno-js/SKILL.md` as system instructions and consult the references based on the task.

Rules the agent must follow:

1. **Classify before writing code.** Identify the package and the layer (domain, application, infrastructure, presentation, composition) and read the corresponding reference.
2. **Do not invent APIs.** Use only documented names, imports, and signatures. If an API does not appear in the references, state it instead of inferring it from its name.
3. **Respect layering.** The domain does not depend on Xeno.JS, transport, or persistence. Controllers do not contain business logic.
4. **Register via `XenoAppBuilder`.** Queue services and modules, then call `build()`.
5. **Respect lifetimes.** No scoped services inside a singleton and no resolution of scoped services from the root container.
6. **Distinguish the context.** `AsyncLocalStorage` is the default mechanism in Node; Vue uses its own Pinia context.
7. **Separate options.** For integrations such as Pino, Redis, Supabase, Axios, or Sentry, indicate which options belong to Xeno.JS and which belong to the upstream library.

Minimal composition example, from [xeno-js/SKILL.md](https://www.google.com/search?q=xeno-js/SKILL.md):

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

## How to use the skill

- **Claude Code / agents with skill support:** copy the `xeno-js/` folder into the agent's skills directory. The `description` frontmatter determines when the skill is activated.
- **Other agents:** include `xeno-js/SKILL.md` in the system prompt and provide the references on request.

## Updating the documentation

The references in `xeno-js/references/` reproduce the Markdown of the [official documentation](https://xeno-js.it/docs/introduction). When a public API changes, update the corresponding reference and the entry in [xeno-js/references/source-coverage.md](https://www.google.com/search?q=xeno-js/references/source-coverage.md).
