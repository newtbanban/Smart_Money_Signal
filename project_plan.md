# Smart Money Signal 研究專案 — V1 規劃(內部人集中買進)

## 專案定位
用SEC EDGAR官方內部人交易(Form 4)資料,研究「內部人集中買進(cluster buying)」是否對後續股價有預測力,並用嚴謹的event study + 樣本外backtest驗證。目標不是「證明能穩定獲利」,而是「展現完整、誠實、可重現的量化研究方法論」。V2再疊加Reddit散戶情緒做成smart/dumb money分歧指標。

---

## Phase 0 — 環境設定(0.5–1天)

- [ ] Python環境(建議用venv或conda),GitHub repo建好骨架(`data/`, `src/`, `notebooks/`, `README.md`)
- [ ] 決定股票範圍(universe):不要一開始就做全市場。建議先抓S&P 500或類似的大型股清單,樣本夠多、流動性夠好、資料乾淨度較高
- [ ] 決定回測期間:建議至少8-10年(例如2015–2024),因為後面要切in-sample/out-of-sample
- [ ] 核心套件:`pandas`, `numpy`, `edgartools`(或直接用SEC bulk data), `yfinance`(股價), `matplotlib`/`plotly`(視覺化)

**⚠️ 重要提醒(survivorship bias)**:如果你用「現在的」S&P 500成分股清單去回溯2015年的資料,會有存活者偏誤(當年可能有公司後來下市/被剔除,你的清單看不到它們)。V1可以先誠實在README註明這個限制,不用花時間解決point-in-time成分股問題,但一定要寫出來,面試官問到才不會被抓到破綻。

---

## Phase 1 — 資料取得(1–2天)

- [ ] 下載SEC官方「Insider Transactions Data Sets」:免費、季度更新、涵蓋2006年至今的Form 3/4/5結構化資料,來源:`sec.gov/data-research/sec-markets-data/insider-transactions-data-sets`。這是最省時間的路徑,不用自己爬蟲、不用處理rate limit
- [ ] 保留欄位:申報日(filedAt)、交易/事件日(period of report)、發行公司CIK/股票代號、申報人姓名/CIK、身份別(officer/director/10%owner)、交易代碼(P/S/A/D/F/M)、股數、價格、交易後持股、是否為10b5-1計畫交易(較新申報書會有這個flag)
- [ ] 用`yfinance`抓對應股票在回測期間的每日價格
- [ ] 如果需要更細節的單筆filing(例如footnote文字),可以用`edgartools`這個Python套件輔助解析

---

## Phase 2 — 資料清理與特徵工程(2–3天)

這是整個專案含金量最高的部分,務必做扎實:

- [ ] **只保留交易代碼P(公開市場買進)**,先排除A(獎勵/授予)、M(選擇權行使)、F(繳稅扣股)、D(其他處分)——這些不是「主動用自己的錢買」,雜訊很大
- [ ] **排除標記為10b5-1計畫的交易**——這些是幾個月前就排定好的,不反映"現在"的資訊
- [ ] **計算相對交易規模**:不要只看絕對金額,算這筆交易相對於這個人平常交易規模、或相對於其既有持股比例的異常程度
- [ ] **偵測cluster buying**:同一家公司,多位不同的officer/director,在短時間窗口內(例如5–10個交易日)都出現買進申報——這是訊號強度最高的情境
- [ ] CIK對應股票代號時要小心處理股票代號變更、下市、合併的情況

---

## Phase 3 — Event Study / 訊號衰減分析(2天,決定V1能不能成立的關鍵步驟)

- [ ] 對每個cluster buy事件,以**申報日(filing date)為第0天**(不是交易日!),計算相對大盤(用SPY或該股票的beta調整)的累積異常報酬(CAR),範圍抓事件前10天到事件後30-60個交易日
- [ ] 畫出平均CAR曲線,回答這個關鍵問題:**異常報酬主要發生在交易日到申報日之間(你看不到、複製不了的部分),還是在申報日之後還有延續(你能複製的部分)?**
- [ ] 如果申報日之後幾乎沒有殘留的異常報酬,代表這個訊號對「事後跟單」的你已經沒有意義了——這個結果本身就是有價值的研究發現,誠實寫出來,不用勉強做出一個有效的backtest

---

## Phase 4 — Backtest設計(2–3天)

- [ ] **切資料**:例如2015–2021當in-sample(拿來調參數、選門檻值),2022–2024完全鎖起來當out-of-sample,**在最終驗證前不准偷看**
- [ ] 策略邏輯:買進近期出現cluster buy訊號的股票組合,持有期依Phase 3的訊號衰減結果決定(例如20–60個交易日)
- [ ] **務必用申報日當進場點**,不是交易日——這是避免look-ahead bias的紅線
- [ ] 加入合理的交易成本假設(例如單邊10-20 bps),不要用零成本回測,那樣的結果沒有意義
- [ ] 鎖定參數後,**只在out-of-sample跑一次**,不要反覆調整後再測——那樣就等於偷看了

---

## Phase 5 — 評估(1天)

- [ ] 指標:年化報酬、Sharpe ratio、最大回撤、勝率
- [ ] 對照組:(1) buy-and-hold SPY (2) 不做任何過濾、所有insider買進訊號的「素樸版」——證明你Phase 2做的過濾有沒有真的提升訊號品質
- [ ] 做個簡單的統計顯著性檢驗(例如bootstrap重抽樣),不要只看點估計值

---

## Phase 6 — 交付物(2天)

- [ ] GitHub README:方法論、資料來源、關鍵發現、**誠實列出限制**(survivorship bias、樣本外表現、揭露延遲影響)
- [ ] 互動dashboard(可以用Claude Cowork或React做,呈現訊號跟backtest結果)
- [ ] 一篇簡短分析文章,講清楚"為什麼這樣設計"跟"發現了什麼",而不是只貼圖表

---

## Phase 7 — 未來方向(V2,不用現在做,但README可以提)

- [ ] 疊加dumb money訊號(Reddit散戶情緒),做成smart/dumb money分歧指標
- [ ] 用Alpaca之類的免費paper trading API做實盤模擬驗證
- [ ] 考慮point-in-time成分股清單解決survivorship bias

---

## 貫穿全程的方法論紅線(隨時自我檢查)

1. 用申報日,不是交易日,做為訊號可得的時間點
2. in-sample跟out-of-sample徹底分開,鎖定參數後不回頭調
3. 一定要有交易成本假設
4. 誠實報告"沒用"的發現,不要只留下"有用"的結果(這叫selective reporting,是研究誠信的大忌)
5. 每個過濾規則(10b5-1排除、cluster定義)都要能講出為什麼,不是憑感覺調出來的