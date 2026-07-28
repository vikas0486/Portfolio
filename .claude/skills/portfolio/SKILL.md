---
name: portfolio
description: Work on the Vikash-Portfolio Next.js 15 site — dev/build/lint, resume Markdown→PDF pipeline, engineering case studies, the "Ask Engineer" BM25 knowledge-base chat, and Vercel deploys. Use when the user asks to run, build, edit, or ship anything in this portfolio project.
---

# Vikash Portfolio

Enterprise-grade personal portfolio built with **Next.js 15 (App Router) + React 19 + TypeScript + TailwindCSS 4 + Framer Motion**. It showcases Platform Engineering / DevOps / Cloud / GenAI work, an automated resume system, engineering case studies, and a build-time RAG-style "Ask Engineer" chat.

Live site: https://vikash-portfolio-alpha.vercel.app

## ⚠️ Read this first

This project pins **Next.js 15** and follows the note in `AGENTS.md`: the installed Next.js may differ from training data. **Before writing App Router / routing / config code, read the relevant guide in `node_modules/next/dist/docs/`** and heed deprecation notices. Do not assume older Next.js conventions.

## Repository map

**Kept current as of 2026-07-29** — this had drifted stale before (missing the `/projects` page, the accordion components, and a since-deleted chat component); if anything below looks off, grep the actual file before trusting it.

```
app/
  page.tsx                     # home page: Navbar → Hero → StatsBar → Terminal → Skills → Timeline → DreamVenture → Contact
  layout.tsx                   # root layout; FloatingChat renders here (every page)
  projects/page.tsx            # dedicated Projects & Platforms page
  engineering/page.tsx         # case studies index (9 sections)
  engineering/[slug]/page.tsx  # dynamic case study page — params is a Promise, always `await params`
  api/ask-engineer/route.ts    # "Ask Engineer" chat API (BM25 over knowledge base, template-fallback answer builder)
components/
  Hero.tsx, Navbar.tsx, StatsBar.tsx, Terminal.tsx, Contact.tsx
  Timeline.tsx, Projects.tsx   # both accordions (click-to-expand, matching Skills.tsx's pattern)
  Skills.tsx                   # 12 skill categories, accordion
  ResumeDownload.tsx, InfraArchitecture.tsx (13 Mermaid diagrams), SkillSnapshot.tsx (jsPDF)
  FloatingChat.tsx             # "ArchForge" — in-place popup chat, global, never navigates away
  DreamVenture.tsx             # "AI Forge Solutions LLP" section
  (EngineeringChat.tsx was removed — chat now lives only in FloatingChat, don't re-add a page-local one)
content/case-studies/*.md      # 9 files, 1:1 with the `sections` array in app/engineering/page.tsx
lib/
  bm25.ts                      # BM25 ranking used by the chat
  knowledge-base.json          # indexed knowledge chunks — static snapshot of forge-kb's master, manually synced
  profile.ts                   # SINGLE SOURCE OF TRUTH for identity/roles/skills — never hardcode these in components
  projects-data.ts             # PROJECTS/ACCENT/TAG_COLOR/DOT_COLOR
resume/
  resume.md                    # ← SOURCE OF TRUTH for the DETAILED resume (full, ~15+ pages)
  resume-professional.md       # ← SOURCE OF TRUTH for the ATS PROFESSIONAL resume (~2 pages)
build-scripts/generate-resume-pdf.js  # Markdown → PDFs via markdown-it + puppeteer
public/
  resume-detailed.pdf          # generated from resume.md
  resume-professional.pdf      # generated from resume-professional.md
  resume.pdf                   # alias/copy of resume-detailed.pdf (legacy path)
.github/workflows/resume-pdf.yml      # CI that regenerates the resume PDFs on push to resume/*.md
```

## Common tasks

### Run the dev server
```bash
npm install        # first time / after dependency changes
npm run dev        # http://localhost:3000  (falls back to :3001/:3002 if busy)
```

### Build & lint (always before deploy)
```bash
npm run build      # uses NODE_OPTIONS=--max-old-space-size=4096
npm run lint       # eslint .
```
A clean deploy requires the build to compile with lint + types passing.
**If `next dev` is already running, `pkill -f "next dev"` before `rm -rf .next && npm run build`** —
building while dev is still writing to `.next/` produces spurious "Cannot find module" errors.

### Regenerate the resume PDFs
There are **two resumes**, each authored **only** in Markdown. Never edit the PDFs directly.
- `resume/resume.md` → **Detailed** resume (full detail, diagrams) → `public/resume-detailed.pdf`
- `resume/resume-professional.md` → **ATS Professional** resume (brief, all companies + key projects, ~2 pages) → `public/resume-professional.pdf`

```bash
node build-scripts/generate-resume-pdf.js   # writes both PDFs + resume.pdf alias
```
The script also copies the detailed PDF to `public/resume.pdf` as a legacy alias.
Keep the Professional resume within **3–4 pages** and ATS-clean (no mermaid, simple tables/lists).
Keep both resumes consistent with the site's current role (see below).
If puppeteer hangs, prefer `waitUntil: "domcontentloaded"` over `networkidle0` (already set in the script).
The `resume-pdf.yml` workflow needs `permissions: contents: write` at the repo level (Settings →
Actions → Workflow permissions) to auto-commit the regenerated PDFs — if it's failing silently,
check that setting before debugging the workflow YAML itself.

### Add / edit an engineering case study
1. Add or edit a Markdown file in `content/case-studies/` (e.g. `kubernetes.md`).
2. It surfaces at `/engineering` and `/engineering/[slug]` — the slug is the filename without `.md`.
3. If the content should be answerable by the chat, update `lib/knowledge-base.json` accordingly.

### Work on "Ask Engineer" chat
- API: `app/api/ask-engineer/route.ts` — ranks `lib/knowledge-base.json` chunks with `lib/bm25.ts`.
- UI: `components/FloatingChat.tsx` (global popup — the only chat surface; a page-local one on
  `/engineering` was removed).
- This is build-time / static retrieval — **no external vector DB, no runtime API cost**. Keep it that way unless explicitly asked otherwise.
- **`GEMINI_API_KEY` has never been set anywhere** — the template-fallback answer builder is what's actually live in production, not a rarely-hit fallback path. Don't remove or neglect it thinking the LLM branch is primary.

### Deploy (Vercel)
```bash
npm run build      # verify locally first
vercel             # preview
vercel --prod      # production
```

## Conventions & guardrails

- **Current role**: Vikash is **Consultant – Platform Engineering at Hitachi Group** (since Jul 2026) — onboarding governance, modernizing legacy DevOps practices with AI-native processes, and automation (**NOT** Terraform Provider development — that's a personal lab project, unrelated to the Hitachi role, see the `terraform-plugin-framework` case study). Prior role: Lead DevOps/SRE at Devo Technology (Apr 2023 – Jun 2026). Keep the role consistent across `components/*`, `lib/knowledge-base.json`, `lib/profile.ts`, and both resumes.
- **Resume**: two sources — edit `resume/resume.md` (detailed) and `resume/resume-professional.md` (ATS) only; regenerate PDFs via the build script. Never hand-edit the PDFs. Keep both in sync with each other and the site.
- **Next.js 15**: consult `node_modules/next/dist/docs/` before routing/config changes (see `AGENTS.md`).
- **Styling**: TailwindCSS 4 utility classes; animations via Framer Motion, always `viewport={{ once: true }}`.
- **Internal links**: use `next/link`'s `<Link>`, not a plain `<a href="/...">` — ESLint's `no-html-link-for-pages` rule fails the build otherwise.
- **Knowledge base**: keep retrieval static (BM25 over local JSON) — don't introduce Pinecone/Weaviate/paid infra without an explicit request. Never add PII (e.g. a home address) to `knowledge-base.json` — it's publicly searchable via the live chat.
- After editing `lib/knowledge-base.json`, validate it parses (`node -e "JSON.parse(require('fs').readFileSync('lib/knowledge-base.json','utf8'))"`).
- Run `npm run build` and `npm run lint` before declaring UI/route work done.

## What to do when invoked

Ask the user (or infer from their request) which task they want, then follow the matching section above. If they just say "run it" → start the dev server. If they mention the resume → the PDF pipeline. If they mention case studies or the chat → the content/knowledge-base sections.
