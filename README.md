# marrylikeceo.com

## 1. Overview

This repository is the marketing/landing site for **Marry Like a CEO**, a coaching brand run by Ariel Yankelewitz. It is a single-page site (`src/app/page.tsx`) built with Next.js: hero, brand-strategy sections, an "Experience" (monthly live event) section, a book announcement, community links, FAQ, and a final call-to-action — with three email-capture forms that write to a Notion database.

The site is deployed on **Vercel**, deploying from this Git repository. The production domain is `marrylikeceo.com` (DNS registered at Squarespace, pointed at Vercel — see [DNS](#7-dns)).

## 2. Stack

- **Next.js 16.2.3** (App Router), **React 19.2.4** / **react-dom 19.2.4**, **TypeScript 5** (strict mode)
- **Tailwind CSS v4** (via `@tailwindcss/postcss`)
- **motion** (`motion/react`, formerly Framer Motion) for animation
- **react-hook-form** + **@hookform/resolvers** for form state/validation wiring
- **zod v4** for schema validation (shared between client and server)
- **@notionhq/client v5** for the email-capture integration
- **clsx** + **tailwind-merge** (combined in `src/lib/utils.ts` as `cn()`) for classname merging
- **lucide-react** for icons
- **@vercel/analytics** and **@vercel/speed-insights** — both mounted in `src/app/layout.tsx`
- **@playwright/test** — installed as a dev dependency, but **unused**: confirmed by searching the full git history across every branch, there has never been a `playwright.config.*` or a `*.spec.*`/`*.e2e.*` file in this repo. `ci.yml` has only ever had one version (added in commit `3564f32`, never modified since) and it has never included an e2e or Playwright step, despite CLAUDE.md describing an "e2e-preview" CI check. `@playwright/test` and `playwright` were both added in the very first scaffold commit (`41fde09`, "initialize Next.js 16 project scaffold with full stack setup") — this looks like leftover starter-template tooling that was never wired up, not a removed feature. It's safe to either build real e2e tests on top of it or remove it; neither choice affects anything else in the app.

**Node version:** the CI workflow pins `node-version: 20` (see `.github/workflows/ci.yml`). There is no `.nvmrc` and no `engines` field in `package.json`. Use Node 20 to match CI. (Local development machine used to write this README ran Node 25.9.0 without issue, but match CI for anything you'll deploy.)

**Package manager:** **npm** — the repo has a committed `package-lock.json` and CI runs `npm ci`. Don't introduce a yarn/pnpm lockfile.

## 3. Local development

```bash
git clone <repo-url>
cd marrylikeceo.com
npm install
```

Create a `.env.local` file in the repo root (see [Environment variables](#4-environment-variables) for values):

```bash
cp .env.example .env.local
# then fill in NOTION_TOKEN and NOTION_SUBSCRIBERS_DB_ID
```

Run the dev server:

```bash
npm run dev
```

This starts Next.js on **http://localhost:3000** (Next.js default port; no custom port is configured anywhere in this repo).

Other scripts (from `package.json`):

```bash
npm run build       # production build (next build)
npm run start        # serve the production build (next start)
npm run lint          # eslint
npm run typecheck   # tsc --noEmit
```

Without a valid `.env.local`, `npm run dev` and `npm run build` will **crash at startup** — see the next section.

## 4. Environment variables

All server env vars are parsed and validated at first access through a single Zod schema in `src/env/server.ts` (`serverEnvSchema.parse(process.env)`). If a required variable is missing or fails validation, this **throws immediately** the first time any server code touches `env` — for `npm run dev` and `npm run build` that means the app/build fails to start, not a graceful in-app error.

| Variable | Purpose | Where to obtain it | Required? | Missing behavior |
|---|---|---|---|---|
| `NOTION_TOKEN` | Auth token for the `@notionhq/client` integration that writes new subscribers to Notion (`src/lib/notion.ts`). | Create a Notion internal integration at notion.so/my-integrations, copy its "Internal Integration Secret". | **Required** | `getEnv()` throws (Zod `.min(1)` fails) → server crashes at startup / on first request. |
| `NOTION_SUBSCRIBERS_DB_ID` | The Notion database ID that subscriber pages are created in. | The database ID segment of the Notion database's URL. | **Required** | Same as above — startup crash. |
| `ALLOWED_ORIGINS` | Comma-separated allowlist of request `Origin` headers accepted by `POST /api/subscribe` (`src/lib/validation.ts`). Requests with an `Origin` header not in this list get a `403`. | Set to your deployed domain(s). | Optional | Defaults to `"https://marrylikeceo.com,https://www.marrylikeceo.com,https://marrylikeceocom.vercel.app"` if unset — no crash. |
| `NODE_ENV` | Standard Next.js environment flag. | Set by the runtime (Next.js/Vercel sets this automatically; you generally don't need to set it by hand). | Optional | Defaults to `"production"` per the Zod schema if somehow unset — no crash. |
| `RESEND_API_KEY` | Declared optional in the env schema (`z.string().optional()`), but **not used anywhere in the codebase** — confirmed by searching every commit on every branch for a Resend import or usage; only the declaration itself has ever existed, present since the very first project-scaffold commit (`41fde09`). It's leftover starter-template scaffolding for a transactional-email feature that was never built, not an active integration. | — | Optional, currently unused | No effect either way. |

The project's `CLAUDE.md` also lists `NEXT_PUBLIC_SENTRY_DSN` as an optional env var. Confirmed this is inaccurate/aspirational documentation: **no Sentry package is installed** (`package.json` has no `@sentry/*` dependency) and no code references it, on any branch, at any point in history. It was documented as "Optional" in CLAUDE.md's very first commit (`1dbe9d7`) but never implemented. Worth telling whoever owns CLAUDE.md, since it currently documents a non-existent feature as available.

`.env.example` (copy to `.env.local` for local dev):

```bash
# Required — Notion integration
NOTION_TOKEN=secret_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
NOTION_SUBSCRIBERS_DB_ID=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Optional — defaults shown are what the code falls back to
ALLOWED_ORIGINS=http://localhost:3000
NODE_ENV=development

# Optional, currently unused in code
RESEND_API_KEY=
```

## 5. Notion integration

Email capture is a single path used by all three forms on the site:

```
EmailCapture component (source: "hero" | "experience" | "book" | "final")
  → react-hook-form + Zod client-side validation (src/lib/schemas.ts)
  → POST /api/subscribe (src/app/api/subscribe/route.ts)
  → withValidation: rate limit (5 requests / 10 min per IP, in-memory) + origin check + honeypot + server-side Zod (src/lib/validation.ts)
  → addSubscriber() (src/lib/notion.ts) → notion.pages.create() into the Subscribers database
  → 200 JSON { success: true, message } on success
```

**Where forms actually appear today:** only two of the four valid `source` values are wired into the UI — `EmailCapture source="book"` in `src/components/sections/Book.tsx`, and `EmailCapture source="final"` in `src/components/sections/FinalCTA.tsx`. `"hero"` and `"experience"` are valid per the Zod schema and the Notion source map but no current component passes them — the Hero section uses a plain WhatsApp link button instead of an email form (see `src/components/sections/Hero.tsx`).

**Notion page properties written by `addSubscriber()`** (`src/lib/notion.ts`):

| Notion property | Type | Value written |
|---|---|---|
| `Name` | title | The submitted `firstName`, or the email address if no name was collected/submitted |
| `Email` | email | The submitted email |
| `Source` | select | One of `"Community"`, `"Experience"`, `"Book"` — mapped from the form's `source` field, see below |

**Source-value mapping** — the form's `source` field (from `subscribeSchema` in `src/lib/schemas.ts`) is translated to a Notion `Source` select option via a lookup table in `src/lib/notion.ts`:

| Form `source` | Notion `Source` select value |
|---|---|
| `hero` | `Community` |
| `experience` | `Experience` |
| `book` | `Book` |
| `final` | `Experience` |

Note that `final` maps to `Experience`, not to a `Final`/`Community` option — this is intentional in the current code (any unrecognized `source` value falls back to `"Community"`, the `DEFAULT_SOURCE` constant), but it means the FinalCTA form (bottom of page) and the Experience section conceptually both land under the same `Experience` select option in Notion. If this is wrong, it's a one-line change in the `sourceMap` object.

**Notion-side setup required before this works, even with correct credentials:**

1. Create a Notion database with a `Name` (title), `Email` (email), and `Source` (select) property. You do **not** need to pre-create the `Community`, `Experience`, `Book` select options by hand: per Notion's official API reference, "If the select [property] doesn't have an option by that name yet, then the name is added to the [database] schema if the connection also has write access to the parent [database]" — and write access is already required for `pages.create()` to work at all, so the first successful subscriber write for each source will create its option automatically. Pre-creating them is fine too (e.g. if you want to control their colors), just not required.
2. Create an internal integration at notion.so/my-integrations, generate its secret, and use it as `NOTION_TOKEN`.
3. **Explicitly connect the integration to the target database from inside Notion** — open the database, use the `•••` menu → "Connections" (or "Add connections") → select your integration. **This step is easy to miss and is not optional**: without it, API calls with a perfectly valid `NOTION_TOKEN` and correct `NOTION_SUBSCRIBERS_DB_ID` will fail (Notion returns an "object not found" style error for the integration, because from the integration's point of view the database doesn't exist until it's shared with it).
4. Copy the database ID out of its URL into `NOTION_SUBSCRIBERS_DB_ID`.

## 6. Deployment

Deployment is via **Vercel**, connected to this Git repository. Production deploys from **`main`** — confirmed via `git remote show origin` (`HEAD branch: main`, GitHub's configured default branch) and by the absence of any `vercel.json` or `.vercel/` directory in the repo that would override Vercel's default (which is to deploy Production from the connected repo's default branch); this also matches CI (`ci.yml` triggers on push/PR to `main`). Vercel also builds preview deployments per branch/PR (the default `ALLOWED_ORIGINS` value includes `https://marrylikeceocom.vercel.app`, suggesting a Vercel preview/production alias is already assumed by the code).

**Environment variables to set in the Vercel dashboard** (Project Settings → Environment Variables), for Production (and Preview if you want previews to be able to write to Notion):

- `NOTION_TOKEN` (required)
- `NOTION_SUBSCRIBERS_DB_ID` (required)
- `ALLOWED_ORIGINS` (optional — only needed if the default value in `src/env/server.ts` doesn't cover your actual domain(s))
- `NODE_ENV` — Vercel sets this automatically for its environments; you shouldn't need to set it manually.

**CI** (`.github/workflows/ci.yml`, GitHub Actions, runs on push/PR to `main`): four jobs — `preflight` (checkout + `npm ci`), `lint` (`npm run lint`), `typecheck` (`npx tsc --noEmit`), and `build` (`npm run build`, with `NOTION_TOKEN`/`NOTION_SUBSCRIBERS_DB_ID` set to the literal string `placeholder` and `ALLOWED_ORIGINS=http://localhost:3000` — just enough to satisfy the required-env-var check at build time, not real credentials). `lint` and `typecheck` both depend on `preflight`; `build` depends on both `lint` and `typecheck`. This CI workflow does not run tests (no `npm test`/Playwright step is defined here, despite `@playwright/test` being a dev dependency — see the note under [Stack](#2-stack)).

## 7. DNS

Domain is registered at **Squarespace** (Squarespace Domains), not Google Domains — DNS records must be added under **Custom Records** in the Squarespace domain management panel, not under any "Google Records" section.

| Type | Host | Value |
|---|---|---|
| A | `@` | `76.76.21.21` |
| CNAME | `www` | `cname.vercel-dns.com` |

These point the apex domain and `www` subdomain at Vercel, and are the values this domain was set up with. **Important correction from Vercel's current documentation (checked against `vercel.com/docs` and `vercel.com/kb` directly):** Vercel no longer issues one universal A/CNAME pair to every project. `76.76.21.21` is still cited as Vercel's "general-purpose" apex value, but Vercel's own docs now say "the correct value for your project is whatever your domain card shows," and note that newer projects can be issued a different anycast address (their example: `216.198.79.1`). Same for the CNAME: newer projects get a **per-project** target like `d1d4fc829fe7bc7c.vercel-dns-017.com` instead of the old shared `cname.vercel-dns.com`.

Practically: since `marrylikeceo.com` is already live and working, whatever is currently set in Squarespace's Custom Records is the correct, working configuration for this project right now — don't change it speculatively. The two values above are what to use as a starting point only if you're re-adding the domain from scratch (e.g. after an accidental deletion) or setting up a new domain on this same Vercel project. In either of those cases, **read the exact A/CNAME values off this project's own domain card** (Vercel dashboard → Project → Settings → Domains → click the domain) rather than trusting the table above, since Vercel may hand this specific project a different pair than the general-purpose one.

## 8. Project structure

```
src/
  app/
    page.tsx           # the entire homepage — imports and orders every section
    layout.tsx          # root layout: fonts, Navbar, Footer, ScrollProgress, EntitySchema, Analytics, SpeedInsights
    globals.css         # Tailwind import + CSS custom properties (brand colors, easing) + a couple of global utility classes
    error.tsx            # client error boundary
    not-found.tsx        # 404 page
    loading.tsx           # route-level loading spinner
    robots.ts              # robots.txt generator — explicitly allows AI crawlers (GPTBot, ClaudeBot, PerplexityBot, etc.)
    sitemap.ts              # sitemap.xml generator — single URL (the homepage)
    apple-icon.png, icon.png  # favicons
    api/subscribe/route.ts     # the only API route — the Notion email-capture endpoint

  components/
    sections/            # one file per homepage section (Hero, StrategyGap, TheSystem, Story, QuoteBand,
                          # Experience, Book, Community, FAQ, FinalCTA) — page.tsx just composes these in order
    layout/               # Navbar, Footer, ScrollProgress — used once each, in layout.tsx
    ui/                    # reusable primitives: Button, Section, Eyebrow, FormField, EmailCapture,
                            # DuotoneImage, Accordion, Logo, SystemIcon
    SEO/                    # EntitySchema (JSON-LD Organization/Person/WebSite/Book graph), FAQSchema (JSON-LD FAQPage)

  constants/
    copy.ts               # ALL site copy lives here — headlines, body text, button labels, nav links,
                           # social URLs, footer text. To change any text on the site, edit this file,
                           # not the component files.

  lib/
    schemas.ts             # Zod schema for the subscribe form (shared client/server)
    validation.ts           # withValidation() wrapper: rate limiting, origin check, honeypot, Zod parsing
    notion.ts                # Notion client + addSubscriber() + the source → Notion-select-option map
    faq-data.ts               # FAQ question/answer content (kept separate from copy.ts)
    utils.ts                   # cn() classname helper

  env/
    server.ts               # Zod-validated process.env accessor — the single source of truth for server env vars

  types/
    index.ts                  # shared TS interfaces (SubscribePayload, ApiResponse, SectionProps)

public/
  images/                     # site photography (each hero-style photo has a color + a "-bw" black/white
                               # variant, used by DuotoneImage for the scroll-triggered color reveal effect)
  llms.txt                     # plain-text summary file aimed at LLM crawlers (GEO — see robots.ts note)

.github/workflows/ci.yml         # CI: preflight / lint / typecheck / build
```

## 9. Known issues and gotchas

**`#retreat` hash redirect (`src/components/sections/Experience.tsx`, lines 38–42).** On mount, the Experience section checks `window.location.hash === "#retreat"` and, if true, calls `history.replaceState(null, "", "#experience")`. This exists because the section was originally called "Retreat" and was renamed to "Experience" (see git history — commit `997c947`, "rename Retreat to Experience"). Any old marketing links, bookmarks, or external references still pointing at `#retreat` land on the right section but get their URL silently rewritten to `#experience` so the address bar matches what's actually on screen. If you ever rename this section again, either keep this pattern or intentionally remove old redirects — don't delete it thinking it's dead code.

**Manual `object-position` crops on hero-style photography** (`object-[50%_18%]` in `Hero.tsx`, `object-[50%_20%]` in `Experience.tsx`, `object-[28%_30%]` in `QuoteBand.tsx`). These are hand-tuned per-image crop points (as percentages, `object-[x%_y%]`), not a generic `object-center`. They exist because each photo needs its subject's face kept in frame across the very wide range of aspect ratios these hero-style images render at (full-bleed panel on desktop, cropped-height band on mobile). If you swap in a new photo for one of these slots, you'll likely need to re-tune its `object-[…]` value by eye — don't assume `object-center` will look right.

**`DuotoneImage` component** (`src/components/ui/DuotoneImage.tsx`) renders a color photo and a black-and-white version of the same photo stacked on top of each other, then fades the B&W layer out on load or on scroll-into-view (`revealOnLoad` vs. `whileInView`, both using the same fade mechanics). This means every "duotone" photo on the site needs **two exported image files** (a normal version and a `-bw` version, matching the naming convention already used in `public/images/`, e.g. `ariel-hero.jpg` / `ariel-hero-bw.jpg`). If you add a new photo section using this component and only supply one image, it will silently render only the color layer with no reveal animation error — nothing will crash, it just won't have the intended effect.

**Origin check is opt-in, not opt-out** (`src/lib/validation.ts`, `checkOrigin`). If a request has **no** `Origin` header at all, `checkOrigin` returns `true` (allowed) — the check only blocks requests that *do* send an `Origin` header and it's not in `ALLOWED_ORIGINS`. This is normal for server-to-server / non-browser requests (which often omit `Origin`), but it means the origin allowlist is not a complete access-control mechanism on its own — the rate limiter and honeypot field are the other two legs of this endpoint's abuse protection.

**In-memory rate limiting** (`src/lib/validation.ts`, `rateLimitMap`). The 5-requests-per-10-minutes limiter is a plain in-memory `Map` inside the route module, keyed by the `x-forwarded-for` IP. On serverless platforms (Vercel functions), this resets whenever a fresh function instance is spun up and is not shared across concurrently running instances — it throttles casual abuse from a single warm instance, not a hardened distributed rate limit. Worth knowing if you're debugging why the rate limit doesn't seem to trigger consistently under load testing.

**`"final"` source maps to the `"Experience"` Notion select value**, not a distinct `"Final"` option — see the mapping table in [section 5](#5-notion-integration). This is easy to misread as a bug when looking at Notion data (subscribers from the bottom-of-page FinalCTA form show up tagged `Experience`, same as subscribers from the actual Experience section content) — it is what the code currently does, not a data-import artifact.

**`RESEND_API_KEY` and `NEXT_PUBLIC_SENTRY_DSN` are documented as env vars but unused in code** — see the flags in [section 4](#4-environment-variables). Don't spend time looking for where they're consumed; as of this snapshot, nothing reads them.
