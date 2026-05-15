# The Palm Reader

> Upload photos of your palms and receive a beautifully designed, AI-guided symbolic palm reading — delivered as a downloadable visual report.

**Demo → [The Palm Reader](https://www.loom.com/share/91bbff78405d45da93405f898aa9f9e8)**

---

## What it does

The Palm Reader is a multi-step AI pipeline that turns two palm photos into a premium infographic. The user photographs their left and right hands, selects their dominant hand, and the app:

1. Runs the images through a GPT-5 vision analysis against a classical palmistry framework
2. Generates five historical figure portraits matched to the reading's personality traits
3. Removes photo backgrounds and produces an AI-styled composite hand illustration
4. Composes everything into a single downloadable infographic with annotated palm lines, comparative hand analysis, mount clues, and historical parallels

The output is for entertainment and reflection — the design intentionally echoes vintage palmistry manuscripts.

---

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | React 18, TypeScript, Vite (SWC), Tailwind CSS, shadcn/ui, React Router v6 |
| Auth | Supabase Auth — Google OAuth |
| AI (text) | OpenAI GPT-5 — structured JSON output via strict Zod schema |
| AI (images) | OpenAI gpt-image-2 — portrait illustration & hand composite |
| Background removal | remove.bg API |
| Poster rasterization | resvg-js (SVG → PNG, Node.js only) |
| Edge orchestration | Supabase Edge Functions (Deno) |
| API layer | Vercel Serverless Functions (Node.js) |
| Database | Supabase PostgreSQL with RLS |
| Storage | Supabase Storage (two buckets) |
| Deployment | Vercel (frontend + Node API) · Supabase (DB + Edge Functions) |

---

## Architecture

The app is split across two serverless runtimes because of a hard constraint: `resvg-js` (the SVG-to-PNG rasterizer) only runs on Node.js, not Deno. All heavy orchestration and OpenAI calls live in Supabase Edge Functions (Deno); all CPU-intensive rendering runs on Vercel (Node.js).

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Browser (React SPA)                        │
│                                                                     │
│  Start → SignIn (Google OAuth) → Right Hand → Left Hand →           │
│  Dominant Hand → Review → Generating → Report                       │
│                                                                     │
│  Polls Supabase DB every 5s for artifact URLs as they arrive        │
│  Renders PalmReadingInfographic (React) → html2canvas → PNG download│
└────────────────┬───────────────────────────────────────────────────┘
                 │ supabase.functions.invoke()
                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│           Supabase Edge Function: generate-palm-reading-report      │
│                         (Deno · ~350s budget)                       │
│                                                                     │
│  1. Load session row from DB                                        │
│  2. Call OpenAI GPT-5 with both palm images + palmistry framework   │
│  3. Validate structured JSON output via Zod schema                  │
│  4. Write report_json + content_json to DB, set status = "ready"   │
│  5. Fire-and-forget three parallel pipelines ──────────────────┐   │
└─────────────────────────────────────────────────────────────────┼───┘
                                                                  │
          ┌───────────────────────┬──────────────────────────────┤
          ▼                       ▼                              ▼
┌─────────────────┐  ┌────────────────────────┐  ┌──────────────────────┐
│ Vercel API      │  │ Supabase Edge Function  │  │ Vercel API           │
│ remove-bg       │  │ generate-historical-    │  │ generate-hands-image │
│                 │  │ busts  (Deno)           │  │                      │
│ Calls remove.bg │  │                         │  │ Calls OpenAI         │
│ API for each    │  │ Calls gpt-image-2 x5    │  │ gpt-image-2 /edits   │
│ hand photo      │  │ in parallel             │  │ with both hand photos│
│                 │  │ (engraving-style)       │  │                      │
│ Stores PNGs in  │  │ Stores JPEGs in         │  │ Stores PNG in        │
│ Supabase        │  │ poster_assets/          │  │ Supabase Storage     │
│ Storage         │  │                         │  │                      │
│                 │  │ Chains to               │  │ Writes               │
│ Writes          │  │ generate-poster-base    │  │ hands_image_url      │
│ left/right_hand │  └──────────┬─────────────┘  │ to DB                │
│ _bg_url to DB   │             │                 └──────────────────────┘
└─────────────────┘             ▼
                  ┌─────────────────────────────┐
                  │ Supabase Edge Function       │
                  │ generate-poster-base (Deno)  │
                  │                              │
                  │ Resolves static base URL     │
                  │ Writes poster_base_url to DB │
                  │ Chains to ─────────────────┐ │
                  └────────────────────────────┼─┘
                                               ▼
                  ┌─────────────────────────────┐
                  │ Supabase Edge Function       │
                  │ composite-and-qa-poster      │
                  │ (Deno)                       │
                  │                              │
                  │ Assembles all artifact URLs  │
                  │ POSTs to Vercel ───────────┐ │
                  └───────────────────────────┼─┘
                                              ▼
                  ┌─────────────────────────────┐
                  │ Vercel API                   │
                  │ render-poster-v2 (Node.js)   │
                  │                              │
                  │ Builds deterministic SVG     │
                  │ Rasterizes via resvg-js      │
                  │ Uploads final poster PNG     │
                  │ Writes final_poster_url      │
                  │ to DB                        │
                  └─────────────────────────────┘
```

### Data model (key columns on `palm_reading_reports`)

| Column | Purpose |
|---|---|
| `status` | Session lifecycle: `pending` → `generating` → `ready` / `failed` |
| `poster_generation_status` | Reading generation: `generating` → `ready` |
| `poster_pipeline_status` | Bust/poster pipeline: `busts_running` → `base_resolved` → `done` |
| `report_json` / `content_json` | GPT-5 structured output (v2.0.0 schema) |
| `bust_image_urls` | JSONB array (length 5, index-preserving) of `{url, name, short_label}` |
| `left/right_hand_bg_url` | Background-removed hand PNGs |
| `hands_image_url` | AI-generated composite hand illustration |
| `final_poster_url` | Rasterized poster PNG in storage |

### Storage buckets

| Bucket | Public | Content |
|---|---|---|
| `palm-reading-assets` | via RLS | Background-removed hand PNGs, AI hands composite |
| `poster_assets` | via RLS SELECT policy | Bust portrait JPEGs, poster base, final poster PNG |

---

## Repository structure

```
├── src/
│   ├── pages/flow/          # Step-by-step UX: RightHand → LeftHand → Dominant → Report
│   ├── components/
│   │   ├── palm-reader/     # PalmReadingInfographic, PalmOrnaments (SVG ornaments, BustMedallion)
│   │   └── ui/              # shadcn/ui component library
│   ├── context/             # AuthContext (Google OAuth), ReadingContext (session state)
│   ├── lib/palm-reader/     # Zod schemas, prompt versions, poster artifact helpers
│   └── server/palm-reader/  # resvg rendering engine, SVG layout, text-fit algorithm
│
├── api/palm-reading/        # Vercel serverless functions (Node.js)
│   ├── remove-bg.ts         # remove.bg background removal
│   ├── generate-hands-image.ts  # gpt-image-2 composite hand illustration
│   ├── render-poster.ts     # V1 SVG → PNG rasterizer
│   └── render-poster-v2.ts  # V2 deterministic overlay rasterizer
│
├── supabase/
│   ├── functions/           # Deno edge functions
│   │   ├── generate-palm-reading-report/   # Main GPT-5 orchestrator
│   │   ├── generate-historical-busts/      # gpt-image-2 portrait pipeline
│   │   ├── generate-poster-base/           # Poster base URL resolver
│   │   ├── composite-and-qa-poster/        # Routes to Vercel rasterizer
│   │   ├── render-palm-poster-overlay/     # SVG overlay builder
│   │   └── retry-bg-removal/               # Manual retry for bg removal
│   └── migrations/          # PostgreSQL schema + RLS policies
│
└── vercel.json              # Function timeouts, SPA rewrite rule
```

---

## Key engineering decisions

**Why two runtimes?**
`resvg-js` (the Rust-based SVG rasterizer) requires a Node.js environment. Everything else runs in Deno (Supabase Edge Functions) for lower cold-start latency and simpler secret management. The `composite-and-qa-poster` edge function acts as the bridge — it assembles all artifact URLs and POSTs to the Vercel `render-poster-v2` endpoint.

**Why fire-and-forget pipelines?**
GPT-5 for the main reading takes up to 3 minutes. Background removal, bust generation, and hand illustration can all run in parallel after the reading text is ready. Firing them independently (no await) lets the user see the report text immediately while images load in progressively via 5-second polling.

**Why parallel bust generation?**
Sequential gpt-image-2 calls for 5 figures take ~300s — exceeding the 150s Supabase Edge Function idle timeout. `Promise.allSettled` across all 5 brings total time to ~30–50s. Failed individual figures fall back to placeholder images without blocking the others.

**Index-preserving bust array**
`bust_image_urls` is stored as a fixed-length array of 5 elements (nulls for failed slots). This ensures `bustImageUrls[i]` always corresponds to `historicalFigures[i]` in the report, even when individual generations fail.

**Schema versioning**
The report JSON follows a `contentSchemaVersion: "2.0.0"` contract validated by Zod on both the edge function (write path) and the Vercel rasterizer (read path). This prevents silent structural drift between the AI output and the rendering engine.

---

## Environment variables

### Vercel
| Variable | Purpose |
|---|---|
| `OPENAI_API_KEY` | Text + image generation |
| `SUPABASE_URL` | Supabase project URL |
| `SUPABASE_SERVICE_ROLE_KEY` | Server-side DB writes |
| `PALM_RENDERER_SECRET` | Shared secret between Edge Functions and Vercel API |
| `REMOVE_BG_API_KEY` | remove.bg background removal |

### Supabase Edge Functions (Secrets)
| Variable | Purpose |
|---|---|
| `OPENAI_API_KEY` | Text + image generation |
| `OPENAI_CALL_TIMEOUT_MS` | AbortController timeout (default 350000) |
| `SUPABASE_SERVICE_ROLE_KEY` | DB reads/writes from Deno |
| `PALM_RENDERER_URL` | Base URL of the Vercel deployment |
| `PALM_RENDERER_SECRET` | Shared secret to authenticate Vercel calls |

### Frontend (`.env`)
| Variable | Purpose |
|---|---|
| `VITE_SUPABASE_URL` | Supabase project URL |
| `VITE_SUPABASE_PUBLISHABLE_KEY` | Supabase anon key (public) |

---

## Local development

```bash
# Install dependencies
npm install

# Start Vite dev server
npm run dev

# Run edge functions locally (requires Docker)
npx supabase start
npx supabase functions serve

# Run tests
npm test
```

> **Note:** Local bust generation and poster rendering require a running Supabase instance and a valid `OPENAI_API_KEY`. Set `POSTER_PLACEHOLDER_BUSTS=1` to skip OpenAI image calls during development.

---

## Deployment

The project deploys automatically on push to `main`:

- **Vercel** builds the React SPA and deploys the `api/` serverless functions
- **Supabase** edge functions are deployed via CLI: `npx supabase functions deploy <function-name>`
- Database migrations: `npx supabase db push`

---

*For entertainment and reflection purposes. Interpretation inspired by classical and modern palmistry references.*
