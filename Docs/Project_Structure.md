# Project_Structure.md

## 專案目錄結構（重構後）

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
│  └─ System/
│     ├─ ERH.ERH
│     └─ SYSTEM.ERB
├─ Docs/
│  ├─ System_Config.md
│  ├─ Base_Management.md
│  ├─ Battle_System.md
│  ├─ Core Loop.md
│  ├─ Data_Structure.md
│  ├─ Interaction_System.md
│  ├─ Kojo_Standard.md
│  ├─ Progression_Balance.md
│  ├─ UI_Layout.md
│  └─ ...（其餘授權與引擎說明文件）
└─ 其他執行與設定檔
```

## 分類規範

### CSV/
- `CSV/Chara/`：所有角色主資料（`CharaXX.csv`）集中管理。
- 道具核心表 `Item.csv` 放置於 `CSV/` 根目錄。
- 系統核心設定檔統一放置於 `CSV/` 根目錄：`GameBase.csv`、`VariableSize.csv`、`PALAM.csv`、`Item.csv`。

> ⚠️ **Warning**：受限於 Emuera 引擎底層寫死的讀取邏輯，系統核心設定檔（GameBase.csv, VariableSize.csv, PALAM.csv, Item.csv）絕對不可放入子目錄，必須直接放置於 CSV/ 根目錄下，否則引擎將無法讀取並導致陣列越界等崩潰。

### ERB/
- `ERB/System/`：主系統入口、共用常數、全域流程控制（如 `SYSTEM.ERB`, `ERH.ERH`）。
- `ERB/Base/`：基地、房間管理、設施互動等基地域邏輯。

### Docs/
- 原 `read me/` 統一改為 `Docs/`。
- 所有設計規格、平衡筆記、引擎與授權文件統一置於 `Docs/`，避免根目錄分散。

## Emuera 載入規則提醒
- Emuera 對一般資料檔（例如角色、道具）可讀取 `CSV/` 下子目錄。
- 但 `GameBase.csv`、`VariableSize.csv`、`PALAM.csv`、`Item.csv` 屬於歷史遺留的系統核心表，路徑解析為特例，需固定在 `CSV/` 根目錄。
- 若誤放至 `CSV/System/` 等子資料夾，會出現 `GameBase 未定義`、`BASE 陣列越界` 等啟動期錯誤。

## 本次重構搬移對應（舊路徑 -> 新路徑）
- `CSV/System/GameBase.csv` -> `CSV/GameBase.csv`
- `CSV/System/VariableSize.csv` -> `CSV/VariableSize.csv`
- `CSV/System/PALAM.Csv` -> `CSV/PALAM.csv`
- `CSV/Chara00.csv` -> `CSV/Chara/Chara00.csv`
- `CSV/Chara01.csv` -> `CSV/Chara/Chara01.csv`
- `CSV/Chara02.csv` -> `CSV/Chara/Chara02.csv`
- `CSV/Item/Item.csv` -> `CSV/Item.csv`
- `ERB/SYSTEM.ERB` -> `ERB/System/SYSTEM.ERB`
- `ERB/ERH.ERH` -> `ERB/System/ERH.ERH`
- `ERB/BASE_MANAGEMENT.ERB` -> `ERB/Base/BASE_MANAGEMENT.ERB`
- `read me/` -> `Docs/`
