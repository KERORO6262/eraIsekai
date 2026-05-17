# Data_Structure.md

> 本文件定義 ERA ISEKAI 當前資料層結構，重點針對「雙軸卡牌戰鬥系統（Attribute + Shape）」的常數、暫存旗標與配方陣列。
> **注意**：舊版卡牌資料結構（`CARD_ENERGY_COST` / `CARD_BASE_POWER` / `CARD_BASE_ACC`）已退役，不再作為戰鬥資料來源。

---

## 1. 戰鬥核心常數（ERH.ERH）

### 1.1 雙軸常數表（Shape + Attribute）

| 常數 | 值 | 類型 | 用途 |
|---|---:|---|---|
| `#DEFINE CARD_PACK_BASE` | `10` | 打包基底 | 卡牌編碼公式 `CARD = ATTRIBUTE*10 + SHAPE` |
| `#DEFINE SHAPE_PHYS` | `1` | Shape ID | 物理（技能配方軸） |
| `#DEFINE SHAPE_MAG` | `2` | Shape ID | 魔法（技能配方軸） |
| `#DEFINE SHAPE_DEF` | `3` | Shape ID | 防禦（技能配方軸） |
| `#DEFINE SHAPE_SPEC` | `4` | Shape ID | 特殊（技能配方軸） |
| `#DEFINE ATTR_FIRE` | `1` | Attribute ID | 火（Buff 軸） |
| `#DEFINE ATTR_WATER` | `2` | Attribute ID | 水（Buff 軸） |
| `#DEFINE ATTR_WIND` | `3` | Attribute ID | 風（Buff 軸） |
| `#DEFINE ATTR_EARTH` | `4` | Attribute ID | 土（Buff 軸） |

### 1.2 卡牌壓縮格式（TFLAG 雙軸解析）

- 儲存欄位：`TFLAG:HAND0~HAND3`、`TFLAG:TEMP_CARD`。
- 空槽：`0`。
- 非空卡牌：`11~44`，由 Attribute（十位）與 Shape（個位）構成。

| 卡牌顯示 | 編碼 | 解碼 Attribute | 解碼 Shape |
|---|---:|---:|---:|
| `[火·物理]` | `11` | `11 / 10 = 1` | `11 - 1*10 = 1` |
| `[水·防禦]` | `23` | `23 / 10 = 2` | `23 - 2*10 = 3` |
| `[土·特殊]` | `44` | `44 / 10 = 4` | `44 - 4*10 = 4` |

> 採十位/個位商數餘數法，優點是 Emuera 執行成本低（整數運算）且除錯時可直接肉眼判讀。

### 1.3 戰鬥槽位與流程索引（TFLAG）

| 索引 | 常數 | 用途 | 備註 |
|---:|---|---|---|
| 10 | `BATTLE_TURN` | 戰鬥回合數 | `[430]` 成功出招後 +1 |
| 11 | `BATTLE_STRATEGY` | 策略保留欄位 | 後續 AI 策略用 |
| 12~15 | `HAND0`~`HAND3` | 四格手牌槽卡牌編碼 | `0` 空槽；非 0 為 `ATTR*10+SHAPE` |
| 16 | `TEMP_CARD` | Temp 槽卡牌編碼 | `0` 空槽；非 0 為 `ATTR*10+SHAPE` |
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
| 30 | `TEMP_BUFF_PATK` | 百分比 | 火屬性卡牌數量 × 10 | 臨時物攻加成 |
| 31 | `TEMP_BUFF_MATK` | 百分比 | 水屬性卡牌數量 × 10 | 臨時魔攻加成 |
| 32 | `TEMP_BUFF_PDEF` | 百分比 | 風屬性卡牌數量 × 10 | 臨時物防加成 |
| 33 | `TEMP_BUFF_MDEF` | 百分比 | 土屬性卡牌數量 × 10 | 臨時魔防加成 |

> 這些欄位由 `@BATTLE_CALC_BOOST` 寫入，並在新回合抽牌 (`@BATTLE_DRAW_CARDS`) 時清空。

---

## 4. 技能配方陣列（全域 #DIM）

戰鬥配方為「Shape 需求表」，以技能 ID 作為索引。

```erb
#DIM SKILL_REQ_SHAPE_PHYS, 30000
#DIM SKILL_REQ_SHAPE_MAG, 30000
#DIM SKILL_REQ_SHAPE_DEF, 30000
#DIM SKILL_REQ_SHAPE_SPEC, 30000
```

### 4.1 陣列語意

| 陣列 | 說明 | 範例 |
|---|---|---|
| `SKILL_REQ_SHAPE_PHYS:ID` | 技能 `ID` 所需物理 Shape 數 | `SKILL_REQ_SHAPE_PHYS:20001 = 2` |
| `SKILL_REQ_SHAPE_MAG:ID` | 技能 `ID` 所需魔法 Shape 數 | `SKILL_REQ_SHAPE_MAG:20001 = 1` |
| `SKILL_REQ_SHAPE_DEF:ID` | 技能 `ID` 所需防禦 Shape 數 | `SKILL_REQ_SHAPE_DEF:20002 = 3` |
| `SKILL_REQ_SHAPE_SPEC:ID` | 技能 `ID` 所需特殊 Shape 數 | `SKILL_REQ_SHAPE_SPEC:20005 = 1`（示意） |

### 4.2 讀取與比對規範
- 比對範圍：`20000~20010`（目前實作掃描區間）。
- 有效技能條件：四陣列需求總和 `> 0`。
- 命中條件：玩家本回合選取卡牌的 **Shape 計數** 滿足四維 `>=`。
- 無命中回傳 `0`，走普攻保底。

---

## 5. 初始化資料（SYSTEM.ERB）

全域資料與技能配方初始化由 `ERB/System/SYSTEM.ERB` 統一控管（包含 `@INIT_SKILL_DATA`）。

`@INIT_SKILL_DATA` 會先清空 `20000~20010` 的 `SKILL_REQ_SHAPE_*`，再註冊測試技能：

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
1. 卡牌槽位（`HAND0~HAND3`, `TEMP_CARD`，值為 `ATTR*10+SHAPE`）
2. UI 勾選旗標（`UI_HAND*_SEL`, `UI_TEMP_SEL`）
3. 配方陣列（`SKILL_REQ_SHAPE_*`）
4. 臨時 Buff 旗標（`TEMP_BUFF_*`）
