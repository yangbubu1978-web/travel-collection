# 旅圖書籤匣 — 權限與資安說明

## 權限模型（現行）

僅 **兩位白名單帳號** 可讀寫（新增 / 修改 / 刪除 / 讀取）全部卡片：

- 雅布大人：`yabu.san@gmail.com`
- 小布：`yang.bubu1978@gmail.com`

其他登入者無法讀寫。判斷邏輯放在 **Firestore Security Rules**（`firestore.rules`），前端只負責依 `isEditor()` 秀出編輯 / 刪除鈕（不作真正防護）。

## 資安修正紀錄

### 2026-09-09
- **XSS 修復**：`title / note / tags / address / hours / link / image` 等動態欄位經 `escapeHtml()` 轉義後才插入 DOM。
- **權限白名單**：移除硬編碼 `OWNER_EMAIL`，改為 `ALLOWED_EDITORS` email 白名單 + 前端 `isEditor()`。
- **Firestore rules**：`allow read, write` 限定兩位白名單 email。

## ⚠️ 部署須知

- 前端（HTML）改完 push 即自動更新 GitHub Pages（等 build）。
- Firestore rules 需 deploy 才生效：
  ```
  firebase deploy --only firestore:rules
  ```
  需要 Firebase CLI 登入 + 該專案權限（`travel-collection-34302`）。

> 權限以「登入帳號 email」判斷，**不依賴卡片資料欄位**，故無舊資料遷移風險，現有資料天然可讀。
