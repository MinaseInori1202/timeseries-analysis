# fpp3 可選工具（fpp3 options）

本檔列出 SKILL.md 各步驟的 **fpp3 替代做法**，**只在使用者選擇時使用，不預設**。
這些工具偏向自動化（自動判定差分階數、自動定階、多模型並列、交叉驗證），適合使用者想要**可重現的自動流程、資料會持續進來、以預測表現挑模型**的情況；
但自動化的建議只是參考，**最後由使用者決定**，並在報告註明採用了什麼。預設的手動流程見 [`manual-workflow.md`](manual-workflow.md)。
本檔以 Rob Hyndman《Forecasting: Principles and Practice (3rd ed.)》的 tidyverts 生態系為準。
出處：<https://otexts.com/fpp3/>。

## 生態系與套件

一次載入 `fpp3` 就會帶入下列核心套件（全部走 tidyverse 管線 `|>`）：

- **tsibble**：時間序列資料結構（`index` + `key` + 規則間隔）。
- **feasts**：視覺化與特徵萃取（`autoplot`、`gg_season`、`gg_subseries`、`gg_lag`、`ACF`、`features`）。
- **fable**：建模與預測（差分階數自動判定會用到 `features()`）。
- 另含 `dplyr`、`ggplot2`、`lubridate` 等。

```r
library(fpp3)
```

## 1. 建立 tsibble

tsibble 的三個要素：**index（時間索引）**、**key（分組鍵，可多個）**、**interval（規則間隔）**。
即使沒 `select()`，tsibble 也會自動保留 index 與 key，以維持合法的時間結構。

依頻率選對時間類別函數：

| 頻率 | 函數 |
|------|------|
| 年 | 直接用整數年份 |
| 季 | `yearquarter()` |
| 月 | `yearmonth()` |
| 週 | `yearweek()` |
| 日 | `as_date()` / `lubridate::ymd()` |
| 次日（時/分/秒） | `as_datetime()` / `lubridate::ymd_hms()` |

```r
# 由既有 data frame 轉換
tsbl <- raw_data |>
  mutate(month = yearmonth(date)) |>
  as_tsibble(index = month, key = c(store, product))
```

### 隱含缺失值：先顯性化，不要隨手補

tsibble 的一大好處是能把「該有卻不見」的時間點抓出來。這些是 tsibble 套件本身的函數
（在 fpp3 正文各章不一定逐一示範，但屬 tidyverts 標準做法）：

```r
tsbl |> has_gaps()          # 有沒有間隔缺失
tsbl |> scan_gaps()         # 列出缺哪些時間點
tsbl |> count_gaps()        # 每條序列缺幾個
tsbl <- tsbl |> fill_gaps() # 把隱含缺失補成顯性的 NA（值先留 NA，之後再決定怎麼補）
```

`fill_gaps()` 只是把「跳號」補成一列值為 `NA` 的顯性缺失，**不是幫你填數值**。
要不要填、怎麼填（如 `tidyr::fill()` 前向填補、或 `imputeTS` 等）留到分解前再依資料性質決定，並在報告中記錄。

### 股價資料：先轉對數報酬

若是股價／價格型資產，依 SKILL.md 的通則，在建好 tsibble 後**先轉對數報酬再往下做**：

```r
tsbl <- tsbl |> mutate(log_return = difference(log(close)))
```

之後把 `log_return` 當成新對象**完整重跑探索分析**，別只轉換就跳過（見 `references/financial.md`的「必做清單」）：
報酬時間圖、**五數統計 + 箱形圖**、六項摘要表（mean/sd/skewness/excess kurtosis/min/max）、
`ACF(log_return)` 與 `ACF(log_return^2)` ＋ `FinTS::ArchTest()`、以及**平穩性一律做 ADF 與 KPSS 兩者**（見 SKILL.md 第 5 階段；fpp3 版的 KPSS 為 `features(., unitroot_kpss)`）。

## 2. 視覺化探索

```r
tsbl |> autoplot(value)                       # 時間圖
tsbl |> gg_season(value)                      # 季節圖
tsbl |> gg_subseries(value)                   # 子序列圖
tsbl |> gg_lag(value, geom = "point")         # 落後圖
tsbl |> ACF(value) |> autoplot()              # 自相關圖（correlogram）
```

ACF 判讀：藍色虛線是顯著界線；有趨勢時小 lag 大且正、緩慢衰減；有季節時在季節倍數 lag 出現高峰；
趨勢+季節會呈現緩降且帶「扇貝狀」的形狀。

## 3. 平穩性與差分（自動化）

（本 skill 預設 ADF 與 KPSS 兩者都做，見 SKILL.md；本節是 fpp3 的自動化做法。）fpp3 的做法是用 **KPSS**（`unitroot_kpss()`，**H0 = 平穩**）搭配 **自動決定差分階數**，
而非人工反覆看圖猜：

```r
tsbl |> features(value, unitroot_nsdiffs)  # 建議做幾次季節差分（預設判準為套件內建的季節強度，見下）

# 使用者選擇處理季節時，一般差分 d 要對「季節差分後」的序列判斷（先看圖，再檢定）
tsbl <- tsbl |> mutate(s_value = difference(value, lag = 12))
tsbl |> gg_tsdisplay(s_value, plot_type = "partial")   # 序列圖、ACF、PACF：ACF 緩慢遞減 → 提示仍不平穩
tsbl |> features(s_value, list(unitroot_kpss, unitroot_ndiffs))
#   KPSS：p < 0.05 → 需要再做一般差分；unitroot_ndiffs：建議一般差分次數

# 差分（先季節差分，再一階差分）
tsbl <- tsbl |>
  mutate(d_value = difference(difference(value, lag = 12), lag = 1))
```

判讀與注意：

- **不要把原序列直接丟進 `unitroot_ndiffs()` 決定 d**（沒有先扣掉季節）。實測（log(AirPassengers)）：原序列建議 1 次；季節差分後 KPSS p = 0.091（不拒絕平穩）、建議 0 次。p 值落在邊際（約 0.05–0.10）時，註明為邊際結果，並以差分後 ACF 與殘差診斷檢視是否過度差分。

- KPSS 的 `p` 值被截在 0.01–0.1 之間；`p < 0.05` 代表**拒絕平穩**、需要差分。
- **順序**：先季節差分再一階差分——季節差分後有時就已平穩，不必再做一階。
- **過度差分**會誘發實際不存在的假自相關，`unitroot_ndiffs()` 的建議就是用來避免差過頭。
- **`unitroot_nsdiffs()` 預設不是單根檢定**：它用套件內建的季節強度判準（強度 < 0.64 才免差分）。
  本 skill 預設不處理季節，細節見 `manual-workflow.md`「季節效應與季節差分 D」。也可再以 `unitroot_nsdiffs(..., unitroot_fn = ~ ...)` 換成自訂判準。

## 4. 變數轉換（Box–Cox / log）

當季節波動幅度**隨水準放大**時，先做 Box–Cox（λ=0 即 log）穩定變異。用 Guerrero 法自動選 λ，
再把轉換**直接寫進模型公式**——fable 會在預測時**自動反轉換並做 bias 校正**（回傳分布均值），不用自己 `exp()`：

```r
lambda <- tsbl |> features(value, guerrero) |> pull(lambda_guerrero)
tsbl |> autoplot(box_cox(value, lambda))
# 模型公式內用 box_cox(value, lambda) 或 log(value)，預測自動還原
```

> **先確認再轉換，別無腦套 Guerrero。** 當序列的**季節振幅其實很穩定、不隨水準放大**時，Guerrero 可能挑出
> **貼著邊界的極端 λ（如 ≈ −0.9）**，套進模型反而會讓預測出現 `NA`／`NaNs produced`。動手前先**目視**：振幅有隨水準
> 變大才轉；若沒有明顯的乘法型變異，就**不要轉換**（維持原尺度），並在報告說明理由。λ 靠近 0 用 `log` 較穩健。
>
> **「目視確認」要用這些程式碼，並附量化數字**：
> ```r
> tsbl |> autoplot(value)       # 時間圖：季節起伏的幅度是否隨整體水準放大（乘法型）、還是大致固定（加法型）
> tsbl |> gg_season(value)      # 季節圖：各年曲線平行（加法型）或逐年張開（乘法型）
> # 量化：逐年年內全距 vs 年平均水準（把 Quarter 換成你的時間欄；轉換前後各算一次）
> tsbl |> as_tibble() |> mutate(yr = year(Quarter)) |> group_by(yr) |>
>   summarise(range = diff(range(value)), mean = mean(value))
> ```
> 實測（`aus_production` 啤酒 1992–2010）：年內全距 122 → 90，年平均 451 → 430，**幅度沒有隨水準放大**，Guerrero 卻給出 λ = −0.900（貼著邊界）、套進模型後預測全為 `NA`——這種資料就不該轉換。報告要把圖與這組數字都放進去。

## 5. tidy forecasting workflow（fable 六步驟）

fpp3 的建模是一條標準管線：**tidy → visualise → specify → estimate → evaluate → forecast**。
核心是 `model()`（輸出 **mable**，可多模型並列）與 `forecast()`（輸出 **fable**）。

```r
# 先切訓練/測試（留最後約 20%，至少 ≥ 預測 horizon）——之後所有估計只用 train
train <- tsbl |> filter_index(. ~ "2007 Q4")   # 依你的時間範圍調整

fit <- train |>
  model(
    snaive = SNAIVE(value),                 # 季節樸素法，當基準線
    ets    = ETS(value),                    # 指數平滑，AICc 自動選 E/T/S
    arima  = ARIMA(value)                   # ARIMA，Hyndman–Khandakar 自動定階
  )
```

- **簡單基準法**：`MEAN()`、`NAIVE()`、`SNAIVE()`、`RW(value ~ drift())`——一定要放一個當比較基準。
- **`ETS(value)`**：省略設定即以 AICc 自動選；也可指定 `ETS(value ~ error("A") + trend("Ad") + season("M"))`
  （Trend：`N`/`A`/`Ad` 阻尼；Season：`N`/`A`/`M`）。
- **`ARIMA(value)`**：自動定階（用 KPSS 決定 d、最小化 AICc 搜 p,q）；手動用 `ARIMA(value ~ pdq(2,1,0) + PDQ(1,1,1))`；
  放寬搜尋加 `stepwise = FALSE, approximation = FALSE`。

## 6. 殘差診斷

好的模型其 innovation residuals 應**不相關、均值零**（最好等變異、常態）：

```r
fit |> select(arima) |> gg_tsresiduals()               # 殘差時序 + ACF + 直方圖 三合一
fit |> select(arima) |> report()                       # 先看實際選出的階數，決定 dof
# dof 必須等於「這個模型實際估計的 AR/MA 參數個數」＝非季節 p+q ＋ 季節 P+Q（不含差分 d、D）
augment(fit) |> features(.innov, ljung_box, lag = 8, dof = 1)   # lag/dof 為示例，請依你的模型改
```

- **`dof` 千萬別硬編數字**：它要對到 `report()` 顯示的實際階數（含季節項）。例如自動選出
  `(0,0,0)(0,1,1)[4]` 只有一個季節 MA → `dof = 1`；抄一個固定的 `dof = 3` 會統計上錯誤。
- `lag`：非季節建議 `10`，季節建議 `2*m`（m＝季節週期，如季資料 m=4 → `lag = 8`）。
- **判讀**：看 `lb_pvalue`——**p 大（> 0.05）→ 殘差近白噪音、模型足夠**；p 小 → 仍有殘留自相關。
- **要在 fable 工作流檢 ARCH**（見 `arima-garch-bridge.md` 的兩問決策）：先取 innovation 殘差再檢——
  `res <- augment(fit) |> filter(.model == "arima") |> pull(.innov)`，然後 `acf(res^2)` 與 `FinTS::ArchTest(res)`。

> **套件遮蔽陷阱（實測踩過）**：`library(fpp3)` 之後再 `library(FinTS)` 會**遮蔽 fable 的 `ARIMA()`**（公式介面報
> `unused arguments`）。解法：**先載 `FinTS` 再載 `fpp3`**，或全程用**命名空間呼叫** `FinTS::ArchTest()` 而不 `library(FinTS)`。

## 7. 模型比較與訓練／測試、交叉驗證

```r
# 同一模型類別內用資訊準則（fit 已在第 5 節於 train 上估計）
glance(fit) |> arrange(AICc) |> select(.model, AIC, AICc, BIC)

# 對「測試集」比較準確度：forecast 出 horizon 期，再和完整資料對
fc <- fit |> forecast(h = "3 years")
accuracy(fc, tsbl) |> arrange(RMSE)        # 用測試集比較誤差（tsbl 為含測試段的完整資料）
```

- 誤差量測：`RMSE`/`MAE`（尺度相依）、`MAPE`（近零值有問題）、**`MASE`/`RMSSE`（跨序列比較首選）**。
- **重要**：AICc **只能在同類別內比**；**ETS 與 ARIMA 之間不可用 AICc 比較**，要改用測試集或交叉驗證的 RMSE 挑贏家。

**時間序列交叉驗證（rolling origin）**：

```r
tsbl |>
  stretch_tsibble(.init = 10, .step = 1) |>   # 初始訓練長度 .init、每次擴張 .step
  model(ETS(value), ARIMA(value)) |>
  forecast(h = 1) |>
  accuracy(tsbl) |> select(.model, RMSE, MAE, MASE)
```

> 註：接近序列尾端的折疊可能噴 `future dataset is incomplete` 警告，屬**無害、預期內**，可忽略。
> 另外，測試集 RMSE 與 CV RMSE 未必給出同一名次（本就正常），報告時如實並陳、擇一為主即可。

## 8. 預測與預測區間（點估計 + 區間圖）

```r
fc <- fit |> forecast(h = "2 years")     # 或 h = 8
fc |> autoplot(tsbl) +                   # 點預測 + 陰影區間疊在歷史上（預設畫 80%/95%；本 skill 只給 95%：`autoplot(fc, tsbl, level = 95)`）
  ggplot2::labs(colour = "模型",
                title = "預測：點估計與 95% 預測區間")   # fable 預設就有圖例（模型色 + 區間層級 level）
fc |> hilo(level = 95)                   # 取數值化的 95% 預測區間
# 殘差非常態時用 bootstrap 區間（不對稱）：
fit |> select(arima) |> forecast(h = 8, bootstrap = TRUE, times = 1000) |> hilo()
```

- 多數模型假設常態，區間 = `ŷ ± c·σ̂_h`（95%→c≈1.96）；區間隨 horizon 變寬。
- 若公式裡用了 `box_cox()`/`log()`，`forecast()`/`autoplot()` 會自動反轉換回原尺度。
- **多模型並列時** `fc |> autoplot(tsbl)` 會把所有模型的預測都疊上去；只想看一個就先篩選，例如
  `fc |> filter(.model == "arima") |> autoplot(tsbl)`。
- **要把預測區間放進 flextable**：`hilo()` 產生的 `95%` 欄是 **hilo 物件**，不能直接 `colformat_double`；
  先拆成數值再製表，例如 `... |> mutate(lower = hilo95$lower, upper = hilo95$upper)`
  或 `... |> unpack_hilo("95%")`，取出 `lower`/`upper` 兩個數值欄。

> **ETS 還是 ARIMA？** fpp3 不給硬性偏好——兩者各有對方沒有的模型。實務上**兩個都 `model()` 進去**，
> 再用第 7 節的測試集或 CV 之 RMSE 挑贏家（不能用 AICc 跨類別比）。

## 預設手動流程與 fpp3 選項的對照（供使用者選擇時參考）

| 面向 | fpp3 選項 | 預設手動流程 |
|------|-----------|----------|
| 資料結構 | tsibble（index+key，時間語意由結構保證） | ts / zoo / xts 或排序好的 data.frame |
| 缺失處理 | `fill_gaps()` 先顯性化，再視需要填 | 直接統計插補（線性/季節/前向） |
| 平穩性檢定 | KPSS（H0=平穩） | ADF（H0=非平穩），兩者 p 值解讀相反 |
| 差分階數 | `unitroot_ndiffs()`/`nsdiffs()` 自動決定 | 人工看 ACF/PACF 與檢定反覆判斷 |
| 建模／選模 | `model()` 多模型（ETS/ARIMA/SNAIVE）並列，`accuracy()`/CV 挑贏家 | 逐一配 `Arima()`/`auto.arima()`，AIC/BIC + 殘差診斷 |
| 預測區間 | `forecast()` + `autoplot()`（自動反轉換） | `forecast()`/`predict()` + 手動 ±1.96·SE |
