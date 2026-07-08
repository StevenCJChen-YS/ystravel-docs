# Industry PDF Match Analysis

- PDF: `C:\Users\HPC091\Downloads\dcc6e98cd08a917119a8cb177b1eb52e (1).pdf`
- Excel: `C:\Users\HPC091\Desktop\客戶分析報表_綜合.xlsx`
- Parsed company names from PDF: `4,032`
- Unique `來源帳號` in Excel: `6,766`
- Source accounts with some match candidate: `2,784`
- Source accounts with no current match: `3,982`

## Matching Strategy

- `raw_exact`: Excel 與 PDF 公司名稱完全相同
- `normalized_exact`: 去掉 `股份有限公司`、`有限公司`、分公司、尾端聯絡人後相同
- `root_or_contains`: 再進一步去掉 `旅行社`、`國際`、`旅遊` 等常見字後相同或互相包含

## Top Excel Source Account Match Samples

- `永信旅行社股份有限公司` (2204): `raw_exact` -> 永信旅行社股份有限公司
- `心典旅行社` (2196): `normalized_exact` -> 心典旅行社股份有限公司
- `興國旅行社` (1920): `no_match` -> []
- `易遊網` (1577): `root_or_contains` -> 易遊網旅行社股份有限公司, 易遊網旅行社(股)公司高雄分公司, 國際網旅行社有限公司
- `威順旅行社-田慈慧` (1179): `normalized_exact` -> 威順旅行社有限公司
- `瘋玩客-卓秀玲` (1151): `root_or_contains` -> 瘋玩客國際旅行社有限公司
- `好漾國際` (1119): `root_or_contains` -> 好漾國際旅行社股份有限公司
- `旅遊經理人-陳涬燕` (994): `normalized_exact` -> 旅遊經理人
- `全聯旅行社-何思霈` (981): `normalized_exact` -> 全聯旅行社股份有限公司
- `旅遊經理人` (902): `raw_exact` -> 旅遊經理人
- `易遊網-杜文馨` (868): `root_or_contains` -> 易遊網旅行社股份有限公司, 易遊網旅行社(股)公司高雄分公司, 國際網旅行社有限公司
- `星盟旅行社` (793): `normalized_exact` -> 星盟旅行社股份有限公司
- `第一旅行社` (708): `normalized_exact` -> 第一旅行社股份有限公司
- `加勒比海-尤月英` (655): `root_or_contains` -> 加勒比海旅行社有限公司
- `大吉旅行社-李君福` (629): `normalized_exact` -> 大吉旅行社有限公司
- `旅遊經理人-王翾沂` (626): `normalized_exact` -> 旅遊經理人
- `富友旅行社` (622): `normalized_exact` -> 富友旅行社有限公司
- `永勁` (580): `root_or_contains` -> 永勁旅行社股份有限公司, 永勁旅行社股份有限公司高雄分公司
- `嘉誠旅行社` (560): `normalized_exact` -> 嘉誠旅行社有限公司
- `噶瑪蘭-劉運春` (543): `root_or_contains` -> 噶瑪蘭旅行社股份有限公司
- `快安` (530): `root_or_contains` -> 快安旅行社有限公司
- `建成旅行社` (515): `normalized_exact` -> 建成旅行社有限公司
- `華適旅行社` (463): `normalized_exact` -> 華適旅行社有限公司
- `羊春麵-吳子寧` (460): `root_or_contains` -> 羊春麵旅行社有限公司
- `天一旅行社` (401): `normalized_exact` -> 天一旅行社股份有限公司
- `六千金` (394): `root_or_contains` -> 六千金旅行社有限公司
- `永信旅行社` (390): `normalized_exact` -> 永信旅行社股份有限公司桃園分公司, 永信旅行社股份有限公司, 永信旅行社股份有限公司高雄分公司, 永信旅行社股份有限公司台中分公司
- `鈺禾旅行社` (361): `normalized_exact` -> 鈺禾旅行社有限公司
- `建成旅行社有限公司` (355): `raw_exact` -> 建成旅行社有限公司
- `百翔旅行社` (350): `normalized_exact` -> 百翔旅行社有限公司, 百翔旅行社有限公司台南分公司

## Key Judgment Principle

- `首都旅行社` vs `首都` 這類情況，不會直接當作 100% 自動命中。
- 我會先做名稱正規化，把公司尾碼、分公司、聯絡人後綴拿掉。
- 若只剩短根字相同，會列為 `candidate`，而不是直接當正式命中。
- 真正要落地進同業/直客分類時，建議分成 `auto-match`、`review`、`no-match` 三層。

## Suggested Production Rule

- 完全相同 or 正規化後相同 -> 可自動判為同業
- 只有短根字相同、包含關係、或名稱過短 -> 進人工覆核
- 無法對應 -> 暫列直客候選，但保留覆核欄位
