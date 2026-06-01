# GitHub Profile README Redesign — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the `Patizy-tel/Patizy-tel` profile `README.md` with a bold, product-led, lead-generation profile featuring Wondabox, Daily Sale, and RapidDev Labs, backed by a self-hosted github-readme-stats instance on Render for reliable per-year + all-time commit/PR counts (including private repos).

**Architecture:** Two phases. **Phase A** deploys `github-readme-stats` as a Node web service on the owner's Render account (token with private-repo read access) so stat/commit/PR images are reliable and accurate. **Phase B** writes a single static `README.md` that composes hosted images (capsule-render banner, readme-typing-svg, shields.io badges, the Render stats instance, demolab streak, komarev views) using GitHub-safe HTML (centered `div`s + a table for product cards — no CSS/JS).

**Tech Stack:** Markdown + GitHub-sanitized HTML; image services: capsule-render, readme-typing-svg (demolab), shields.io, streak-stats (demolab), komarev; self-hosted github-readme-stats (Node ≥22, Express) on Render; `gh` CLI; Render MCP tools.

**Reference spec:** `docs/superpowers/specs/2026-06-01-github-profile-readme-redesign-design.md`

---

## File Structure

- **Create:** `README.md` (overwrite the existing profile README — the only file the profile repo needs).
- **External (no file in this repo):**
  - A **fork** of `anuraghazra/github-readme-stats` on the owner's GitHub (`Patizy-tel/github-readme-stats`).
  - A **Render web service** deploying that fork, env `PAT_1` set to a GitHub token.
- **Backup:** `README.old.md` (temporary copy of the current README, deleted at the end).

---

## Phase A — Deploy github-readme-stats on Render

### Task A1: Create a GitHub token with private-repo read access

**Files:** none (manual GitHub action by the owner).

- [ ] **Step 1: Create a classic Personal Access Token**

Go to GitHub → **Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token (classic)**.
- Note: `render-readme-stats`
- Expiration: 90 days (or No expiration)
- Scopes: check **`repo`** (Full control of private repositories — needed so commit/PR counts include private repos and the `daily-sale-shop` org) and **`read:user`**.
- Generate and **copy the token** (starts with `ghp_…`). Keep it for Task A3.

- [ ] **Step 2: Verify the token works and can see private repos**

Run (paste the token in place of `ghp_xxx`):
```bash
curl -s -H "Authorization: token ghp_xxx" "https://api.github.com/user/repos?visibility=private&per_page=1" -o /dev/null -w "%{http_code}\n"
```
Expected: `200` (a `401` means the token is wrong; a `200` with empty body still confirms auth).

### Task A2: Fork github-readme-stats to the owner's account

**Files:** none (creates `Patizy-tel/github-readme-stats` on GitHub).

- [ ] **Step 1: Fork via gh CLI**

Run:
```bash
gh repo fork anuraghazra/github-readme-stats --clone=false --org=""
```
Expected: output like `✓ Created fork Patizy-tel/github-readme-stats`.

- [ ] **Step 2: Verify the fork exists and has express.js**

Run:
```bash
gh api repos/Patizy-tel/github-readme-stats --jq '.full_name, .default_branch'
curl -s -o /dev/null -w "%{http_code}\n" "https://raw.githubusercontent.com/Patizy-tel/github-readme-stats/master/express.js"
```
Expected: `Patizy-tel/github-readme-stats`, a branch name (e.g. `master`), and `200`.

### Task A3: Create the Render web service from the fork

**Files:** none (creates a Render web service).

> Uses Render MCP tools. If the MCP is unavailable, do this in the Render dashboard with the same settings.

- [ ] **Step 1: Select the Render workspace**

Call `mcp__render__list_workspaces`, then `mcp__render__select_workspace` with the owner's workspace ID.
Expected: workspace selected (the owner's account that also hosts `pateltanaka.onrender.com`).

- [ ] **Step 2: Create the web service**

Call `mcp__render__create_web_service` with:
- `name`: `github-readme-stats`
- `repo`: `https://github.com/Patizy-tel/github-readme-stats`
- `branch`: `master` (use the default branch from Task A2)
- `runtime`: `node`
- `buildCommand`: `npm ci`
- `startCommand`: `node express.js`
- `plan`: `free`
- `region`: `frankfurt` (closest to Zimbabwe; or `oregon` if unavailable)
- `envVars`: `[{ "key": "PAT_1", "value": "ghp_xxx" }, { "key": "NODE_VERSION", "value": "22" }]`

Expected: a service is created and returns a service ID + a `onrender.com` URL. **Record the URL** as `RENDER_STATS_URL` (e.g. `https://github-readme-stats-xxxx.onrender.com`).

- [ ] **Step 3: Wait for the first deploy to go live**

Call `mcp__render__list_deploys` for the service until the latest deploy `status` is `live` (poll a few times; first build takes a few minutes).
Expected: a deploy with `status: live`.

### Task A4: Verify the live stats endpoint returns real data

**Files:** none.

- [ ] **Step 1: Hit the all-time endpoint**

Run (replace `RENDER_STATS_URL`):
```bash
curl -s -o /dev/null -w "%{http_code}\n" "RENDER_STATS_URL/api?username=Patizy-tel&include_all_commits=true&count_private=true&show_icons=true"
curl -s -o /dev/null -w "%{http_code}\n" "RENDER_STATS_URL/api/top-langs/?username=Patizy-tel&layout=compact"
```
Expected: `200` for both. (If the first call is slow, the service was cold-starting — retry once; it should be fast after.)

- [ ] **Step 2: Confirm private commits are counted**

Run:
```bash
curl -s "RENDER_STATS_URL/api?username=Patizy-tel&include_all_commits=true&count_private=true" | grep -o 'Total Commits[^<]*' | head -1
```
Expected: an SVG snippet containing a "Total Commits" label with a non-zero number (sanity check that the token + private counting work).

### Task A5 (optional): Keep the free service warm

**Files:** none. Skip if cold-start blips are acceptable.

- [ ] **Step 1: Create a Render cron job to ping the service every 10 minutes**

Call `mcp__render__create_cron_job` with:
- `name`: `keep-stats-warm`
- `schedule`: `*/10 * * * *`
- `runtime`: `node`
- `command`: `node -e "fetch('RENDER_STATS_URL/api?username=Patizy-tel').then(()=>process.exit(0))"`
- `plan`: `free`

Expected: cron job created. (Alternative: an external uptime pinger like UptimeRobot hitting `RENDER_STATS_URL/api?username=Patizy-tel`.)

---

## Phase B — Build the README

### Task B0: Back up the current README

**Files:**
- Create: `README.old.md`

- [ ] **Step 1: Copy current README to a backup**

Run:
```bash
cd "c:/Users/Magnificent Tello/Documents/Patizy-tel"
cp README.md README.old.md
```
Expected: no output; `README.old.md` now exists.

### Task B1: Write the hero, contact badges, and intro

**Files:**
- Modify: `README.md` (full overwrite happens across B1–B4; write the whole file in B1, then append-verify in later tasks — or write B1–B4 content in one Write).

> Implementation note: GitHub strips `<style>`, `class`, and JS. Use only `<div align>`, `<p align>`, `<img>`, `<a>`, `<table>`, `<h3>`, `<sub>`, `<br>`. The mockup's CSS is for preview only.

- [ ] **Step 1: Write the top of `README.md`**

Write this exact content as the start of `README.md`:
```markdown
<!-- Profile README — Magnificent Tello 🔥 -->
<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:6D28D9,100:DB2777&height=210&section=header&text=Magnificent%20Tello%20%F0%9F%94%A5&fontSize=48&fontColor=ffffff&animation=fadeIn&fontAlignY=40&desc=Founder%20%C2%B7%20I%20build%20products%20people%20actually%20use&descSize=18&descAlignY=62" width="100%" alt="Magnificent Tello"/>

<p>
  <img src="https://img.shields.io/badge/%F0%9F%A4%96%20Wondabox-6D28D9?style=for-the-badge" alt="Wondabox"/>
  <img src="https://img.shields.io/badge/%F0%9F%9B%92%20Daily%20Sale-0EA5E9?style=for-the-badge" alt="Daily Sale"/>
  <img src="https://img.shields.io/badge/%E2%9A%A1%20RapidDev%20Labs-F59E0B?style=for-the-badge" alt="RapidDev Labs"/>
</p>

<a href="https://github.com/Patizy-tel">
  <img src="https://readme-typing-svg.demolab.com?font=Fira+Code&weight=600&size=22&duration=3000&pause=800&color=DB2777&center=true&vCenter=true&width=640&lines=Founder+building+real+products;Wondabox+%C2%B7+Daily+Sale+%C2%B7+RapidDev+Labs;From+idea+to+reality%2C+fast.+%F0%9F%9A%80" alt="What I build"/>
</a>

<p>📍 Zimbabwe 🇿🇼</p>

<p>
  <a href="https://www.linkedin.com/in/patel-tanaka-3355a6100/"><img src="https://img.shields.io/badge/LinkedIn-Connect-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white" alt="LinkedIn"/></a>
  <a href="https://x.com/PatizyTel"><img src="https://img.shields.io/badge/@PatizyTel-000000?style=for-the-badge&logo=x&logoColor=white" alt="X"/></a>
  <a href="https://pateltanaka.onrender.com/"><img src="https://img.shields.io/badge/Portfolio-DB2777?style=for-the-badge&logo=vercel&logoColor=white" alt="Portfolio"/></a>
  <a href="mailto:pateltanaka22@gmail.com"><img src="https://img.shields.io/badge/Hire%20me-Email-6D28D9?style=for-the-badge&logo=gmail&logoColor=white" alt="Email"/></a>
  <img src="https://komarev.com/ghpvc/?username=Patizy-tel&color=db2777&style=for-the-badge&label=PROFILE+VIEWS" alt="views"/>
</p>

</div>

---

### 👋 About

I build and run **🤖 Wondabox** and **🛒 Daily Sale**, and ship custom software for clients through **⚡ RapidDev Labs**. Where others see problems, I see solutions — and a project. Currently turning ideas into products across **AI, e-commerce, and custom software**.
```

### Task B2: Write the product cards (3 ventures)

**Files:**
- Modify: `README.md` (append).

- [ ] **Step 1: Append the products section**

Append this exact content to `README.md`:
```markdown

---

### 🚀 What I'm Building

<table>
  <tr>
    <td width="33%" valign="top" align="center">
      <h3>🤖 Wondabox <sub>· FLAGSHIP</sub></h3>
      <p>AI-powered customer engagement for <b>WhatsApp &amp; Messenger</b>. Automate conversations with AI.</p>
      <sub>Python · TypeScript · AI / LLM</sub>
      <br/><br/>
      <a href="https://wondabox.ai/"><img src="https://img.shields.io/badge/Visit-wondabox.ai-6D28D9?style=for-the-badge" alt="wondabox.ai"/></a>
    </td>
    <td width="33%" valign="top" align="center">
      <h3>🛒 Daily Sale</h3>
      <p>A full <b>e-commerce platform</b> — storefront, delivery &amp; admin — built for the Zimbabwean market.</p>
      <sub>TypeScript · Node · Express</sub>
      <br/><br/>
      <a href="https://dailysale.co.zw/"><img src="https://img.shields.io/badge/Visit-dailysale.co.zw-0EA5E9?style=for-the-badge" alt="dailysale.co.zw"/></a>
    </td>
    <td width="33%" valign="top" align="center">
      <h3>⚡ RapidDev Labs</h3>
      <p><i>"From idea to reality, fast."</i> Agency shipping web &amp; mobile apps — 20+ client projects delivered.</p>
      <sub>React · Angular · NestJS</sub>
      <br/><br/>
      <a href="https://rapidevlabs.com/"><img src="https://img.shields.io/badge/Work%20with%20us-rapidevlabs.com-F59E0B?style=for-the-badge" alt="rapidevlabs.com"/></a>
    </td>
  </tr>
</table>
```

### Task B3: Write the tech stack + stats (this year + all-time)

**Files:**
- Modify: `README.md` (append).

> Replace **every** `RENDER_STATS_URL` below with the URL recorded in Task A3 (e.g. `https://github-readme-stats-xxxx.onrender.com`). Note: github-readme-stats shows **commits per-year by default** and **all-time with `include_all_commits=true`**; **Total PRs is all-time in both cards** (the card does not split PRs by year).

- [ ] **Step 1: Append the tech stack and stats sections**

Append this exact content to `README.md` (then do a find-replace of `RENDER_STATS_URL`):
```markdown

---

### 🛠️ Tech Stack

<div align="center">

![NestJS](https://img.shields.io/badge/NestJS-E0234E?style=for-the-badge&logo=nestjs&logoColor=white)
![Angular](https://img.shields.io/badge/Angular-DD0031?style=for-the-badge&logo=angular&logoColor=white)
![React](https://img.shields.io/badge/React-20232A?style=for-the-badge&logo=react&logoColor=61DAFB)
![Node.js](https://img.shields.io/badge/Node.js-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)
![Express](https://img.shields.io/badge/Express-000000?style=for-the-badge&logo=express&logoColor=white)
![MongoDB](https://img.shields.io/badge/MongoDB-47A248?style=for-the-badge&logo=mongodb&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=for-the-badge&logo=typescript&logoColor=white)
![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=for-the-badge&logo=javascript&logoColor=black)
![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white)

</div>

---

### 📊 By the Numbers

<div align="center">

<img height="175" src="RENDER_STATS_URL/api?username=Patizy-tel&include_all_commits=true&count_private=true&show_icons=true&hide_border=true&theme=radical&bg_color=0D1117&title_color=DB2777&icon_color=6D28D9&text_color=C9D1D9&custom_title=All-Time%20Impact" alt="All-time stats" />
<img height="175" src="https://streak-stats.demolab.com?user=Patizy-tel&theme=radical&hide_border=true&background=0D1117&ring=DB2777&fire=6D28D9&currStreakLabel=DB2777&sideLabels=C9D1D9&dates=8B949E" alt="Streak" />

<br/>

<img height="175" src="RENDER_STATS_URL/api?username=Patizy-tel&count_private=true&show_icons=true&hide_border=true&theme=radical&bg_color=0D1117&title_color=DB2777&icon_color=6D28D9&text_color=C9D1D9&custom_title=This%20Year" alt="This-year stats" />
<img height="175" src="RENDER_STATS_URL/api/top-langs/?username=Patizy-tel&layout=compact&hide_border=true&theme=radical&bg_color=0D1117&title_color=DB2777&text_color=C9D1D9&langs_count=8" alt="Top languages" />

</div>
```

- [ ] **Step 2: Find-replace the stats URL**

Run (replace the example with the real URL from Task A3):
```bash
cd "c:/Users/Magnificent Tello/Documents/Patizy-tel"
sed -i 's#RENDER_STATS_URL#https://github-readme-stats-xxxx.onrender.com#g' README.md
grep -c "RENDER_STATS_URL" README.md
```
Expected: final `grep` prints `0` (no placeholders left).

### Task B4: Write the CTA footer

**Files:**
- Modify: `README.md` (append).

- [ ] **Step 1: Append the footer**

Append this exact content to `README.md`:
```markdown

---

<div align="center">

## 💼 Got a project? Let's build it. 🚀

**Open to clients, collaborations &amp; consulting.**

<a href="mailto:pateltanaka22@gmail.com"><img src="https://img.shields.io/badge/%F0%9F%93%A9%20Get%20in%20touch-DB2777?style=for-the-badge" alt="Get in touch"/></a>

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:DB2777,100:6D28D9&height=120&section=footer" width="100%" alt="footer"/>

</div>
```

### Task B5: Verify all image URLs resolve

**Files:** none.

- [ ] **Step 1: Extract every image URL from README and check status**

Run:
```bash
cd "c:/Users/Magnificent Tello/Documents/Patizy-tel"
grep -oE 'src="[^"]+"' README.md | sed 's/src="//;s/"$//' | while read u; do
  code=$(curl -s -o /dev/null -w "%{http_code}" -L --max-time 25 "$u"); echo "[$code] $u";
done
```
Expected: every line shows `[200]`. (A `RENDER_STATS_URL` line shows the real onrender.com domain — if it shows a cold-start delay, retry once.)

### Task B6: Commit, push, and verify on GitHub

**Files:**
- Delete: `README.old.md`

- [ ] **Step 1: Remove the backup and stage**

Run:
```bash
cd "c:/Users/Magnificent Tello/Documents/Patizy-tel"
rm README.old.md
git add README.md
git status --short
```
Expected: `M README.md` (and `D README.old.md` only if it had been committed; if it was never committed, just `M README.md`).

- [ ] **Step 2: Commit**

Run:
```bash
git commit -m "$(cat <<'EOF'
Redesign profile README: product-led founder profile

Bold Founder layout featuring Wondabox, Daily Sale, and RapidDev Labs.
Self-hosted github-readme-stats (Render) for reliable per-year and
all-time commit/PR counts including private repos. Removes the broken
committers.top badge and prior typos.

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
EOF
)"
```
Expected: one commit created.

- [ ] **Step 3: Push**

Run:
```bash
git push origin main
```
Expected: push succeeds.

- [ ] **Step 4: Verify the live profile renders**

Open `https://github.com/Patizy-tel` in a browser (or fetch and inspect). Confirm against the Acceptance Criteria in the spec:
- Hero shows **Magnificent Tello 🔥** + three venture pills; no Flostec/title text.
- Wondabox card is first and flagged FLAGSHIP; all three CTA buttons link to wondabox.ai / dailysale.co.zw / rapidevlabs.com.
- LinkedIn / X / Portfolio / Email badges appear near the top and all work.
- "All-Time Impact" and "This Year" stat cards both render (radical purple/pink theme) and show commit/PR numbers; streak + top-langs render.
- No raw `<style>`/`class` artifacts; product cards sit side-by-side on desktop and stack on mobile.
- The old `committers.top` badge and "Chekc" typo are gone.

---

## Self-Review (completed by plan author)

- **Spec coverage:** Hero (B1), contact badges (B1), intro (B1), 3 product cards (B2), tech stack (B3),
  stats with per-year + all-time commits/PRs (A1–A4 + B3), streak/top-langs (B3), CTA footer (B4),
  Render self-host incl. private (Phase A), trophy dropped (not present). ✅
- **Placeholder scan:** `RENDER_STATS_URL` is an intentional deferred value produced by Task A3 and
  resolved in Task B3 Step 2 (verified `grep -c` == 0). No other placeholders. ✅
- **Consistency:** `RENDER_STATS_URL` used identically in B3; `PAT_1` env name matches express.js
  (`api/*` read `PAT_1`); colors `6D28D9`/`DB2777`/`0D1117` consistent throughout; `main` is the
  profile repo's branch (confirmed from git status) while `master` is the stats fork's default branch. ✅
- **Known caveat documented:** github-readme-stats Total PRs is all-time in both cards (no per-year PR
  split); Render free tier cold-starts (mitigated by optional Task A5). ✅
