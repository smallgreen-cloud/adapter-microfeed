# AGENTS.md — microfeed 部署引導（跨 CLI，無 R2 降級版）

你是導遊不是裁判：每一關綠燈由機械驗證判定。同一步驟失敗 3 次即停，記錄卡點。使用者確認不超過 3 次。

## 前置

- Node >= 22.12（上游 package.json engines；README 建議 Node 24）、corepack（上游 yarn 4.18.0，`packageManager` 欄位鎖定，一律用 `corepack yarn` 呼叫）
- `CLOUDFLARE_API_TOKEN`＋`CLOUDFLARE_ACCOUNT_ID` 環境變數（Workers Scripts:Edit＋D1:Edit＋Account Settings:Read；**不需 R2 權限——本 adapter 即為 R2 未開通/無權限帳號的降級版**）
- 帳號需已註冊 workers.dev 子網域
- `export MICROFEED_DISABLE_R2=1`（patch 0001 降級開關——init 全程需要）
- **E53 註記**：上游 README.md:181-182 明言不支援 unattended API-token 部署，但 B run 實測全程非互動可行（文件反向漂移）。若 CLI 的 scope 檢查（manage-cli/lib/cloudflare.ts hasRequiredScopes）擋下並嘗試開瀏覽器登入，即 token 權限不足——回頭補 token 權限，不要走瀏覽器 OAuth。

## 步驟

### 1. 說明與同意（確認 1）

讀 `.smallgreen/profile.yaml` 向使用者說明：這是自架 CMS＋feed 平台；內容存自己帳號 D1（珍貴資料，維護契約含備份義務）；本版為無 R2 降級——**媒體上傳停用（回 503），純文字內容、JSON feed、RSS、admin 後台全功能**；帳號 R2 可用時不設 `MICROFEED_DISABLE_R2` 即回復完整功能。確認 instance 名與 admin 登入 email。

### 2. 組裝

```bash
git clone https://github.com/microfeed/microfeed.git microfeed && cd microfeed
git checkout fa0dcbd2ccd63444f704b8902b6850398b664d2b   # 與 UPSTREAM.md 一致
git apply <本repo>/patches/0001-manage-cli-no-r2-degraded-mode.patch
git apply <本repo>/patches/0002-media-endpoints-nonfatal.patch
git apply <本repo>/patches/0003-remove-version-metadata-binding.patch
git apply <本repo>/patches/0004-local-d1-database-name.patch
corepack yarn install
```

### 3. 部署（確認 2）

```bash
MICROFEED_DISABLE_R2=1 corepack yarn manage init \
  --yes \
  --instance <name> \
  --account-id "$CLOUDFLARE_ACCOUNT_ID" \
  --project-name <name> \
  --d1-name <name>-db \
  --r2-name <name>-media \
  --admin-path admin \
  --admin-auth built-in \
  --owner-email <使用者的登入 email> \
  --no-open
```

- **E54 陷阱**：`--yes` 只接受「非 secret 預設」，**不涵蓋資源名/admin path 提示**——`--project-name` `--d1-name` `--r2-name` `--admin-path` 缺一即掉入互動 askText（headless 下卡死）；`--owner-email` 缺席時 `--yes` 直接報錯（上游 commands.ts:546-551）。上述指令已把全部旗標寫死，照抄替換 `<name>` 即可。旗標實名出處：上游 `manage-cli/help.ts:56-70`。
- `--r2-name` 在降級模式仍必給（進 config 與生成範本），但不會建 bucket、部署設定會剝除 binding。
- init 內部流程：建 D1 → 跑 migrations → typecheck/test/build → 自動產生並注入 `UPLOAD_SIGNING_KEY`＋`BETTER_AUTH_SECRET` → `wrangler deploy` → 打 `/.well-known/microfeed.json` 驗證部署。
- **密碼設定**：init 尾聲 CLI 會印出一組 **30 分鐘、單次有效**的一次性設密碼連結。把連結原样轉交使用者自行開啟設定；**絕不使用 `--admin-password`**（上游 README 明令 agent 禁用），不要求使用者把連結或密碼貼回對話。設定完成前 `/admin/` 回 403 屬預期。
- **Pages 類通用雷（跨 adapter 通識，C run E58/E61）**：detached HEAD 下 `wrangler pages deploy` 必帶 `--branch main`，否則掛 preview 分支非 production。本 adapter 走 `wrangler deploy`（Workers），此雷不適用本流程；上游另有 `migrate-pages` 指令處理舊 Pages 安裝遷移。

### 4. 驗收（確認 3）

照 `.smallgreen/acceptance.yaml`（`<url>` 為 init 輸出的 workers.dev 網址）：

1. `GET /` → 200
2. `GET /json/` → 200 且含 `jsonfeed.org/version`；`GET /rss/` → 200 且為合法 XML（**注意尾斜線**：`/json`、`/rss` 回 308，Python urllib 不跟 308）
3. `GET /.well-known/microfeed.json` → 200 且 `product: microfeed`
4. 未認證 `POST /api/items/` → 401
5. 使用者設完密碼後：登入 admin、建一筆純文字 item、`/json/` 含該 item
6. **降級驗證點**：admin 上傳圖片 → PUT `/media-upload/...` 回 503「Media storage is disabled on this deployment」（明確失敗，非 500 崩潰）

全過才算完成（Build 成功不算）。

## 維護與移除

照 `.smallgreen/maintenance.yaml`。**移除前必先 `wrangler d1 export`（內容是珍貴資料）**；移除走上游 `yarn manage destroy --instance <name>`——先 `--dry-run` 審計畫再 `--confirm <name>`（該指令拒絕 `--yes`，需明給 confirm 名）。
