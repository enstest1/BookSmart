<xml>
  <overview>
    <![CDATA[
    **MVP** — *Smart Bookmark Janitor (SBJ)*: a Chrome‑extension + companion worker that scans a user’s unruly bookmark tree (≈1 000+) and reorganises it into semantic folders powered by Gemini 1.5.  

```
**Key architecture**
  1. Chrome extension (MV3): UI + Bookmarks API + background SW.
  2. Companion **worker** (choose **one**):  
     • *Local Native‑Messaging host* (packaged Node binary via `pkg`) **OR**  
     • *Serverless* (Cloud Run/FastAPI) with REST endpoints.  
  3. SQLite queue shared via Native‑Messaging or HTTPS.  
  4. Playwright crawler + Gemini Flash→Pro classifier.  
  5. Folder‑tree builder → Bookmark moves or clean HTML export.  

**Safety nets**: automatic full backup before first re‑org and one‑click Undo.
]]>
```

  </overview>

  <!--────────────────────────────────────────────────────────────-->

  <packages>
    <![CDATA[
    # Core runtime
    - node@20 (ESM)  &  pnpm
    - typescript@5

```
# Extension bundling
- vite
- react  +  react‑dom
- tailwindcss  +  postcss  +  autoprefixer
- webextension‑polyfill  # Promise‑based Chrome API
- esbuild‑crx            # CRX pack & Hot‑reload in dev

# Worker runtime
- playwright              # headless Chromium
- better‑sqlite3          # queue DB
- google‑ai               # Gemini client
- pino                    # JSON logs
- dotenv                  # config
- pkg                     # compile native host binary (if local mode)
]]>
```

  </packages>

  <!--────────────────────────────────────────────────────────────-->

  <dataContracts>
    <![CDATA[
    // src/shared/types.ts
    ------------------------------------------------------------
    import { z } from "zod";

```
export const BookmarkRec = z.object({
  id: z.string(),
  title: z.string().optional(),
  url: z.string().url(),
  parentFolder: z.string().optional()
});

export const MetaSnippet = z.object({
  url: z.string().url(),
  title: z.string().optional(),
  description: z.string().optional(),
  ogTitle: z.string().optional(),
  ogDescription: z.string().optional(),
  bodySnippet: z.string().max(2000)
});

export const LabelRes = z.object({
  summary: z.string().max(280),
  primary: z.string(),
  secondary: z.string().optional(),
  confidence: z.number().min(0).max(1)
});

/** Extension ⇄ Worker messages */
export const Msg = z.discriminatedUnion("type", [
  z.object({ type: z.literal("enqueue"), rec: BookmarkRec }),
  z.object({ type: z.literal("progress"), done: z.number(), total: z.number() }),
  z.object({ type: z.literal("apply"), tree: z.any() }),
  z.object({ type: z.literal("backup"), html: z.string() }),
  z.object({ type: z.literal("error"), err: z.string(), url: z.string() })
]);
]]>
```

  </dataContracts>

  <!--────────────────────────────────────────────────────────────-->

  <stateDiagrams>
    <![CDATA[
    **Queue row lifecycle**
    [pending] → fetchMeta → [fetched] → label → [labelled]
       ↘ error                     ↙
          [error]

```
**Extension event flow (MV3)**
┌──── popup UI ────┐   click Scan        progress
│                  │──────────────▶ bg service‑worker ────────────▶ popup UI
└──────────────────┘                    │           ▲
       ▲ apply tree                     │native msg │enqueue new
       │                                ▼           │
chrome.bookmarks.*◀────── bg SW ◀──── native host / cloud worker
]]>
```

  </stateDiagrams>

  <!--────────────────────────────────────────────────────────────-->

  <interfaces>
    <![CDATA[
    ▸ **manifest.json** (root)
    ```json
    {
      "manifest_version":3,
      "name":"Smart Bookmark Janitor",
      "version":"0.1.0",
      "description":"AI‑powered bookmark organiser",
      "permissions":["bookmarks","storage","nativeMessaging"],
      "host_permissions":["https://*/*"],
      "background":{"service_worker":"src/bg.ts"},
      "action":{"default_popup":"src/popup.html"}
    }
    ```

````
▸ **Background SW (src/bg.ts)** — *pseudo‑interface*
  - `onInstalled`: dump bookmark tree & send `enqueue` list to worker.
  - `chrome.bookmarks.onCreated`: relay new URL to worker.
  - `chrome.runtime.onMessage`: {cmd:'scan'|'apply'|'undo'} → forward.
  - Bridge: `browser.runtime.sendNativeMessage` **or** `fetch('https://api.sbjanitor.dev/enqueue')`.

▸ **Popup UI**
  - small React app. Components: `ApiKeyForm`, `ProgressBar`, `ActionButtons`.
  - uses `browser.runtime.sendMessage({cmd:'scan'})` etc.

▸ **Native host descriptor** (`com.sbjanitor.host.json` in `~/Library/Application Support/Google/Chrome/NativeMessagingHosts`)
```json
{
  "name":"com.sbjanitor.host",
  "description":"SBJ local worker",
  "path":"<install_dir>/sbj-host",
  "type":"stdio",
  "allowed_origins":["chrome-extension://__EXT_ID__/"]
}
```
]]>
````

  </interfaces>

  <!--────────────────────────────────────────────────────────────-->

  <geminiUsage>
    <![CDATA[
    // worker/src/llm.ts  (unchanged from previous version)
    ]]>
  </geminiUsage>

  <!--────────────────────────────────────────────────────────────-->

  <crawlerSnippet>
    <![CDATA[
    // worker/src/crawler.ts  (unchanged)
    Note: Playwright runs **inside the worker**, never in‑extension.
    ]]>
  </crawlerSnippet>

  <!--────────────────────────────────────────────────────────────-->

  <bridgeSnippet>
    <![CDATA[
    // Extension side (bg.ts)
    import browser from "webextension-polyfill";

```
async function sendToHost(msg) {
  /* if local mode */
  return browser.runtime.sendNativeMessage("com.sbjanitor.host", msg);
}

async function sendToCloud(msg) {
  await fetch("https://api.sbjanitor.dev/msg", {
    method:"POST", headers:{"content-type":"application/json"}, body:JSON.stringify(msg)
  });
}
]]>
```

  </bridgeSnippet>

  <!--────────────────────────────────────────────────────────────-->

  <pseudocode>
    <![CDATA[
    BG::onInstalled ⇒ backupBookmarks() ⇒ seedQueue()

```
Worker::loop(batch=100):
  rows = queue.next('pending')
  parallel rows { meta = fetchMeta(url); label = classify(meta); queue.update(row) }
  postMessage({type:'progress', done, total})

BG::onMessage 'apply' ⇒ tree = buildTree(labelledRows); applyFolders(tree); saveUndoMap()
]]>
```

  </pseudocode>

  <!--────────────────────────────────────────────────────────────-->

  <logging>
    <![CDATA[
    • **Worker**: pino JSON to stdout → file (if local) or Cloud Logging (if serverless).  Levels: DEBUG(fetch), INFO(batch done), WARN(retry), ERROR(fatal).
    • **Extension BG**: `console.info` for lifecycle, send errors to popup via `browser.runtime.sendMessage({type:'error', err})`.
    ]]>
  </logging>

  <!--────────────────────────────────────────────────────────────-->

  <acceptance>
    <![CDATA[
    1. `pnpm dev` launches extension in Chrome, hot‑reloads popup.
    2. First run backs up bookmarks to `backup_*.html` and queue `>0` rows.
    3. After Scan→Apply, tree has ≤30 folders and Undo restores original state.
    4. End‑to‑end test (vitest) with 20 mock URLs completes <120 s.
    5. CRX passes Chrome Web Store validation (MV3, no remote code) and native host installer runs on macOS + Windows.
    ]]>
  </acceptance>
</xml>
