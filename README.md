# healthly

**DigiLocker for your medical records, with an AI companion that answers from them.**

healthly turns years of scattered prescriptions, lab reports, scans and discharge summaries into one longitudinal health record you can search, chart and question. Every number and every AI answer links back to the document it came from.

> Medical documents → structured health data → longitudinal timeline → retrieval & analysis

## Who it's for

People who manage their own (or a parent's) care across many doctors, labs and cities, and who end up re-explaining their history at every new appointment from a folder of PDFs and phone photos.

## What's in the product

| Screen | What it does |
| --- | --- |
| **Home** | Health snapshot, recent activity, current medications, trend cards with sparklines, global search (⌘K) |
| **Timeline** | Every record, chronologically, filterable by consultation / diagnostic / medication / hospitalization / procedure, with year jump and a detail drawer |
| **Records** | The document vault: drag-and-drop multi-file upload with a staged processing pipeline, filters, sort and search |
| **Record detail** | Original document beside the extracted data, extraction confidence, and a review mode to correct values |
| **Health Data** | Biomarkers (longitudinal chart + measurement history with clickable sources), conditions, allergies and vitals |
| **Medications** | Current medications and a longitudinal history: a Gantt of every course plus dose-change timelines |
| **Ask AI** | Questions across the whole record; answers separate **facts from records**, **calculated comparisons** and **AI interpretation**, with inline charts, gaps and numbered sources |
| **Medical Summary** | A printable, doctor-ready summary for any period, where every line cites its source documents |

### Clinical honesty, built in

- **Reference ranges are stored per report**, never assumed universal. The TSH chart's shaded band steps between labs (SRL 0.35–5.5, Apollo 0.27–4.2, Metropolis 0.4–4.5) and each value is flagged against its own report.
- **Units are never silently mixed.** A Vitamin D result in nmol/L is converted with an explicit, labelled factor; a unit with no safe conversion is listed but not charted.
- **Gaps are stated, not filled.** "I found reports from March 2025 and September 2026, but I don't have TSH results covering the 18 months between them."
- **Hearsay isn't data.** A TSH value a doctor noted from an outside lab is called out, not charted.
- **Low-confidence extractions need review.** Anything under 90% confidence stays out of trends and AI answers until the user confirms or corrects it, and corrections are marked.
- Absence on a newer prescription isn't treated as "stopped" when a different doctor wrote it.

## Demo patient

Aarav Sharma, 35, Bengaluru (previously Pune): **47 documents from 2023 to 2026** across 3 labs, 4 clinics, 2 hospitals and an imaging centre. They show a rising TSH that leads to a diagnosis of subclinical hypothyroidism and a levothyroxine dose change, a statin started for cholesterol, recurrent vitamin D deficiency, a penicillin allergy after a drug rash, a 5-day pneumonia admission, and a handwritten prescription scan.

Try asking:
- "How have my thyroid levels changed over the last 3 years?"
- "What antibiotics have I been prescribed over the past three years?"
- "Show me every time I was diagnosed with a respiratory infection."
- "When did my TSH first become abnormal?"
- "What changed between my last two prescriptions?"
- "Summarize my treatment history for my new doctor."

## Architecture

```
src/
  app/                      Next.js App Router screens + API routes
    api/extract             document processing endpoint
    api/ask                 LLM answer composition endpoint
    api/status              which services are connected
  lib/
    types.ts                domain model (documents own all extracted facts)
    data/                   repository boundary: demo dataset ↔ Postgres (Prisma)
    health/                 pure read-model: biomarker series, unit conversion,
                            medication courses, conditions, search
    ai/engine.ts            retrieval engine: intent routing → grounded answers
    ai/digest.ts            citation-ready record digest for an LLM
    processing/             extraction contract, Claude extractor, demo extractor,
                            reconciliation into the history
    llm/anthropic.ts        Claude provider (structured outputs)
  components/               shell, shadcn-style primitives, charts, records, ask
prisma/                     schema + seed for PostgreSQL
artifact/                   single-file build (in-memory router, esbuild + Tailwind)
scripts/                    developer utilities
.github/workflows/          CI and GitHub Pages deploy
```

**Modular services, graceful fallbacks.** Every external service is optional:

| Service | With configuration | Without |
| --- | --- | --- |
| Storage | PostgreSQL via Prisma (`DATABASE_URL`) | bundled demo patient; uploads and corrections persist in the browser |
| Document extraction | Claude reads PDFs and images (OCR, handwriting) into a strict JSON schema (`ANTHROPIC_API_KEY`) | demo extractor infers a plausible document from the file name, labelled as such and routed to review |
| Ask AI | Claude writes the answer, grounded on the record digest **and** the local engine's computed facts; cited ids are validated server-side | deterministic retrieval engine answers from structured data, entirely on-device |

The retrieval engine is useful by itself and also acts as a guardrail for the LLM: the numbers come from code, and the model is told to use them.

## Getting started

Requirements: Node.js 22 (see `.nvmrc`) and npm. PostgreSQL and an Anthropic API key are optional.

```bash
git clone https://github.com/tamanna-ail/healthly.git
cd healthly
npm install            # also generates the Prisma client
npm run dev            # http://localhost:3000, demo data, no setup needed
```

### Optional services

Copy `.env.example` to `.env` and fill in what you need:

```bash
# PostgreSQL: create the tables and load the demo patient
DATABASE_URL="postgresql://user:pass@localhost:5432/healthly"
npm run db:push && npm run db:seed

# Claude: real document extraction (PDF/image OCR) and AI-written answers
ANTHROPIC_API_KEY="sk-ant-..."
```

Without them, the app runs on the bundled demo patient, the labelled demo extractor and the on-device retrieval engine.

## Development

| Command | What it does |
| --- | --- |
| `npm run dev` | Start the dev server |
| `npm run build` / `npm start` | Production build and server |
| `npm run lint` | ESLint |
| `npm run typecheck` | TypeScript, no emit |
| `npm run db:push` / `npm run db:seed` | Sync the Prisma schema to Postgres / load the demo patient |
| `npm run ask -- "When did my TSH first become abnormal?"` | Query the retrieval engine from the terminal |
| `npm run build:artifact` | Bundle the whole app into one self-contained HTML file (`artifact/healthly.html`) |

Notes for contributors:

- This uses **Next.js 16**, which has breaking changes from older versions (async `params`, `proxy` instead of `middleware`, and more). Check `node_modules/next/dist/docs/` before relying on older patterns; `AGENTS.md` says the same for coding agents.
- Business logic lives in `src/lib` as pure functions over a `HealthSnapshot` (no React, no I/O), so the UI, the AI engine and the server share it. Start there for any change to how records are interpreted.
- Every clinical fact belongs to the document it came from. Keep that invariant: new data types should hang off `MedicalDocument.extracted`, and anything shown to the user should be traceable to a document id.
- Colours are theme tokens in `src/app/globals.css` (light and dark). Use the token classes (`bg-surface`, `text-ink-2`, `bg-t-green-bg`, and so on) rather than literal colours.
- CI (`.github/workflows/ci.yml`) runs lint, typecheck and build on every push to `main` and on pull requests.

## Deploy

**Vercel (full version):** import the repository and deploy; no configuration is needed for the demo. Add `ANTHROPIC_API_KEY` for real extraction and AI-written answers, and `DATABASE_URL` (available at build time) for Postgres.

**GitHub Pages (static demo):** `.github/workflows/deploy-pages.yml` publishes a static export on every push to `main`. Enable it once under Settings → Pages → Source: **GitHub Actions**; the site is then served at https://tamanna-ail.github.io/healthly/. Pages has no server, so that build leaves out the API routes and the app runs on its on-device engines. Records uploaded in the browser are rendered through a client-side `404.html` fallback.

To try the static build locally: `rm -rf src/app/api && STATIC_EXPORT=true BASE_PATH=/healthly npm run build`, then restore `src/app/api` with `git checkout src/app/api`.

**Single file:** `npm run build:artifact` produces one self-contained HTML page with an in-memory router. It uses the on-device engines and invented clinic and lab names in its demo data.

## Stack

Next.js 16 (App Router) · React 19 · TypeScript · Tailwind CSS v4 · shadcn/ui-style components on Radix · Lucide · Recharts · PostgreSQL + Prisma 7 · Anthropic SDK

---

healthly organises and explains records. It is not a doctor and does not give medical advice.
