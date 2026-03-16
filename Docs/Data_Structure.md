# Data_Structure.md

> 本文件定義 ERA ISEKAI 當前資料層結構，重點針對「四象元素編織戰鬥系統」的常數、暫存旗標與配方陣列。  
> **注意**：舊版卡牌資料結構（`CARD_ENERGY_COST` / `CARD_BASE_POWER` / `CARD_BASE_ACC`）已退役，不再作為戰鬥資料來源。

---

## 1. 戰鬥核心常數（ERH.ERH）

### 1.1 四象元素常數表

| 常數 | 值 | 類型 | 用途 |
|---|---:|---|---|
| `#DEFINE ELEM_PHYS` | `1` | Element ID | 物理元素 |
| `#DEFINE ELEM_MAG` | `2` | Element ID | 魔法元素 |
| `#DEFINE ELEM_DEF` | `3` | Element ID | 防禦元素 |
| `#DEFINE ELEM_SPEC` | `4` | Element ID | 特殊元素 |

### 1.2 戰鬥槽位與流程索引（TFLAG）

| 索引 | 常數 | 用途 | 備註 |
|---:|---|---|---|
| 10 | `BATTLE_TURN` | 戰鬥回合數 | `[430]` 成功出招後 +1 |
| 11 | `BATTLE_STRATEGY` | 策略保留欄位 | 後續 AI 策略用 |
| 12~15 | `HAND0`~`HAND3` | 四格手牌槽元素 ID | 0 代表空槽 |
| 16 | `TEMP_CARD` | Temp 槽元素 ID | 0 代表空槽 |
| 17 | `TEMP_USED_THIS_TURN` | Temp 本回合使用鎖 | 0 可用 / 1 已用 |
| 18 | `DECK_SIZE` | 相容保留欄位 | 元素系統下可作統計值 |
| 19 | `DISCARD_SIZE` | 相容保留欄位 | 元素系統下可作統計值 |

---

## 2. 戰鬥 UI 狀態變數（TFLAG 20~24）

| 索引 | 常數 | 值域 | 意義 |
|---:|---|---|---|
| 20 | `UI_HAND0_SEL` | 0/1 | H0 勾選狀態 |
| 21 | `UI_HAND1_SEL` | 0/1 | H1 勾選狀態 |
| 22 | `UI_HAND2_SEL` | 0/1 | H2 勾選狀態 |
| 23 | `UI_HAND3_SEL` | 0/1 | H3 勾選狀態 |
| 24 | `UI_TEMP_SEL` | 0/1 | Temp 勾選狀態 |

### 2.1 使用規則
- `0`：未選取（UI 顯示 `[ ]`）
- `1`：已選取（UI 顯示 `[X]`）
- 在 `@BATTLE_DRAW_CARDS` 開始新回合時，以上旗標全部重置為 `0`。

---

## 3. 臨時 Buff 變數（TFLAG 30~33）

| 索引 | 常數 | 單位 | 計算來源 | 當前用途 |
|---:|---|---|---|---|
| 30 | `TEMP_BUFF_PATK` | 百分比 | 物理元素數量 × 10 | 臨時物攻加成 |
| 31 | `TEMP_BUFF_MATK` | 百分比 | 魔法元素數量 × 10 | 臨時魔攻加成 |
| 32 | `TEMP_BUFF_PDEF` | 百分比 | 防禦元素數量 × 10 | 臨時物防加成 |
| 33 | `TEMP_BUFF_MDEF` | 百分比 | 特殊元素數量 × 10 | 臨時魔防加成 |

> 這些欄位由 `@BATTLE_CALC_BOOST` 寫入，並在新回合抽牌 (`@BATTLE_DRAW_CARDS`) 時清空。

---

## 4. 技能配方陣列（全域 #DIM）

戰鬥配方改為「元素需求表」，以技能 ID 作為索引。

```erb
#DIM SKILL_REQ_PHYS, 30000
#DIM SKILL_REQ_MAG, 30000
#DIM SKILL_REQ_DEF, 30000
#DIM SKILL_REQ_SPEC, 30000
```

### 4.1 陣列語意

| 陣列 | 說明 | 範例 |
|---|---|---|
| `SKILL_REQ_PHYS:ID` | 技能 `ID` 所需物理元素數 | `SKILL_REQ_PHYS:20001 = 2` |
| `SKILL_REQ_MAG:ID` | 技能 `ID` 所需魔法元素數 | `SKILL_REQ_MAG:20001 = 1` |
| `SKILL_REQ_DEF:ID` | 技能 `ID` 所需防禦元素數 | `SKILL_REQ_DEF:20002 = 3` |
| `SKILL_REQ_SPEC:ID` | 技能 `ID` 所需特殊元素數 | `SKILL_REQ_SPEC:20005 = 1`（示意） |

### 4.2 讀取與比對規範
- 比對範圍：`20000~20010`（目前實作掃描區間）。
- 有效技能條件：四陣列需求總和 `> 0`。
- 命中條件：玩家本回合選取元素滿足四維 `>=`。
- 無命中回傳 `0`，走普攻保底。

---

## 5. 初始化資料（SYSTEM.ERB）

全域資料與技能配方初始化由 `ERB/System/SYSTEM.ERB` 統一控管（包含 `@INIT_SKILL_DATA`）。

`@INIT_SKILL_DATA` 會先清空 `20000~20010` 的 `SKILL_REQ_*`，再註冊測試技能：

| 技能 ID | 名稱 | `PHYS` | `MAG` | `DEF` | `SPEC` |
|---:|---|---:|---:|---:|---:|
| 20001 | 重力斬擊 | 2 | 1 | 0 | 0 |
| 20002 | 絕對防禦 | 0 | 0 | 3 | 0 |

---

## 6. 舊結構退役聲明

下列舊結構已不在戰鬥流程中使用：
- 卡牌 ID 專用能耗 / 威力 / 命中陣列
- 依 `ITEM.csv` 卡牌資料作為核心戰鬥判定來源

戰鬥結算已完全遷移到：
1. 元素槽位（`HAND0~HAND3`, `TEMP_CARD`）
2. UI 勾選旗標（`UI_HAND*_SEL`, `UI_TEMP_SEL`）
3. 配方陣列（`SKILL_REQ_*`）
4. 臨時 Buff 旗標（`TEMP_BUFF_*`）

