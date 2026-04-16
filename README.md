# DPO Central

An open-source privacy management platform for Data Protection Officers, privacy professionals, and in-house teams who need to run a GDPR-grade privacy program without buying into a six-figure OneTrust contract.

Built with Next.js 16, tRPC, Prisma, and PostgreSQL. Runs on Vercel, Railway, your own box, or anywhere Next.js runs.

## What you get

DPO Central ships with the modules every privacy program actually needs:

- **Data Inventory** — assets, data elements, processing activities, auto-generated data flow maps
- **ROPA** (Article 30 Records of Processing) — per-activity with linked assets, transfers, and data categories
- **DSAR portal** — a public intake page per organization, rate-limited, with consent capture, audit trail, and auto-redaction after retention
- **Incident register** — breach tracking with timeline, tasks, notifications, and a GDPR Article 33 export
- **Assessment engine** — LIA and custom assessments out of the box. DPIA templates are available as a premium add-on.
- **Vendor management** — processors, DPAs, contracts, risk tiers, and a vendor questionnaire workflow
- **International transfers** — SCCs, TIAs, adequacy lookups
- **AI governance** — EU AI Act-aligned AI system register with risk tier and human-oversight metadata
- **Regulatory landscape** — jurisdiction-aware compliance dashboard
- **Eight PDF report exports** — ROPA, Data Inventory, DSAR Performance, Breach Register, Vendor Register, Assessment Portfolio, Regulatory Landscape, and individual Assessment reports. All with a Graphviz-rendered data flow diagram embedded as a page.

## Quick start

### Prerequisites

- Node.js 20 or newer
- A PostgreSQL database (local Postgres, Neon, Supabase, Railway — anything)
- A [Resend](https://resend.com) account for magic-link sign-in (free tier is fine)
- Optional: Google OAuth credentials for Google sign-in

### Install

```bash
git clone https://github.com/RINDOGATAN/dpo.git
cd dpo
npm install
cp .env.example .env.local
```

Edit `.env.local` and fill in:

```bash
DATABASE_URL="postgresql://..."
NEXTAUTH_URL="http://localhost:3001"
NEXTAUTH_SECRET="<generate with: openssl rand -base64 32>"
RESEND_API_KEY="re_..."
EMAIL_FROM="noreply@yourdomain.com"

# Optional — Google OAuth
GOOGLE_CLIENT_ID=""
GOOGLE_CLIENT_SECRET=""
```

### Set up the database

```bash
npx prisma db push        # creates tables
npm run db:seed            # seeds system templates (LIA, Custom, DPIA placeholders)
npm run db:seed-vendors    # seeds the public SaaS vendor catalog (~1000 entries)
```

### Run it

```bash
npm run dev
# open http://localhost:3001
```

In development, sign in at `/sign-in` with any email. Email magic links go via Resend if configured, otherwise via the dev credentials provider.

### Seed demo data (optional)

Five realistic industry verticals ship as demo scenarios:

```bash
npm run db:seed-demo-saas          # CloudForge Labs (SaaS)
npm run db:seed-demo-healthcare    # Nordic Health Group (healthcare)
npm run db:seed-demo-fintech       # Vega Financial (fintech)
npm run db:seed-demo-media         # Herald Digital Media (publisher)
npm run db:seed-demo-proserv       # Alder & Stone Consulting (professional services)
```

Each one creates a fully populated organization so you can evaluate the UI end-to-end without having to fill in forms yourself.

## Architecture

```
prisma/schema.prisma              # ~56 models
src/server/routers/privacy/       # tRPC routers (multi-tenant, org-scoped)
src/server/services/export/       # 8 PDF report components
src/config/                       # Feature flags, AI Act, jurisdictions, vendor catalog
src/app/(dashboard)/privacy/      # Dashboard pages
src/app/(public)/                 # Public landing + docs
src/app/dsar/                     # Public DSAR intake portal
src/app/api/export/               # PDF export routes
```

**Multi-tenancy**: all models are scoped by `organizationId`. tRPC procedures enforce this via the `organizationProcedure` middleware. Adding a new module means using `organizationProcedure` and you automatically get tenant isolation.

**Auth**: NextAuth v4, JWT strategy. Google OAuth + Email Magic Link out of the box. Custom `sendVerificationRequest` uses Resend's HTTP API instead of SMTP.

**PDF pipeline**: `@react-pdf/renderer` for layout, `@hpcc-js/wasm-graphviz` + `@resvg/resvg-js` for the data flow diagrams. The flow diagram is pre-rendered to PNG before the React-PDF tree runs, then embedded as an `<Image>`.

## Open core

DPO Central follows an open-core model:

- **Core** (this repo, AGPL-3.0): Data Inventory, ROPA, DSAR, Incidents, LIA and custom assessments, Vendor management, AI governance, public portal, all 8 PDF reports.
- **Premium** (separate commercial packages): DPIA templates with scoring, enhanced security module with advanced RBAC. The core is designed so that premium packages are dynamically imported if installed — see `src/lib/skills/loader.ts` and `src/lib/security/loader.ts`. The app runs fully without them, with open-source defaults.

If you want premium, check the project website. If you don't, you have everything you need to run a complete privacy program for free.

## Deployment

### Vercel

1. Push the repo to your GitHub account
2. Import the project into Vercel
3. Set the environment variables from `.env.local` in Vercel's Project Settings → Environment Variables
4. Set up your Postgres connection (Neon, Supabase, Railway, Vercel Postgres — any)
5. Deploy

The included `vercel.json` is a neutral default (empty `crons`). If you want to run the DSAR auto-redaction cron, add it to `vercel.json` pointing at `/api/cron/dsar-redaction`.

### Self-hosted

```bash
npm run build
npm start
```

The app listens on port 3001 by default. Put it behind a reverse proxy (nginx, Caddy, Traefik) and you're done.

### Docker

A basic `docker-compose.yml` is included that spins up the app + a local Postgres. It's a starting point, not a hardened production recipe.

## Contributing

Pull requests welcome. For anything substantial, open an issue first so we can discuss the approach before you spend time on it.

The codebase has a few conventions worth knowing:

- **List queries that feed a picker must paginate** via `useInfiniteQuery` with auto-fetch. Silently truncating at the default `max(100)` server cap is the most common bug class in this codebase — check existing pickers before adding a new one.
- **Multi-tenancy is enforced at the tRPC procedure level.** Never query Prisma without an `organizationId` filter in a router.
- **Dynamic imports of optional packages must catch `MODULE_NOT_FOUND` gracefully.** See `src/lib/skills/loader.ts` for the pattern.

## License

AGPL-3.0-or-later. See `LICENSE`.

In plain English: you can use it, you can modify it, you can run it as a SaaS. If you run a modified version as a network service, you have to offer your users the source of your modifications under the same license. If that's not compatible with your use case, contact us about a commercial license.

## Security

Found a vulnerability? See `SECURITY.md` for disclosure instructions. Please don't open public issues for security reports.
