# CLAUDE.md — Digital Sovereignty Project

## What is this project?
Two websites (digitaltoberoende.se + digitalsovereignty.eu) helping European small businesses reduce dependency on US/Chinese digital services. Includes educational content, a self-assessment tool, and a certification program.

## Tech Stack
- **Framework:** Next.js 14+ (App Router, TypeScript)
- **Styling:** Tailwind CSS
- **CMS:** Sanity CMS (for blog posts and directory entries)
- **Analytics:** Matomo (self-hosted)
- **Newsletter:** Listmonk (self-hosted)
- **Deployment:** Docker on Hetzner Cloud
- **Payment:** Stripe + Swish (Sweden)

## Project Structure
```
/
├── apps/
│   ├── web-se/              # digitaltoberoende.se (Swedish)
│   │   ├── app/             # Next.js App Router pages
│   │   ├── components/      # Site-specific components
│   │   └── content/         # Swedish content/copy
│   └── web-eu/              # digitalsovereignty.eu (English, phase 2)
├── packages/
│   ├── ui/                  # Shared UI components
│   ├── assessment/          # Self-assessment tool logic
│   ├── certification/       # Certification framework/scoring
│   └── config/              # Shared Tailwind, TS config
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml   # Full stack: app + matomo + listmonk + sanity
├── docs/
│   ├── certification-criteria.md
│   ├── assessment-questions.md
│   └── eu-alternatives-directory.md
└── CLAUDE.md
```

## Monorepo Setup
Use Turborepo. Both sites share the `packages/*` code.

## Commands
```bash
pnpm install              # Install dependencies
pnpm dev --filter web-se  # Run Swedish site locally
pnpm build                # Build all
pnpm lint                 # Lint all
pnpm test                 # Run tests
```

## Code Conventions
- All components: TypeScript with explicit prop types
- Use server components by default, 'use client' only when needed
- CSS: Tailwind utility classes, no custom CSS files unless absolutely necessary
- Swedish user-facing text lives in `/apps/web-se/content/` as structured objects (not hardcoded in components)
- Comments in English (code), Swedish (user-facing content)
- Commit messages: conventional commits (feat:, fix:, docs:, etc.)
- All interactive elements must be accessible (WCAG 2.1 AA)

## Design System
- **Colors:** Navy #1B2A4A, Green #2D5A3D, White #FAFAF8, Amber #D4A843, Light gray #E8E8E4
- **Heading font:** Playfair Display (or Merriweather) via Google Fonts
- **Body font:** Source Sans 3 (or Nunito Sans)
- **Style:** Clean Scandinavian, authoritative, grounded. NOT generic SaaS/startup aesthetic.
- **No stock photos of people pointing at screens.** Use illustrations, diagrams, icons, or photography of real European landscapes/cities.

## Key Business Rules
1. Self-assessment tool must work without login/signup
2. Assessment results can optionally be saved (requires email)
3. Affiliate links must be visibly marked ("Affiliatelänk")
4. All data stays in EU — no Google Analytics, no US-hosted services in production
5. Certification scoring: 8 categories × 12.5 points = 100 max
6. Nivå 1 ≥ 25, Nivå 2 ≥ 50, Nivå 3 ≥ 85

## Content Tone
- Factual, never fear-mongering
- Acknowledge trade-offs of EU alternatives honestly
- Empowering ("you can do this") not condescending
- Cite real legislation (CLOUD Act, FISA 702, Schrems II, GDPR Art. 44–49)

## Environment Variables (expected)
```
NEXT_PUBLIC_SITE_URL=
NEXT_PUBLIC_MATOMO_URL=
NEXT_PUBLIC_MATOMO_SITE_ID=
SANITY_PROJECT_ID=
SANITY_DATASET=
STRIPE_SECRET_KEY=
STRIPE_PUBLISHABLE_KEY=
LISTMONK_API_URL=
LISTMONK_API_USER=
LISTMONK_API_PASSWORD=
```

## Deployment
- Docker Compose on Hetzner Cloud (CPX21 or similar)
- Reverse proxy: Caddy (automatic HTTPS)
- CI/CD: GitHub Actions → build → push to Hetzner via SSH

## Important: Infrastructure Sovereignty
This project must practice what it preaches:
- No Vercel, Netlify, AWS, GCP, Azure for production hosting
- No Cloudflare (US-based) for DNS/CDN — use Hetzner DNS or deSEC
- No Google Fonts CDN in production — self-host font files
- No US-based analytics or tracking
- Exception: GitHub for code hosting (acknowledged as US-based, acceptable for code repos)
- Exception: Stripe for payments (acknowledged, Swish offered as alternative)
