<!-- ╔═══════════════════════════════════════════════════════════╗ -->
<!-- ║           SMART BOOKMARK JANITOR - Prompt v 0.5           ║ -->
<!-- ╚═══════════════════════════════════════════════════════════╝ -->

<overview>
  <![CDATA[
  **MVP — Smart Bookmark Janitor (SBJ)**  
  Chrome extension that scans ≤ 5 000 existing bookmarks, deduces topics via
  Gemini 2 Flash Lite, and rebuilds the tree into ≤ 30 semantic folders.
  
  **Key architecture**
    1. Chrome extension (MV3) — popup UI, Bookmarks API, background SW.
    2. Companion **worker** — *default:* **Local Native-Messaging host**
       (Node binary packaged with `pkg`).  
       • Cloud-Run variant is out-of-scope for v0.1, tracked in `roadmap.md`.
    3. SQLite queue (better-sqlite3) on the worker’s file-system.
    4. Fast metadata fetch → **Playwright fallback** (only if meta empty).  
    5. Batched Gemini Flash Lite (25 URLs/call); escalate to
       Gemini 2 Pro *only* if `confidence < 0 .75`, max 30 URLs total.
    6. Folder-builder → `chrome.bookmarks.move` or clean HTML export.

  **Safety nets** — full backup (`AllBookmarks.html`) on first run & one-click
  Undo (stores original parent IDs).
  ]]>
</overview>

<!-- ──────────────────────────────────────────────────────────── -->

<packages>
  <![CDATA[
  # Core runtime
  - node@20 (ESM)   pnpm
  - typescript@5   @types/chrome  zod-to-ts

  # Extension bundling
  - vite  react  react-dom
  - tailwindcss  postcss  autoprefixer
  - webextension-polyfill # Promise-based Chrome API
  - esbuild-crx     # CRX pack & hot-reload in dev

  # Worker runtime
  - playwright@1.44.0   # headless Chromium
  - better-sqlite3    # queue DB
  - google-ai      # Gemini client
  - pino        # JSON logs
  - dotenv      # config
  - pkg        # compile native host binary
  ]]>
</packages>

<!-- ──────────────────────────────────────────────────────────── -->

<dataContracts>
  <![CDATA[
  // src/shared/types.ts
  --------------------------------------------------------------
  import { z } from "zod"

  export const BookmarkRec = z.object({
    id: z.string(),
    title: z.string().optional(),
    url: z.string().url(),
    parentFolder: z.string().optional(),
    createdAt: z.number().int()            // ms since epoch
  })

  export const MetaSnippet = z.object({
    url: z.string().url(),
    title: z.string().optional(),
    description: z.string().optional(),
    ogTitle: z.string().optional(),
    ogDescription: z.string().optional(),
    bodySnippet: z.string().max(2_000)
  })

  export const LabelRes = z.object({
    summary: z.string().max(280),
    primary: z.string(),
    secondary: z.string().optional(),
    confidence: z.number().min(0).max(1)
  })

  /** Extension ⇄ Worker messages */
  export const Msg = z.discriminatedUnion("type", [
    z.object({ type: z.literal("enqueue"), rec: BookmarkRec }),
    z.object({ type: z.literal("progress"), done: z.number(), total: z.number() }),
    z.object({ type: z.literal("apply"), tree: z.any() }),
    z.object({ type: z.literal("backup"), html: z.string() }),
    z.object({ type: z.literal("error"), kind: z.string(), err: z.string(), url: z.string() })
  ])
  ]]>
</dataContracts>

<!-- ──────────────────────────────────────────────────────────── -->

<stateDiagrams>
  <![CDATA[
  **Queue row lifecycle**
  [pending] → fetchMeta → [fetched] → label → [labelled] → buildTree
       ↘ error(fetch/network/ai)            ↙
            [error]

  **Extension event flow (MV3)**
  ┌──── popup UI ────┐  click Scan          progress
  │                  │──────────────▶ bg service-worker ───────────▶ popup UI
  └──────────────────┘                    │            ▲
        ▲ apply tree                      │native msg  │enqueue new
        │                                 ▼            │
  chrome.bookmarks.*◀────── bg SW ◀──── native host (worker)
  ]]>
</stateDiagrams>

<!-- ──────────────────────────────────────────────────────────── -->

<interfaces>
  <![CDATA[
  ▸ **manifest.json**
  ```json
  {
    "manifest_version": 3,
    "name": "Smart Bookmark Janitor",
    "version": "0.1.0",
    "description": "AI-powered bookmark organiser",
    "icons": { "128": "icon128.png" },
    "permissions": ["bookmarks", "storage", "nativeMessaging"],
    "host_permissions": ["https://*/*"],
    "background": { "service_worker": "src/bg.ts" },
    "action": { "default_popup": "src/popup.html" }
  }
