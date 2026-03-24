# Project_Structure.md

## 專案目錄結構

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
│     ├─ CORE_LOOP.ERB
│     ├─ CHARA_SYSTEM.ERB
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
> 系統：System Layer
> 設計原則：啟動入口、流程控制、角色系統、除錯工具分離

| 模組檔案 | 核心職責 | 主要函式 / 內容 |
|---|---|---|
| `SYSTEM.ERB` | 遊戲啟動與全域初始化路由 | `@EVENTFIRST`, `@INIT_BASE_STATE`, `@TITLE_SCREEN` |
| `CORE_LOOP.ERB` | 每日流程狀態機與階段推進 | `@GAME_LOOP`, `@PHASE_MORNING`, `@PHASE_DAY`, `@PHASE_NIGHT`, `@SYS_RECOVER`, `@TRIGGER_NIGHT_EVENT` |
| `CHARA_SYSTEM.ERB` | 角色基礎數值與顯示字串 | `@INIT_CHARA_STATS`, `@GET_RACE_STR`, `@GET_JOB_STR` |
| `DEBUG_MENU.ERB` | 開發者測試與除錯操作 | `@DEBUG_MAIN_MENU`, `@TEST_ADD_RANDOM_NPC`, NPC/資源編輯流程 |
| `ERH.ERH` | 全域常數、索引、陣列定義 | `#DEFINE`, `#DIM` |

### `ERB/Battle/`
**職責：戰鬥域完整封裝。**

- `BATTLE_MAIN.ERB`（核心）
  - 戰鬥主循環與入口：`@BATTLE_ENTER`, `@BATTLE_UI_MAIN`
  - UI 勾選邏輯：`TFLAG:UI_HAND0_SEL~UI_TEMP_SEL`
  - Temp 槽操作：每回合一次存/取與交換、精確單選防呆
  - 同色堆疊計算：`@BATTLE_CALC_BOOST`（寫入 `TFLAG:TEMP_BUFF_*`）
  - 技能配方映射：`@BATTLE_MATCH_SKILL`（對 `SKILL_REQ_*` 做 `>=` 比對）
  - 出招結算與清理：`@BATTLE_PROCESS_ACTION`

> 設計規格：所有 `@BATTLE_` 函式集中於 `ERB/Battle/`，維持流程層與戰鬥層邊界清晰。

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
   - `SYSTEM.ERB` 負責啟動、初始化與標題流程路由。
   - `CORE_LOOP.ERB` 負責 `DAY_PHASE` 的主循環分派。
   - `BATTLE_MAIN.ERB` 實作戰鬥細節。
3. **文件與程式碼同步更新**
   - 戰鬥規則改動時，同步更新 `Docs/Battle_System.md` 與 `Docs/Data_Structure.md`。

---

## 長期維護架構規範（Evergreen Architecture Rules）

### 1) 模組邊界與單一職責 (Module Boundaries & SRP)
- `System`：僅負責啟動、總初始化流程路由與標題流程，不承載基地/戰鬥/角色細節實作。
- `Core Loop`：僅負責日循環狀態機與時機廣播，不直接處理任何單一領域的底層資料細節。
- `Base`：負責設施、房間、工坊佇列與基地資源規則。
- `Battle`：負責戰鬥資料、戰鬥流程、戰鬥結算與技能配方資料。
- `Chara`：負責角色屬性、狀態與角色日常結算。
- 依賴方向必須維持為：`System/Core Loop -> Domain Modules(Base/Battle/Chara)`，禁止反向依賴與跨域耦合。
- **嚴禁跨領域直接操作對方底層陣列**（例如 Core Loop 直接改動 `WORK_Q_REMAIN`）。

### 2) 生命週期委派設計 (Lifecycle Delegation Pattern)
- 任何全域生命週期節點（如：初始化、晨間/晚間、換日）由核心層負責「呼叫 Hook」。
- 實際狀態變更由對應子系統負責（例如基地晨間事件在 Base 模組實作，角色晚間結算在 Chara 模組實作）。
- 新增生命週期行為時，優先擴充既有 Hook，不在 Core Loop 內堆疊領域細節流程。

### 3) 介面命名公約 (API Naming Conventions)
- 跨模組初始化入口：`@<MODULE>_INIT`（例如 `@BASE_INIT`, `@BATTLE_INIT`）。
- 跨模組生命週期事件：`@<MODULE>_EVENT_<TIMING>`（例如 `@BASE_EVENT_MORNING`, `@CHARA_EVENT_NIGHT`）。
- 非跨模組內部輔助函式應保持在模組內命名域，避免對外暴露不必要 API。

---

## CSV 路徑提醒（Emuera 特例）

以下核心表 **必須** 放在 `CSV/` 根目錄：
- `GameBase.csv`
- `VariableSize.csv`
- `PALAM.csv`
- `Item.csv`

若誤放進子資料夾，可能造成啟動期解析錯誤（如核心變數未定義、陣列越界）。
