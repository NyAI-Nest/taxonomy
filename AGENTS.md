# AGENTS.md — Taxonomy

## What this is

**Taxonomy** is shadcn's open-source demo/starter of a full SaaS-style app built on the Next.js 13 App Router (canary), aimed at developers learning server-component patterns. It has public marketing pages, an MDX docs site and blog, and an authenticated dashboard where users create and edit posts in an Editor.js editor. The free plan is capped at 3 posts; a Stripe PRO subscription unlocks unlimited posts. Stack: TypeScript, Next.js 13 App Router, React 18, Prisma + MySQL (PlanetScale), NextAuth.js (GitHub + email via Postmark), Stripe, Contentlayer/MDX, Tailwind CSS, Radix UI/shadcn/ui, Zod, Editor.js.

The README marks the project as **archived**: it intentionally uses early App Router patterns and is not recommended for production. Work within the existing patterns; do not modernize it.

## Setup, run, build

Requires Node v16.18.0 (see `.nvmrc`) and pnpm.

```sh
nvm use
pnpm install                 # postinstall runs prisma generate
cp .env.example .env.local   # then fill in the values
pnpm dev                     # contentlayer dev + next dev, concurrently
```

Env vars are validated by `env.mjs`; the app fails without them. Groups: app (`NEXT_PUBLIC_APP_URL`); NextAuth/GitHub (`NEXTAUTH_URL`, `NEXTAUTH_SECRET`, `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET`, `GITHUB_ACCESS_TOKEN`); database (`DATABASE_URL`, MySQL); Postmark (`SMTP_FROM`, `POSTMARK_API_TOKEN`, `POSTMARK_SIGN_IN_TEMPLATE`, `POSTMARK_ACTIVATION_TEMPLATE`); Stripe (`STRIPE_API_KEY`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_PRO_MONTHLY_PLAN_ID`).

Other scripts from `package.json`:

```sh
pnpm build     # contentlayer build && next build
pnpm start     # next start
pnpm lint      # next lint
pnpm preview   # next build && next start
```

There is no test suite. Check changes with `pnpm lint` and a clean `pnpm build`, then exercise the flow manually with `pnpm dev`. Dashboard/editor flows need a reachable MySQL at `DATABASE_URL`; auth and billing flows need real NextAuth/Postmark/Stripe keys.

## Layout

- `app/` — App Router tree in route groups: `(marketing)` (landing, `blog/`, `pricing/`, `[...slug]` catch-all for MDX pages), `(auth)` (`login/`, `register/`), `(dashboard)` (`dashboard/` posts list, `billing/`, `settings/`, with `loading.tsx` files), `(editor)` (`editor/[postId]/`), `(docs)` (`docs/[[...slug]]/`, `guides/[...slug]/`).
- `app/api/` — route handlers: `posts/` (list/create; POST enforces the 3-post free-plan limit), `posts/[postId]/` (PATCH/DELETE with ownership check), `users/[userId]/` (name update), `users/stripe/` (checkout/portal session), `webhooks/stripe/` (records subscription state), `og/` (OG image generation), `auth/[...nextauth]/_route.ts` (inactive copy, see rules).
- `pages/api/auth/[...nextauth].ts` — the NextAuth handler actually served (pages router, kept as a workaround for a Prisma packaging issue).
- `middleware.ts` — auth gating: unauthenticated users redirected to `/login` (with a `from` param), signed-in users redirected away from auth pages; matcher covers `/dashboard/:path*`, `/editor/:path*`, `/login`, `/register`.
- `lib/` — core logic: `auth.ts` (NextAuth config), `db.ts` (cached Prisma client exported as `db`), `session.ts` (`getCurrentUser()`), `stripe.ts`, `subscription.ts` (plan lookup), `exceptions.ts` (`RequiresProPlanError`), `utils.ts` (`cn`, `formatDate`, `absoluteUrl`), `toc.ts`, `validations/` (Zod schemas: `auth.ts`, `og.ts`, `post.ts`, `user.ts`).
- `prisma/` — `schema.prisma` (MySQL: `User` with Stripe columns, `Post` with JSON content and `published` flag, `Account`, `Session`, `VerificationToken`) and `migrations/`.
- `contentlayer.config.js` — MDX document types `Doc`, `Guide`, `Post`, `Author`, `Page`; rehype/remark plugins for code highlighting and heading anchors.
- `content/` — MDX sources: `docs/`, `guides/`, `blog/`, `authors/`, `pages/` (privacy, terms).
- `config/` — static config: `site.ts`, `marketing.ts`, `docs.ts`, `dashboard.ts` (nav structures), `subscriptions.ts` (free/pro plan definitions).
- `components/` — app components (`editor.tsx`, `post-operations.tsx`, `billing-form.tsx`, `nav.tsx`, `shell.tsx`, ...) plus `components/ui/` with the shadcn/ui Radix primitives.
- `env.mjs` — typed env validation via `@t3-oss/env-nextjs` + Zod.
- `styles/` — `globals.css`, `editor.css`, `mdx.css`.
- `types/` — `index.d.ts` (shared types) and `next-auth.d.ts` (session/JWT module augmentation).
- `hooks/` — `use-lock-body.ts`, `use-mounted.ts`.
- Generated, never edit: `.contentlayer/` (Contentlayer output, mapped in `tsconfig.json`).

## Conventions

- Path alias `@/*` maps to the repo root.
- Server components by default: async components fetch via `db` and check auth via `getCurrentUser()`; on missing user, redirect to sign-in via `authOptions?.pages?.signIn` (see `app/(dashboard)/dashboard/page.tsx`, `app/(editor)/editor/[postId]/page.tsx`).
- Client components opt in with the `use client` directive (editor, forms, navs).
- Route handlers share one pattern: try/catch; Zod-parse route params and body; respond `403` for unauthorized/forbidden, `402` for `RequiresProPlanError`, `422` with `error.issues` on `ZodError`, else `500`; plain `new Response(...)` bodies. Ownership checks query by both resource id and `authorId`/user id.
- Validate payloads with schemas from `lib/validations/`; extend those files instead of inlining new schemas.
- UI: compose from `components/ui/` primitives, merge classes with `cn()` from `lib/utils.ts`, variants via class-variance-authority. Tailwind tokens are CSS variables in `styles/globals.css`; dark mode is class-based (`tailwind.config.js`).
- Nav and plan data are config-driven: edit `config/*.ts` (typed in `types/index.d.ts`), not hardcoded JSX.
- Prisma models map to snake_case tables via `@map`/`@@map`; `Post.content` stores Editor.js blocks as JSON.
- MDX frontmatter must match the document types in `contentlayer.config.js` (`title`, `description`, `published`, `date`, `authors`, ...). Blog/docs OG images go through `/api/og` because `opengraph-image.tsx` does not work in catch-all routes (see README known issues).
- Formatting and commits: Prettier (`prettier.config.js`, sorted imports, tailwind class order), ESLint (`next/core-web-vitals`, `prettier`, tailwindcss plugin); husky runs `pretty-quick --staged` on pre-commit and commitlint (conventional commits, `.commitlintrc.json`) on commit-msg.

## Rules for agents

- Archived demo on a Next 13 canary: do not upgrade Next/React or other dependencies, do not migrate to current App Router idioms, do not enable full `strict` TypeScript. Keep changes consistent with what is already there.
- Do not touch:
  - `pages/api/auth/[...nextauth].ts` and `app/api/auth/[...nextauth]/_route.ts` — the pages-router handler is the live one; the underscore-prefixed app file is deliberately not treated as a route.
  - The free-plan limit check in `app/api/posts/route.ts` and the auth redirects in `middleware.ts` and the dashboard/editor layouts and pages.
  - Stripe webhook signature verification in `app/api/webhooks/stripe/route.ts`.
  - Existing folders under `prisma/migrations/` — add a new migration instead.
  - `.contentlayer/` generated output.
- Treat `components/ui/` as the shared primitive library: reuse it; if a primitive must change, keep its shadcn/ui API intact.
- New env var: add it to `.env.example` and to `env.mjs` (server or client section plus `runtimeEnv`), or the build fails.
- Schema changes: edit `prisma/schema.prisma`, create a migration with the Prisma CLI (e.g. `pnpm prisma migrate dev --name <change>`), and regenerate the client (`pnpm prisma generate`).
- Never commit secrets; `.env.local` stays local.
- Before finishing: `pnpm lint` and `pnpm build` must pass; for auth, billing, or post CRUD changes, run the flow manually with `pnpm dev`.

## Product docs

The product's user-facing documentation is the MDX content itself: `content/docs/` (docs site), `content/guides/`, `content/blog/`, `content/pages/` (legal), `content/authors/`; their navigation lives in `config/docs.ts` and `config/marketing.ts`. There are no `_ideas`, `_features`, `_designs`, `_adr`, `_plans`, `_tasks`, or `_releases` folders in this repo; the only engineering docs are `README.md` and this file.
