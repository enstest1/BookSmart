<xml>
<!-- ╔═══════════════════════════════════════════════════════════╗ -->
<!-- ║            CODER-AGENT BRIEF – Smart Bookmark Janitor     ║ -->
<!-- ║              Author: System Architect v0.5 (2025-05-23)   ║ -->
<!-- ╚═══════════════════════════════════════════════════════════╝ -->

<overview>
  <![CDATA[
  **Problem (MVP)**  
  Chrome users sit on thousands of uncategorised bookmarks. *Smart Bookmark
  Janitor (SBJ)* re-labels ≤ 5 000 bookmarks, clusters them into ≤ 30 semantic
  folders via **Gemini 2 Flash Lite**, then applies the structure with a click.
  Safety: full HTML backup + one-click Undo.

  **Value** — runs locally through a packaged Node worker → negligible cloud
  cost; advanced users may inject their own Gemini key.
  ]]>
</overview>

<packages>
  <![CDATA[
  ## Node tool-chain
  - node@20  pnpm
  - typescript@5  @types/chrome  zod-to-ts

  ## Extension build
  - vite
  - react  react-dom
  - tailwindcss  postcss  autoprefixer
  - webextension-polyfill   # Promise Chrome API
  - esbuild-crx              # CRX pack & hot-reload

  ## Worker runtime
  - playwright@1.44.0        # headless Chromium
  - better-sqlite3           # queue DB
  - google-ai                # Gemini client
  - pino                     # JSON logs
  - dotenv                   # env config
  - pkg                      # compile native host binary
  ]]>
</packages>

<dataContracts>
  <![CDATA[
  // src/shared/types.ts  (validated with Zod)
  ------------------------------------------------
  import { z } from "zod"

  export const BookmarkRec = z.object({
    id: z.string(),
    title: z.string().optional(),
    url: z.string().url(),
    parentFolder: z.string().optional(),
    createdAt: z.number().int()               // ms epoch
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

  /** Extension ⇄ Worker envelope */
  export const Msg = z.discriminatedUnion("type", [
    z.object({ type: z.literal("enqueue"),  rec: BookmarkRec }),
    z.object({ type: z.literal("progress"), done: z.number(), total: z.number() }),
    z.object({ type: z.literal("apply"),    tree: z.any() }),
    z.object({ type: z.literal("backup"),   html: z.string() }),
    z.object({ type: z.literal("error"),    kind: z.string(), err: z.string(), url: z.string() })
  ])
  ]]>
</dataContracts>

<stateDiagrams>
  <![CDATA[
  **Queue row lifecycle**
  [pending] → fetchMeta → [fetched] → label → [labelled] → buildTree
       ↘ error(network/ai)               ↙
            [error]

  **Extension event flow**
  ┌──── popup UI ────┐  click Scan            progress
  │                  │─────────────▶ bg service-worker ─────────▶ popup UI
  └──────────────────┘                      │          ▲
        ▲ apply tree                        │nativeMsg │enqueue new
        │                                   ▼          │
  chrome.bookmarks.*◀──── bg SW ◀──── local worker (native host)
  ]]>
</stateDiagrams>

<interfaces>
  <![CDATA[
  ▸ manifest.json
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
