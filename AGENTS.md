# Shopify app development

This app is scaffolded from a Shopify app template. See the README for framework-specific details.

Use the [Shopify AI Toolkit](https://shopify.dev/docs/apps/build/ai-toolkit) for all Shopify API and platform work. If missing, install it in the agent host per that page (or `npx skills add Shopify/shopify-ai-toolkit --list` for skill-compatible hosts) — do not add tooling to this repo.

## Critical operating constraints

- **Never push to GitHub and never deploy/push to Shopify** (`git push`, `npm run deploy`, `shopify app deploy`, `shopify app config link/use`) under any circumstances unless explicitly asked in that specific message.
- **Do not run `npm run dev` / `shopify app dev` or `npm run build`** unless explicitly asked to do so.
- Before implementing anything, read `docs/plan_V1.md` — it is the current implementation plan (V1) for this app and should be treated as the source of truth for scope, data model, and architecture decisions. Update it as decisions are confirmed or change during implementation.

## What this app is

A public Shopify App Store app ("whatsapp order confirmation") that, on order placement, puts the order's fulfillment order(s) on hold, sends the customer a WhatsApp template message asking them to confirm (YES) or cancel (NO), and releases the hold or cancels the order based on the reply (or auto-cancels after a timeout). See `docs/plan_V1.md` for full details: data model, webhook design, WhatsApp Cloud API integration, admin UI, billing, and milestones.

Right now the repo is the **unmodified Shopify App Template (React Router)** scaffold — none of the WhatsApp/hold/confirmation logic in the plan has been built yet. Treat `app/routes/app._index.tsx` and `app/routes/webhooks.app.uninstalled.tsx` as reference examples of the template's patterns, not as app-specific business logic.

## Commands

```shell
npm run dev            # shopify app dev — DO NOT RUN unless explicitly asked
npm run build           # react-router build — DO NOT RUN unless explicitly asked
npm run setup           # prisma generate && prisma migrate deploy (run after schema changes)
npm run lint            # eslint, cached
npm run typecheck       # react-router typegen && tsc --noEmit
npm run graphql-codegen # regenerate GraphQL types after editing admin.graphql queries
npm run deploy          # shopify app deploy — DO NOT RUN, ever, unless explicitly asked
```

There is no test runner configured in this repo.

Package manager is npm (there's also a `pnpm-workspace.yaml` scoping `extensions/*` as a workspace, but root scripts assume npm/the Shopify CLI).

## Architecture

- **Framework:** `@shopify/shopify-app-react-router` (a React Router port of the Shopify Remix app template) handles OAuth, session storage, webhook registration, and embedded-app boilerplate. The configured singleton lives in `app/shopify.server.ts` and exports `authenticate`, `unauthenticated`, `login`, `registerWebhooks`, `sessionStorage`, `apiVersion` — import from here, don't reconstruct the Shopify client elsewhere.
- **Routing:** file-based routes via `@react-router/fs-routes` (`app/routes.ts` just calls `flatRoutes()`). Route files live flat in `app/routes/` using dot-segment naming (e.g. `app.additional.tsx` → `/app/additional`, `webhooks.app.uninstalled.tsx` → `/webhooks/app/uninstalled`). Folder form (`app/routes/app._index/route.tsx`) is also valid for routes needing colocated files (e.g. `styles.module.css`).
- **Auth boundary:** `app/routes/app.tsx` is the layout for all embedded admin pages — its loader calls `authenticate.admin(request)` and wraps children in Polaris `AppProvider`/`s-app-nav`. Any new admin page route should nest under this (i.e. be named `app.<something>.tsx`).
- **Webhooks:** subscriptions are declared declaratively in `shopify.app.toml` under `[[webhooks.subscriptions]]` (uri + topics), not registered imperatively in code — this is the template's chosen pattern (see README "Webhooks: shop-specific webhook subscriptions aren't updated") and per the plan should be followed for the new `orders/create` and `fulfillment_orders/order_routing_complete` subscriptions too. Each webhook topic gets its own route file under `app/routes/webhooks.*.tsx` that calls `authenticate.webhook(request)`.
- **Session storage:** Prisma (`PrismaSessionStorage`) against the `Session` model in `prisma/schema.prisma`, currently SQLite (`dev.sqlite`) — the plan calls for migrating this datasource to PostgreSQL before production/multi-tenant use. `app/db.server.ts` exports the Prisma client singleton.
- **GraphQL:** Admin API queries/mutations are typed via `@shopify/api-codegen-preset`, configured in `.graphqlrc.ts`. Run `npm run graphql-codegen` after adding/changing inline `admin.graphql(...)` calls so generated types stay in sync.
- **Extensions workspace:** `extensions/` is currently empty (`.gitkeep` only) but is wired as an npm/pnpm workspace for future Shopify CLI-generated extensions (`npm run generate`).
- **App config:** `shopify.app.toml` defines scopes (`write_products,write_metaobjects,write_metaobject_definitions` — these are template defaults and will need updating per the plan's actual scope needs, e.g. `write_orders`, fulfillment hold scopes), webhook subscriptions, and the template metafield/metaobject examples (`demo_info`, `example`) — these are scaffold examples, not app-specific, and can be removed once real functionality replaces them.
- Embedded-app navigation must use Polaris/`react-router` `Link` or `redirect` from `authenticate.admin`, never raw `<a>` or React Router's own `redirect` — breaks session continuity inside the iFrame (see README "Gotchas").
