# Case study: a 5-language static marketing site for a dental tourism clinic

A production website for a dental aesthetics & implant clinic in Antalya, Turkey, serving international dental-tourism patients in **five languages (TR / EN / DE / RU / AR, including full RTL)** — built as a fully static Next.js export that runs on inexpensive shared hosting with **no backend, no CMS, and no third-party chat service**. The site evolved from a multi-sector demo template I built earlier and was hardened into a real client deliverable.

> The source code is owned by the client and is not published. This repository documents the problem, the architecture, and the engineering decisions.

---

## Problem

The clinic needed to replace an aging PHP site with something that could:

- **Sell to international patients** — dental tourism visitors arrive speaking Turkish, English, German, Russian, or Arabic, and expect content (and a chat assistant) in their own language.
- **Run on the hosting they already pay for** — a constrained shared cPanel host with FTP access only: no Node runtime, no SSH, no CI runners.
- **Stay maintainable by one developer** — no CMS to patch, no database to back up, no chat SaaS subscription.
- **Feel premium** — the clinic competes on trust; the site needed polished motion design (including an animated implant-anatomy explainer) without sacrificing performance or accessibility.

## Architecture

```
┌──────────────────────── build time (local) ────────────────────────┐
│                                                                    │
│  Next.js 14 App Router ── `output: "export"` ──► out/ (pure HTML/  │
│   │                                               CSS/JS, no SSR)  │
│   ├─ Dict-based i18n: every string is a {tr,en,de,ru,ar} record;   │
│   │  a context provider resolves the active language and flips     │
│   │  <html lang/dir> for RTL — no i18n framework, no route split   │
│   └─ Content as typed TS data modules (treatments, packages, FAQ)  │
│                                                                    │
└───────────────┬────────────────────────────────────────────────────┘
                │ deploy.sh (bash)
                ▼
┌──────────────────────── deploy (shared host) ──────────────────────┐
│  1. npm run build → inject production .htaccess into out/          │
│  2. zip the whole export → single FTPS upload                      │
│  3. cPanel JSON API: server-side extract of the zip                │
│  4. live health check: HTTP status of every top-level route +      │
│     sitemap, with cache-busting query                              │
└───────────────┬────────────────────────────────────────────────────┘
                │
                ▼
┌──────────────────────── runtime (browser) ─────────────────────────┐
│  Static pages + client-side enhancements:                          │
│   • Chat widget — fully offline: regex-heuristic language          │
│     detection on the visitor's own text (Arabic/Cyrillic script    │
│     ranges, TR/DE diacritics + stopwords, EN fallback) routed      │
│     into a clinic persona knowledge base. Zero network calls,      │
│     zero console errors on the static host; the same engine        │
│     exposes a system prompt so it can be upgraded to an            │
│     LLM-backed /api/chat route on any Node host without a          │
│     rewrite.                                                       │
│   • ToothImplantAnatomy — labeled cross-section plate fusing a     │
│     hand-drawn natural tooth with the clinic's real implant        │
│     photo; leader-line draw-in animations and pointer tilt         │
│     (Framer Motion), degrading gracefully under                    │
│     prefers-reduced-motion. Labels are tuned per language so all   │
│     five locales fit the plate margins.                            │
└────────────────────────────────────────────────────────────────────┘
```

**Production `.htaccess` (shipped with every deploy):** ACME-safe canonical HTTPS redirect (loop-safe behind proxies), clean route → `index.html` mapping for the static export, long-lived immutable caching for hashed assets with short revalidation for HTML, standard security headers, and gzip.

## My role

Sole developer — design adaptation, frontend implementation, i18n architecture, the offline chat engine, the deploy tooling, and the production launch (the previous PHP site was archived in place as a rollback path).

## Stack

| Layer | Choice |
|---|---|
| Framework | Next.js 14 (App Router, `output: "export"` — fully static) |
| Language | TypeScript throughout, including all content data |
| Styling | Tailwind CSS + Radix UI primitives |
| Motion | Framer Motion, GSAP, Lenis smooth scroll |
| Forms | react-hook-form + zod |
| i18n | Custom `Dict` type (`{tr,en,de,ru,ar}`) + React context; RTL via `<html dir>` sync |
| Deploy | Bash: zip → single FTPS upload → cPanel JSON API server-side extract → HTTP health check |
| Hosting | Shared cPanel hosting (no Node runtime) |

## Key decisions & why

- **Static export over SSR/CMS.** The host has no Node runtime, and the content changes rarely. A typed-TS content layer gives compile-time safety that a CMS wouldn't, and the attack surface drops to "serve files."
- **Dictionary i18n instead of route-per-locale.** Five locales × route splitting would have quintupled the page count and complicated the shared-host rewrite rules. A single page tree with a client-side language context (persisted per visitor, defaulting to EN for the international audience, with no hydration mismatch) kept the export small and the `.htaccess` trivial.
- **Offline chat instead of a chat SaaS.** The widget answers from a curated clinic knowledge base, detecting the visitor's language from their own message — so a Russian patient typing into an English UI still gets a Russian answer. The engine is isomorphic by design: the same module exports an LLM system prompt, so moving to an Anthropic-backed API route later is a hosting change, not a rewrite. Deliberate guardrails live in the persona: never quote fixed prices (treatment plans are individual), never diagnose — always route to a free consultation.
- **One-zip deploys.** Per-file FTP on this host was impractically slow for a Next.js export (hundreds of files). Uploading a single archive and extracting it server-side via the cPanel API turned deploys from a long, failure-prone file sync into a short, atomic-feeling operation — followed by an automated live health check of every top-level route before the deploy is declared done.
- **Secrets stay out of the repo.** The deploy script refuses to run without the credential provided as an environment variable; nothing sensitive is hardcoded.

## Results

- Live in production, replacing the legacy PHP site with a documented rollback path.
- Five fully translated locales including Arabic RTL — covering the clinic's actual patient demographics — with language-aware chat on a site that has no server at all.
- Runs on the client's existing low-cost shared hosting: zero new infrastructure, zero recurring SaaS fees for chat or CMS.
- Repeatable one-command deploys with built-in post-deploy verification.
- Motion-rich UI that respects `prefers-reduced-motion` and keeps `<html lang>`/`dir` in sync for screen readers and SEO; sitemap and robots generated at build time.

## Lessons learned

1. **Constraints are a design input, not an obstacle.** "Shared hosting, FTP only" shaped the best parts of this build: the static export, the offline chat, and the zip-based deploy are all direct answers to that constraint — and each made the product simpler and cheaper to run.
2. **Detect language from the user, not the UI.** Dental-tourism visitors routinely type in a language other than the one the page is displayed in. Script-range checks (Arabic, Cyrillic) are nearly free and almost never wrong; diacritics + stopword heuristics cover the Latin-script languages well enough for a scoped assistant.
3. **Design the upgrade path even when you don't ship it.** Keeping the chat engine free of browser-only imports meant the LLM-backed server route works today in dev and on any future Node host — the static deployment is a deployment choice, not an architectural ceiling.
4. **Multilingual UI is a layout problem, too.** German compounds and Russian labels are much longer than their Turkish originals; the anatomy plate's labels needed per-language line-breaking and anchor tuning to fit all five locales.
5. **A deploy isn't done until the live site says so.** Baking the health check (every route + sitemap, cache-busted) into the deploy script caught issues that a successful upload alone would have hidden.
