# Web portfolio architecture

*Last updated: 2026-05-19 (revised after full portfolio walk). Forward-looking; subject to revision as the modules and consumer sites evolve.*

## Purpose

This document captures the planned shape of the infodancer web stack -- the set of reusable Go modules under this org that compose into personal and portfolio sites -- and the direction the existing consumer sites are heading. It complements the mail-stack docs in this directory; it is *not* an implementation plan.

The motivating constraint: one developer maintaining four-plus sites who is honest about not enjoying frontend work and would rather build each visual decision once.

## The destination

Every consumer site is a thin shell around a set of reusable modules sharing a common visual identity. Site-specific code is reduced to what is actually unique to that site. Static and dynamic content alike are served from Go services using `html/template`, composed from a library of feature modules and a shared UI layer.

Hugo remains the primary content engine for several sites today and will be for some time. The shared UI layer is designed to serve Hugo and Go-template consumers as equal first-class clients, not to push Hugo out on a deadline. Sites migrate off Hugo when there's a concrete reason -- generally when the blog module is ready and a site's content authoring would benefit from being dynamic -- not on a schedule.

```
infodancer/ui              ← design tokens, base CSS, Go + Hugo partial variants
infodancer/faq             ← Q&A module (full-UI; v0.1 mid-development)
matthewjhunter/contact     ← Contact form (handler-only; in production)
matthewjhunter/newsletter  ← Subscribe handler (handler-only; in production)
matthewjhunter/timeline    ← Interactive timeline (full-UI; pre-release)
infodancer/blog            ← Blog / CMS (full-UI; planned)
infodancer/oidclient       ← OIDC client library (exists)
infodancer/webauth         ← SSO hub (exists)
infodancer/<other>         ← Future modules, extracted from observed duplication

speculativefiction.org     ← Go html/template; catalog + auth + newsletter; will mount faq, blog
oldschoolgamers.org        ← Hugo + Go sidecar; directory + sessions + namegen; will mount faq, blog
matthewjhunter.org         ← Hugo; will mount blog when ready
amyhunter.org              ← Hugo + Go server; mounts contact + newsletter (in production)
hunterfamily.website       ← Hugo + Go server; placeholder; newsletter likely
herald                     ← Go web app; AI-curated feed reader; currently standalone
mail webadmin              ← Go web app; mail-stack admin UI; in active development
mail webmail               ← Go web app; planned; first-class mail-stack web client
infodancer.{com,net,org}   ← Hugo info sites for the three TLDs
```

A consumer site's `main.go` becomes a small assembly: configure database, configure auth, configure shared UI, mount each module under its prefix, register the small handful of routes that are genuinely site-specific.

## The layers

The architecture has three layers above the standard library and a single external dependency layer.

### Layer 0 -- shared Go packages (no UI)

Pure-Go libraries with no visual surface. Extracted from observed duplication across feature modules, not designed up front. Likely candidates as duplication appears:

- `infodancer/oidclient` -- already exists
- `infodancer/markdown` -- goldmark + bluemonday rendering pipeline (currently lives in `infodancer/faq/internal/render`; extract when a second consumer needs it)
- `infodancer/slug` -- slug generation and validation
- `infodancer/csrf` -- middleware
- `infodancer/auth` -- already exists; UserResolver pattern shared by modules

**Principle:** do not create these speculatively. Wait for duplication. The first extraction happens when a second feature module would otherwise copy-paste the code.

### Layer 1 -- shared UI (CSS + template partials, Hugo and Go)

`infodancer/ui` is the only Layer 1 package. v1 ships small: design tokens as CSS custom properties, a base stylesheet for typography and spacing, and a few canonical partials (`nav`, `footer`) that read from the tokens. Components are extracted from duplication later.

The package ships **two parallel partial variants** from v1: Go `html/template` partials for the Go-served sites and modules, and Hugo partials for the Hugo-served sites. The CSS -- both the token file and the base stylesheet -- is identical across both worlds since CSS variables work the same way regardless of who emits the markup. Five of the active consumer sites are Hugo-primary today, so the Hugo variant is not a v2 afterthought; it's a v1 requirement.

Token vocabulary uses role-based names (`--app-color-prose-fg`, `--app-font-display`, `--app-space-tight`, `--app-radius-card`) rather than module-specific names. Feature modules read these tokens by chaining: `--faq-color-fg: var(--app-color-prose-fg, #1a1a1a)`. With `infodancer/ui` loaded, every full-UI module picks up the same palette automatically; without it, each module's defaults apply.

The token vocabulary is the load-bearing artifact in this layer. Implementations of components can be added or rewritten freely; renaming tokens breaks every consumer.

### Layer 2 -- feature modules

Self-contained services with their own data model, routes, and tests. Each module exposes:

- A `New(Config)` constructor returning a mountable type.
- A `Routes() http.Handler` that the consumer mounts under a base path with `http.StripPrefix`.
- A `Migrate(ctx)` entrypoint for goose migrations (optional; consumer may run its own migration loop).
- Authentication via a `UserResolver` the consumer provides, against whatever session mechanism it already runs.

**Two module patterns coexist in the portfolio.** They differ in how much UI the module ships:

**Handler-only modules** are pure backends with an HTTP affordance. They ship no templates and no CSS. Responses are typically tiny HTML fragments (success/error states) or JSON; the host site owns the form markup, the page chrome, and all visual styling. The module is unaware of `infodancer/ui` because it has no visual surface to skin. Examples: **contact**, **newsletter**.

**Full-UI modules** own a complete user-facing surface -- lists, detail pages, forms, search, admin screens. They ship templates and CSS, consume `infodancer/ui` tokens (so the look matches the host site automatically), and expose a template overlay (`Config.Templates fs.FS`) and a static overlay (`Config.Static fs.FS`) for hosts that need to override individual pieces. Examples: **faq**, **timeline**, planned **blog**.

The split is real and load-bearing. A handler-only module is the right choice when the surface is small enough that the host has opinions about every pixel (a contact form, a subscribe widget). A full-UI module is the right choice when the surface is large enough that asking every host to redesign it would be wasteful (a Q&A site, a timeline, a blog). Modules don't change patterns over time; the choice is made up front.

Modules in the portfolio:

- **faq** -- Q&A. Full-UI. v0.1 mid-development. Stack Exchange dump import, subsite slicing, search, voting, comments, admin moderation, FAQ document export (planned for v0.2 Track E).
- **timeline** -- Interactive event timeline. Full-UI. Pre-release. PostgreSQL-backed, htmx-driven, supports massive event counts. CUE-based wire format. Mountable into a host site once the integration surface is finalized.
- **contact** -- Contact form. Handler-only. In production at amyhunter.org. SQLite/Postgres storage, SMTP delivery, pluggable mailer interface.
- **newsletter** -- Newsletter subscribe handler. Handler-only. In production at amyhunter.org. SQLite/Postgres storage, Listmonk and Buttondown adapters, composable anti-bot middleware.
- **blog** -- Blog / lightweight CMS. Full-UI. **Planned and on the near-term critical path.** The immediate driver is osg's session-notes workflow: GitHub + Hugo is ergonomic for someone who lives in software dev, but it's a hard barrier for the campaign players who would otherwise contribute session reports from their character's POV. User adoption is gated on having a proper, user-friendly authoring CMS -- and that CMS is blog-shaped. The blog module's first job is to host osg's session notes; existing markdown bundles migrate into it gradually after launch. Shares token vocabulary with faq, and likely some Layer 0 packages (markdown rendering, slug handling, tag UI) once duplication is honest.

Other modules will likely emerge from the consumer sites once the pattern proves itself. Event / session tracking (osg), namegen (osg), and several of the surfaces in the legacy Java archive (older prototypes of forum, profile, menu, and feed-aggregation ideas) are plausible Layer 2 candidates as duplication appears.

### Layer 3 -- consumer sites

A consumer site is a Go service (sometimes paired with a Hugo build) that imports the modules it needs, threads its own auth/session into them via `UserResolver`, applies its skin via `infodancer/ui` token overrides plus optional partial overrides, and adds the routes that are unique to that site.

Each consumer keeps its own database and its own deployment. Modules can share a database with the consumer (faq writes its `faq_*` tables alongside the host's `sf_*` tables) or run in their own -- the choice is per-deployment, controlled by the connection pool the consumer passes in.

Current and planned consumer sites:

- **speculativefiction.org** -- Active. Go html/template, dynamic-heavy. Will mount faq (M6b), eventually blog. Catalog, auth, newsletter integration.
- **oldschoolgamers.org** -- Active. Hugo + Go sidecar (hybrid). Session-notes reimplementation is already underway; the driver is non-dev contributors (campaign players) who can't author via GitHub-as-CMS. Will mount **blog** as its first major use case, **faq** alongside. Other dynamic surfaces: directory, sessions, namegen, attendance.
- **matthewjhunter.org** -- Active. Hugo only today. Future consumer of blog when ready; no fixed migration deadline.
- **amyhunter.org** -- Active. Hugo + Go server. Already mounts **contact** and **newsletter** in production -- the working reference for handler-only module integration.
- **hunterfamily.website** -- Placeholder ("coming soon"). Hugo + Go server. Newsletter likely; further surface TBD.
- **herald** -- Active development. Go web app with templates and a separate CLI/MCP surface. AI-curated feed reader, currently standalone but a candidate consumer of `infodancer/ui` tokens once they exist.
- **mail webadmin** (infodancer/webadmin) -- Active development. Go web app for mail-stack administration; first-class web consumer alongside the rest of the portfolio.
- **mail webmail** -- Planned. Companion to webadmin; another first-class mail-stack web client. Not yet implemented.
- **infodancer.com / .net / .org** -- Three small Hugo info sites under `~/hugo/infodancer/{com,net,org}`. Likely candidates for `infodancer/ui` Hugo partials adoption once they exist.

The portfolio also contains older content sites (personal blogs, legacy Hugo sites) whose future under this architecture is left open; they can adopt `infodancer/ui` when convenient or keep their existing chrome indefinitely.

### Scaffolds

Four template repos under `github.com/matthewjhunter` bootstrap new sites and modules: **template-go** (Go module scaffold), **template-hugo** / **web-template** (Hugo site scaffolds), and **hmx-template** (htmx-flavored Go web scaffold). Once `infodancer/ui` exists, each scaffold should reference it from the moment a new repo is created -- that's the cheapest way to drive adoption across future sites. Updating the scaffolds is a small task on the `infodancer/ui` v1 ship checklist.

## Key design decisions

### Hugo coexistence, not retirement on a schedule

Hugo was chosen for content sites because writing CSS is unpleasant and Hugo themes look good out of the box. The cost shows up at sites that have grown dynamic features -- every dynamic surface requires a separate Go server alongside the Hugo build, and the two halves share chrome only as well as the host's templates are written to match.

That cost is real but it does not justify a flag-day retirement. Most of the portfolio's content surface is genuinely well-served by Hugo, and `infodancer/ui` is designed to make Hugo a first-class consumer of the same design tokens that Go-served sites use. The architectural direction is **convergence on shared tokens and shared partials, not convergence on a single template engine.**

Migrations off Hugo happen opportunistically, when a site's content authoring would actually benefit from being dynamic:

- **sf** is already Go-only; nothing to migrate.
- **osg** is the leading-edge migration. Session-notes are being reimplemented onto a dynamic blog-shaped CMS because GitHub + Hugo authoring is unworkable for non-dev contributors; Hugo's narrative-markdown ergonomics don't help when the people writing the reports can't operate the toolchain. The blog module's first production deployment will be here. Existing session-note markdown bundles will migrate gradually after the new tool is live. Hugo's role at osg shrinks as this lands; other dynamic features continue to move into the Go sidecar.
- **mjh** stays on Hugo until the blog module exists and there's a clear benefit. No deadline.
- **amyhunter.org**, **hunterfamily.website**, **infodancer.{com,net,org}** stay on Hugo. They're content sites; Hugo is the right tool. They become `infodancer/ui` consumers via the Hugo partials variant.
- **Personal and legacy Hugo sites outside the active portfolio** keep their existing themes indefinitely. Adopting `infodancer/ui` is optional and only worthwhile if such sites get relaunched into the active set.

### Go html/template, not React/Vue/SvelteKit

Server-rendered HTML with htmx for selective interactivity. Reasons:

- No JS build pipeline to maintain.
- No client-side state model to reason about.
- Templates parse in milliseconds; pages render in milliseconds.
- The faq module already proves the pattern at scale.
- One developer who dislikes frontend work is not building a SPA.

htmx 2 is the accepted JS dependency for fragment-based interactivity (already used by faq and sf). No other client-side framework.

### Design tokens, not component frameworks

`infodancer/ui` v1 is CSS variables and a base stylesheet -- *not* a component library. Components (cards, buttons, forms) live in feature modules until duplication forces extraction. This keeps `infodancer/ui` small and the tokens stable. A component library that ships too early dictates feature-module HTML in ways that hurt later.

When a component is extracted, it ships as a Go template partial that reads tokens. The CSS for the component goes in `infodancer/ui`. The partial can be overridden by a consumer site if needed.

### Module extraction from observed duplication

The cheapest mistake to make is creating shared packages on speculation. Faq and the planned blog will both render markdown, both handle slugs, both have tags, both have comments. The temptation is to extract `infodancer/markdown`, `infodancer/slug`, `infodancer/tags`, `infodancer/comments` up front.

The discipline is to not. Build faq's version. When blog is built, compare. Extract what is *actually* identical, leave what diverges in the modules. A premature shared abstraction puts the joints in the wrong places and is expensive to fix.

Exception: `infodancer/ui` is extracted up front because two consumer sites and two feature modules will share its vocabulary within the same quarter. The duplication is foreseeable, not speculative.

### Auth via UserResolver

Each feature module accepts a `UserResolver` from the consumer rather than running its own OIDC flow. The consumer wires its existing `oidclient`/`webauth` session into a 20-line adapter that maps the consumer's user type to the module's user type. This keeps the module unopinionated about how identity is acquired and avoids duplicate auth flows across mounted modules.

### Database per consumer, not per module

A consumer site runs one Postgres instance. Modules mounted into that consumer use the same instance, with their own schema-prefixed tables (`faq_*`, `blog_*`, etc.). This simplifies deployment (one DB to back up, one connection pool to tune) and keeps cross-module queries possible when they make sense (e.g., a "recent activity" widget on a homepage that pulls from both faq and blog).

Modules ship their own migrations via `goose` and run them on startup. The host invokes each module's `Migrate(ctx)` during boot, or the module does it itself if `Config.AutoMigrate` is true.

## Where we are today

**Modules:**
- **infodancer/faq** is at v0.1 M6a-complete on `main`. Subsites work end-to-end inside the module. The consumer-integration step (M6b, sf wiring) is the next planned work after `infodancer/ui` lands. Track E (FAQ document view) is specced for v0.2.
- **matthewjhunter/contact** and **matthewjhunter/newsletter** are in production at amyhunter.org. Handler-only modules; they validate the module pattern at the smallest scale.
- **matthewjhunter/timeline** is pre-release. PostgreSQL-backed, htmx-driven, fully functional standalone; mountable-module shape will land alongside `infodancer/ui` consumption.
- **infodancer/oidclient** and **infodancer/webauth** are in production. They are the reference for the auth integration pattern; any new module consumes them via the `UserResolver` shape.
- **infodancer/ui** does not yet exist. It is the first piece of new work this document anticipates.
- **infodancer/blog** does not yet exist but is on the near-term critical path because osg's session-notes reimplementation depends on it. Its DESIGN.md is planned alongside `infodancer/ui` so the token vocabulary is disciplined by two known full-UI consumers (faq + blog) rather than one -- and so the design is ready when implementation starts in earnest.

**Consumer sites:** sf is the immediate target for faq M6b. osg, mjh, amy, hunterfamily, herald, mail webadmin, and mail webmail are all first-class consumers in different stages. The infodancer.* TLD info sites are smaller-scope adopters once the Hugo partials variant of `infodancer/ui` exists.

The full portfolio walk that informed this revision is complete. The token vocabulary for `infodancer/ui` will be shaped by the surfaces that exist across this set, not by faq alone.

## Open questions

- **Component extraction trigger** -- at what point does a duplicated piece of HTML+CSS across faq, timeline, blog, and the Go-served consumers get extracted into `infodancer/ui`? The honest answer is "when extracting it is cheaper than keeping it duplicated, not before." The first candidate is likely a `q_card`-style content card; the second is a top-nav component once both Go and Hugo consumers agree on a shape.
- **Versioning of infodancer/ui** -- CSS token renames are breaking changes for every consumer. The token vocabulary needs to settle to v1 quickly and then be stable. Component additions are not breaking; token deletions and renames are.
- **Existing infodancer/gotemplate** -- already in the org. Its relationship to `infodancer/ui` and the feature modules needs examination during the `infodancer/ui` v1 spec.
- **Timeline integration shape** -- timeline currently runs standalone. The exact mount surface (Routes() return, UserResolver wiring, template overlay) needs to be finalized as part of its first mounted-module release. Likely follows faq's pattern with minor adjustments.
- **Handler-only modules and `infodancer/ui`** -- contact and newsletter ship no chrome. Should they grow a thin response template that consumes `infodancer/ui` tokens (so success/error fragments aren't unstyled), or do their host sites continue to own those fragments entirely? Current lean: hosts keep owning, the modules stay token-unaware.

### Resolved during the portfolio pass

- **Hugo partials variant of `infodancer/ui`** -- yes, ships in v1. Hugo is primary for five of the active consumer sites; serving Hugo consumers as second-class isn't viable.

## Next steps

1. ~~Repo pass over the full portfolio of consumer sites and supporting modules~~ -- **done 2026-05-19; findings folded into this revision.**
2. Spec `infodancer/ui` v1: token list (informed by the surfaces in faq, timeline, contact/newsletter response fragments, herald, webadmin, sf, osg, amy, hunterfamily, and the infodancer.* info sites), base CSS shape, parallel Go and Hugo partial variants, integration contract for both worlds.
3. Spec `infodancer/blog` DESIGN.md (no implementation, just the design document; its job is to validate the token vocabulary against a second known full-UI consumer).
4. Spec faq M6a-12 (theming hook in the faq module that consumes `infodancer/ui` tokens cleanly).
5. After all three specs are settled, begin implementation in the order that minimizes rework -- `infodancer/ui` v1 first; then blog and faq M6a-12 in parallel (they're independent and both block real consumer work -- blog for osg's session-notes CMS, faq M6a-12 for sf's faq mount); then faq M6b (sf wiring). The blog module is no longer a "wait and see" item -- osg's session-notes reimplementation is already moving and depends on it.
6. As `infodancer/ui` lands, update the four scaffold repos (template-go, template-hugo, web-template, hmx-template) to reference it from new-repo creation.
