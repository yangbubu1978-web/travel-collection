# 旅圖書籤匣 — 權限與資安說明

## 權限模型（現行）

- **讀取**：**任何人皆可**（訪客免登入即可瀏覽，2026-09-23 起）
- **寫入**：僅白名單帳號（新增 / 修改 / 刪除）
  - 雅布大人：`yabu.san@gmail.com`
  - 小布：`yang.bubu1978@gmail.com`
  - Prosaist：`prosaist0101@gmail.com`（2026-09-20 加入）

判斷邏輯放在 **Firestore Security Rules**（`firestore.rules`），前端只負責依 `isEditor()` 秀出編輯 / 刪除鈕（不作真正防護）。

> ⚠️ 開放讀取代表**任何知道專案 ID 的人都能用 Firestore API 讀走全部卡片**（不只是「知道網址的人」）——專案 ID 就寫在網頁原始碼裡。這是刻意的取捨：換來手機訪客完全免登入、不受跨網站 Cookie 限制。若日後要收回，把 `allow read` 改回 `if request.auth != null && request.auth.token.email in [...]` 即可。

> ⚠️ 新增帳號要改 **兩個地方**：`firestore.rules` 的 `request.auth.token.email in [...]`（伺服器端真正的門）＋ `travel-collection.html` 的 `ALLOWED_EDITORS`（前端秀按鈕）。只改一邊會出現「能登入但看不到按鈕」或「看得到按鈕但存不進去」。
> email 一律寫小寫（Google 帳號的 token email 為小寫，Gmail 本身不分大小寫）。

## 資安修正紀錄

### 2026-09-09
- **XSS 修復**：`title / note / tags / address / hours / link / image` 等動態欄位經 `escapeHtml()` 轉義後才插入 DOM。
- **權限白名單**：移除硬編碼 `OWNER_EMAIL`，改為 `ALLOWED_EDITORS` email 白名單 + 前端 `isEditor()`。
- **Firestore rules**：`allow read, write` 限定兩位白名單 email。

## ⚠️ 部署須知

**唯一對外網址（2026-09-24 起）**：
- `https://travel-collection-34302.firebaseapp.com/` ← **主要網址，手機／桌機都用這個**
- `https://travel-collection-34302.web.app/`（同一個站，備用網址）

> GitHub Pages（`yangbubu1978-web.github.io/travel-collection/`）已於 2026-09-24 **停用**。
> 原因：維護兩條路徑沒有任何好處（流量限制較寬的部分實際用不到，真正會先撞到的是 Firestore 額度），
> 而且少一道門才守得住「網址不外流」的策略。若日後需要緊急備援，在 repo Settings → Pages 重新開啟即可。

- 部署指令：`firebase deploy --only hosting`（`public` 指向 repo 根目錄）
- Hosting 的 HTML 已設 `Cache-Control: no-cache`，不會有舊版快取問題
- Firestore rules 需 deploy 才生效：
  ```
  firebase deploy --only firestore:rules
  ```
  需要 Firebase CLI 登入 + 該專案權限（`travel-collection-34302`）。

## 🔑 authDomain 不可以亂改（2026-09-20 實測）

`authDomain` 必須維持 **`travel-collection-34302.firebaseapp.com`**。

- 改 `travel-collection-34302.web.app` → Google 直接回 **`redirect_uri_mismatch`**（OAuth 用戶端只認 firebaseapp.com 的 `/__/auth/handler`），登入全掛。
- 因此「同源登入」的做法是**把 App 也放到 firebaseapp.com 上**（Firebase Hosting 同時服務 `.web.app` 與 `.firebaseapp.com`），而不是改 authDomain。
- 📱 **手機／內建瀏覽器請用 `https://travel-collection-34302.firebaseapp.com/`**：網頁與登入處理頁同一個網域，不會被「跨網站 Cookie 封鎖／儲存空間隔離」擋掉。
- 🆕 **2026-09-23 起訪客不需要登入**：`read` 已開放，單純瀏覽不會碰到同源限制。只有要編輯時才需要登入。
- 🆕 **2026-09-24 起 GitHub Pages 已停用**，只剩 Firebase 一個網址，不必再區分「電腦用／手機用」。

## 🆘 登入失敗的兩種情境（前端已內建提示）

1. **App 內建瀏覽器**（Messenger／LINE／IG 等）：登入頁會顯示橘色警告並提供「複製網址」鈕 → 請改用 Safari／Chrome 開啟。
2. **瀏覽器封鎖跨網站 Cookie**：按過登入卻又跳回登入頁（安靜失敗）→ 頁面會顯示「登入沒有完成」提示與排除步驟。

> 權限以「登入帳號 email」判斷，**不依賴卡片資料欄位**，故無舊資料遷移風險，現有資料天然可讀。

## 🔎 補充景點資料的來源對照（2026-09-24 實測）

給協助編輯者補資料的人（目前是小布）參考。實測條件：一般網路環境、未登入任何服務。

| 來源 | 抓取方式 | 可行性 | 可取得欄位 |
|---|---|---|---|
| 部落格 / 一般網站 | 讀 HTML 的 Open Graph 標籤 | ✅ | `og:title`、`og:description`、`og:image`、`og:site_name` |
| YouTube（含 Shorts） | oEmbed API（免 key、免註冊） | ✅ | 標題、頻道名、縮圖 |
| Instagram（貼文 / Reels） | 無登入抓取 | ❌ | Meta 封鎖，實測 HTTP 200 但無 `og:title` |

### YouTube：oEmbed

```
https://www.youtube.com/oembed?url=<影片網址>&format=json
```

回傳：

```json
{
  "title": "影片標題",
  "author_name": "頻道名稱",
  "thumbnail_url": "https://i.ytimg.com/vi/<id>/hqdefault.jpg"
}
```

`watch?v=` / `/shorts/` / `youtu.be/` 三種形式都支援（網址要 URL-encode）。

縮圖也可直接組，完全不需請求：

```
https://i.ytimg.com/vi/<影片ID>/maxresdefault.jpg   # 最高解析（不一定存在）
https://i.ytimg.com/vi/<影片ID>/hqdefault.jpg       # 一定存在
```

### 部落格 / 一般網站

抓 `<meta property="og:...">`。多數網站為了社群分享預覽都會提供。
用社群爬蟲的 User-Agent（如 `facebookexternalhit/1.1`）成功率較高。

### Instagram：不建議自動化

未登入時公開頁面幾乎不回傳任何 og 標籤。可行的自動化方式都需要「登入 session + 住宅代理」，
成本高、不穩定，且 Meta 明確禁止，帳號會被鎖。
**請改用人工：截圖，或直接手打店名／地址。**

> 目前的流程：Prosaist 貼上原始連結 → 小布協助補完欄位。
> 若日後要做成站內功能（貼網址自動帶入），這張表就是可行性依據。
