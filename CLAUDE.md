# Soft Scents — Project Rules

Fragrance e-commerce for Kenya. Medusa v2 backend + Next.js Starter Storefront, hosted on Medusa Cloud. Payments: Paystack (cards) and M-Pesa (Daraja STK Push). Roadmap and ownership: `plan.md`.

These rules apply to every contributor and every Claude session. They override personal defaults.

## Before Writing Code

1. Read the current sprint in `plan.md`. Work only on tasks assigned to your role (Dev A = `apps/backend`, Dev B = `apps/storefront`). Touching the other area needs a heads-up in the PR description.
2. Load the matching skill **before** coding, plus the reference files it points to:
   - Backend (modules, workflows, routes, links, subscribers, jobs): `medusa-dev:building-with-medusa`
   - Storefront: `medusa-dev:building-storefronts`
   - Admin widgets/routes: `medusa-dev:building-admin-dashboard-customizations`
   - Migrations: `medusa-dev:db-generate`, `medusa-dev:db-migrate`
3. Unsure about an API? Query Context7 (`.mcp.json`) or the Medusa docs. Never guess Medusa, Paystack, or Daraja syntax.
4. Work on a branch: `feature/<name>`, `fix/<name>`, `chore/<name>`. Never commit to `main`.

## Repository Layout

```
apps/backend      Medusa v2 (Dev A)
apps/storefront   Next.js Starter Storefront (Dev B)
plan.md           Sprint plan — tick tasks when merged
```

## Commands

Run from the app directory. A task is not done until build, lint and tests pass.

| Purpose | Backend | Storefront |
|---|---|---|
| Dev server | `npm run dev` | `npm run dev` |
| Build (type check) | `npm run build` | `npm run build` |
| Lint | `npx medusa lint` | `npm run lint` |
| Tests | `npm run test:unit`, `npm run test:integration:modules`, `npm run test:integration:http` | `npm test` |
| Migrations | `npx medusa db:generate <moduleName>` then `npx medusa db:migrate` | — |

<!-- TODO: confirm script names after the Sprint 0 scaffold -->

## Backend Rules (Medusa v2)

**Layering — never bypass:** Module (data + CRUD) → Workflow (business logic, mutations, rollback) → API route (HTTP + validation) → SDK client.

- Every mutation runs through a workflow. Routes never call module services to write data.
- Business rules, ownership and permission checks live in workflow steps, not routes.
- Every step that writes externally (DB, Paystack, Daraja, email) defines a compensation function.
- Modules stay CRUD-only and isolated. Cross-module relations use `defineLink` in `src/links/`; reads use `query.graph()`; filtering across linked modules uses `query.index()`. Run migrations after adding links.
- Module names are camelCase (`fragranceNotes`, `mpesa`), never dashed. Never add `.linkable()`.
- HTTP methods: `GET`, `POST`, `DELETE` only.
- Validate every request body/query with Zod from `@medusajs/framework/zod` (Zod v4 syntax) in `middlewares.ts`. Export both schema and inferred type; type routes as `MedusaRequest<T>` or `AuthenticatedMedusaRequest<T>`.
- Static imports only; no `await import()` in handlers.
- Workflow composition functions: plain `function`, no `async`/`await`, no conditionals (use `when()`), no data manipulation (use `transform()`).
- Subscribers and scheduled jobs stay thin: they only run a workflow.
- File layout: steps in `src/workflows/steps/`, workflows in `src/workflows/`, links in `src/links/`.

## Money & Pricing

- Medusa stores amounts as-is: KES 4,500 is `4500`. **Never** multiply or divide by 100 in Medusa code or the storefront.
- Paystack expects subunits (×100) and Daraja expects whole-shilling integers. Convert **only** inside the respective payment adapter, with a unit test for the conversion.
- Never trust an amount from the client. The server derives it from the cart.
- Currency `kes`, region Kenya, prices tax-inclusive (VAT 16%).

## Payments (Highest-Risk Code)

- Each provider is a payment module provider (`AbstractPaymentProvider`). Raw HTTP calls to Paystack/Daraja live in one adapter class per provider so they can be mocked and swapped.
- Webhooks/callbacks:
  - Paystack: verify `x-paystack-signature` (HMAC-SHA512 of the raw body) before anything else.
  - Daraja: callbacks are unsigned. Treat them as hints and confirm via the transaction status query before authorizing payment.
  - Handlers are idempotent (key on Paystack `reference` / Daraja `CheckoutRequestID`). Duplicate deliveries must be no-ops.
- A payment is only marked authorized/captured after server-side verification with the provider.
- Pending M-Pesa payments are reconciled by a scheduled job. Never leave an order in limbo.
- Every payment path needs tests for: success, failure, timeout, duplicate callback, bad signature.

## Storefront Rules (Next.js)

- All API calls go through the Medusa JS SDK (`sdk.store.*`, `sdk.client.fetch()` for custom routes). No raw `fetch()` to the backend, no `JSON.stringify` on SDK bodies.
- Data fetching stays in the starter's `src/lib/data/` layer. Components stay presentational; logic goes into hooks or server actions.
- Only `NEXT_PUBLIC_*` vars reach the browser (publishable key, Paystack public key). Secret keys never touch the storefront.
- Every async UI handles loading, error and empty states. Disable submit buttons while pending.
- Kenyan phone input normalized to `2547XXXXXXXX` / `2541XXXXXXXX` and validated on both client and server.
- Accessibility: WCAG 2.2 AA. Semantic HTML, labelled inputs, keyboard-usable checkout.
- Long Tailwind class strings go into named constants or variants, not inline in JSX.
- Admin UI: use `@medusajs/ui` components and semantic color classes only.

## Code Style (Both Apps)

- TypeScript strict. No `any`. Use `unknown` and narrow.
- Functions do one thing. Take at most 2 parameters; use an options object beyond that.
- Early returns over nested conditionals. Descriptive names; no abbreviations or magic numbers/strings (use named constants).
- No explanatory comments. Only `// TODO: <action>` or `// MODIFY: <reason>`.
- Delete dead code, unused imports and stale files in the same PR.
- Don't add a dependency when the platform, Medusa, or a few lines cover it. New dependencies are justified in the PR.

## Security & Privacy

- Never log PII: phone numbers, emails, addresses, names, payment payloads. Mask them (e.g. `2547****1234`).
- Secrets live in `.env` (gitignored) locally and in Medusa Cloud/hosting env vars. Keep `.env.example` updated with placeholder values.
- Rate-limit auth and payment-initiation routes. Admin routes require authentication middleware.
- Use parameterized queries only (the Medusa ORM covers this). No string-built SQL.

## Testing

- Write tests first for business logic: workflows, steps, payment adapters, money conversion.
- Mock Paystack and Daraja in unit tests. Integration tests use `@medusajs/test-utils` (`medusaIntegrationTestRunner`, `moduleIntegrationTestRunner`).
- Storefront: E2E (Playwright) for browse → cart → checkout with each payment method.
- Fixing a bug starts with a failing test that reproduces it.

## Git & PRs

- Conventional Commits: `feat(backend): add mpesa provider`, `fix(storefront): cart total rounding`.
- Small PRs. One task per PR. The other dev reviews. CI (build, lint, tests) must be green.
- PR description: what changed, how it was tested, any `plan.md` task ticked, any env var added.
- Schema changes include the generated migration in the same PR.

## Definition of Done

Build, lint and tests pass · no `any` · no PII in logs · migrations included · `.env.example` updated · deployed to staging · `plan.md` updated.
