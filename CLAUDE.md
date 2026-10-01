# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

GitDiagram turns a GitHub repository into an interactive Mermaid architecture diagram (and an optional one-minute narrated video). One Next.js 16 App Router application — no separate backend. Generation API lives in Next.js Route Handlers under `src/app/api/`. Vercel is the only live deployment; the `Dockerfile` and `railway.json` are a dormant Railway disaster-recovery recipe, not a live standby. See `docs/architecture.md` for the production architecture and `docs/dev-setup.md` for setup.

## Commands

Bun is the package manager and runtime (`bun install`; `bun ci` for frozen lockfile).

```bash
bun run dev                       # dev server (Turbopack) at localhost:3000
bun run test                      # all tests (vitest run)
bun run test src/server/generate/graph.test.ts   # single test file
bun run test:watch                # vitest watch mode
bun run lint                      # eslint (fails on any warning)
bun run typecheck                 # TypeScript 7 tsc --noEmit
bun run check                     # lint + typecheck
bun run format:check              # prettier (ts/js/mdx/css/json/yaml)
bun run knip                      # unused files/exports/dependencies
bun audit                         # bun audit (dependency vulnerabilities)
bun run build                     # production build
bun run check:video-tracing       # after build: video routes trace ffmpeg/Chromium only where needed
bun run perf:budget               # after build: bundle budgets
```

Full CI gate: `bun run lint && bun run typecheck && bun run format:check && bun run knip && bun audit && bun run test && bun run build && bun run check:video-tracing && bun run perf:budget`. `bun install` turns on `.githooks/pre-push` (format, lint, typecheck, knip). `workers/presence` has its own lockfile and CI job. `experiments/` (historical scripts) is excluded from typecheck and lint.

Vitest runs two projects (see `vitest.config.ts`): **server** (`node` env: `src/server/**`, `src/app/api/**`) and **client** (`jsdom` env with testing-library: everything else). Tests are colocated `*.test.ts(x)` files. Path alias `~` → `src/`.

## Architecture

### Generation pipeline (the core)

`/api/generate/stream` (`src/app/api/generate/stream/route.ts`, `runtime = "nodejs"`, `maxDuration = 300`) streams SSE through `src/server/generate/`:

1. **Ingestion** (`github.ts`) — fetch default branch, recursive tree, README via GitHub API. A partial (truncated) tree is kept and top-level folders it left out are read one level deep (≤8 requests); only an oversized README (>750 KB) is rejected.
2. **Source context and explanation** (`repository-context.ts`, `source-context.ts`, `source-excerpt.ts`, `source-references.ts`) — rank source files (tests/e2e/demos/generated excluded), read the top 12 plus up to 28 more in two rounds (the second follows what the first imports) within a 12-second enrichment deadline, bound the prompt's source text to 48k characters. Every file is resolved for the repository files it imports or names. Streams an evidence-grounded architecture explanation.
3. **Graph stage** (`graph-planner.ts`, `openai.ts`) — model returns a strict, size-bounded graph AST (groups/nodes/edges/labels/paths), validated by `graph.ts` (identifiers, edge endpoints, limits, every linked path checked against the real repo tree). Each edge carries a nullable `evidencePath`. Structural failures retry with focused feedback up to `MAX_GRAPH_ATTEMPTS`; a graph whose only faults are unresolvable paths is repaired in place (`stripUnknownGraphPaths`). Then `edge-evidence.ts` drops citations of files the model was not shown and fills uncited edges from the reference lists.
4. **Compilation** (`compileDiagramGraph` in `graph.ts`) — deterministic AST→Mermaid compiler with total text escaping and GitHub-only links. `mermaid-validator.ts` is the JSDOM-backed parser used **only by tests** (`mermaid.test.ts` contract tests) to keep the server bundle small — do not import it into production code.
5. **Client rendering** (`src/components/mermaid-diagram.tsx`, `src/features/diagram/mermaid-security.ts`) — sanitize source, render Mermaid with `securityLevel: "antiscript"` and `htmlLabels: false`, sanitize the resulting SVG with DOMPurify, then re-enforce the GitHub-only link allowlist. (`strict` is not usable here: it disables the `click` directives the diagram depends on, so the allowlist enforcement is what carries that weight.)

Other routes: `/api/generate/cost` (pre-run estimate), `/api/generate/cancel` (distributed cancellation via Redis), `/api/diagram-state` (persisted result contract), `/api/healthz`.

`generation-policy.ts` centralizes model/token/effort constants; `model-config.ts` selects the provider (`AI_PROVIDER` = openai | openrouter — both via the OpenAI SDK); `pricing.ts` + `complimentary-gate.ts` handle cost accounting and the free-tier daily token gate.

### Subsystems at a glance

Each subsystem has detailed mechanics somewhere else — this file is the index.

- **Explainer videos** (`src/{server,features,components}/explainer/`, `/api/video/*`, `/[username]/[repo]/video`, `/videos`) — feature-flagged (`VIDEO_EXPLAINER_ENABLED=1`, `NEXT_PUBLIC_VIDEO_EXPLAINER=1`). Director + per-scene designers write a 12–16 beat script; OpenRouter TTS voices it; Chromium + ffmpeg render MP4s as ~5 s segments joined into a loudness-normalized soundtrack. Limits, audience rules, paid-videos flow and storage live in `.env.example` and `src/server/explainer/limits.ts`. The Info panel's **Connections** list shows each arrow with its cited file linked on GitHub.
- **Operator dashboard** (`/admin`, `src/server/admin/`) — owner-only sign-in via `VIDEO_ADMIN_TOKEN`. Live presence is a separate Cloudflare Worker (`workers/presence/`, deployed with `bunx wrangler deploy`). Live switches live in a Redis hash with a 1 s per-instance cache (`controls.ts`). Live feed, "view as", counters, and `view-as` cookie all live under `src/server/admin/`.
- **MCP server** (`src/server/mcp/`, `/mcp`) — public read-only remote MCP server at `https://gitdiagram.com/mcp`. Tools (`server.ts`; edit their descriptions with care — they decide whether agents pick GitDiagram): `get_repository_diagram`, `find_repository_diagrams`, `get_explainer_video`. The MCP App diagram view is `src/mcp-app/diagram-view.ts`, bundled by `scripts/build-mcp-app.mjs` (esbuild) into gitignored `public/mcp-app/`. `server.json` is the MCP Registry entry — keep its `version` equal to `MCP_SERVER_VERSION`.
- **Search engines and AI agents** — repository pages at `src/app/[username]/[repo]/page.tsx`; the Markdown twin at `src/app/[username]/[repo]/llms.txt/route.ts` (rewrite in `src/proxy.ts`, ISR 6 h); guide at `/visualize-codebase` (`src/features/guide/content.ts`); llms.txt at `/llms.txt`; `robots.ts` names the AI crawlers. `JsonLd` (`src/components/json-ld.tsx`) escapes `<`, `>`, `&`.
- **Continue with GitHub** (`src/server/github-connect/`, `/api/github/connect`, `/api/github/callback`) — separate public GitHub App for private repos, gated by `NEXT_PUBLIC_GITHUB_CONNECT=1`. Uses sealed (AES-GCM, key from `CACHE_KEY_SECRET`) HttpOnly cookies; private diagrams stored under `github-user:<id>` (`githubStorageKey`).
- **Abuse control** — per-IP fixed-window limiters in `generate/rate-limit.ts` before the daily quota; both fail open on Redis errors because the daily quota still bounds total spend. `src/server/http/` holds same-origin and credentials helpers (mutating routes require same-origin; user GitHub tokens travel per-request, never persisted server-side).

### Layering

- `src/server/` — server-only code (imports `server-only`): generation pipeline, GitHub auth (`github-auth.ts` supports single PAT, PAT pool, or GitHub App), storage, HTTP guards, OG image generation.
- `src/server/storage/` — R2 (`r2.ts`, `artifact-store.ts`) for diagram artifacts, with a **separate private namespace derived from `CACHE_KEY_SECRET`** for private repos (`cache-key.ts`); Upstash Redis (`upstash.ts`) for quota (`quota-store.ts`), cancellation, short-lived failure state (`status-store.ts`), and `distributed-lock.ts` (newest-session-wins persistence in `generation-persistence.ts`).
- `src/features/` — client/shared domain logic per feature (diagram SSE parsing, export, github-url parsing, credentials, browse catalog). The graph AST schema/types in `src/features/diagram/graph.ts` are shared between server validation and client.
- `src/hooks/useDiagram.ts` + `src/hooks/diagram/` — orchestrate the client generation lifecycle (cost check → stream → render → persist).
- `src/app/[username]/[repo]/` — the diagram page (ISR `revalidate` 6 h plus on-demand invalidation; the root layout's star-count fetch revalidates every 5 min, which caps the effective lifetime; a page rendered after a failed stored-state read is cached 60 s) (and `video/`, the watch page); also `/browse`, `/videos`, `/advertise` (`/sponsor` redirects there), `/admin`, and `/out/[campaign]` (sponsor click-through).

### Environment

Copy `.env.example` → `.env`. Minimum to run generation locally: R2 vars, `CACHE_KEY_SECRET`, Upstash vars, and one AI provider key. `.env.example` lists every setting with its default; `docs/dev-setup.md` groups them and covers deploy. `docs/deployment-failover.md` covers the Railway recovery recipe.

## Conventions

- Prettier with `prettier-plugin-tailwindcss` (`bun run format:write`); ESLint 10 flat config (`eslint.config.mjs`).
- **Server code must not leak into client bundles** — keep it under `src/server/` behind `server-only`.
- **Diagram output safety is defense-in-depth** (server validation → deterministic compiler → client sanitization). Changes to any layer should keep the others intact and are covered by `mermaid-security.test.ts` and the compiler contract tests.

<!-- BEGIN:nextjs-agent-rules -->

# This is NOT the Next.js you know

This version has breaking changes — APIs, conventions, and file structure may all differ from your training data. Read the relevant guide in `node_modules/next/dist/docs/` (resolved from this file's directory; in monorepos the `next` package may not be visible from the repo root) before writing any code. Heed deprecation notices.

This block is written and re-added by `next dev` — verify at `node_modules/next/dist/server/lib/generate-agent-files.js`. Removing it from a diff only re-creates the uncommitted change; committing it with your work keeps the tree clean.

<!-- END:nextjs-agent-rules -->
