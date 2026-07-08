# Customer Excel First Pass Analysis

- Source file: `C:\Users\HPC091\Desktop\客戶分析報表_綜合.xlsx`
- Analyzed sheet: `客戶分析報表220601-260621`
- Rows: `108,535`
- Columns: `40`

## Workbook Structure

- `客戶分析報表220601-260621`: 108,536 rows x 40 columns

## Column List

- `線別`
- `團號`
- `團名`
- `身分`
- `中文姓名`
- `英文姓`
- `英文名`
- `身份證字號`
- `性別`
- `稱謂`
- `出生日期`
- `行動電話`
- `聯絡電話`
- `護照號碼`
- `護照效期`
- `台胞證號碼`
- `台胞證到期日`
- `台胞證號碼.1`
- `台胞證到期日.1`
- `E-Mail`
- `聯絡資料:其他`
- `領隊`
- `團體天數`
- `訂單編號`
- `報名日期`
- `出團日期`
- `旅遊國家`
- `團費金額`
- `付款方式`
- `來源帳號`
- `通路來源`
- `參團方式`
- `參團狀態`
- `業務人員`
- `往來狀態`
- `警示訊息`
- `戶籍地址`
- `聯絡地址`
- `住家地址`
- `公司地址`

## Duplicate Header Bases

- `台胞證號碼` appears `2` times
- `台胞證到期日` appears `2` times

## Highest Missingness

- `戶籍地址`: 108,457 missing (99.93%)
- `聯絡電話`: 108,459 missing (99.93%)
- `警示訊息`: 108,439 missing (99.91%)
- `公司地址`: 108,388 missing (99.86%)
- `聯絡地址`: 108,146 missing (99.64%)
- `聯絡資料:其他`: 107,178 missing (98.75%)
- `E-Mail`: 104,787 missing (96.55%)
- `台胞證到期日`: 104,082 missing (95.9%)
- `台胞證到期日.1`: 104,082 missing (95.9%)
- `台胞證號碼`: 104,082 missing (95.9%)
- `台胞證號碼.1`: 104,082 missing (95.9%)
- `住家地址`: 102,501 missing (94.44%)
- `付款方式`: 97,522 missing (89.85%)
- `行動電話`: 92,370 missing (85.11%)
- `領隊`: 35,570 missing (32.77%)

## Order Statistics

- Rows with order number: `108,535`
- Unique order numbers: `16,286`
- Average rows per order: `6.66`
- Median rows per order: `2`
- Max rows in one order: `99`
- Orders with exactly 1 row: `3,239`
- Orders with >10 rows: `2,912`
- Orders with >20 rows: `1,722`

## Duplicate Column Check

- `台胞證號碼 vs 台胞證號碼.1`: same in `108,535` / `108,535` rows (`100.0%`)
- `台胞證到期日 vs 台胞證到期日.1`: same in `108,535` / `108,535` rows (`100.0%`)

## Traveler-Specific Checks

- `traveler_rows`: `105,813`
- `traveler_missing_id_and_passport`: `337`
- `traveler_with_name_and_birth`: `96,304`
- `traveler_with_any_id_document`: `105,476`
- `placeholder_name_rows`: `7,705`
- `passport_placeholder_rows`: `8,963`

## Key Field Highlights

- `身分`: non-null `108,535`, unique `3`, missing `0.0%`
  Top values: 旅客 (105813), 領隊 (2720), 導遊 (2)
- `訂單編號`: non-null `108,535`, unique `16,286`, missing `0.0%`
  Top values: 00024480 (99), 00025076 (99), 00045568 (99), 00034071 (99), 00010207 (97)
- `團號`: non-null `108,535`, unique `5,029`, missing `0.0%`
  Top values: SNA03261011A (153), LIN02260110S (140), TNN02250913S (136), KEE03RWA251024S (132), MAL01251123S (130)
- `中文姓名`: non-null `108,305`, unique `74,604`, missing `0.21%`
  Top values: 旅客1 (670), 旅客2 (666), 旅客3 (472), 旅客4 (438), 旅客5 (352)
- `身份證字號`: non-null `99,028`, unique `84,471`, missing `8.76%`
  Top values: B221917335 (65), H122579708 (57), H220814402 (56), H121244680 (55), F220749104 (52)
- `護照號碼`: non-null `105,236`, unique `81,862`, missing `3.04%`
  Top values: 未建旅客檔 (8968), 369046607 (65), 367515746 (57), 365677929 (56), 369570711 (55)
- `E-Mail`: non-null `3,748`, unique `2,395`, missing `96.55%`
  Top values: heidi.chenwang@gmail.com (52), sophiahlting@gmail.com (45), threicl132@yahoo.com.tw (43), a0930087010@gmail.com (37), james560412@gmail.com (36)
- `行動電話`: non-null `16,165`, unique `10,349`, missing `85.11%`
  Top values: 0919997092 (65), 0989261234 (58), 0955400225 (56), 0939666193 (55), 0933957103 (53)
- `旅遊國家`: non-null `86,766`, unique `165`, missing `20.06%`
  Top values: AT/CZ (11130), 越南 (4957), 日本 (4811), CH/IT (4656), 中國 (4357)
- `團費金額`: non-null `108,535`, unique `2,298`, missing `0.0%`
  Top values: 0 (9275), 39900 (1682), 69900 (1640), 88500 (1527), 79900 (1498)
- `付款方式`: non-null `11,013`, unique `1`, missing `89.85%`
  Top values: 支票 (11013)
- `通路來源`: non-null `108,534`, unique `8`, missing `0.0%`
  Top values: (1)同業介紹 (70122), (3)業務招攬 (30845), FILLO (3325), 同業介紹 (2212), 旅展銷售 (1473)
- `參團方式`: non-null `108,535`, unique `2`, missing `0.0%`
  Top values: 全程 (100233), JOIN TOUR (8302)
- `參團狀態`: non-null `108,535`, unique `2`, missing `0.0%`
  Top values: 確認 (107032), 預訂 (1503)
- `業務人員`: non-null `108,535`, unique `116`, missing `0.0%`
  Top values: 張伊瑶 (7383), 洪秋好 (6923), 黃亮諺 (6849), 陳怡樺 (6009), 蘇子豪 (5861)

## Initial Data Quality Concerns

- 身份證字號與護照號碼同時缺漏: `337` rows (`0.31%`)
- 行動電話與聯絡電話同時缺漏: `92,306` rows (`85.05%`)
- E-Mail 缺漏: `104,787` rows (`96.55%`)
- 旅遊國家 缺漏: `21,769` rows (`20.06%`)

## Recommendation For Next Round

- Confirm which `身分` values should be counted as valid traveler records.
- Define whether traveler matching should prioritize `身份證字號`, `護照號碼`, or a fallback combination.
- Define which `參團狀態` should count as active / completed / excluded.
- Confirm if analysis should be traveler-centric, order-centric, or both.
- Confirm the exact target outputs you want to extract after this first profiling pass.
