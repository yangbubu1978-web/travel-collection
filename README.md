# 旅圖書籤匣 — 權限與資安說明

## 權限模型（現行）

僅 **白名單帳號** 可讀寫（新增 / 修改 / 刪除 / 讀取）全部卡片：

- 雅布大人：`yabu.san@gmail.com`
- 小布：`yang.bubu1978@gmail.com`
- Prosaist：`prosaist0101@gmail.com`（2026-09-20 加入）

其他登入者無法讀寫。判斷邏輯放在 **Firestore Security Rules**（`firestore.rules`），前端只負責依 `isEditor()` 秀出編輯 / 刪除鈕（不作真正防護）。

> ⚠️ 新增帳號要改 **兩個地方**：`firestore.rules` 的 `request.auth.token.email in [...]`（伺服器端真正的門）＋ `travel-collection.html` 的 `ALLOWED_EDITORS`（前端秀按鈕）。只改一邊會出現「能登入但看不到按鈕」或「看得到按鈕但存不進去」。
> email 一律寫小寫（Google 帳號的 token email 為小寫，Gmail 本身不分大小寫）。

## 資安修正紀錄

### 2026-09-09
- **XSS 修復**：`title / note / tags / address / hours / link / image` 等動態欄位經 `escapeHtml()` 轉義後才插入 DOM。
- **權限白名單**：移除硬編碼 `OWNER_EMAIL`，改為 `ALLOWED_EDITORS` email 白名單 + 前端 `isEditor()`。
- **Firestore rules**：`allow read, write` 限定兩位白名單 email。

## ⚠️ 部署須知

- 前端（HTML）改完 push 即自動更新 GitHub Pages（等 build，`cache-control: max-age=600`）。
- **同一個檔案也部署在 Firebase Hosting**（`firebase deploy --only hosting`）：
  - `https://travel-collection-34302.firebaseapp.com/` ← **建議手機使用者用這個網址**
  - `https://travel-collection-34302.web.app/`
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
- 桌機用 GitHub Pages 或上面任一網址都正常。

## 🆘 登入失敗的兩種情境（前端已內建提示）

1. **App 內建瀏覽器**（Messenger／LINE／IG 等）：登入頁會顯示橘色警告並提供「複製網址」鈕 → 請改用 Safari／Chrome 開啟。
2. **瀏覽器封鎖跨網站 Cookie**：按過登入卻又跳回登入頁（安靜失敗）→ 頁面會顯示「登入沒有完成」提示與排除步驟。

> 權限以「登入帳號 email」判斷，**不依賴卡片資料欄位**，故無舊資料遷移風險，現有資料天然可讀。兩個網域共用同一個 Firestore 專案，資料完全同步。
