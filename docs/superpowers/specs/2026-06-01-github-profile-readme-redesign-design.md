# GitHub Profile README Redesign — Design Spec

**Date:** 2026-06-01
**Repo:** `Patizy-tel/Patizy-tel` (special profile repo — its `README.md` renders at the top of github.com/Patizy-tel)
**Owner:** Magnificent Tello 🔥 (Tanaka Patel)

## Goal

Redesign the profile README into a **client/lead-generation funnel** that puts the owner's
three ventures front and centre. A visitor should, within seconds, understand what is built,
see proof it ships, and have an obvious path to make contact.

## Audience & Positioning

- **#1 audience:** prospective clients & leads.
- **Positioning shift:** from "10-year full-stack dev for hire" → **"a founder who builds and runs real products."**
- Lead with the **products**, not the parent company. (Flostec Digital Solutions and job-title
  framing are intentionally omitted.)

## Visual Direction — "Bold Founder"

- High-energy: gradient hero, punchy taglines, colourful product cards, loud CTAs.
- **Palette:** purple `#6d28d9` → pink `#db2777` gradient; dark surfaces `#0d1117` / `#161b22`.
- Stat cards use the **`radical`** theme tuned to the palette (`title_color=db2777`, `icon_color=6d28d9`).

## Build Approach — Static + self-updating cards

- A single `README.md` (markdown + light inline HTML for layout).
- All dynamic content is rendered via free **hosted image services** (no GitHub Actions, no token,
  no workflow to maintain). Cards refresh themselves whenever GitHub re-fetches the images.
- Services used (public, healthy):
  - **capsule-render** — gradient hero banner + footer wave
  - **readme-typing-svg** (demolab) — animated tagline
  - **shields.io** (`for-the-badge`) — contact + tech-stack badges
  - **streak-stats (demolab)** — contribution streak
  - **komarev** — profile-views counter
- **Self-hosted on Render** (the public github-readme-stats instance is rate-limited — 503 —
  and cannot count private repos):
  - **github-readme-stats** deployed as a Node web service on the owner's Render account.
    Serves the stats card, the commit/PR counts (per-year + all-time), and top-languages from
    `https://<service>.onrender.com/api…`. A GitHub token (`PAT_1`) with **private-repo read
    access** is set on the service so commit/PR counts include private work.
  - **Render free-tier caveat:** services spin down after ~15 min idle (~50s cold start), which
    can briefly break the embedded image. Optional mitigation: a Render cron job pinging the
    service to keep it warm.
- **Dropped:** github-profile-trophy (rate-limited 402; owner chose to skip trophies).

## Confirmed Content

| Field | Value |
|---|---|
| Hero name | **Magnificent Tello 🔥** |
| Tagline | "Founder building products people actually use" |
| Location | 🇿🇼 Zimbabwe |
| Flagship product | 🤖 **Wondabox** → https://wondabox.ai/ — AI-powered customer engagement for WhatsApp & Messenger |
| Product 2 | 🛒 **Daily Sale** → https://dailysale.co.zw/ — full e-commerce platform (storefront, delivery, admin) |
| Product 3 | ⚡ **RapidDev Labs** → https://rapidevlabs.com/ — agency, "from idea to reality, fast" (20+ client projects) |
| Email (Hire me) | pateltanaka22@gmail.com |
| Portfolio | https://pateltanaka.onrender.com/ |
| X / Twitter | @PatizyTel |
| LinkedIn | https://www.linkedin.com/in/patel-tanaka-3355a6100/ |
| GitHub user (for stats) | `Patizy-tel` |

## Layout — 7 stacked sections (top → bottom)

The structure is a deliberate sales funnel: **hook → contact → offer → proof → close.**

1. **Hero banner** — gradient (capsule-render) with name **Magnificent Tello 🔥**, three venture
   pills (🤖 Wondabox · 🛒 Daily Sale · ⚡ RapidDev Labs), an animated typing tagline, and 🇿🇼 Zimbabwe.
2. **Contact / social badges** — LinkedIn · X (@PatizyTel) · Portfolio · Hire me (Email). Placed
   high (above the fold) so the lead path is immediate.
3. **One-line intro** — product-led: "I build and run Wondabox and Daily Sale — and ship custom
   software for clients through RapidDev Labs…"
4. **The products** — 3 cards. **Wondabox first, flagged FLAGSHIP.** Each card: emoji + name,
   one-line pitch, tech line, and a CTA button linking to the live site.
5. **Tech stack** — shields.io badges reflecting real repo languages: NestJS, Angular, React,
   Node.js, Express, MongoDB, TypeScript, JavaScript, Python.
6. **Live GitHub stats** — `radical` theme, served from the **self-hosted Render instance**.
   Must surface **commit and PR counts, both per-year and all-time**:
   - **All-time card:** `…/api?…&include_all_commits=true&count_private=true` → **Total Commits
     (all-time, incl. private)** + **Total PRs** + stars + issues + contributions.
     Custom title "All-Time Impact".
   - **This-year card:** a second card *without* `include_all_commits` (with `count_private=true`) →
     **Total Commits (current year)** + PRs for the year. Custom title "This Year".
   - Plus **streak-stats** (current/longest streak + total contributions, public demolab instance)
     and **top-languages** (compact, from the Render instance `…/api/top-langs`).
   - Because the instance uses a token with private-repo read access, counts reflect **all** work,
     not just public.
7. **CTA footer** — "Got a project? Let's build it. 🚀 — Open to clients, collaborations &
   consulting" + "Get in touch" (email) button + profile-views counter + capsule-render footer wave.
   *(Trophy section dropped per owner decision.)*

## Rendering Notes / Constraints

- GitHub sanitizes HTML in READMEs: **no `<style>`, no `class`, no JS, no `onclick`.** The mockup
  used those for preview only. The real README must achieve layout using:
  - Centered blocks via `<div align="center">` / `<p align="center">`.
  - Product "cards" via a markdown/HTML **table** (the reliable way to get side-by-side columns).
  - Colors/gradients come **only** from the hosted image services (banner, badges, stat cards),
    not from CSS.
- All badges/links use `for-the-badge` style for the bold look.
- Links open the live sites; email uses a `mailto:` link.

## Out of Scope (YAGNI)

- No GitHub Actions in the profile repo (the README itself stays static).
- No `wesetech-digital` org (owner did not prioritize it).
- No blog-feed or WakaTime auto-injection.
- No trophy card (dropped).

## Acceptance Criteria

- [ ] github-readme-stats is deployed and reachable on Render (`https://<service>.onrender.com/api`
      returns 200) with `PAT_1` set to a token that can read the owner's private repos.
- [ ] README renders correctly on github.com/Patizy-tel (verified visually).
- [ ] Hero leads with **Magnificent Tello 🔥** + the three products; no Flostec/title framing.
- [ ] Wondabox appears first and is marked flagship; all three CTAs link to the correct live sites.
- [ ] Contact badges (LinkedIn, X, Portfolio, Email) appear high on the page and all resolve.
- [ ] Stats / streak / top-langs cards render with the purple→pink `radical` theme.
- [ ] Commit & PR counts are visible **both per-year (This Year card) and all-time (All-Time Impact
      card)**, and reflect private work (`count_private=true`).
- [ ] No raw HTML/CSS leaks (no visible `<style>`/`class` artifacts); layout holds on mobile widths.
- [ ] The old broken `committers.top` badge and typos from the previous README are gone.
