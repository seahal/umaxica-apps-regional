# Regional Repository Scope and Boundary (2026-04-25)

## Status

Accepted

## Context

This repository has gone through several iterations of structural decomposition together with the
sibling (now "global") codebase:

1. A "Fat Engine" architecture, with `Identity`, `Zenith`, `Foundation`, and `Distributor`
   implemented as Rails Engines mounted into a single root application
   (see `adr/engine-isolate-namespace-adoption.md`, `adr/four-engine-split.md`).
2. A migration away from fat engines toward four independent Rails applications under
   `apps/identity`, `apps/zenith`, `apps/foundation`, `apps/distributor`
   (see `adr/abolish-fat-engines-move-to-independent-apps.md`).

Both approaches struggled to converge. Engine isolation produced unmanageable nested namespaces,
view-path precedence issues, and Zeitwerk friction. The four-app split, while pragmatic, still
forced cross-app coordination on shared concerns (authentication, sessions, OIDC, account
preferences) and kept regional content surfaces (docs, news, help, and other locale/region-specific
delivery) tangled with the identity surface.

In parallel, the deployment story was re-examined. The IdP surface (historically served on the
`sign.*.{app,com,org}` hosts) and the primary application surface (the `www.*.{app,com,org}` hosts)
share most of their dependencies, data models, and authorization boundaries. The remaining
surfaces — region- and locale-specific content (docs, news, help, and similar) — have a different
release cadence, different operational requirements, and a different audience.

The companion decision in the global repository
(`umaxica-apps-global/adr/split-into-regional-and-global-repos.md`) records the full reasoning,
including the WebAuthn / FIDO2 RP-ID concern that drove the IdP hostname rename from `sign.*` to
`id.*`. This ADR is the regional counterpart: it states what is, and is not, in scope for **this**
repository, and how this repository relates to the global IdP at the data-exchange layer.

(Concrete repository names, brand names, and the apex domain are intentionally omitted; this
decision is independent of any future rename.)

## Decision

This repository is the **regional** half of the two-repository split. It hosts the regional /
locale-specific delivery surfaces — currently understood as `docs`, `news`, `help`, and any other
content surfaces whose audience, cadence, or locale binding makes them inappropriate for the
global IdP + RP application.

The global repository owns:

- the IdP surface, served on `id.*.{app,com,org}` (renamed from `sign.*`), and
- the primary RP surface, served on `www.*.{app,com,org}`,
- together with the functionality previously labeled as the `Zenith` and `Signature` engines.

The regional repository owns:

- the regional content surfaces (`docs`, `news`, `help`, and related locale/region-specific
  delivery) and their content models,
- the views, controllers, jobs, and storage that exclusively serve those surfaces.

### Boundary rules

- **No more Rails Engines in this repository.** `engines/` and the `Jit::<EngineName>::` namespace
  style are abolished here as well. Code that previously lived in engines is folded into the
  standard Rails layout (`app/`, `lib/`, `config/`).
- **No more `apps/<name>/` split.** The four-app plan is superseded for this repository too;
  this repository is one ordinary Rails application.
- **Repository boundary, not engine boundary, is the isolation unit.** Isolation between the
  regional surface and the global IdP + RP surface is enforced by living in two separate
  repositories with their own deploys, dependencies, and ownership — not by in-process module
  boundaries.
- **No IdP responsibilities here.** Authentication, OIDC issuance, passkey enrollment, session
  step-up, and credential storage do not live in this repository. The canonical IdP host is
  `id.*.{app,com,org}` and is served by the global repository.

### Relationship to the global IdP (data exchange)

Because the global repository becomes the IdP, this repository participates as a **consumer** of
identity and identity-derived data. The regional surfaces remain user-aware where they need to be
(for example, personalized docs / news / help experiences, or staff-only operator views), but they
do not authenticate users themselves.

The structural commitments here are:

- This repository acts as an **OIDC relying party** (or equivalent downstream consumer) of the
  global IdP at `id.*.{app,com,org}`. Sign-in flows redirect to the global IdP; this repository
  receives identity assertions and uses them to scope regional content.
- Identity-bearing data (principal IDs, staff/operator roles, locale/region preferences relevant
  to content delivery, and similar) is **received** from the global side through a defined
  contract — OIDC claims, signed tokens, or an explicitly versioned API — rather than read from a
  shared in-process model. This repository must not assume direct access to the global IdP's
  database.
- Conversely, regional events that the global side needs to observe (e.g. content engagement
  signals relevant to account state) are **published** out of this repository through an
  explicitly versioned channel, not by reaching into global tables.
- The exact wire format, claim shape, token lifetime, and transport for this exchange are
  **deferred to implementation ADRs** and are not fixed here. What is fixed is that the boundary
  exists, that it is a remote / contractual boundary (not a shared-database boundary), and that
  this repository is the consumer side of identity, not the issuer.

### Database scope

The current 20-database layout in this repository (`principal`, `operator`, `token`, `preference`,
`guest`, `document`, `news`, `activity`, `occurrence`, `avatar`, `queue`, `cache`, and the
corresponding read replicas) was inherited from the engine / four-app era and reflects assumptions
that no longer hold once the IdP moves out.

A substantial reduction is expected: databases that exist solely to support IdP / RP concerns
(identity, tokens, sessions, IdP-side preferences, IdP-side avatars, etc.) become the global
repository's responsibility and will be removed from this repository as the migration progresses.
The regional repository's persistent state will be limited to what the regional surfaces actually
own (content, regional preferences not surfaced to the IdP, regional jobs / cache).

The exact post-migration database list is **not fixed in this ADR**; it is being worked out and
will be recorded in a follow-up ADR once the surface-by-surface migration is closer to complete.
What is fixed here is the direction: this repository's database footprint shrinks, and any
identity-owning data leaves.

## Consequences

- This repository's migration target is materially simpler than the previous engine / four-app
  plans: one Rails application, one Gemfile, one Zeitwerk load path, scoped to regional content
  surfaces.
- Cross-cutting authentication and session concerns no longer need to be re-implemented or shimmed
  here; they are delegated to the global IdP across a remote contract.
- A clear remote contract with the global IdP becomes a first-class architectural element of this
  repository. Breaking changes on either side now have a versioning surface that did not exist
  when both lived in the same process.
- A meaningful number of databases, models, controllers, and views currently in this tree are
  flagged for removal as the IdP / RP responsibilities migrate to the global repository. The
  exact list and order are out of scope for this ADR and will be tracked separately.
- Earlier ADRs that assume an engine-based or four-app structure inside this repository
  (`adr/engine-isolate-namespace-adoption.md`,
  `adr/four-engine-split.md`,
  `adr/four-engine-restoration-and-base-contract.md`,
  `adr/four-app-wrapper-runtime-and-root-retirement.md`,
  `adr/four-app-solid-cache-and-solid-queue.md`,
  `adr/abolish-fat-engines-move-to-independent-apps.md`,
  `adr/rails-way-engine-architecture-restoration.md`,
  `adr/three-engine-consolidation.md`,
  `adr/current-context-boundary-by-engine.md`,
  `adr/four-engine-restoration-and-base-contract.md`,
  `adr/ongoing-engine-migration-state.md`,
  `adr/distributor-solid-cache-queue-placement.md`,
  `adr/identity-db-scope-reduction-and-solid-setup.md`)
  are superseded by this decision to the extent that they prescribe an engine layout, a four-app
  split, or an in-process IdP inside this repository. Their domain-level reasoning (which data
  belongs together, which boundaries matter) remains useful as background, but the structural
  prescriptions no longer apply here.
- ADRs that describe IdP / authentication mechanics
  (`adr/sign-configuration-sprint-spec.md`,
  `adr/oidc-authn-hardening-implementation-decisions.md`,
  `adr/oidc-claims-decision.md`,
  `adr/refresh-revoke-aal-downgrade-and-replay-hardening.md`,
  `adr/email-otp-race-condition-fixes.md`,
  `adr/turnstile-environment-toggle.md`)
  no longer describe behavior owned by this repository. They are retained as historical context
  and as input to the global repository, where the IdP now lives. From the regional side, only
  the **consumer** view of those concerns (claim shape, token verification, RP-side step-up
  expectations) is relevant, and that view will be captured in follow-up ADRs about the regional
  ↔ global data exchange.
- The IdP hostname rename from `sign.*` to `id.*` is a global-repository concern. This repository
  participates only as a consumer: redirect targets, OIDC discovery URLs, and any cached issuer
  identifiers that this repository holds must be updated to `id.*` during implementation. No
  passkey / WebAuthn RP-ID configuration lives in this repository.
