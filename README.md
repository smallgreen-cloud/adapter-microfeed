# adapter-microfeed

> [microfeed](https://github.com/microfeed/microfeed)（自架輕量 CMS＋feed 平台：網頁、JSON feed、RSS，4,028★）的 SmallGreen 適配 repo（Path C，**無 R2 降級版**）。

[![conformance](https://github.com/smallgreen-cloud/adapter-microfeed/actions/workflows/conformance.yml/badge.svg)](https://github.com/smallgreen-cloud/adapter-microfeed/actions/workflows/conformance.yml)

**驗證等級：Discovered**（收錄 ≠ 驗證）

上游程式碼不在本 repo：[UPSTREAM.md](UPSTREAM.md) 鎖定 commit、[.smallgreen/](.smallgreen/) 契約三檔、[patches/](patches/)（manage CLI 無 R2 降級開關＋媒體端點防呆＋wrangler 設定對齊——R2 未開通/無權限帳號適配；降級模式屬上游缺口，建議回饋 upstream）、[AGENTS.md](AGENTS.md) 非互動部署路徑。

## 資料流向（信任揭露）

內容（items、channel 設定）與 admin 帳號存部署者自己帳號的 D1（珍貴資料——維護契約含備份義務）。admin 後台以內建 better-auth（email＋密碼）保護，首組密碼走 CLI 產生的 30 分鐘一次性連結。無外連、無遙測。

降級語義：媒體上傳與媒體檔服務停用（端點回 503 明確訊息）；純文字內容、前台網頁、JSON feed、RSS、admin 後台全功能。帳號 R2 可用時部署不設 `MICROFEED_DISABLE_R2` 即回復完整功能，無需撤 patch（見 UPSTREAM.md）。

License：AGPL-3.0（隨上游 LICENSE 檔；上游 package.json 誤標 MIT，已列建議回饋）。
