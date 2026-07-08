# Ystravel 設計系統 — Foundation（全公司共用）

| 項目 | 內容 |
|---|---|
| 文件性質 | 跨系統設計系統來源（source of truth） |
| 適用範圍 | Ystravel-AuthPortal、Ystravel-CRM-Frontend、未來 EIP 前端 |
| 技術 | NuxtUI v4 + Tailwind CSS v4 |
| 狀態 | 草稿（2026-07-08） |
| 前身 | 抽自 `Ystravel-AuthPortal/docs/AUTHPORTAL_UI_FOUNDATION.md`（2026-07-06，已相當成熟），升級為全公司共用版並加上「各系統主題層」 |

---

## 0. 這份文件是什麼

你**不是從零開始**——AuthPortal 已經有一套成熟的 UI foundation。這份文件做兩件事：
1. 把那套 foundation **抽成全公司共用的基礎**，讓 CRM、EIP 都沿用同一套底。
2. 加上「**各系統不同主色**」的主題層（你 2026-07-08 的決定）。

**核心觀念：設計 Token 分兩層**
- **Base（共用底層）**：字體、間距、圓角、灰階、語意色（成功/警告/錯誤/資訊）、密度、按鈕階層 → **全公司一致**。
- **Theme（各系統主題層）**：只有 `primary` 主色每個系統覆寫 → **各系統不同**。

> 業界（Shopify、Atlassian、Material）都是這樣：共用一套底，各產品換主色。

---

## 1. 各系統主色（Theme 層，定案 2026-07-08）

| 系統 | primary 主色 |
|---|---|
| **AuthPortal** | **Blue** |
| **EIP**（未來） | **Teal** |
| **CRM** | **Violet** |

- 三色在色環上大致等距，最好辨識「我在哪個系統」。
- 三個主色都**避開了語意色**（見 §3），不會跟成功/警告/資訊訊息撞色。
- 每個系統只覆寫 `primary`，其餘全部沿用 Base。

---

## 2. Base — Typography（共用）

- **字體**：`--font-sans: "Aptos", "Noto Sans TC", "Segoe UI Variable", "Segoe UI", system-ui, sans-serif`；mono：`JetBrains Mono`。
- **文字層級**（內部系統以穩定可掃讀為優先，body 不隨斷點頻繁變動）：

| 用途 | 大小 |
|---|---|
| Page title | mobile `text-xl`，`sm+` `text-2xl` |
| Section title | mobile `text-base`，`lg+` `text-lg` |
| Body / Help | 固定 `text-sm` |
| Meta / code / timestamp | 固定 `text-xs` |

---

## 3. Base — 色彩（共用語意色）

### 3.1 語意色盤（各系統一致，**不覆寫**）
| 語意 | 色 |
|---|---|
| secondary | sky |
| success | emerald |
| info | sky |
| warning | amber |
| error | rose |
| neutral | slate |

> ⚠️ 各系統的 `primary` 主色（Blue/Teal/Violet）**不得使用上面任何一個**，否則主色會跟語意訊息撞色。

### 3.2 中性底色分層（不只白底一層）
| Token | 用途 |
|---|---|
| `bg-default` | 整頁背景、sidebar、header、main 容器 |
| `bg-muted` | toolbar、hover、表頭、次層背景 |
| `bg-elevated` | card、panel、浮起表面 |
| `bg-accented` | 選取態、active nav、當前項目 |

### 3.3 文字與邊框語意
- 文字：`text-highlighted`（標題）／`text-default`（內文）／`text-toned`（中層）／`text-muted`（輔助）
- 邊框：`border-muted`（一般分隔/card）／`border-toned`（需更明顯）／`border-accented`（強調/選取）

---

## 4. Base — 其他基礎（共用）

- **圓角**：`--ui-radius: 0.5rem`
- **斷點**：base `<640` / sm `≥640` / md `≥768` / lg `≥1024` / xl `≥1280` / 2xl `≥1536`
- **Layout tokens**：`--app-sidebar-width: 240px`、`--app-sidebar-collapsed-width: 4rem`、`--app-header-height: 70px`
- **密度**（各系統可依任務調整）：
  - Login：`xl`
  - Auth Admin：mobile/tablet `lg`、desktop `md`、row action `sm`
  - CRM：compact，desktop 表單/filter/table 以 `sm`/`md` 為主，不複製 login 的 `xl`
- **按鈕階層**：
  - Primary page action：`color="primary" variant="solid"`
  - Secondary：`color="neutral" variant="outline"`
  - Toolbar/utility：`color="neutral" variant="ghost"`
  - Filled neutral：`color="neutral" variant="soft"`（`neutral solid` 會近黑，避免當一般灰按鈕）
  - Risky：`warning soft`；Destructive：`error soft`/`outline`
  - 同一 view 只保留一個最強 primary。

---

## 5. Light / Dark Mode（共用）

- **兩個模式一開始就一起做**，禁止新增只支援 light mode 的硬寫色。
- 顏色偏好共用 storage key：`ystravel.platform.color-mode`（Auth/CRM/EIP 共用）。
- 新頁面優先用 semantic class + NuxtUI color/variant，不硬寫 `bg-white`/`text-slate-*`。
- **主色的 dark mode**：主色用中間調（如 blue/teal/violet 的 500/600），深色模式自動對應到淺一階（如 400），明暗兩模式對比都足夠——這也是當初不選 sky 當主色的原因（sky 太淺，暗色模式會刺眼/糊）。

---

## 6. 例外：LoginPage 品牌入口頁

- `LoginPage` 是**特殊品牌入口**，可保留較強的品牌視覺與頁面局部 CSS variable（現有 6 色 palette），**不當作內頁模板**。
- Auth Admin、CRM、EIP 的內頁一律走本文件的 semantic token，不沿用 LoginPage 的 palette。

---

## 7. 各 repo 怎麼套用（實作位置）

| 用途 | 位置 |
|---|---|
| 全站 NuxtUI theme 覆寫（含 primary） | 各 repo `vite.config.ts` → `ui({ ui: { colors: { primary: '...' }, ... } })` |
| 全域 CSS（字體、icon 線條、layout token） | 各 repo `src/assets/css/main.css` 的 `@theme` / `:root` |
| 頁面局部 override | `.vue` 的 `size` / `ui` prop / `class` |

- **共用底層**（字體、語意色、密度、按鈕階層…）三個 repo 應保持一致。
- **主題層**只有 `primary` 每個 repo 不同（Auth=Blue / CRM=Violet / EIP=Teal）。
- 目前**不改任何現有 code**；本文件先作為規範，實際套用在各系統做 UI 時逐步落實。

---

## 8. 與既有文件的關係

- 本文件 = 全公司**共用來源**（Base + Theme 決策）。
- `Ystravel-AuthPortal/docs/AUTHPORTAL_UI_FOUNDATION.md` = AuthPortal 的**詳細實作規範**（shell、shared 元件、drawer、dropdown 等細節），持續有效，作為本文件的參考實作。
- `docs/reference/AUTHPORTAL_NUXTUI_COMPONENT_SCALE.md` = `UButton` 等元件尺寸客製的技術背景。
- CRM / EIP 未來各自可有 `docs/<system>-ui.md` 記錄自己的延伸，但**不得牴觸本文件的 Base**。

---

## 9. 待決策 / 待辦
- [ ] 三系統 primary 的**確切色階**（例如 blue-600 或 blue-500 當亮色主色）——做 UI 時實測 light/dark 對比後定。
- [ ] 是否需要把 Base token 做成一個共用 npm 套件 / 共用 CSS 檔，讓三 repo 真正共用而非各自複製（現階段先文件對齊，未來重複痛了再抽套件）。
