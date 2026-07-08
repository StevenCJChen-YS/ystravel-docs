# Release Checklist

用在把合併後的 `main` 部署到 staging / production 之前。

## 部署前

- [ ] 確認這次改動涉及的所有環境變數都已在目標環境設定(`.env` 不進 repo)
- [ ] 確認是否有 Prisma migration,先在 staging 跑過 `prisma migrate deploy` 驗證
- [ ] 確認 build 成功(`npm run build`)
- [ ] 確認 `main` 上的測試全部通過

## 部署

- [ ] 依 [06-git-and-release-flow.md](./06-git-and-release-flow.md) 的部署路徑部署到 staging
- [ ] 在 staging 手動驗證這次改動的核心情境(至少對照 PRD 的 Acceptance Criteria 走一次)
- [ ] 確認沒有影響到既有功能(至少檢查同一系統的其他主要頁面)
- [ ] 部署到 production

## 部署後

- [ ] 觀察錯誤 log / 監控是否有異常
- [ ] 若涉及敏感資料或權限異動,確認 audit log 有正常寫入
- [ ] 通知相關使用者(如果這次改動會影響到他們的操作方式)

## 回滾方案

- [ ] 這次改動如果出問題,回滾方式是什麼?(revert commit + 重新部署上一版 / 關閉 feature flag / 還原 migration)
- [ ] 若有資料庫 migration,是否有對應的 rollback migration 或還原步驟?

## 何時需要更完整的 Release 流程

目前是內部系統、部署頻率不高,這份 checklist 已足夠。當團隊變大、部署變頻繁時,再考慮:

- 自動化 CI/CD(GitHub Actions 自動 build + 部署)
- Feature flag,讓大改動可以分階段開放
- 正式的 staging/UAT 簽核紀錄
