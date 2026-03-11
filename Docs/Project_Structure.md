# Project_Structure.md

## 專案目錄結構（重構後）

```text
/
├─ CSV/
│  ├─ Chara/
│  │  ├─ Chara00.csv
│  │  ├─ Chara01.csv
│  │  └─ Chara02.csv
│  ├─ Item/
│  │  └─ Item.csv
│  └─ System/
│     ├─ GameBase.csv
│     ├─ PALAM.Csv
│     └─ VariableSize.csv
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
- `CSV/System/`：核心系統定義與全域表（如 `GameBase`, `VariableSize`, `PALAM`）。
- `CSV/Item/`：道具相關資料。

### ERB/
- `ERB/System/`：主系統入口、共用常數、全域流程控制（如 `SYSTEM.ERB`, `ERH.ERH`）。
- `ERB/Base/`：基地、房間管理、設施互動等基地域邏輯。

### Docs/
- 原 `read me/` 統一改為 `Docs/`。
- 所有設計規格、平衡筆記、引擎與授權文件統一置於 `Docs/`，避免根目錄分散。

## Emuera 載入規則提醒
- Emuera 會遞迴讀取 `ERB/` 與 `CSV/` 之下檔案；只要檔案仍位於這兩個根目錄中，移動到子資料夾不會影響載入。
- 目錄分類僅影響維護性與協作可讀性，不改變腳本執行語意。


## 本次重構搬移對應（舊路徑 -> 新路徑）
- `CSV/Chara00.csv` -> `CSV/Chara/Chara00.csv`
- `CSV/Chara01.csv` -> `CSV/Chara/Chara01.csv`
- `CSV/Chara02.csv` -> `CSV/Chara/Chara02.csv`
- `CSV/GameBase.csv` -> `CSV/System/GameBase.csv`
- `CSV/VariableSize.csv` -> `CSV/System/VariableSize.csv`
- `CSV/PALAM.Csv` -> `CSV/System/PALAM.Csv`
- `CSV/Item.csv` -> `CSV/Item/Item.csv`
- `ERB/SYSTEM.ERB` -> `ERB/System/SYSTEM.ERB`
- `ERB/ERH.ERH` -> `ERB/System/ERH.ERH`
- `ERB/BASE_MANAGEMENT.ERB` -> `ERB/Base/BASE_MANAGEMENT.ERB`
- `read me/` -> `Docs/`
