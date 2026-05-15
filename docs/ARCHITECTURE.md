# Suno Tracks Exporter — 架構分析

> 來源版本：1.0.5 (`pooodjgdoajgkhcnjfabfbdifeakoocm`)
> 作者：VogelCodes（buymeacoffee 連結指向 `VogelCodes`）
> Manifest V3 Chrome Extension

---

## 1. 高層架構

這是一個典型的 **MV3 三層 Chrome Extension**，靠著「**MITM 自家頁面的 Authorization header**」這招把使用者瀏覽器借來當成 Suno API 的代理：

```
┌─────────────────────────────────────────────────────────────────────────┐
│ popup.html / popup.js               (使用者按 toolbar icon 觸發)         │
│   ↓ chrome.tabs.sendMessage("showDownloader")                            │
├─────────────────────────────────────────────────────────────────────────┤
│ content.js  (injected into https://suno.com/*)                           │
│   • 在頁面內 hook window.fetch 和 XMLHttpRequest                          │
│   • 渲染 modal UI（純 DOM/inline-style，非 React）                        │
│   • 透過 chrome.runtime.sendMessage 呼叫 service worker                   │
│   • 用 URL.createObjectURL + <a download> 處理大檔下載                    │
├─────────────────────────────────────────────────────────────────────────┤
│ background.js  (service worker)                                          │
│   • chrome.webRequest.onBeforeSendHeaders 攔截 auth token                 │
│   • 直接 fetch Suno API（帶 Bearer token + browser-token + device-id）    │
│   • chrome.downloads.download 處理檔案落地                                │
│   • importScripts("metadata.js") → ID3v2 / RIFF tag 嵌入                  │
└─────────────────────────────────────────────────────────────────────────┘
       ↕  chrome.storage.local                ↕  Suno REST API
  (authToken, deviceId, tracks_<ws>, ...)    (studio-api-prod.suno.com)
```

兩個核心 trick：

1. **Token 收集**：popup 不知道你的 token，content script 也不直接看，而是 `webRequest.onBeforeSendHeaders` 在背景靜態監聽。只要使用者瀏覽 suno.com，頁面自己會發 API call，header 裡的 `Authorization: Bearer ...` 就被剝下來存進 `chrome.storage.local.authToken`。
2. **大檔回傳的迂迴**：service worker 不能用 `URL.createObjectURL`，所以 background 把音檔 fetch 回來、嵌好 ID3 tag、轉 base64，再透過 `tabs.sendMessage` 丟給 content script，由 content script 在 DOM 中 `<a download>` 觸發儲存。檔案 > 50 MB 時自動 fallback 成「直接下載 CDN URL（沒嵌 metadata）」。

### 完整元件圖

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║                                 CHROME BROWSER                                    ║
║                                                                                   ║
║  ┌─────────────────────────┐         ┌─────────────────────────────────────────┐ ║
║  │   TOOLBAR POPUP         │         │  SUNO.COM TAB  (https://suno.com/*)     │ ║
║  │   ──────────────────    │         │  ─────────────────────────────────────  │ ║
║  │   popup.html (400 px)   │         │                                          │ ║
║  │   popup.js              │         │   ┌──────────────────────────────────┐ │ ║
║  │   ┌───────────────────┐ │         │   │  Page Main World (Suno React)    │ │ ║
║  │   │ ✓ Token Found     │ │         │   │  - 自己發 fetch() to Suno API    │ │ ║
║  │   │ [Extract Token]   │ │         │   │  - 帶 Authorization: Bearer xxx  │ │ ║
║  │   │ [Open Downloader] │ │         │   └─────────────┬────────────────────┘ │ ║
║  │   └────────┬──────────┘ │         │                 │ (in transit)         │ ║
║  └────────────┼────────────┘         │                 ▼                      │ ║
║               │ "showDownloader"     │   ┌──────────────────────────────────┐ │ ║
║               │ tabs.sendMessage     │   │ Content Script (isolated world)  │ │ ║
║               └────────────────────► │   │ ──────────────────────────────── │ │ ║
║                                      │   │ content.js  (4742 lines)         │ │ ║
║                                      │   │ • Modal UI (inline-style DOM)    │ │ ║
║                                      │   │ • JSZip → CSV/JSON/scripts.zip   │ │ ║
║                                      │   │ • Hook fetch/XHR (ineffective:   │ │ ║
║                                      │   │   isolated world — webRequest    │ │ ║
║                                      │   │   below is what actually works)  │ │ ║
║                                      │   │ • Large-file save via            │ │ ║
║                                      │   │   URL.createObjectURL + <a dl>   │ │ ║
║                                      │   └────────┬─────────────────────────┘ │ ║
║                                      └────────────┼───────────────────────────┘ ║
║                                                   │ runtime.sendMessage           ║
║                                                   ▼                               ║
║  ┌──────────────────────────────────────────────────────────────────────────────┐║
║  │             BACKGROUND SERVICE WORKER  (background.js, MV3)                  │║
║  │             ─────────────────────────────────────────────────                │║
║  │                                                                              │║
║  │  ┌────────────────────────┐  ┌──────────────────────┐  ┌──────────────────┐│║
║  │  │  webRequest Sniffer    │  │  Message Router      │  │  Download Mgr    ││║
║  │  │  ────────────────────  │  │  ──────────────────  │  │  ──────────────  ││║
║  │  │  onBeforeSendHeaders   │  │  getTracks           │  │  chrome.downloads││║
║  │  │   → 攔 Authorization   │  │  getToken/saveToken  │  │   .download()    ││║
║  │  │   → storage.authToken  │  │  downloadFile        │  │  downloadFile-   ││║
║  │  │  (host-permission 內   │  │  initiateWavConv     │  │   names Map      ││║
║  │  │   被動觀察，不修改)    │  │  pollWavFile         │  │   (URL → name)   ││║
║  │  └──────────┬─────────────┘  │  getWorkspaces(Page) │  │  onDetermining-  ││║
║  │             │                │  refresh* / cache*   │  │   Filename       ││║
║  │             │                │  fetchAllMetadata    │  │  onChanged →     ││║
║  │             │                │  getTrackMetadata    │  │   markDownloaded ││║
║  │             │                └──────────┬───────────┘  └────────┬─────────┘│║
║  │             │                           │                       │          │║
║  │             │            ┌──────────────▼───────────────────────▼────────┐ │║
║  │             │            │  Suno API Client + Rate-Limit Engine          │ │║
║  │             │            │  ──────────────────────────────────────────── │ │║
║  │             │            │  retryWithBackoff(fn, ctx)                    │ │║
║  │             │            │  • 429 → exp backoff + jitter (2 → 30 s)      │ │║
║  │             │            │  • 首次重試把所有 delay × 2 並持久化          │ │║
║  │             │            │  • Headers: Bearer + browser-token +          │ │║
║  │             │            │    device-id + origin + referer               │ │║
║  │             │            └──────────────┬────────────────────────────────┘ │║
║  │             │                           │                                   │║
║  │             │            ┌──────────────▼────────────────────────────────┐ │║
║  │             │            │  metadata.js  (importScripts)                  │ │║
║  │             │            │  ──────────────────────────────────────────── │ │║
║  │             │            │  • extractMetadataFromTrack(apiData)           │ │║
║  │             │            │  • embedMetadata()  ── 手刻 ID3v2.3 encoder   │ │║
║  │             │            │  • Frames: TIT2 TPE1 TALB TDRC TCON TBPM       │ │║
║  │             │            │            TKEY COMM USLT APIC                 │ │║
║  │             │            │  • embedWavMetadata() ── ID3 append at EOF     │ │║
║  │             │            │  • encodeSynchsafe / fetchCoverArt             │ │║
║  │             │            └────────────────────────────────────────────────┘ │║
║  │             ▼                                                                │║
║  └──────────────────────────────────────────────────────────────────────────────┘║
║                                                                                   ║
║  ┌──────────────────────────────────────┐  ┌─────────────────────────────────┐ ║
║  │  chrome.storage.local                │  │  localStorage (suno.com only)   │ ║
║  │  ────────────────────────             │  │  ─────────────────────────────  │ ║
║  │  authToken                           │  │  downloaded_<clipId>_mp3        │ ║
║  │  deviceId          (UUID v4)         │  │  downloaded_<clipId>_wav        │ ║
║  │  rateLimitConfig   (persistent)      │  │  filterStems / filterUpload /   │ ║
║  │  tracks_<workspaceId>                │  │   filterLiked / filterNotLiked  │ ║
║  │  tracks_<workspaceId>_timestamp      │  │                                 │ ║
║  │  track_metadata_<clipId>             │  │                                 │ ║
║  └──────────────────────────────────────┘  └─────────────────────────────────┘ ║
╚═══════════════════════════════════════════╤═══════════════════════════════════╝
                                            │ HTTPS (host_permissions)
                                            ▼
   ┌──────────────────────────────────────────────────────────────────────────┐
   │                            SUNO BACKEND                                   │
   │  ─────────────────────────────────────────────────────────────────────    │
   │  studio-api-prod.suno.com                                                 │
   │   ├─ GET  /api/project/me?page=N            → list workspaces             │
   │   ├─ POST /api/feed/v3                      → list tracks (cursor paged)  │
   │   ├─ GET  /api/feed/?ids=<clipId>           → single track full metadata  │
   │   ├─ POST /api/gen/<id>/convert_wav/        → kick off WAV conversion     │
   │   ├─ GET  /api/gen/<id>/wav_file/           → poll until 200 + wav_url    │
   │   └─ POST /api/billing/clips/<id>/download/ → official download counter   │
   │                                                                           │
   │  cdn1.suno.ai / cdn2.suno.ai                                              │
   │   ├─ /<clipId>.mp3                          → audio file (CDN)            │
   │   └─ <image_url>                            → cover art (JPEG)            │
   │                                                                           │
   │  clerk.suno.com / auth.suno.com                                           │
   │   └─ Clerk (auth provider) — Bearer token 在這簽發，被 extension 觀察    │
   └──────────────────────────────────────────────────────────────────────────┘
```

### MP3 下載 lifecycle（橫向資料流）

```
 USER ─click→ [Download MP3] ──┐
                               │
                               ▼
 ┌──────────── content.js ─────────────┐    ┌─────────── background.js ──────────────┐
 │ downloadSelectedTracks(tracks)      │    │                                         │
 │   for each track:                   │    │                                         │
 │     ↓                               │    │                                         │
 │   sendMessage("downloadFile",       ├───►│ downloadFile → downloadFileWithMetadata │
 │     {url, filename, clipId, "mp3"}) │    │   ↓                                     │
 │                                     │    │ ┌─ POST /api/billing/.../download/ ──┐ │
 │                                     │    │ │   (non-critical, swallow errors)   │ │
 │                                     │    │ └────────────────────────────────────┘ │
 │                                     │    │   ↓                                     │
 │                                     │    │ fetchTrackMetadata(clipId)              │
 │                                     │    │   GET /api/feed/?ids=<clipId>           │
 │                                     │    │   ↓                                     │
 │                                     │    │ fetch(audio_url) → Blob                 │
 │                                     │    │   ↓                                     │
 │                                     │    │ embedMetadata(blob, metadata)           │
 │                                     │    │   (ID3v2.3 + APIC + USLT + COMM)        │
 │                                     │    │   ↓                                     │
 │                                     │    │ blob → base64                           │
 │                                     │    │   ↓                                     │
 │ handleLargeFileDownload(req) ◄──────┼────┤ tabs.sendMessage("downloadLargeFile-    │
 │   base64 → Blob                     │    │   WithMetadata", {blobData,             │
 │   URL.createObjectURL → <a download>│    │   filename, metadata})                  │
 │   .click()                          │    │                                         │
 │   ↓                                 │    │                                         │
 │ also save .txt sidecar (raw JSON)   │    │                                         │
 │   ↓                                 │    │                                         │
 │ markTrackAsDownloaded(clipId,"mp3") │    │                                         │
 │ refreshTrackUI(clipId)              │    │                                         │
 │   (灰掉、加 ✓、disable buttons)     │    │                                         │
 └─────────────────────────────────────┘    └─────────────────────────────────────────┘

 Fallback chain（任何一層爆掉就 degrade）：
   blob→base64 too large (>50 MB) ──► direct chrome.downloads.download(audio_url)
                                       (no metadata embedding)
   tabs.sendMessage failed ─────────► same fallback as above
   "Receiving end does not exist" ──► same fallback as above
   tab is not on suno.com ──────────► data: URL（service worker 自己下載，
                                       但 >2 MB 又會 fallback 回 CDN URL）
```

### WAV 轉檔 lifecycle（差別在多兩個 round-trip + 跳過 metadata 嵌入）

```
 USER ─click→ [Download WAV]
   ↓
 content.js: convertAndDownloadWav(clipId, filename)
   ↓
   ① initiateWavConversion(clipId)
        → POST /api/gen/<clipId>/convert_wav/             (期待 204 No Content)
   ↓
   ② pollWavFile(clipId)
        → loop max 60 × 2 s:
            GET /api/gen/<clipId>/wav_file/
            404 → continue
            200 + { wav_file_url } → return url
   ↓
   ③ downloadTrack(wavUrl, filename, clipId, "wav")
        → 走 downloadFileWithMetadata 但 format === "wav" 時跳過 ID3 嵌入，
          直接 chrome.downloads.download(wavUrl)  + .txt sidecar
```

---

## 2. 檔案清單與責任

| 檔案 | 大小 | 角色 |
|------|-----:|------|
| `manifest.json` | 1.6 KB | MV3 宣告：service worker、content script、host_permissions、CSP |
| `background.js` | 49 KB | Service worker — token 攔截、Suno API 代理、下載排程、rate-limit 退避 |
| `content.js` | 176 KB / 4742 行 | 注入 suno.com 的 UI + 大檔下載 helper + token hook |
| `content.css` | <1 KB | 只有 modal hover 微樣式（主要樣式是 inline） |
| `metadata.js` | 19 KB | ID3v2.3 tag builder（MP3）+ RIFF/ID3 hybrid（WAV） |
| `popup.html` / `popup.js` | 7 KB | Toolbar 浮窗：顯示 token 狀態 + 「Open Downloader」按鈕 |
| `api.js` | 2 KB | **未使用**的孤兒檔（Auth-Sharing backend RPC client，沒被任何檔案 import） |
| `jszip.min.js` | 97 KB | 第三方 — content script 用來打包 CSV/JSON/腳本成 .zip |
| `_locales/{en,es,pt_BR,pt_PT}/messages.json` | 25-27 KB | i18n 字串（chrome.i18n） |
| `create-icons.html` | 2.9 KB | 開發用工具（產 PNG icon），不會被使用者載入 |
| `_metadata/` | - | Chrome Web Store 自動產生的 hash 簽名 |

---

## 3. Manifest 設計

```jsonc
{
  "manifest_version": 3,
  "permissions": [
    "storage",        // ← 存 token、cache、設定
    "downloads",      // ← chrome.downloads API
    "tabs",           // ← 找 active suno.com tab 來 sendMessage
    "webRequest",     // ← 攔截 Authorization header（被動觀察，不修改）
    "unlimitedStorage" // ← 大量 tracks/metadata cache
  ],
  "host_permissions": [
    "https://suno.com/*",
    "https://*.suno.com/*",
    "https://*.suno.ai/*",
    "https://clerk.suno.com/*",        // ← Clerk 是 Suno 的 auth provider
    "https://auth.suno.com/*",
    "https://studio-api.prod.suno.com/*",
    "https://studio-api-prod.suno.com/*", // ← 兩種主機名都列（API 主要用後者）
    "https://cdn1.suno.ai/*",
    "https://cdn2.suno.ai/*"
  ],
  "content_security_policy": {
    "extension_pages": "script-src 'self'; object-src 'self'"
  },
  "content_scripts": [{
    "css": ["content.css"],
    "js":  ["jszip.min.js", "content.js"],  // ← JSZip 先載
    "matches": ["https://suno.com/*"]
  }],
  "background": { "service_worker": "background.js" }
}
```

注意：

- **沒有要 `scripting` 權限**（README 寫錯）— 整支 extension 不用 `chrome.scripting.executeScript`，全靠靜態注入的 content script。
- `key` 欄位是固定 public key，這樣本機 unpacked 安裝跟 Web Store 上裝的 extension ID 一致（`pooodjgdoajgkhcnjfabfbdifeakoocm`）。
- `webRequest` 只用 `onBeforeSendHeaders`（讀取），**沒有用 `webRequestBlocking`**（MV3 已不允許 blocking）。

---

## 4. Token 攔截機制（雙保險）

這是整個 extension 的命門。設計成 **defense in depth**：

### 4a. Background — `webRequest` 被動嗅探（`background.js:26-53`）

```js
chrome.webRequest.onBeforeSendHeaders.addListener((details) => {
  if (details.url.includes("studio-api-prod.suno.com")
      || details.url.includes("auth.suno.com")
      || details.url.includes("clerk.suno.com")) {
    const authHeader = details.requestHeaders.find(h =>
      h.name.toLowerCase() === "authorization");
    if (authHeader) {
      const token = authHeader.value.replace("Bearer ", "");
      chrome.storage.local.set({ authToken: token });
    }
  }
}, { urls: [...] }, ["requestHeaders"]);
```

被動、零干擾，不需要使用者按任何按鈕。

### 4b. Content script — `fetch` / XHR monkey-patch（`content.js:18-89`）

```js
window.fetch = function (...args) {
  // 檢查 url 是否 suno API + options.headers 是否有 Authorization
  // 有的話 chrome.runtime.sendMessage({ action: "saveToken", token })
  return originalFetch.apply(this, args);
};
```

註解寫 *"Method 3 removed to avoid inline script CSP violations"*——原本應該還有第三招透過 inject `<script>` 到頁面 main world，但因 suno.com 的 CSP 被擋下。

實際上，**因為 content script 跑在 isolated world，它 hook 的是 isolated world 的 fetch，看不到頁面 main world 自己的 fetch call**。所以實質上 token 主要還是靠 4a 的 webRequest 拿到——4b 是個失效但無害的備援。

### 4c. Popup 的「Extract Token」按鈕（`popup.js:73-126`）

是個 fallback UX：把當前 tab 導向 `https://suno.com/?wid=default`，那頁面會主動發 API call，於是 4a 就抓到 token。然後 popup 每秒輪詢 background，最多 10 次。

---

## 5. Suno API 呼叫對照表

所有 API 都從 background 發出，headers 必帶這四件套：

```
authorization: Bearer <token>
browser-token: {"token":"<base64({timestamp})>"}    ← 每次重新算
device-id: <uuid v4>                                ← 一次性生成存 storage
origin: https://suno.com
referer: https://suno.com/
```

| 用途 | Method | URL | 來源函式 |
|------|--------|-----|----------|
| 列 workspaces（含分頁） | GET | `studio-api-prod.suno.com/api/project/me?page=N&sort=created_at&show_trashed=false` | `fetchWorkspacesPage` (`background.js:413`) |
| 列某 workspace 的 tracks | POST | `studio-api-prod.suno.com/api/feed/v3` body `{cursor, limit:100, filters:{workspace:{workspaceId}, disliked:"False", trashed:"False"}}` | `fetchTracks` (`background.js:522`) |
| 單一 clip 的詳細 metadata | GET | `studio-api-prod.suno.com/api/feed/?ids=<clipId>` | `fetchTrackMetadata` (`background.js:1352`) |
| 啟動 WAV 轉檔 | POST | `studio-api-prod.suno.com/api/gen/<clipId>/convert_wav/` | `initiateWavConversion` (`background.js:1128`) |
| 輪詢 WAV 結果 | GET | `studio-api-prod.suno.com/api/gen/<clipId>/wav_file/` → `{wav_file_url}` | `pollWavFile` (`background.js:1158`)，每 2s × 最多 60 次 |
| 觸發官方 download counter | POST | `studio-api-prod.suno.com/api/billing/clips/<clipId>/download/` | `downloadFileWithMetadataFromBillingEndpoint` (`background.js:672`)；錯了也不擋 |
| 抓 MP3 / 封面 | GET | `cdn1.suno.ai/<clipId>.mp3` 或 `track.audio_url` 或 `track.image_url` | 同上 |

**那個 `studio-api.prod.suno.com`（帶點的）只出現在 content.js token-sniff 的 URL match 裡，所有實際 fetch 都用 `studio-api-prod.suno.com`（dash）。**

---

## 6. Rate-limit 處理（`background.js:292-407`）

一個**有狀態、會學習**的 backoff 系統：

```js
let rateLimitConfig = {
  baseDelay: 300,        // 每筆 request 之間
  workspaceDelay: 500,   // workspace 之間
  trackDelay: 300,       // tracks 分頁之間
  metadataDelay: 500,    // metadata 抓取之間
  maxRetries: 5,
  initialBackoff: 2000,
  maxBackoff: 30000,
  backoffMultiplier: 2,
};
```

`retryWithBackoff(fn, context)` 包住每個 API call：

1. 接到 429 → 算 backoff `min(initialBackoff × 2^attempt, maxBackoff) + jitter(0-1000ms)`
2. **第一次 retry 時順便把所有 delay × 2**（最多 capping），並寫回 storage——下次冷啟 extension 也會 carry over 這個保守設定
3. notify content script 顯示 toast（`showRateLimitNotification`，content.js:2097）
4. 最多重試 5 次後 throw

設計很扎實——這是個會被使用者跑批次的工具，被 ban 過顯然不只一次。

---

## 7. UI 流程（content.js modal）

整個 UI 是 **inline-style 純 DOM**（沒用任何 framework），透過 `prefers-color-scheme: dark` query 即時讀深淺色。Modal 是個 `position: fixed` 全螢幕罩，內部兩個 tab：

### Tab 1: All Tracks（主流程）

```
loadTracksIntoModal()
  ├─ attemptTokenExtraction()           ← 等 500ms 讓頁面自然觸發 API
  ├─ for page in pages:
  │    chrome.runtime.sendMessage("getWorkspacesPage")
  ├─ for workspace in workspaces:
  │    1) getCachedTracks(workspace.id)  ← 先看 storage cache（沒 TTL）
  │    2) 若無 cache 才呼叫 getTracks 分頁拉完
  │    3) cacheTracks(workspace.id, tracks)
  ├─ filter(status === "complete" && audio_url)
  ├─ 渲染 workspace-group 摺疊清單
  ├─ createFilterComponent()             ← Hide: stems/upload/liked/notLiked
  ├─ startBackgroundMetadataFetch()      ← 非阻塞，逐首抓 detail metadata
  └─ 註冊一票 event listeners
```

UI 元素：

- **每首 track**：checkbox、標題、MP3 button、WAV button、unlock button（🔓 重設下載狀態）
- **workspace 標題**：摺疊箭頭、全選 checkbox、track 數、🔄 refresh、cache timestamp
- **左側 panel**：總計、Information warning、metadata fetch progress、download progress、四顆 action button
  - Download Selected MP3
  - Convert & Download Selected WAV
  - Download Track List（按下後變成「Download All Metadata」，需 metadata fetch 完成）
  - Reset Downloads

### Tab 2: Custom Download

使用者把 `missing-tracks.txt` 拖進來（這是 Tab 1 的 export 跑 `check-library.bat/.sh` 後產出的清單），extension 只下載這份清單中的曲子。支援 per-format missing：

```
missing-tracks.txt 格式：
trackId,mp3Found,wavFound,txtFound
abc123,true,false,true     ← 缺 WAV，只下 WAV
def456,false,true,false    ← 缺 MP3 和 sidecar，只下 MP3
```

這是個 **library sync** workflow：本地檔案系統當 source of truth，extension 只補差。

### 下載 lifecycle（單檔 MP3）

```
content.js downloadSelectedTracks
  → chrome.runtime.sendMessage("downloadFile", {url, filename, clipId, format})

background.js downloadFile → downloadFileWithMetadata → downloadFileWithMetadataFromBillingEndpoint
  1. POST /api/billing/clips/<id>/download/         （非關鍵，失敗也繼續）
  2. fetchTrackMetadata(clipId)                     （拿完整 metadata）
  3. fetch(audio_url) → blob
  4. embedMetadata(blob, metadata)                  （ID3v2.3，含 APIC 封面）
  5. blob → base64 → tabs.sendMessage("downloadLargeFileWithMetadata")
       ↓ (若 base64 > 50 MB)
       fallback: chrome.downloads.download(audio_url) 不嵌 metadata

content.js handleLargeFileDownload
  6. base64 → Blob → URL.createObjectURL → <a download>.click()
  7. 同步產出 .txt sidecar（人類可讀的 metadata dump）
  8. markTrackAsDownloaded(clipId, format) → localStorage["downloaded_<id>_mp3"]
  9. refreshTrackUI(clipId)                         （灰掉、改文字、加✓）

background.js onChanged downloadId
  10. tabs.sendMessage("markDownloaded")            （冗餘的二次確認）
```

WAV 的差別：第 4 步跳過 metadata 嵌入（檔太大送 message 會爆，直接走 `downloadFromUrl(audioUrl, ...)`，sidecar 還是有）。

---

## 8. Metadata 嵌入（`metadata.js`）

**100 % 自幹的 ID3v2.3 encoder**，沒用任何 third-party library（jsmediatags 在程式碼裡只當讀取的 fallback，沒被實際走到）。

支援的 frames：

| Frame ID | 內容 | 來源欄位 |
|----------|------|----------|
| `TIT2` | Title | `track.title` |
| `TPE1` | Artist | `track.metadata.artist` / fallback `"Suno AI"` |
| `TALB` | Album | `track.metadata.album` |
| `TDRC` | Year | `new Date(track.created_at).getFullYear()` |
| `TCON` | Genre | `track.metadata.genre` |
| `TBPM` | BPM | `track.metadata.bpm` / `tempo` |
| `TKEY` | Musical key | `track.metadata.key` / `musical_key` |
| `COMM` | Prompt（"comment" 欄位用來塞 styles tags） | `track.metadata.tags` / `gpt_description_prompt` |
| `USLT` | Lyrics | `track.metadata.infill_lyrics` 優先 |
| `APIC` | Cover art | `track.image_url` → fetch as JPEG → 嵌入；MIME 編碼用 ISO-8859-1（Windows Media Player 相容） |

`encodeSynchsafe()` / `decodeSynchsafe()` 處理 ID3 size 欄位的同步安全整數。

**WAV** 不修原始 RIFF，而是把整段 ID3v2 tag **附加在檔案尾**（行 156-161）。大部分 player 接受這個 hybrid，但這不是 spec-compliant，有些嚴格的 parser 會無視。

**Sidecar `.txt`** 同步產出（content.js:2411 / background.js:622），裡面除了人類可讀的欄位之外，**完整附上原始 API response JSON**——這份對逆向 Suno API 是金礦。

---

## 9. 儲存策略

### `chrome.storage.local`（由 background 控制）

| Key 模式 | 內容 | TTL |
|----------|------|-----|
| `authToken` | Bearer token 字串 | 永久（過期就會被新值覆蓋） |
| `deviceId` | UUID v4 | 永久 |
| `rateLimitConfig` | 動態調整的 delays | 永久 |
| `tracks_<workspaceId>` | 該 workspace 的所有 clips array | 永久（使用者按 🔄 refresh 才更新） |
| `tracks_<workspaceId>_timestamp` | unix ms | 同上 |
| `track_metadata_<clipId>` | 個別 track 完整 metadata | 永久 |

### `localStorage`（由 content script 控制，只在 suno.com 頁面下能看到）

| Key 模式 | 內容 |
|----------|------|
| `downloaded_<clipId>_mp3` | 下載時間戳，UI 用來灰掉按鈕 |
| `downloaded_<clipId>_wav` | 同上 |
| `filterStems` / `filterUpload` / `filterLiked` / `filterNotLiked` | "Hide" filters 的勾選狀態 |

**Reset Here** 按鈕（modal 右上角橘色）會 `chrome.storage.local.clear()` + 清掉所有 `downloaded_*` / `filter*` 的 localStorage。

---

## 10. 一些觀察與小坑

### 還算乾淨的設計

- **Sequential downloads with jittered delays**（1-3s MP3、2-4s WAV）—— 不嘗試平行化，明顯是被 ban 過後得到的教訓。
- **Cache-first**：track listing 從不自動 refresh，使用者按 🔄 才打 API；省 quota 也減少觸發 rate limit。
- **完整 fallback chain**：blob too large → content script via `tabs.sendMessage` → message too large → 直接 `chrome.downloads.download(url)`（無 metadata）→ tab 找不到 → data URL 硬塞。一路有 plan B。

### 程式碼味道

- **`content.js` 4742 行單檔**，所有 modal HTML 全是樣板字串 + inline style。可讀性差到值得逆向工程獎章。
- **大量重複的 HTML 字串**——Tab 1 跟 Tab 2 的 track-item 樣板幾乎一模一樣，渲染了至少三次（render、updateWorkspaceTracksInUI、custom-download tab）。
- **`api.js` 是死碼**——看起來原本想做 "Auth Sharing"（多人分享 Suno 帳號 credentials 的 OTP-based 流程），有 backend RPC client 但沒任何檔案 import 它。可能是先放著的未來功能。
- **`content.js` token hook 大概沒效**：isolated world fetch 看不到 main world 的 fetch（除非 page 把 fetch 從 isolated world 借去用，很少見）。實際靠 webRequest 在 background 拿。
- **`browser-token` 用 `btoa(JSON.stringify({timestamp}))` 自己編**——這顯然不是 Suno 後端真正要的格式（真實 token 應該由 Clerk 簽過），但 API 似乎也接受任何值。猜測這欄位後端可能還沒嚴查。

### 安全/隱私面

- Token 只存在本地 `chrome.storage.local`，沒上傳任何外部 server。
- **沒有遠端 telemetry**，也沒有外部分析。買披薩按鈕純圖片，不會 phone home。
- `cover art` 是用 `fetch(coverUrl, {credentials: "omit"})` 抓，因此 CDN 必須允許 CORS（`cdn1.suno.ai` 看起來允許）。
- 沒有用 `eval` / `Function()`，CSP 也只開到 `'self'`，做 supply-chain 攻擊面很小。
- 主要被動風險：**JSZip 97 KB 的 minified blob 沒人審過**——這是個流行套件，但 vendor 版本沒鎖。

### 你（gary）如果要 fork 或學的點

- **想要 Suno API 完整 schema** → 抓一份 track，看 sidecar `.txt` 末尾的 raw API response，這是最完整的非官方文件。
- **想加自己的格式（FLAC、Opus）** → 真正路徑短：先在 background 加一個新 action（類似 `initiateWavConversion`），其餘 progress/UI 框架直接 reuse。
- **想加自動上傳 / push 到雲端** → metadata.js 跟 download flow 是耦合最緊的部分，你想插就插在 `downloadFromUrl` 寫檔之前（拿著 blob + metadata）。
- **想觀察 Suno API token 格式** → 載入 extension 後 `chrome.storage.local.get("authToken")` 在 background console（`chrome://extensions/` → 該 extension → service worker → inspect）就拿得到。

---

## 11. 看 source code 的快速導覽建議

讀順序（從上而下會比較有 context）：

1. `manifest.json` — 看權限與 host 範圍（30 秒）
2. `background.js:1-244` — 看所有 message action 註冊；這就是 background 的「API 表」
3. `background.js:413-619` — 看 Suno API 怎麼呼叫（workspaces, tracks, metadata）
4. `metadata.js` 全檔 — ID3 builder，全部都是手寫 byte 操作，看完就懂 Suno track 物件結構
5. `content.js:99-308`（`showDownloadModal`）— modal 進入點
6. `content.js:648-1719`（`loadTracksIntoModal`）— 主資料流；一條龍從 fetch → render → wire up event
7. `content.js:1946-2046` + `2147-2304` — 兩條下載 pipeline（MP3 直拉、WAV poll-then-pull）

---

研究結束。Sources 已搬入 `D:\ulovemusic\suno-exporter\research\`，本檔放在 parent。
