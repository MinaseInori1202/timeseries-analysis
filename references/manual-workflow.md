# 預設的手動流程（Manual workflow）

本檔是 SKILL.md 各步驟的**預設做法**：走經典的 Box–Jenkins / ARIMA 思維，以 **ADF 與 KPSS 兩個平穩性檢定**與明確的統計轉換為主，
每個決策都由使用者確認，並在每一步都附上統計解讀與領域直覺（見 SKILL.md 的「每個結果都要解釋意義」原則）。
使用者若選擇 fpp3 的自動化工具（tsibble、`unitroot_kpss`／`unitroot_ndiffs`、`ARIMA()` 自動定階等），改讀 [`fpp3-options.md`](fpp3-options.md)。

> 平穩性檢定**ADF 與 KPSS 兩者都做**，兩者 H₀ 方向相反，結果依四種情況判讀（見下方第 5 節與 SKILL.md）；檢定前先看圖，選 c（無趨勢）或 ct（有趨勢）設定。

## 生態系與套件

| 用途 | 套件 / 函數 |
|------|-------------|
| 時間序列物件 | `stats::ts()`、`zoo`、`xts` |
| 單位根檢定 | `tseries::adf.test()`（主）、`fUnitRoots::adfTest()` / `unitrootTest()`（可指定 type、較彈性）、`tseries::pp.test()`（Phillips–Perron，可交叉驗證）、`urca::ur.df()` |
| 缺失值插補 | `zoo::na.approx()`（線性）、`zoo::na.locf()`（前向填補）、`imputeTS`（季節性插補） |
| 變異數穩定 | `log()`（指數成長序列）、`forecast::BoxCox()` |
| 結構識別 | `stats::acf()`、`stats::pacf()`、`TSA::eacf()`（ARMA 定階）、`stats::ar()`（AIC 自動選階） |
| 差分 | `base::diff()`、`forecast::ndiffs()`、`forecast::nsdiffs()` |
| ARCH 效果檢定（金融序列） | `FinTS::ArchTest()`（Engle's LM test，檢定是否有波動聚集） |
| 殘差診斷 | `stats::Box.test(type = "Ljung-Box")`、`qqnorm()`/`qqline()`、`stats::shapiro.test()` |
| 波動度建模 | `rugarch`（GARCH，金融常用）、`fGarch::garchFit()`（ARCH/GARCH 家族） |

本流程多用 R base graphics（`plot()`、`acf()`、`pacf()`），與統計諮詢報告的慣例一致；
ACF／PACF 一律用 base R 風格（`forecast::Acf()`／`Pacf()`：有外框、無格線、不畫 lag 0、`lag.max = 20`、橫軸標 `Lag`），並排時用下方的 `acf_pacf()`。

```r
library(tseries)
library(zoo)
library(TSA)      # eacf
```

## 1. 建立 ts 物件

```r
# frequency 依季節週期設定：月 = 12、季 = 4、週 = 52、日(週季節) = 7
y <- ts(raw_data$value, start = c(2015, 1), frequency = 12)
```

若時間有跳號，先在 data.frame 階段把完整時間軸建好、對齊，再轉 `ts`——
**不要靠刪列來讓長度湊齊**，那會破壞時間間隔。

> **長季節（日資料的年週期 s≈365、或同時有週季節 7）本 skill 不處理。** 不要用 `ts(frequency = 365)` 去配 SARIMA(s=365)——階數太高、幾乎估不動、也不穩；告知使用者超出涵蓋範圍，並在報告註明未處理季節。

## 2. 缺失值處理（給使用者選項，由使用者決定）

> 流程上，缺失值在資料健康檢查（與要做預測時的預測設定）之後、視覺化之前處理（SKILL.md 第 2 階段）；有樣本外資料時只對訓練段判斷與補值。程式上補值函數需要 `ts` 物件，所以本檔放在建立 ts 之後。

有缺失時，先判斷缺失是否帶有資訊（如假日休息），再把選項與依據簡述給使用者（完整選項表見 SKILL.md 第 2 階段，依據為 fpp3 §13.9、`arima()` 說明文件、`forecast::na.interp()` 說明文件、Moritz & Bartz-Beielstein 2017）。選定方法後的寫法：

```r
# 選項 A：不補，直接建模（arima() 在 method = "ML" 下可處理 NA）
# fit <- forecast::Arima(y, order = c(1,1,1))
# acf(y, na.action = na.pass)                  # 有 NA 時 ACF／PACF 預設會報錯，要加 na.action = na.pass
# Box.test(residuals(fit), type = "Ljung-Box") # 內部已用 na.pass，可直接用；不要 na.omit（ts 會報錯，刪掉又破壞時間間隔）

# 選項 B：線性內插（缺口短、序列平滑）
y <- zoo::na.approx(y)            # 或 forecast::na.interp(y)

# 選項 C：前向填補（階梯狀、狀態延續的序列）
# y <- zoo::na.locf(y)

# 選項 D：季節性插補（季節強、缺口短）
# y <- imputeTS::na_seasplit(y)
```

偏離值只用時間圖目視（本 skill 不做異常值的統計檢驗）；被看到的點不等於資料錯誤，是否處理要有領域理由，並在報告記錄。

## 3. 時間圖與變異數穩定（log / Box–Cox）

先畫原始序列的時間圖，看趨勢、季節、循環與變異數是否隨水準放大：

```r
plot(y, col = "steelblue", lwd = 1.5,
     main = "", ylab = "數值（單位）", xlab = "時間")   # 不放圖內主標題，圖名由 fig-cap 在下方給
```

**判斷是否需要 log 轉換**：若序列呈**指數成長**、或季節波動幅度隨水準等比放大（乘法型季節），
先取對數把它轉成近似線性趨勢、加法型季節，穩定變異數後再往下做：

```r
log_y <- log(y)     # 指數成長 / 乘法季節時使用
```

**判斷時要用圖與量化數字（不要只憑感覺）**：先看原始尺度的時間圖（上面的 `plot(y)`）與季節圖（`forecast::ggseasonplot(y, continuous = TRUE)`），再量化「波動幅度是否隨水準增大」：把序列切成連續的數段（有年週期的月、季資料以「年」為一段，讓每段含完整季節；沒有年週期或不到兩年時，切成等長的連續數段），計算各段的**標準差（波動幅度）與平均（水準）**，看標準差是否隨平均上升，轉換前後各算一次：

```r
# 分段：有年週期且至少兩年 → 以年為一段；否則切成等長的連續數段（段數在報告註明）
seg <- if (frequency(y) > 1 && length(y) >= 2 * frequency(y)) floor(time(y)) else cut(seq_along(y), breaks = 5, labels = FALSE)
s  <- tapply(y, seg, sd)                                # 各段標準差（波動幅度）
mu <- tapply(y, seg, mean)                              # 各段平均（水準）
rbind(sd = s, mean = mu)                                # 標準差隨平均上升 → 變異數隨水準增大，考慮對數／Box–Cox
```

> 解讀提醒：log 轉換不是為了好看，而是讓「變異數固定」這個平穩性前提更接近成立；
> 之後的差分、ADF、ARIMA 都在 log 尺度上進行，預測值最後再 `exp()` 轉回原尺度並說明。

（股價／價格型資產請改用對數報酬 `diff(log(price))`，見 SKILL.md 特例；
報酬序列可再用 `FinTS::ArchTest()` 檢定 ARCH 效果／波動聚集。）

## 4. 視覺化與結構識別（ACF / PACF / EACF / ar）

這是判斷序列結構、為後續定階鋪路的核心。四樣工具搭配使用：

```r
# 最方便：一次畫出「序列圖 + ACF + PACF」，定階時很好用
acf_pacf <- function(x, lag.max = 20) {              # ACF 與 PACF 並排（base R：有外框、無格線、不畫 lag 0）
  op <- par(mfrow = c(1, 2), mar = c(4.2, 4.2, 1, 1)); on.exit(par(op))
  forecast::Acf(x,  lag.max = lag.max, main = "", xlab = "Lag", ylab = "ACF")
  forecast::Pacf(x, lag.max = lag.max, main = "", xlab = "Lag", ylab = "PACF")
}
acf_pacf(y)

# 或分開畫：
par(mfrow = c(1, 2))
par(mfrow = c(1, 1))

TSA::eacf(as.numeric(y))          # 擴展 ACF，找 ARMA(p,q) 候選頂點
a <- ar(y, method = "mle")        # 以 AIC 選 AR 階數：a$order；a$aic 為各階數與最小 AIC 的差（< 2 的也列為選項）
```

**判讀規則（Box–Jenkins）**：

- ACF 拖尾、PACF 在 p 階後截斷 → 偏 **AR(p)**。
- ACF 在 q 階後截斷、PACF 拖尾 → 偏 **MA(q)**。
- 兩者都拖尾 → 偏 **ARMA**，用 **EACF** 找「o 三角形頂點」定 (p, q)，再以 AIC/BIC 裁決。
- 有季節時：**先依週期長度做季節差分再定階**，並分「季節部分 (P,Q)：只看 s,2s,3s 週期點」與
  「非季節部分 (p,q)：只看前半段短期 lag」兩段判讀——完整規則見 SKILL.md 的「SARIMA 定階規則」。

`ar()` 常選出較高階數（反映序列持續性）；報告時可**兼顧可解釋性**，挑選有明確圖形依據的較精簡階數，並說明取捨理由。

（上面是識別結果；模型配適與診斷的銜接見 `arima-garch-bridge.md`。）

## 5. 平穩性與差分（ADF 與 KPSS 兩者都做）

兩個檢定的 H₀ 方向相反：**ADF 的 H₀ = 有單位根（非平穩），p < 0.05 → 平穩；KPSS 的 H₀ = 平穩，p < 0.05 → 不平穩**。

**第一步：先畫圖，選 c 或 ct 設定。** 有明顯趨勢 → ct（含常數與趨勢）；沒有明顯趨勢、圍繞某個固定水準 → c（只含常數）。ADF 與 KPSS 用對應設定（c 對 `Level`、ct 對 `Trend`）：

```r
plot(y); acf_pacf(y)                                           # 先看圖：序列圖、ACF、PACF

# 有明顯趨勢（ct）
fUnitRoots::adfTest(y, lags = 1, type = "ct")    # 落後期數 lags 依 PACF 判斷（Tsay 講義例：由一階差分序列的樣本 PACF 選 p）
tseries::kpss.test(y, null = "Trend")
# 或 tseries::adf.test(y)：預設同時含常數與線性趨勢（等同 ct），落後期數預設 trunc((n-1)^(1/3))

# 無明顯趨勢、圍繞某個水準（c）
# fUnitRoots::adfTest(y, lags = 1, type = "c")
# tseries::kpss.test(y, null = "Level")
```

- 嚴謹一點也可用 `urca::ur.df()` 指定 `type = c("none", "drift", "trend")`（實測 `tseries::adf.test()` 的統計量與 `urca::ur.df(type = "trend")` 完全相同），並在報告說明選了哪一型與理由。`pp.test(y)`（Phillips–Perron）是另一個單位根檢定，可交叉驗證。

**第二步：兩個檢定的結果組合，分四種情況判讀**（完整表格與限制見 SKILL.md 第 4 階段、`textbook-methods.md`）：

| 情況 | ADF | KPSS | 處理 |
|---|---|---|---|
| 一 | 拒絕 H₀ | 不拒絕 H₀ | 明確平穩，d = 0（**ct 設定時為趨勢平穩，須先處理趨勢**，見下）|
| 二 | 不拒絕 H₀ | 拒絕 H₀ | 明確非平穩，差分後重新看圖、選 c／ct、再檢定 |
| 三 | 不拒絕 H₀ | 不拒絕 H₀ | 資訊不足：增加樣本、看圖找異常點與明顯轉折；列出各項結果，由使用者決定 d |
| 四 | 拒絕 H₀ | 拒絕 H₀ | 可能趨勢平穩或分數整合：看圖有無趨勢、檢查 c／ct 設定、可嘗試去趨勢；列出各項結果，由使用者決定 |

**用 ct 設定得到情況一時是「趨勢平穩」**：序列圍繞一條趨勢線平穩，不能直接以 d = 0 配 ARMA，要先處理趨勢（例如納入時間趨勢項、或先去趨勢；改用差分要留意過度差分），由使用者決定。

不平穩就差分。**一般差分階數 d 的判斷：先看圖，再用 ADF＋KPSS 的四種情況判斷**。**若使用者選擇處理季節效應**，d 要對「季節差分後」的序列判斷（原序列的檢定受季節與趨勢干擾；季節的處理見下一小節）；**預設（不處理季節）**則對原序列判斷，並在報告註明季節性可能干擾檢定結果：

```r
plot(y); acf_pacf(y)                                           # 先看圖：ACF 緩慢遞減 → 提示仍不平穩
# 依圖選 c／ct，兩個檢定都做（見上）→ 依四種情況判讀
y_d <- diff(y, differences = 1)                  # 情況二：再做一般差分
plot(y_d); acf_pacf(y_d)                                       # 差分後重新看圖、重新選 c／ct（差分後常見為圍繞固定水準 → c），兩個檢定再做一次
# 若使用者選擇處理季節：先 y_s <- diff(y, lag = 12)（季資料 lag = 4），把上面的 y 換成 y_s
```

- **順序（使用者選擇處理季節時）**：先季節差分 → **看季節差分後的 ACF／PACF**（實測 log(AirPassengers)：ACF 自 lag 1 的 0.71 起緩慢遞減，連續到 lag 8 才落回信賴區，提示需要再差分）→ ADF 佐證 → 必要時一階差分並重驗。
- **不要直接用 `ndiffs()` 的預設值決定 d**：它只看 KPSS，且直接丟原序列時沒有先扣掉季節。
- 報告寫明 ADF 與 KPSS 的設定：**含常數與趨勢項與否（c 或 ct）**、**落後期數**。常數與趨勢的選擇依時間圖判斷（序列是否圍繞某個固定水準、有無明顯斜率）。
- p 值落在邊際（約 0.05–0.10）時註明為邊際結果。實測（季節差分後的 log(AirPassengers)，lags = 5）：三種設定（無常數無趨勢、含常數、含常數與趨勢）的統計量為 −1.00、−2.69、−3.19，5% 臨界值為 −1.95、−2.88、−3.43，**三種設定都不拒絕單根**，本例結論不受設定影響。
- **實測（log(AirPassengers)）**：原序列 ADF p = 0.01（誤判平穩，該序列明顯有趨勢與季節）；季節差分後 p = 0.093（邊際、不顯著 → 再一般差分）；季節＋一般差分後 p = 0.01（平穩）→ D = 1、d = 1。同一份季節差分序列，兩種設定（皆用 R 實測，落後期數 k = 5）：ct 設定 ADF 統計量 −3.19、p = 0.093，KPSS（`null = "Trend"`）p > 0.10；c 設定 ADF 統計量 −2.69、p = 0.082，KPSS（`null = "Level"`）p = 0.091。兩種設定都是 ADF 不拒絕、KPSS 也不拒絕，屬**情況三（資訊不足）**，不能自動決定 d，要列出各項結果並以差分後 ACF 與殘差診斷裁決、由使用者決定。
- **設定會改變結論（實測，原序列 log(AirPassengers)，有趨勢與季節）**：ct 設定 ADF p < 0.01、KPSS（Trend）p > 0.10（情況一）；c 設定 ADF p = 0.64、KPSS（Level）p < 0.01（情況二）。同一份資料，c 與 ct 給出相反的結論，所以要先看圖選設定。季節與趨勢同時存在時，ADF 與 KPSS 都不是為季節設計的，結論要與圖形對照。

### 季節效應與季節差分 D（預設不處理）

**本 skill 預設不處理季節效應**（不做季節差分、不配季節項），看到明顯季節性時要**先告訴使用者**，並詢問是否需要處理。
標準 ADF 只檢定**零頻率（非季節）單根**，對季節頻率的單根不敏感，也可能被確定性季節誤導（報告中常見：未差分序列 ADF 也拒絕 H₀），所以 ADF 通過不代表季節已消除。

**使用者需要處理季節時：提供 D 的建議，並附 ACF／PACF 圖。**

```r
library(forecast)
fit <- auto.arima(y)                       # 讓套件自動選階
arimaorder(fit)                            # c(p, d, q, P, D, Q, m)：讀其中的 D 當建議
plot(y); acf_pacf(y)                                     # 附圖一：原序列的序列圖、ACF、PACF
acf_pacf(diff(y, lag = 12))                              # 附圖二：季節差分後（月資料 lag = 12；季資料 lag = 4）
```

- **這只是套件自動選階的建議**：`auto.arima()` 預設用套件內建的季節強度判準（不是單根檢定）。**實測 log(AirPassengers)**：建議 D = 1（`arimaorder` 為 0,1,1,0,1,1,12）。
- 附圖後，**由使用者依圖與分析目的決定 D**（0 或 1）；報告如實寫出採用的 D 與依據。
- **過度差分**要避免——差過頭會製造假自相關、放大變異數。差分後重畫 ACF/PACF 確認結構，並決定 (d, D)。

## 建模（提示）

探索與前處理完成後，通常接著配適 ARIMA/SARIMA、依準則比較、並做**殘差診斷**才算完整：

- **選模準則先問使用者，並說明依據**（提問範本與出處見 SKILL.md「選模準則要先問使用者」）：以預測為主 → AIC／AICc 或交叉驗證；想要較精簡、可解釋的結構 → 可考慮 BIC（但 BIC 並非解釋專用）。


- 殘差白化檢定：`Box.test(resid, lag = 10, type = "Ljung-Box", fitdf = p + q)`，p > 0.05 表殘差已白化。
- 殘差常態性：`qqnorm()` + `qqline()`、`shapiro.test()`。

**預測：點估計 + 預測區間圖（務必畫出來）**。用 `forecast` 套件最省事——`forecast()` 產生點預測與 95% 預測區間（本 skill 只給 95%，用 `level = 95`），`autoplot()` 直接把預測疊在歷史序列上：

```r
library(forecast); library(ggplot2)
fc <- forecast(fit, h = 12, level = 95)   # fit 為 Arima()/auto.arima() 物件；h = 預測期數
autoplot(fc) +
  autolayer(fc$mean, series = "點預測") +          # 讓「點預測」進圖例
  labs(title = "預測：點估計與 95% 預測區間",
       x = "時間", y = "數值",
       colour = "序列", fill = "預測區間") +        # 區間層級(95)列在 fill 圖例、點預測列在 colour 圖例
  theme(legend.position = "right")
# 取數值：fc$mean（點預測）、fc$lower / fc$upper（level = 95 時各只有 95% 一欄）
```

> **一定要有圖例**：`forecast::autoplot(fc)` 的陰影區間會有 `level` 圖例（`level = 95` 時只有 95）；再用
> `autolayer(fc$mean, series = "點預測")` 讓點預測也出現在圖例，讀者才分得清點估計與區間。

- **log／Box–Cox 尺度**：用 `Arima(y, lambda = 0)` 或 `forecast(fit, lambda = 0, biasadj = TRUE)`，`autoplot` 會**自動反轉換回原尺度**、區間也一併還原（不用自己 `exp()`）。
- **手動法**（如作業風格，base plot）：`p <- predict(fit, n.ahead = 12); upper <- p$pred + 1.96*p$se; lower <- p$pred - 1.96*p$se`，
  再把歷史末點接到預測首點畫線、加上區間虛線，並用 `legend()` 標出「觀測值 / 點預測 / 95% 區間」：
  ```r
  legend("topleft", legend = c("觀測值", "點預測", "95% 區間"),
         col = c("black", "tomato", "tomato"), lty = c(1, 1, 2), lwd = c(1.5, 2, 1), bty = "n")
  ```
- **判讀**：預測區間會**隨 horizon 變寬**（不確定性累積）；金融報酬的點預測常接近常數，重點在區間寬度與波動（條件變異數／GARCH 的區間更有意義）。

先把上述識別、平穩化、預測結果整理清楚，讓建模階段能直接接手。

## 產出提醒

本流程的重點是「每個決策可檢視」，報告裡每個決策都要寫清楚**用了什麼檢定、統計量與 p 值、為何這樣轉換／補值、
為何差分幾次**，並附上領域直覺解讀（報告本文以客觀敘述呈現，不寫「以便讀者複核」之類字句）。
