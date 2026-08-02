# UPSTREAM

| 欄位 | 值 |
|---|---|
| 上游 | [microfeed/microfeed](https://github.com/microfeed/microfeed) |
| 鎖定 commit | `fa0dcbd2ccd63444f704b8902b6850398b664d2b` |
| License | AGPL-3.0（LICENSE 檔與 GitHub API 一致；package.json 誤標 MIT，見「建議回饋上游」） |
| 鎖定日期 | 2026-08-02 |

本 commit 的 microfeed 已是 Astro on Workers 架構（`wrangler deploy` 部署 Worker，非舊版 Pages；舊 Pages 安裝走上游 `migrate-pages` 指令），admin 認證為內建 better-auth（email＋密碼，存 D1 `auth_user`），部署走 `yarn manage` CLI（manage-cli/）。

## patches 對照

| patch | 內容 | 為什麼 |
|---|---|---|
| 0001-manage-cli-no-r2-degraded-mode.patch | manage-cli/lib/cloudflare.ts 加 `MICROFEED_DISABLE_R2=1` 降級開關，三處改碼：(1) `r2BucketExists`（上游 cloudflare.ts:729）降級時直接回 false，並對 10042 拋帶解法的明確錯誤；(2) `createR2`（cloudflare.ts:807）降級時 no-op；(3) `deploy`（cloudflare.ts:929）降級時先剝除 dist/server/wrangler.json 的 `r2_buckets` 再部署 | R2 未開通/無權限帳號跑 init，前置檢查 `r2BucketExists`（commands.ts:1544-1547 的 Promise.all）遇 10042 非「not found」措辭即 throw 中止全流程＝R2 硬依賴（B run E52）。B agent 當時改 3 處達 D4-degraded：10042 視為不存在／createR2 no-op／部署前剝 r2 binding；本 patch 同 3 點但改顯式 env 開關——避免「10042 靜默當不存在」的半降級狀態，且不設變數即上游完整行為，無需撤 patch 還原 |
| 0002-media-endpoints-nonfatal.patch | 媒體端點 2 處防呆：src/pages/media-upload/[...key].ts PUT 與 src/server/media/media.ts getMediaResponse 在 `MEDIA_BUCKET` binding 缺失時回 503 明確訊息 | 部署設定剝除 binding 後 `env.MEDIA_BUCKET` 為 undefined，媒體上傳/讀取會 TypeError 500（上游僅此 2 檔直接觸 MEDIA_BUCKET）。503＋明確訊息＝「媒體上傳明確失敗」的降級如實；binding 存在時防呆不觸發，完整模式無害 |
| 0003-remove-version-metadata-binding.patch | wrangler.jsonc 與 wrangler.template.jsonc 移除 `version_metadata` binding；src/pages/.well-known/microfeed.json.ts:13 的 `env.CF_VERSION_METADATA.timestamp` 改為空字串；tests/unit/manage-cli/WranglerTemplate.test.ts:50 斷言同步改 undefined | `version_metadata` 不在 SmallGreen 免費層允許清單（conformance SAP-2 的 WRANGLER_RESOURCE_KEYS 判定），保留即 fail。CLI 部署驗證只消費 identity 的 `product`＋`instanceId`（commands.ts:996-1004），不讀 `deployedAt`——空字串安全。上游 runChecks 於部署時跑 test（commands.ts:1373），故 test 斷言必須同步改 |
| 0004-local-d1-database-name.patch | root wrangler.jsonc 的 FEED_DB 補 `database_name: "microfeed"` | root 設定原只有 binding 名（上游註明僅供 build/型別生成），本地 D1 工具（conformance migration 重放、`wrangler d1 execute <name> --local`）需要 database_name 定位資料庫。不影響雲端部署——CLI 一律用生成的 per-instance 設定（wrangler.template.jsonc 渲染） |

**還原完整模式**：帳號 R2 可用時，部署時不設 `MICROFEED_DISABLE_R2` 即回復媒體功能（patch 0001 為 env-gated，patch 0002 防呆在 binding 存在時不觸發）——四支 patch 全數保留無害。

**已知降級副作用**（記錄非缺陷）：降級部署的 Worker 無 MEDIA_BUCKET binding，`yarn manage connect` 的既有安裝探索（cloudflare.ts discoverMicrofeedWorkers 要求 r2 binding 存在，:596）不會列出它；`yarn manage status` 顯示 R2 bucket 不存在屬預期。

## B/C run 實測對照（納入設計的證據）

- **E52**（B run）：R2 硬依賴——init 的 r2BucketExists 前置檢查遇 10042 即中止全流程。→ patch 0001 的存在理由。
- **E53**（B run）：上游 README.md:181-182 明言「Hosted/headless agents and unattended API-token deployment are not supported」，但實測 `CLOUDFLARE_API_TOKEN` 全程非互動可行＝文件反向漂移（文件比實作保守）。→ AGENTS.md 走非互動路徑並註記此漂移。
- **E54**（B run）：CLI `--yes` 不涵蓋資源名/admin path 提示——`resourceName`/`configuredAdminPath`（commands.ts:194-224）不讀 `--yes`，無旗標即 askText 互動提示；`--owner-email` 缺席時 `--yes` 直接 throw（commands.ts:546-551）。實測旗標實名（manage-cli/help.ts:56-70）：`--instance` `--account-id` `--project-name` `--d1-name` `--r2-name` `--admin-path` `--admin-auth <built-in|none>` `--owner-email` `--admin-password`（不安全，禁用）`--no-open` `--reuse-d1` `--reuse-r2` `--preview` `--local` `--yes`。→ AGENTS.md 指令把全部資源名旗標寫死。
- **E58/E61**（C run，Pages 類通用雷）：detached HEAD 下 `wrangler pages deploy` 不帶 `--branch main` 會掛到 preview 分支。本 adapter 走 `wrangler deploy`（Workers）不受此雷，列入 AGENTS.md 供跨 adapter 通識。

## 同步程序

1. `gh api repos/microfeed/microfeed/commits/main --jq .sha` 取新 commit，與鎖定值 diff 審閱（重點：manage-cli/lib/cloudflare.ts 的 R2 呼叫點增減 vs patch 0001、`env.MEDIA_BUCKET`/`runtimeEnv.MEDIA_BUCKET` 觸點 vs patch 0002、wrangler 設定資源 key vs patch 0003）
2. 對新 commit 依序 `git apply --check patches/000{1,2,3,4}-*.patch`
3. 組裝樹重跑 conformance 三腳本＋上游 `yarn typecheck && yarn test && yarn build`
4. 更新本檔與 profile.yaml 的 `upstream.commit`（兩處一致，CON-6 驗證）

## 建議回饋上游

1. **無 R2 降級模式**（patch 0001＋0002）適合開 upstream issue/PR：R2 未開通帳號目前完全無法使用 microfeed，即使只發布純文字內容；env-gated 降級讓 feed/CMS 核心功能與 R2 解耦。
2. **License 元資料矛盾**：LICENSE 檔為 AGPL-3.0（GitHub API spdx 同），package.json 卻標 `"license": "MIT"`（package.json:6）——應修正 package.json。
