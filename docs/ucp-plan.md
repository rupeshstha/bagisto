# Bagisto UCP Module — Scope & Merge Plan

Oct 15, 2026 · @rupeshstha

> **Working document.** Planning notes for the UCP feature branch. Remove this file before opening a pull request to `bagisto/bagisto` (AGENTS.md: "Do not create documentation files unless explicitly requested").

## Summary and merge verdict

Build UCP as a first-party package, `packages/Webkul/UCP`, and land it as a series of small PRs against `master` (2.5). Start with the discovery manifest and read-only Catalog. Only write code after a maintainer agrees to the approach on [issue #11328](https://github.com/bagisto/bagisto/issues/11328).

**Merge likelihood (my estimate, not a measured rate):**

| Approach | Chance it merges | Why |
| --- | --- | --- |
| One big "full UCP" PR, unannounced | ~10–15% | Large external feature PRs get parked: the Paytm gateway [#11190](https://github.com/bagisto/bagisto/pull/11190) has tests and an approval but sits on **Hold**. ~40 `feat` PRs have been closed without merging. |
| Approach agreed on the issue, then phased PRs | ~45–55% for Phase 1; lower for checkout | Maintainers merge outside contributions quickly (only 5 PRs open, against ~4,800 merged), including features such as WCAG [#11318](https://github.com/bagisto/bagisto/pull/11318). |
| Standalone `bagisto-ucp` package you own | 100% shipped, 0% in core | No sign-off needed. Webkul can adopt it later. |

The biggest single risk isn't code quality. It's **direction**: Webkul may want UCP in the separate `bagisto/bagisto-api` repo, or as a paid extension. That's why step one is a comment on the issue, not a branch.

## What UCP needs from Bagisto

UCP is an agent-facing API that sits beside the storefront. It doesn't replace Bagisto's cart. A store publishes a profile at `/.well-known/ucp` listing its services, capabilities and payment handlers. Agents (AI Mode, Gemini, others) then call the capability endpoints. Spec revision: [2026-04-08](http://ucp.dev/2026-04-08/specification/overview/), date-versioned and still changing.

| Capability | UCP id | Bindings in spec | Bagisto source to wrap | Phase |
| --- | --- | --- | --- | --- |
| Discovery profile | `/.well-known/ucp` | HTTPS GET | Channel config, `core()->getCurrentChannel()` | 1 |
| Catalog (search, lookup, detail) | `dev.ucp.shopping.catalog` | REST, MCP | `ProductRepository`, product flat index, Elasticsearch | 1 |
| Cart | `dev.ucp.shopping.cart` | REST, MCP, Embedded | `Webkul\Checkout\Cart` facade, `CartRepository` | 2 |
| Checkout session | `dev.ucp.shopping.checkout` | REST, MCP, A2A, Embedded | Cart → `OrderRepository::create()`, Shipping, Tax | 3 |
| Discount (extension) | `dev.ucp.shopping.discount` | extends checkout | `CartRule` coupons | 3 |
| Fulfillment (extension) | `dev.ucp.shopping.fulfillment` | extends checkout | `Shipping` carriers | 3 |
| Order + webhooks | `dev.ucp.shopping.order` | REST, MCP | `Sales` events (`checkout.order.save.after`, shipment, refund) | 4 |
| Identity linking | `dev.ucp.common.identity_linking` | OAuth 2.0 | `Customer` + an OAuth server (decision needed) | 5 |
| AP2 mandates | `dev.ucp.shopping.ap2_mandate` | extension | Payment methods | Later |

**Rules the spec makes non-negotiable:**

- The profile must include `ucp.version`, `ucp.services` and `ucp.payment_handlers`, even when empty. It must be served over HTTPS with no redirects and `Cache-Control: public, max-age>=60`.
- Versions are negotiated: the server picks the highest date both sides support, per capability.
- Webhooks must be signed with HTTP Message Signatures (RFC 9421), and keys are published as JWK in `signing_keys[]`. Bagisto already requires `web-token/jwt-framework`, so no new dependency is needed.
- Payment credentials travel only from platform to business, as handler tokens, and must never be echoed back.
- Business outcomes return 200 with a `messages[]` array. HTTP errors are for protocol problems only.

## Design considerations

These are the decisions that will decide the review. Settle each one in the issue thread or the PR description before you code it.

1. **Home: core package or `bagisto-api`.** UCP is an API, and Bagisto's REST/GraphQL API lives in the separate [bagisto/bagisto-api](https://api-docs.bagisto.com/) repo. Ask the maintainers which one they want. Core is easier to argue for: UCP has its own wire format and doesn't fit API Platform resources.
2. **Off by default.** Use an admin toggle under *Configure → UCP* (`Config/system.php`), set per channel. Advertise only the capabilities that are enabled. A disabled store returns 404 on `/.well-known/ucp`, so nothing changes for existing installs.
3. **Channels, locales, currencies.** Resolve the channel from the request host, as Bagisto already does. The manifest, prices and availability must use that channel's base currency and locale. This is what the external package couldn't do.
4. **Money precision.** UCP uses integer minor units. Convert from Bagisto's decimal prices with the currency's `decimal` setting. Never use float maths. Write a test for zero-decimal currencies (JPY) and three-decimal ones (KWD).
5. **Protocol layer separate from domain.** Thin controllers, then immutable DTOs, then per-capability `Contracts` that call Bagisto repositories. Keep the version-specific JSON in versioned transformers (`V20260408\CatalogTransformer`), so a new spec date means a new transformer, not a rewrite.
6. **Cart sessions are not customer carts.** An agent cart is a Bagisto `Cart` row tagged with a `ucp_session` id. It's guest-scoped unless identity is linked, and it expires on a schedule. It must never merge into a logged-in shopper's storefront cart.
7. **Payments: delegate, don't capture.** For checkout completion, map UCP payment-handler tokens onto an existing Bagisto payment method, starting with Stripe. Don't build a new gateway. Card data never touches Bagisto.
8. **Auth and abuse.** Accept the spec's options: API key per agent (admin-issued, hashed, a DataGrid to revoke), with OAuth later. Rate-limit every endpoint. Require an `Idempotency-Key` on writes, stored with a TTL.
9. **Signing keys.** Store ES256/Ed25519 keys encrypted, rotate them from the admin panel, and publish public JWKs in the manifest. Sign outbound order webhooks with RFC 9421.
10. **Inventory.** Decide whether an agent checkout reserves stock. The simplest version Webkul will accept: no reservation, and stock is re-validated at `complete`, matching storefront behaviour.
11. **House rules (non-negotiable for review).** Contract + Model + Proxy + Repository for new tables. Repositories only, no direct model queries. Docblocks on every member and no comments in method bodies. `trans()` with all 23 locales. Code must work on MySQL 8, MariaDB 10.11 and PostgreSQL 16 using `db_grammar()`. No new Composer dependency without approval.

## Module architecture

```
                 ┌──────────── packages/Webkul/UCP (new) ────────────┐   ┌──── Existing Bagisto packages ────┐
┌───────────┐    │  Routes + middleware                               │   │  Webkul\Core                      │
│ AI agent  │───▶│  API key, rate limit, idempotency                  │   │  Channel, locale, currency        │
│ Gemini,   │    │        │                                           │   │                                   │
│ AI Mode   │    │        ▼                                           │   │  Webkul\Product                   │
└───────────┘    │  Capability controllers                            │   │  Repository, flat index, ES       │
      ▲          │  Manifest, Catalog, Cart, Checkout                 │   │                                   │
      │          │        │                                           │   │  Webkul\Checkout                  │
      │          │        ▼                                           │   │  Cart facade, ucp_sessions        │
      │          │  Capability contracts ─────────────────────────────┼──▶│                                   │
      │          │  Call Bagisto repositories only                    │   │  Pricing rules                    │
      │          │        │                                           │   │  Shipping, Tax, CartRule          │
      │          │        ▼                                           │   │                                   │
      │          │  Versioned transformers                            │   │  Sales + Payment                  │
      │          │  V20260408: DTO to UCP JSON                        │   │  Orders, events, Stripe           │
      │ signed   │                                                    │   │        │                          │
      │ webhooks │  Webhook signer (queued)  ◀────────────────────────┼───┼────────┘                          │
      └──────────┼─ RFC 9421, JWK in manifest                         │   │                                   │
                 └────────────────────────────────────────────────────┘   └───────────────────────────────────┘
```

Agent requests pass down through the UCP layers, and only the contracts touch Bagisto, through repositories. Order events flow back from Sales as signed webhooks. Because every Bagisto dependency sits behind a contract, the package could later move to `bagisto-api` or a standalone repo without a rewrite.

## Phased scope

Each phase is one PR that a reviewer can merge on its own. Don't open the next PR until the previous one is merged. That keeps you out of the Hold queue.

### Phase 0 — Agreement (no code)

- [ ] Comment on #11328 with the proposal (draft in the Merge strategy section) and ask to be assigned
- [ ] Get a maintainer answer on: core vs `bagisto-api`, package name, target branch `master`
- [ ] Rename the issue to "Add UCP (Universal Commerce Protocol) support" if maintainers agree

### Phase 1 — Package skeleton, manifest, Catalog (read-only)

- `packages/Webkul/UCP` scaffold: ServiceProvider, `Config/system.php` (per-channel enable plus capability toggles), ACL, routes, `ucp.php` config, 23-locale lang files
- `GET /.well-known/ucp`: spec-valid profile with `version`, `services`, empty `payment_handlers`, `capabilities: [catalog]`, and cache headers
- `POST /catalog/search`, `/catalog/lookup`, `/catalog/product`: simple and configurable products mapped to UCP product and variants, with price range, availability and images
- **Acceptance:** the profile and responses validate against the official UCP JSON schemas (`ucp-schema`) in a Pest test. Disabled channel returns 404. Pest feature tests run on all three DB engines. Pint is clean and the translations check passes.
- Target size: under 1,500 lines including tests

### Phase 2 — Cart

- `POST /carts`, `GET|PUT /carts/{id}`, `POST /carts/{id}/cancel` backed by the `Cart` facade with a `ucp_sessions` table (Contract, Model, Proxy, Repository)
- API-key auth for agents (admin DataGrid to issue and revoke), plus rate limiting and idempotency keys
- **Acceptance:** an agent cart never appears in or merges with a storefront session. Expired sessions are pruned by a scheduled command. Stock and price are re-validated on every read.

### Phase 3 — Checkout, Discount, Fulfillment

- `POST /checkout-sessions`, `GET|PUT /checkout-sessions/{id}`, `…/complete`, `…/cancel`
- Shipping options from enabled carriers, tax from `Tax`, coupons from `CartRule`
- One payment handler mapped to the existing Stripe package, through a `PaymentHandler` contract so others can plug in
- **Acceptance:** `complete` creates a normal Bagisto order (visible in admin, invoices work, emails fire). Business errors come back as `messages[]` with status 200. No card data is stored or echoed back.

### Phase 4 — Orders and signed webhooks

- `GET /orders/{id}` plus webhook delivery on order, shipment and refund events through queued jobs
- Signing keys generated and stored encrypted, public JWK published in the profile, RFC 9421 signatures
- **Acceptance:** retry with backoff, a failed-delivery log visible in admin, signature verified in tests

### Phase 5 — Identity linking (only if maintainers want it)

- OAuth 2.0 discovery and authorisation for linking a Bagisto customer to an agent. This probably needs `laravel/passport`, which is a new dependency, so it has to be approved first.
- Ship it as a separate decision. Phases 1–4 are useful without it.

## Merge strategy

The [contribution guide](https://github.com/bagisto/bagisto/blob/master/CONTRIBUTING.md) is explicit: comment and wait to be assigned, and agree the approach on the feature request first. Their warning: "A feature that arrives as a surprise pull request is far more likely to be turned down on direction than on code."

1. **Post the proposal on #11328** and ask to be assigned (draft below). Tag `devansh-webkul`, who triaged the issue.
2. **Wait for a yes on direction.** No answer in 2 weeks: ask once more in the thread, then start the standalone-package fallback (Risks section) so the work isn't wasted.
3. **Ship Phase 1 as a minimal, fully tested PR** to `master`. Use the PR template. List the gates you ran (`pint --test`, `pest`, `bagisto:translations:check`), and ask for the *Need Playwright Testing* label if you add admin config UI.
4. **Make review cheap.** Use Conventional Commit subjects (`feat(ucp): …`). Add a CHANGELOG entry. Don't touch other packages except to register providers, and don't commit compiled assets. Main merges need **2 approving reviews**, so reply to review comments within a day.
5. **Show it working.** Link a short screen recording of an agent calling `/.well-known/ucp` and `/catalog/search` against a seeded store, or a UCP playground run.
6. **Repeat per phase.** Each later PR references the issue and the merged previous phase.

**Draft comment for #11328**

```markdown
I'd like to take this on. Proposal, so we can agree the direction before code:

- A first-party package `packages/Webkul/UCP` (no third-party UCP dependency), disabled by default, enabled per channel under Configure.
- Spec revision 2026-04-08, with version-specific transformers so later spec dates are additive.
- Delivered in small PRs to `master`: (1) discovery manifest + read-only Catalog, (2) Cart, (3) Checkout + discount/fulfillment using existing payment methods (Stripe first), (4) Order + signed webhooks. Identity linking (OAuth) only if you want it.
- Follows AGENTS.md conventions (repositories, proxies, 23 locales, Pest on MySQL/MariaDB/PostgreSQL).

Two questions: should this live in core or in bagisto-api? And is there any internal or commercial UCP work I'd be duplicating?

Happy to adjust scope. Could you assign this to me if the approach works?
```

## Claude CLI setup

Bagisto is already set up for coding agents: `CLAUDE.md` imports [AGENTS.md](https://github.com/bagisto/bagisto/blob/master/AGENTS.md), which tells the agent to load Bagisto's own skills before writing code. Use those skills. They encode exactly what reviewers send back.

**One-time setup**

```bash
git clone git@github.com:rupeshstha/bagisto.git && cd bagisto
git remote add upstream https://github.com/bagisto/bagisto.git
git fetch upstream
git checkout feat/ucp-support          # this branch, created from upstream/master
composer install && cp .env.example .env && php artisan bagisto:install
npx skills add bagisto/agent-skills --skill '*' -a claude-code
git clone https://github.com/Universal-Commerce-Protocol/ucp ../ucp-spec   # schemas for local validation
```

**Keep your own notes out of the PR.** Put UCP-specific context in `CLAUDE.local.md` (add it to `.git/info/exclude` so it is never committed) rather than editing the shared `CLAUDE.md`:

```markdown
# UCP module — working notes
- Package: packages/Webkul/UCP, namespace Webkul\UCP. Target branch: master (2.5).
- Spec: ../ucp-spec, revision 2026-04-08. Validate every response against its JSON schemas.
- Current phase: 1 (manifest + catalog). Do NOT implement cart/checkout yet.
- Never add composer dependencies. Never edit other packages except provider registration.
- Money = integer minor units from the channel currency; no floats.
- Done = bagisto-change-verification passes (pint --test, pest on mysql + pgsql, translations:check).
```

**Prompt pattern per phase.** Run in plan mode first, review the plan, then let it build.

```text
Load bagisto-coding-standards, bagisto-package-development and bagisto-pest-testing.
Read ../ucp-spec for the discovery profile and catalog capability (2026-04-08).
Plan Phase 1 of the UCP package as described in CLAUDE.local.md: list every file you
will create, the config keys, the routes, and the Pest tests. Do not write code yet.
```

```text
Implement the approved plan. Write the Pest tests first and confirm they fail, then
implement until they pass. Finish by loading bagisto-change-verification and running
every gate. Report what you ran and anything you skipped.
```

```text
Load bagisto-code-review and review the diff against upstream/master as a strict Webkul
maintainer would. List anything they would send back, then fix it.
```

**Habits that keep the PR mergeable**

- Give each phase its own branch and session. Run `/clear` between phases so context from one doesn't leak into the next.
- Before each PR: `git fetch upstream && git rebase upstream/master`, then `git diff --stat upstream/master` and check that nothing outside `packages/Webkul/UCP`, `composer.json` autoload, `bootstrap/providers.php` and `config/concord.php` changed.
- Write the PR description yourself, using the agent's summary as a draft only.

## Risks and open questions

| Risk | Likelihood | Mitigation |
| --- | --- | --- |
| Webkul is building UCP internally or plans a paid extension | Medium. They sell extensions and have [written about UCP](https://webkul.com/blog/universal-commerce-protocol-ucp/), though no Bagisto module is announced. | Ask directly in the Phase 0 comment, before writing code |
| Maintainers want it in `bagisto-api`, not core | Medium | Keep the protocol layer framework-agnostic, so moving repos is a namespace change |
| PR parked on Hold, like Paytm #11190 | High for big PRs | Phase-sized PRs, agreement first, quick replies to review |
| Spec moves past 2026-04-08 | High. Breaking changes are announced 2 weeks ahead in [UCP Discussions](https://ucp.dev/versioning/). | Versioned transformers, plus a schema-validation test that fails loudly |
| Checkout or payment bugs touch real money | Medium | Delegate to the existing Stripe package. Disabled by default. Phase 3 behind its own toggle. |
| UCP rollout is US-first (Canada and Australia next, UK later) | Certain | Fine for open source, but NZ merchants can't use AI Mode checkout yet ([Search Engine Land](https://searchengineland.com/google-expands-universal-commerce-protocol-and-launches-new-agentic-shopping-tools-478113)) |

**Fallback if core says no or goes quiet:** publish the same code as `rupeshstha/bagisto-ucp`, a Composer package that registers into Bagisto the same way. Merchants can use it today, and Webkul can absorb it later. Nothing in the design above blocks this.

**Open questions for the maintainers**

- Core or `bagisto-api`?
- MCP binding as well as REST? The spec offers both for catalog and cart, and Bagisto already ships `laravel/ai`.
- Is identity linking (OAuth) in scope for core, given it likely needs a new dependency?

**Sources**

- [Issue #11328](https://github.com/bagisto/bagisto/issues/11328) · [CONTRIBUTING.md](https://github.com/bagisto/bagisto/blob/master/CONTRIBUTING.md) · [AGENTS.md](https://github.com/bagisto/bagisto/blob/master/AGENTS.md) · [PR #11190](https://github.com/bagisto/bagisto/pull/11190) · [PR #11318](https://github.com/bagisto/bagisto/pull/11318)
- [UCP spec overview 2026-04-08](http://ucp.dev/2026-04-08/specification/overview/) · [UCP versioning](https://ucp.dev/versioning/) · [UCP GitHub](https://github.com/Universal-Commerce-Protocol/ucp)
- [Bagisto agent skills](https://devdocs.bagisto.com/ai/agent-skills.html) · [Bagisto API docs](https://api-docs.bagisto.com/) · [tcgunel/ucp-laravel](https://github.com/tcgunel/ucp-laravel)
