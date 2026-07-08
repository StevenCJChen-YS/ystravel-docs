# Features

每個功能一個資料夾,`<feature-slug>` 與程式碼那邊的 branch 名稱、`.feature` 檔名保持一致。

```text
docs/features/<feature-slug>/
  prd.md
  sa.md
  sd.md
  api-spec.md      # 有新增/異動 API 才需要
  test-plan.md
```

模板在 [../development-process/](../development-process/00-overview.md)。流程說明從 [00-overview.md](../development-process/00-overview.md) 開始看。

## Current Features

- [auth-foundation](./auth-foundation/architecture-decisions.md) — SSO/Auth 地基決策；目前主線為 native auth v2。
- [auth-native-login-v2](./auth-native-login-v2/README.md) — 當前 Phase 1a：AuthService 主導登入與權限，AuthPortal 回到原登入頁體驗。
- [auth-logto-login](./auth-logto-login/prd.md) — Logto 實驗紀錄；保留供追溯，不是目前主線。
