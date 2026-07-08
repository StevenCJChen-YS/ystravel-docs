# AuthPortal Nuxt UI 元件尺寸規範

更新日期：2026-07-06  
適用專案：`Ystravel-AuthPortal`（Vue + Vite + Nuxt UI v4 + Tailwind CSS v4）  
相關文件：[Ystravel-AuthPortal/docs/AUTHPORTAL_UI_FOUNDATION.md](../../Ystravel-AuthPortal/docs/AUTHPORTAL_UI_FOUNDATION.md)

## 1. 文件目的

記錄 AuthPortal 目前對 **Nuxt UI 元件尺寸** 的客製化決策與背景，供後續開發參考。

目前狀態：

- **已完成**：`UButton` 全站 size scale 客製化
- **尚未完成**：`UInput`、`USelect`、`USelectMenu` 等表單元件仍使用 Nuxt UI 預設尺寸
- **策略**：其他元件遇到並排對齊問題時，再依本文件同一套設計原則調整

## 2. 設定位置

| 用途 | 檔案 |
|------|------|
| 全站 Nuxt UI theme 覆寫 | `Ystravel-AuthPortal/vite.config.ts` → `ui({ ui: { ... } })` |
| 全域 CSS（字體繼承、icon 線條） | `Ystravel-AuthPortal/src/assets/css/main.css` |
| Nuxt UI 各元件預設 theme 參考 | `node_modules/.nuxt-ui/ui/<component>.ts` |
| 頁面局部 override | 各 `.vue` 的 `size` prop、`ui` prop、`class` |

### Override 優先順序（高 → 低）

1. 單一元件的 `ui` prop / `class`
2. `vite.config.ts` 全域 theme
3. Nuxt UI 內建預設

## 3. 設計原則

### 3.1 視覺方向

- 比 Nuxt UI 預設更 **compact（緊湊）**、更 **輕（font-normal）**
- Icon 略小、線條略細（`stroke-width: 1.5`）
- 字體 / icon 縮小後，用 **padding（尤其 `py`）** 補回可點擊高度，避免按鈕過扁

### 3.2 Size 梯度邏輯

| 層級 | 字體 | 用途 |
|------|------|------|
| xs ~ md | `text-xs`（12px） | 密集 admin 操作、預設按鈕 |
| lg ~ xl | `text-sm`（14px） | 較重要的操作、header CTA、Login 頁 |

相鄰 size 若字體 / icon 相同，主要靠 **padding** 區分（與 Nuxt UI 原本 lg/md 邏輯類似）。

### 3.3 全站預設 size

- `UButton` 預設：`md`（在 `vite.config.ts` 的 `defaultVariants`）

## 4. UButton 目前設定

### 4.1 全尺寸共用

| 項目 | Nuxt UI 預設 | 目前設定 |
|------|-------------|---------|
| 字重 | `font-medium`（500） | `font-normal`（400） |
| Icon 線條 | Lucide 預設 `stroke-width: 2` | `stroke-width: 1.5`（`main.css`） |

`main.css` 片段：

```css
@layer base {
  [data-slot="leadingIcon"] svg,
  [data-slot="trailingIcon"] svg {
    stroke-width: 1.5;
  }
}
```

### 4.2 各 size 對照表

| size | 字體 | Icon | Padding（base） | Gap | 備註 |
|------|------|------|-----------------|-----|------|
| **xs** | `text-xs`（12px） | `size-3`（12px） | `px-2 py-1` | `gap-1` | 最小 |
| **sm** | `text-xs`（12px） | `size-3.5`（14px） | `px-2.5 py-1.5` | `gap-1.5` | row action 等 |
| **md**（預設） | `text-xs`（12px） | `size-3.5`（14px） | `px-2.75 py-2` | `gap-1.5` | admin 預設密度 |
| **lg** | `text-sm`（14px） | `size-4`（16px） | `px-3.25 py-2` | `gap-2` | mobile / tablet 表單區 |
| **xl** | `text-sm`（14px） | `size-4`（16px） | `px-3.75 py-2.5` | `gap-2` | Login、header 主要 CTA |

### 4.3 與 Nuxt UI 預設差異（重點）

| size | 項目 | Nuxt UI 預設 | 目前設定 | 變化 |
|------|------|-------------|---------|------|
| md | 字體 | `text-sm`（14px） | `text-xs`（12px） | ↓ |
| md | Icon | `size-5`（20px） | `size-3.5`（14px） | ↓ |
| md | 字重 | 500 | 400 | ↓ |
| md | 垂直 padding | `py-1.5` | `py-2` | ↑ 補高度 |
| lg | 字體 / Icon | 同 md（14px / 20px） | 升級為 14px 字 + 16px icon | 與 md 拉開 |
| xl | 字體 | `text-base`（16px） | `text-sm`（14px） | ↓ |
| xl | Icon | `size-6`（24px） | `size-4`（16px） | ↓ |
| xl | 水平 padding | `px-3` | `px-3.75` | 微調 |

## 5. 重要技術背景

### 5.1 按鈕高度如何決定

`UButton` 為 `inline-flex items-center`，高度 ≈ **上下 padding + max(icon 高度, 文字 line-height)**。

因此：

- 只改 padding、不改 icon / 字體 → 高度幾乎不變
- icon / 字體縮小 → 按鈕變矮（即使 padding 相同）
- 解法：縮小內容後，適度增加 `py` 補高度

### 5.2 Nuxt UI 原生 size 行為（未客製化前）

Nuxt UI 的 `size` **不是等比例放大所有內容**：

| size | 字體 | Icon | 主要差異 |
|------|------|------|---------|
| xs / sm | `text-xs` | `size-4` | padding |
| md / lg | `text-sm` | `size-5` | **僅 padding 不同** |
| xl | `text-base` | `size-6` | 字體與 icon 才明顯變大 |

### 5.3 `font: inherit` 必須放在 `@layer base`

`main.css` 中：

```css
@layer base {
  button, input, select, textarea {
    font: inherit;
  }
}
```

若寫在 `@layer` **外面**，會蓋過 Tailwind / Nuxt UI 的 `text-xs`、`text-sm`、`font-medium`，導致按鈕字體大小與字重異常（例如永遠 16px）。

Icon 不受此影響（`UIcon` 是獨立元素，用 `size-*` class 控制）。

### 5.4 Tailwind CSS v4 spacing 乘數

本專案使用 **Tailwind v4**，spacing 為乘數系統：

```css
/* 例如 px-3.75 */
padding-inline: calc(var(--spacing) * 3.75);
/* --spacing 預設 = 0.25rem = 4px */
```

**只要是 0.25 的倍數都有效**，包含 `2.75`、`3.25`、`3.75` 等，不必侷限 v3 時代的固定 scale。

實測：`px-3.75` 與 `px-3.5` 相差 2px 總寬度（每側 1px），符合預期。

參考：[Tailwind CSS v4 padding 文件](https://tailwindcss.com/docs/padding)

## 6. 與其他元件的對齊（尚未客製化）

### 6.1 目前狀態

只改了 `UButton`，`UInput` / `USelect` / `USelectMenu` 仍為 Nuxt UI 預設。

**同一 `size` prop 並排時，高度可能不完全一致。** 例如：

| 元件 | md 字體 | md Icon | md 垂直 padding |
|------|--------|---------|----------------|
| `UButton`（已客製） | 12px | 14px | `py-2` |
| `UInput`（預設） | 16px（`text-base/5`） | 20px | `py-1.5` |
| `USelect`（預設） | 14px | 20px | `py-1.5` |

### 6.2 已知影響場景

- Admin 頁：Input 左側 + 確認 Button 右側並排
- Login 頁：`UFormField size="xl"` 的 Input vs `UButton size="xl"`
- `UFieldGroup`：Input 與 Button 緊密相連

### 6.3 處理策略（遇到再調）

1. 先確認視覺上是否真的有問題（差距 1~2px 有時可接受）
2. 若需對齊，依 **本文件同一套原則** 客製化對應元件的 `variants.size`
3. 優先同步調整：`input`、`select`、`selectMenu`（trigger 部分）
4. 並排時仍應使用 **相同 `size` prop**，必要時用 `UFieldGroup`

## 7. 未來擴展指南

當需要客製化 `UInput` 等元件時，在 `vite.config.ts` 加入類似結構：

```ts
input: {
  variants: {
    size: {
      md: {
        base: 'px-2.75 py-2 text-xs gap-1.5',  // 對齊 button md 的思路
        leadingIcon: 'size-3.5',
        trailingIcon: 'size-3.5',
      },
      // ...
    },
  },
  defaultVariants: {
    size: 'md',
    color: 'neutral',
    variant: 'outline',
  },
},
```

建議步驟：

1. 打開 `node_modules/.nuxt-ui/ui/<component>.ts` 看預設 slot 名稱
2. 以 `UButton` 同一 size 的 padding / 字體 / icon 為目標參考
3. 改完後在「Input + Button 並排」的實際頁面驗證高度
4. 更新本文件對照表

### 不建議

- 只改 button、不改 input，卻在大量並排表單場景期待完美對齊
- 在每個 `.vue` 逐顆加 `class` 調高度（應優先走全域 theme）
- 用非 0.25 倍數的 spacing（Tailwind v4 不支援，需改用 `px-[11px]` 任意值）

## 8. 使用建議

### 8.1 何時用哪個 size

| 場景 | 建議 size |
|------|----------|
| Admin 一般操作按鈕 | `md`（預設，可不寫） |
| Table row action | `sm` |
| Mobile / tablet 表單區 | `lg` |
| Login 提交、Header 登出等主要 CTA | `xl` |
| 僅 icon 的 utility button | `sm` + `square` |

### 8.2 局部 override 範例

單顆按鈕特殊圓角（不影響全站）：

```vue
<UButton size="xl" class="rounded-sm" />
```

## 9. 修訂紀錄

| 日期 | 內容 |
|------|------|
| 2026-07-06 | 初版：記錄 UButton 客製化 scale、Tailwind v4 spacing、對齊策略 |
