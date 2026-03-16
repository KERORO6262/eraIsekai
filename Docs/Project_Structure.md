# Project_Structure.md

## 專案目錄結構（模組化重構版）

```text
/
├─ CSV/
│  ├─ GameBase.csv
│  ├─ VariableSize.csv
│  ├─ PALAM.csv
│  ├─ Item.csv
│  └─ Chara/
│     ├─ Chara00.csv
│     ├─ Chara01.csv
│     └─ Chara02.csv
├─ ERB/
│  ├─ Base/
│  │  └─ BASE_MANAGEMENT.ERB
│  ├─ Battle/
│  │  └─ BATTLE_MAIN.ERB
│  └─ System/
│     ├─ ERH.ERH
│     ├─ SYSTEM.ERB
│     └─ DEBUG_MENU.ERB
├─ Docs/
│  ├─ Battle_System.md
│  ├─ Base_Management.md
│  ├─ Data_Structure.md
│  ├─ System_Config.md
│  ├─ Core Loop.md
│  ├─ Interaction_System.md
│  ├─ Progression_Balance.md
│  ├─ UI_Layout.md
│  └─ ...（授權與引擎說明）
└─ 其他執行與設定檔
```

---

## 目錄責任切分（Single Responsibility）

### `ERB/System/`
**職責：系統層主循環與初始化。**

- `SYSTEM.ERB`
  - 進入點 `@EVENTFIRST`
  - 日/夜階段推進、基地流程跳轉
  - `@INIT_BASE_STATE` / `@INIT_SKILL_DATA` 等初始化
- `ERH.ERH`
  - 全域常數與索引註冊（`#DEFINE`）
  - 全域資料陣列配置（`#DIM`）

### `ERB/Battle/`
**職責：戰鬥域完整封裝。**

- `BATTLE_MAIN.ERB`（核心）
  - 戰鬥主循環與入口：`@BATTLE_ENTER`, `@BATTLE_UI_MAIN`
  - UI 勾選邏輯：`TFLAG:UI_HAND0_SEL~UI_TEMP_SEL`
  - Temp 槽操作：每回合一次存/取與交換、精確單選防呆
  - 同色堆疊計算：`@BATTLE_CALC_BOOST`（寫入 `TFLAG:TEMP_BUFF_*`）
  - 技能配方映射：`@BATTLE_MATCH_SKILL`（對 `SKILL_REQ_*` 做 `>=` 比對）
  - 出招結算與清理：`@BATTLE_PROCESS_ACTION`

> 模組化目標：所有 `@BATTLE_` 函式集中在 `ERB/Battle/`，避免 `SYSTEM.ERB` 膨脹，並降低跨功能改動的耦合風險。

### `ERB/Base/`
**職責：基地管理域。**

- `BASE_MANAGEMENT.ERB`
  - 訓練室 / 工坊 / 休息室 / 房間槽位管理
  - 與戰鬥透過流程呼叫銜接，不直接承擔戰鬥結算細節

---

## 載入與依賴原則

1. **資料定義先於功能使用**
   - `ERH.ERH` 的 `#DEFINE` / `#DIM` 必須覆蓋戰鬥與基地所需索引。
2. **流程層呼叫、模組層實作**
   - `SYSTEM.ERB` 只做 `CALL BATTLE_ENTER` 等流程路由。
   - `BATTLE_MAIN.ERB` 才實作戰鬥細節。
3. **文件與程式碼同步更新**
   - 戰鬥規則改動時，同步更新 `Docs/Battle_System.md` 與 `Docs/Data_Structure.md`。

---

## CSV 路徑提醒（Emuera 特例）

以下核心表 **必須** 放在 `CSV/` 根目錄：
- `GameBase.csv`
- `VariableSize.csv`
- `PALAM.csv`
- `Item.csv`

若誤放進子資料夾，可能造成啟動期解析錯誤（如核心變數未定義、陣列越界）。

