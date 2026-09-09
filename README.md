# 旅圖書籤匣 — 資安說明

## 本次修正內容（2026-09-09）

### 1. 前端（travel-collection.html）
- **XSS 修復**：`title` / `note` / `tags` / `address` / `hours` / `link` / `image` 等動態欄位改經 `escapeHtml()` 轉義後才插入 DOM，避免注入 `<img onerror=...>`。
- **權限改用 uid**：移除前端硬編碼的 `OWNER_EMAIL`，改為每張卡片寫入 `ownerUid`（= 建立者的 Firebase uid），編輯/刪除按鈕依 `ownerUid` 判斷。
- **舊資料遷移**：新增 `migrateLegacyOwner()`，登入歷史 owner 帳號時自動把無 `ownerUid` 的舊卡片補上 `ownerUid`。

### 2. 後端（firestore.rules）— ⚠️ 須部署才生效
依 `ownerUid` 限制：登入者只能讀 / 新建 / 改 / 刪自己擁有的卡片。

```json
// firebase.json
{ "firestore": { "rules": "firestore.rules" } }
```

## ⚠️ 重要部署順序（不可顛倒）

1. **先** push 前端（HTML 更新）。
2. 用**原擁有者帳號**（yabu.san@gmail.com）登入，讓 `migrateLegacyOwner()` 跑一次，把既有卡片全部補上 `ownerUid`。
3. **最後**才 deploy Firestore rules 鎖庫：
   ```
   firebase deploy --only firestore:rules
   ```
   如果先在 open rules 下沒跑遷移就鎖規則，**所有舊卡片會立即讀不到**（因為沒有 ownerUid）。

> 需要 Firebase CLI 登入該專案（`travel-collection-34302`）才能 deploy rules。
