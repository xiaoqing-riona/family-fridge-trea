# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

"我家冰箱" (Family Fridge) is a Chinese-language family fridge management web app. It lets family members track inventory, get freshness alerts, generate AI-powered meal recipes, and import shopping orders — all in a single HTML file with a Vercel serverless API backend.

## Architecture

This is a **zero-build-step project**:

- **`/index.html`** — The entire frontend: ~730 lines of vanilla JS and inline CSS. No framework, no bundler. All application state is in-memory JS variables (no localStorage, no database).
- **`/api/qwen.js`** — Vercel serverless function that proxies requests to Alibaba Cloud's DashScope API (Qwen AI models, OpenAI-compatible format). Requires `DASHSCOPE_API_KEY` environment variable.
- **`/vercel.json`** — Routes `/api/*` and sets CORS headers for API calls.
- **`/fridge-api/`** — Duplicate standalone deployment of the API only (its own `vercel.json` + `api/qwen.js`). Differs from root `/api/qwen.js` in that it lacks the OPTIONS preflight CORS handler.

## Development Commands

There is no build, lint, or test toolchain. To develop locally:

```bash
# Serve the frontend (any static server works)
npx serve .
# or
python3 -m http.server 8080
```

To test the Vercel API locally:

```bash
npm install -g vercel
DASHSCOPE_API_KEY=your_key vercel dev
```

The frontend's `callAI()` function always calls `/api/qwen` (relative URL), so it works against either the local Vercel dev server or a deployed environment.

## Key Frontend Conventions

**State**: All inventory data lives in the `inv` array of objects `{id, name, e (emoji), qty, u (unit), days (freshness), min (alert threshold)}`. State is reset on page reload — there is no persistence layer.

**AI calls**: All AI features go through `callAI(body, onSuccess, onCorsError, onOtherError)`. It wraps the `/api/qwen` fetch and normalizes errors. Text generation uses `qwen-plus`; image OCR uses `qwen-vl-max` with base64-encoded images in `image_url` format.

**JSON parsing**: `safeParseJSON(raw)` strips markdown fences and tries multiple extraction strategies — always use it when handling AI responses.

**Demo fallback**: When AI fails, the recipe feature falls back to `makeDemoResult()` using `DEMO_POOL`. OCR failures show a manual entry form. Order parsing falls back to `localParseOrder()` (regex-based). These fallbacks are intentional for offline/demo use.

**Views**: Three panels — `#panel-dad` (shopping/inventory), `#panel-mom` (freshness + recipe), `#panel-kid` (gamified vegetable tracker). Switching is done via `switchView(v)`.

**Sync badge**: `doSync()` is purely cosmetic — it animates a "syncing" spinner for 1.1 s then shows "已同步". There is no actual cloud sync.

## Deployment

The app deploys to Vercel. The required environment variable is:

- `DASHSCOPE_API_KEY` — Alibaba Cloud DashScope API key for Qwen model access

The root `vercel.json` configures both URL rewriting and CORS headers. The `/fridge-api/` subdirectory is a separate Vercel project intended for deploying the API independently.
