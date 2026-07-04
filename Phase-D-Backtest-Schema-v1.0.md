# smc-pro Phase D · 自動化回測系統 Data Schema

**版本**：v1.0 · 設計日期 2026-07-04
**適用**：Notion Database / Google Sheets / PostgreSQL
**啟用時機**：Pine 指標上線後開始記錄

---

## 設計原則

1. **一次收齊，避免補漏** — 每筆進場都完整記錄，即便暫時用不到
2. **自動優先，人工兜底** — Pine 能算的絕不手打
3. **失敗要分類** — 因為失敗是最珍貴的反指標數據
4. **可交叉分析** — 每個「條件」都是獨立欄位，未來能 A/B 測試

---

## A. Trade ID & Meta（5 欄位 · 全自動）

| 欄位 | 型別 | 說明 | 範例 | 來源 |
|---|---|---|---|---|
| trade_id | UUID | 唯一識別碼 | `t-2026-07-04-001` | Pine auto |
| datetime | Datetime | 進場時間戳 | `2026-07-04 15:23:45 (UTC+8)` | Pine auto |
| symbol | Text | 交易對 | `BTC-USDT-SWAP` | Pine auto |
| exchange | Select | 交易所 | `OKX` | Pine auto |
| dashboard_version | Text | smc-pro 版本 | `v0.26.0` | Pine auto |

---

## B. 市場環境（4 欄位）

| 欄位 | 型別 | 說明 | 範例 | 來源 |
|---|---|---|---|---|
| kz_session | Select | KZ 時段 | `London / NY / 亞盤 / KZ 外` | 🤖 auto by time |
| timeframe | Select | 週期 | `5M` (固定) | 🤖 auto |
| htf_bias | Select | 1H 偏向 | `多 / 空 / 盤整` | ✋ manual |
| volatility_regime | Select | 波動狀態 | `低 / 正常 / 高 / 極高` | 🤖 auto (ATR 分位) |

---

## C. 七項條件狀態（7 欄位 · 核心）

| 欄位 | 型別 | 說明 | 判定 | 來源 |
|---|---|---|---|---|
| c1_kz_pass | Boolean | ① 時段過關 | 是否在 London/NY KZ 內 | 🤖 auto |
| c2_bias_aligned | Boolean | ② HTF 偏向對齊 | 進場方向 == HTF 偏向 | ✋ manual |
| c3_zone_correct | Boolean | ③ 折扣/溢價位置 | Long 在 discount / Short 在 premium | ✋ manual |
| c4_score | Number | ④ 分數（0-100） | smc-pro 打分 | 🤖 auto from dashboard |
| c5_ob_confirmed | Boolean | ⑤ OB 確認 | 是否有明確 OB | ✋ manual |
| c6_sl_correct | Boolean | ⑥ SL 貼 OB | SL 是否在 OB 底部/頂部 | 🤖 auto (價位比對) |
| c7_rr_pass | Boolean | ⑦ R:R ≥ 2:1 | (TP1 - Entry) / (Entry - SL) ≥ 2 | 🤖 auto |
| conditions_pass_count | Number | 過關數（0-7） | 上述 7 項 sum | 🤖 auto |
| all_conditions_pass | Boolean | 全過關 | conditions_pass_count == 7 | 🤖 auto |

---

## D. 訊號來源（2 欄位）

| 欄位 | 型別 | 說明 | 選項 |
|---|---|---|---|
| pine_triggered | Boolean | Pine 是否自動觸發 | Yes / No |
| manual_verified | Select | 事後手動驗證 | `七條件確實過 / 部分過 / 強行進 / 未驗證` |

---

## E. 進場數據（6 欄位 · 全自動）

| 欄位 | 型別 | 說明 | 範例 |
|---|---|---|---|
| direction | Select | 方向 | `Long / Short` |
| entry_price | Decimal | 進場價 | `60379.8` |
| sl_price | Decimal | 停損價 | `60339.6` |
| tp1_price | Decimal | 目標 1（2R） | `60460.2` |
| tp2_price | Decimal | 目標 2（3R+） | `60540.6` |
| position_size_usdt | Decimal | 倉位（USDT） | `500` |
| position_size_pct | Decimal | 帳戶百分比 | `2.5%` |
| planned_rr | Decimal | 計畫 R:R | `2.0` |

---

## F. SMC / ICT 結構（5 欄位 · 全自動）

| 欄位 | 型別 | 說明 | 選項 |
|---|---|---|---|
| ob_type | Select | 訂單塊 | `Bull OB / Bear OB / None` |
| fvg_type | Select | 公允價值缺口 | `Bull FVG / Bear FVG / None` |
| ob_fvg_confluence | Boolean | OB × FVG 疊合 | Yes / No |
| mss_completed | Boolean | MSS 已完成 | Yes / No |
| sweep_type | Select | 流動性獵取 | `SSL / BSL / None` |

---

## G. 諧波輔助（3 欄位 · v0.26 新增）

| 欄位 | 型別 | 說明 | 範例 |
|---|---|---|---|
| harmonic_pattern | Select | 諧波型態 | `Gartley / Bat / Butterfly / ... / None` |
| harmonic_match_pct | Number | 匹配度（0-100） | `85` |
| harmonic_bonus_used | Boolean | 是否用作共振加分 | Yes / No |

---

## H. 結果（5 欄位）

| 欄位 | 型別 | 說明 | 選項 |
|---|---|---|---|
| exit_method | Select | 出場方式 | `TP1 / TP2 / SL / 手動 / 未觸發` |
| exit_price | Decimal | 出場價 | 🤖 OKX API |
| pnl_usdt | Decimal | 損益（USDT） | 🤖 計算 |
| actual_rr | Decimal | 實際 R:R | 🤖 計算 |
| hold_time_mins | Number | 持倉分鐘 | 🤖 計算 |

---

## I. 失敗自動分類（1 欄位 · SL 觸發才填）

**優先序判定邏輯**：

```pseudocode
IF exit_method == "SL":
  IF c4_score < 65:
    failure_type = "假訊號"  # 進場當下就有問題
    
  ELIF 進場後任一根 5M K 收盤突破反向結構:
    failure_type = "結構失效"  # BOS 反向
    
  ELIF 任一根 5M K body 穿過 OB body > 0.5%:
    failure_type = "OB 破壞"  # 訂單塊失守
    
  ELIF 5M K body 完全填補進場時 FVG:
    failure_type = "FVG 反填"  # 缺口被吞
    
  ELIF KZ 結束時價格在進場價 ±0.3% 內:
    failure_type = "KZ 時效"  # 時段結束未達 TP1
    
  ELSE:
    failure_type = "其他 · 待人工復盤"
```

| 欄位 | 型別 | 選項 |
|---|---|---|
| failure_type | Select | `假訊號 / 結構失效 / OB 破壞 / FVG 反填 / KZ 時效 / 其他` |

**為什麼要優先序？** 一筆 SL 可能同時滿足多個條件，但**根因只有一個**。「假訊號」放最前面是因為它反過來質疑「這筆到底該不該進」；後面才是「進了但失守」的分類。

---

## J. 事後檢討（2 欄位 · 復盤時填）

| 欄位 | 型別 | 說明 | 範例 |
|---|---|---|---|
| tags | Multi-select | 標籤 | `教科書型 / 強行進 / 情緒進 / 共振爆炸 / 教訓案例` |
| review_notes | Long text | 復盤筆記 | 自由文字 |

---

## 完整 Pine `alert()` JSON 輸出格式

當 Pine 指標觸發時，webhook payload 建議格式：

```json
{
  "trade_id": "t-{{time}}-{{ticker}}-{{plot_0}}",
  "datetime": "{{timenow}}",
  "symbol": "{{ticker}}",
  "dashboard_version": "v0.26.0",
  "kz_session": "London",
  "volatility_regime": "normal",
  "c1_kz_pass": true,
  "c4_score": 78,
  "c6_sl_correct": true,
  "c7_rr_pass": true,
  "direction": "Long",
  "entry_price": {{close}},
  "sl_price": {{plot_0}},
  "tp1_price": {{plot_1}},
  "tp2_price": {{plot_2}},
  "planned_rr": 2.0,
  "ob_type": "Bull OB",
  "fvg_type": "Bull FVG",
  "ob_fvg_confluence": true,
  "mss_completed": true,
  "sweep_type": "SSL",
  "pine_triggered": true
}
```

其餘欄位（手動、結果、失敗分類）在後續補上。

---

## 分析範本：這個 schema 能回答的問題

| 問題 | 用哪些欄位交叉分析 |
|---|---|
| 七項條件哪一項最有效？ | C 條件 × H 勝率 |
| 分數 75+ 真的比 65+ 好嗎？ | c4_score 分段 × pnl_usdt |
| 哪個 KZ 最賺？ | kz_session × actual_rr |
| OB+FVG 疊合真的加分嗎？ | ob_fvg_confluence × 勝率 |
| 諧波共振加分項有沒有用？ | harmonic_bonus_used × 勝率 |
| 我最常在哪種失效上輸？ | failure_type 分布 |
| 「強行進」比正規進場差多少？ | tags 包含「強行進」× pnl |
| 高波動時勝率有變化嗎？ | volatility_regime × 勝率 |
| 諧波配 OB 疊合的組合最強？ | (harmonic × ob_fvg_confluence) × 勝率 |

---

## 下一步

1. **v0.26.0 上線後 1 週**：熟悉節奏，累積手感
2. **開始寫 Pine 指標**：把上表 🤖 標記的欄位當 `alert()` 輸出規格
3. **Pine 上線那天**：把這份 schema 開 Notion Database 或 Google Sheet
4. **累積 30 筆**：第一次坐下來看數字聊調整
5. **累積 100 筆**：Phase D 系統化開發正式啟動

---

*版本歷史*
- v1.0 · 2026-07-04 · 初版 · 38 欄位 · 10 分類
