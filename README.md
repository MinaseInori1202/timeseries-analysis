# timeseries-analysis

時間序列分析的 **Claude Skill**，以 R 語言撰寫。流程由**使用者逐步掌控**：每個階段做完先給你看結果，遇到需要選擇的步驟（檢定、差分階數、補值、模型階數、選模準則…）會說明各選項與依據，由你決定。

給 Claude 一句話說明資料與目的，它會一步一步帶你走完分析，最後交付一份可重跑的 Quarto 報告（`.qmd` 原始檔 + 編譯好的 `.html`）。

---

## 特色

- **自主可控，不替你選方法** —— 預設是手動、逐步確認的流程（ADF 與 KPSS、看 ACF/PACF 定階）；每個需要選擇的步驟會簡述選項，由你決定
- **fpp3 自動化工具是「可選項」** —— `tsibble`／`feasts`／`fable`、`unitroot_ndiffs()`、`ARIMA()` 自動定階、交叉驗證等，在你選擇時才使用，並說明依據與限制
- **資料有多個變數時先問你** —— 只分析一個、分別分析；想用其他欄位解釋某個變數（外生變數）會在健康檢查後問你；多對象的長表會先問是否協助整理，不自行合併
- **金融序列專用分支** —— 金融資產價格預設轉對數報酬（並告訴你理由）、六項摘要統計、ARCH effect 檢定、GARCH 方向判斷
- **建模到預測的完整流程** —— 候選模型配適與比較（選模準則先問你）→ 殘差診斷 → 預測與 95% 預測區間（取過對數會反轉換回原尺度）
- **殘差兩層診斷** —— 一階動差（Ljung–Box）與二階動差（ARCH），**兩層結論都必須寫進報告**
- **預設不處理季節效應，並會告知你** —— 看到明顯季節性時，先說明限制並詢問是否處理；你需要時才給季節差分 D 的建議（`auto.arima()`），並附 ACF／PACF 圖，兩種判準的建議不一致時都告訴你
- **長季節不處理** —— 日資料年週期等長季節／多重季節不套 SARIMA(365)，會告知使用者本 skill 不處理
- **外生變數先問你** —— 健康檢查後問分析目標與要不要考慮外生變數；要的話預設用資料中其他欄位，做迴歸加 ARIMA 誤差或迴歸加 GARCH 誤差
- **檢定寫法有規範** —— Ljung–Box 必須寫出 H₀／H₁、滯後期數與結果（依 Tsay 講義：一般 m = 10；GARCH 後標準化殘差及其平方各報 Q(10)）
- **報告格式已規範** —— 三線表、中英分開字型、單一自足 HTML、預測圖必附圖例、圖內不放主標題、程式碼可折疊

---

## 環境需求

| 項目 | 說明 |
|---|---|
| R | 4.x 以上 |
| Quarto | 用來把 `.qmd` 編譯成 `.html`（[下載](https://quarto.org/docs/get-started/)）|

R 套件（依實際用到的做法安裝，不必全裝）：

```r
# 共通
install.packages(c("flextable", "e1071", "gridExtra"))

# 預設手動流程
install.packages(c("forecast", "tseries", "TSA", "zoo", "urca", "lmtest", "sandwich"))

# 選用：fpp3 工具（你選擇時才需要）
install.packages("fpp3")          # 內含 tsibble / feasts / fable

# 金融序列（股價、報酬、波動度）
install.packages(c("quantmod", "FinTS", "rugarch", "fGarch"))

# 選配
install.packages(c("gtsummary", "imputeTS"))
```

> **注意**：`library(fpp3)` 之後再 `library(FinTS)` 會遮蔽 `fable` 的 `ARIMA()`。
> 請**先載入 `FinTS`**，或改用命名空間呼叫 `FinTS::ArchTest()`。

---

## 安裝

### 方法一：請 Claude 幫你裝（最簡單）

1. 點本頁綠色 **Code** → **Download ZIP**
2. 開啟 Claude，直接跟它說：

```
我下載了 timeseries-analysis-skill-main.zip，請幫我安裝這個 skill
```

Claude 會自動解壓並放到正確位置。

### 方法二：手動放置

解壓後把資料夾放到 Claude 的 skills 目錄，並命名為 `timeseries-analysis`（**資料夾名稱必須與 `SKILL.md` 的 `name` 相同**）：

```
~/.claude/skills/timeseries-analysis/     # 個人層級（所有專案都可用）
.claude/skills/timeseries-analysis/       # 專案層級（只在該專案可用）
```

Windows 的 `~` 是 `C:\Users\<你的帳號>`。

### 確認安裝成功

資料夾結構應如下，且 **`SKILL.md` 必須在最外層**：

```
timeseries-analysis/
├── SKILL.md
├── assets/
├── docs/
├── examples/
└── references/
```

放好後**重新啟動 Claude**，skill 才會生效。

---

## 使用方式

輸入 `/timeseries-analysis` 觸發時，skill 會先問要分析哪份資料：範例 1（NVIDIA 日資料）、範例 2（美國公債利率週資料），或你自己的資料。

也可以直接用自然語言說明**資料**與**目的**：

```
我要分析 R 內建的 AirPassengers（1949–1960 每月國際航空旅客數）。
這份要寫進統計諮詢報告，需要嚴謹、能交代統計依據，請幫我做時間序列分析。
```

Skill 會：

1. 看過資料後，說明資料的樣貌；資料有多個變數或多個對象時，**先問你要分析什麼**
2. **逐階段**完成探索、前處理、平穩性與差分、定階、配適與比較、殘差診斷、預測，每階段做完先給你看結果
3. 遇到**需要選擇的步驟**時，簡述選項（預設的手動做法，以及可選的 fpp3 做法），由你決定
4. 產出 `report.qmd` + `report.html` + 乾淨資料

其他例子：

```
用 quantmod 抓台積電近三年股價，幫我做時間序列分析。
我有每小時的伺服器流量，之後會持續進資料，想建可自動重跑的預測流程。
```

---

## 預設做法與可選項

| 需要選擇的步驟 | 預設（手動） | 可選（fpp3 或自動化）|
|---|---|---|
| 缺失值 | 確認缺口存在後，以明確的統計方法補並記錄 | `fill_gaps()` 先顯性化 |
| 平穩性檢定 | **ADF 與 KPSS 兩者都做**（先看圖選 c 或 ct 設定，結果依四種情況判讀）| `unitroot_kpss()`／`unitroot_ndiffs()` |
| 季節效應與 D | **預設不處理**（會告知你）；你需要時以 `auto.arima()` 給 D 的建議並附 ACF／PACF 圖 | `nsdiffs()`／`unitroot_nsdiffs()` |
| 一般差分 d | 先看 ACF/PACF 與圖，再以 ADF＋KPSS 的四種情況判斷（處理季節時對季節差分後的序列判斷）| `unitroot_ndiffs()` |
| 模型階數 | ACF / PACF 人工判讀，比較候選模型 | `auto.arima()`／`ARIMA()` 自動定階、ETS、多模型並列 |
| 選模準則 | 由你選（AIC／AICc／BIC／交叉驗證，附出處說明）| `glance()`／`accuracy()`、交叉驗證 |

> 兩種平穩性檢定的虛無假設**方向相反**（ADF 的 $H_0$ 是非平穩、KPSS 的 $H_0$ 是平穩），
> 報告中若同時引用務必註明，避免 p 值誤讀。

---

## 產出

| 檔案 | 說明 |
|---|---|
| `report.qmd` | Quarto 原始檔，可複核、修改、重跑 |
| `report.html` | 單一自足檔（CSS/JS/圖片內嵌），可直接寄送或列印 |
| 乾淨資料 | 已處理缺失／轉換，附前處理紀錄 |

報告格式規範：三線表（`flextable` + `theme_booktabs`）、中文標楷體 + 英數 Times New Roman、
圖說在下／表說在上、預測圖必附圖例、**預設不含 `sessionInfo()` 附錄**。

**報告內建程式碼折疊**：HTML 預設只顯示圖、表與解讀，每段程式碼收合在「顯示程式碼」列，讀者點擊即可展開或收合（YAML 為 `code-fold: true` 搭配 `execute: echo: true`）。

---

## 檔案結構

```
timeseries-analysis/
├── SKILL.md                          # 主流程（預設做法與可選項、各階段、報告規範）
├── assets/
│   └── report-template.qmd           # Quarto 報告範本（排版、輔助函數已設定）
├── docs/
│   ├── skill-guide.html              # 時間序列分析指南（觀念、流程圖、操作規格、範例資料說明），用瀏覽器開啟
│   └── workflow.png                  # 流程圖圖檔
├── examples/
│   └── data/                         # 測試用範例資料（NVIDIA 日資料、美國公債利率週資料），說明見指南「範例資料集」
└── references/
    ├── manual-workflow.md            # 預設的手動流程（含季節效應的 D 建議）
    ├── fpp3-options.md               # fpp3 可選工具
    ├── arima-garch-bridge.md         # 建模銜接：定階、殘差診斷、ARCH/GARCH、auto.arima（可選）
    ├── financial.md                  # 金融資產價格資料：對數報酬與必做清單
    ├── ljung-box-lags.md             # Ljung–Box 滯後期數 m 的依據（Tsay 講義查證）
    ├── decision-tree.md              # 分析流程決策樹（含流程圖，涵蓋配適比較與預測）：何時問使用者、常見誤用
    ├── workflow.drawio               # 流程圖的可編輯原始檔（draw.io）
    └── textbook-methods.md           # 報告圖表與檢定說明，給非統計背景的讀者（是什麼、何時用、怎麼看）
```

採**漸進式載入**：`SKILL.md` 為核心，`references/` 只在需要時才讀取。`docs/` 與 `examples/` 給使用者閱讀與測試，skill 執行時不會讀取。

---

## 參考資料

- Hyndman, R.J. & Athanasopoulos, G. (2021). *Forecasting: Principles and Practice*, 3rd ed. <https://otexts.com/fpp3/>
- Tsay, R.S. (2010). *Analysis of Financial Time Series*, 3rd ed.（Ch.2 迴歸模型含時間序列誤差；Ch.3 條件異質變異模型）
- tidyverts（tsibble / feasts / fable）<https://tidyverts.org>
- Quarto <https://quarto.org>
