# Development_Roadmap.md

> 專案：ERA ISEKAI（Emuera 相容）養成 + 策略經營 + 雙軸卡牌戰鬥  
> 文件版本：2026-05-17  
> 說明：本路線圖依「底層地基 → 可玩 MVP」邏輯排序，每階段標註完成狀態、可執行任務清單與長期架構規範提醒。  
> 狀態圖例：✅ 已完成　🚧 進行中　📋 待實作　🎨 設計完整，待編碼

---

## 整體開發階段總覽

| 階段 | 名稱 | 主要檔案 | 狀態 |
|:---:|---|---|:---:|
| Phase 0 | CSV 資料地基與全域常數 | `CSV/*.csv` + `ERH.ERH` | ✅ |
| Phase 1 | 系統啟動與基礎路由 | `SYSTEM.ERB` | ✅ |
| Phase 2 | 每日核心循環狀態機 | `CORE_LOOP.ERB` | ✅ |
| Phase 3 | 戰鬥框架 MVP | `BATTLE_MAIN.ERB` | ✅ |
| Phase 4 | 基地經營框架與 UI | `BASE_MANAGEMENT.ERB` | ✅ |
| Phase 5 | 互動系統核心與數值計算 | `BASE_MANAGEMENT.ERB` 互動段落 | ✅ |
| Phase 6 | 口上系統（KOJO） | 各 `KOJO_XXXX.ERB` | 📋 |
| Phase 7 | 進階戰鬥機制擴充 | `BATTLE_MAIN.ERB` | 🎨 |
| Phase 8 | 系統配置、存檔與畫廊 | `SYSTEM.ERB` + `SYSTEM_CONFIG.ERB` | 📋 |
| Phase 9 | 數值平衡校準與 Debug 精煉 | `DEBUG_MENU.ERB` + 各模組 | 📋 |

---

## Phase 0：CSV 資料地基與全域常數 ✅

### 完成狀態
已完成。ERH.ERH 完整定義雙軸常數、TFLAG 索引、配方陣列；四張核心 CSV 放置在正確目錄。

### 實現方式
→ 詳見 [Data_Structure.md](Data_Structure.md)（第 1–4 節）、[Project_Structure.md](Project_Structure.md)（CSV 路徑提醒）

### 已實作清單
- `CSV/GameBase.csv`、`VariableSize.csv`、`PALAM.csv`、`Item.csv`（根目錄就位）
- `CSV/Chara/Chara00.csv`、`Chara01.csv`、`Chara02.csv`
- `ERH.ERH`：`SHAPE_PHYS/MAG/DEF/SPEC`、`ATTR_FIRE/WATER/WIND/EARTH`、`CARD_PACK_BASE=10`
- `ERH.ERH`：`TFLAG` 索引 10–33（`BATTLE_TURN`、`HAND0~3`、`TEMP_CARD`、`UI_HAND*_SEL`、`TEMP_BUFF_*`）
- `ERH.ERH`：`#DIM SKILL_REQ_SHAPE_PHYS/MAG/DEF/SPEC, 30000`

### MVP 檢核點
> 引擎啟動不報「未定義變數」、「陣列越界」錯誤；所有 TFLAG 索引在 Emuera 除錯模式下可正常讀寫。

### 架構規範提醒
- **資料定義先於功能使用**：任何新增的索引或常數必須先寫入 ERH.ERH，再在 ERB 中引用。
- CSV 核心表**嚴禁**移至子資料夾，否則引擎解析期報錯（見 Project_Structure.md）。

---

## Phase 1：系統啟動與基礎路由 ✅

### 完成狀態
已完成。啟動流程、初始化入口、技能資料初始化、角色系統基礎均已落地。

### 實現方式
→ 詳見 [Project_Structure.md](Project_Structure.md)（System 層職責表）、[Data_Structure.md](Data_Structure.md)（第 5 節初始化資料）

### 已實作函式
| 函式 | 檔案 | 職責 |
|---|---|---|
| `@EVENTFIRST` | `SYSTEM.ERB` | 遊戲首次啟動入口，路由至標題畫面 |
| `@TITLE_SCREEN` | `SYSTEM.ERB` | 標題選單（新遊戲/讀檔/設定） |
| `@INIT_BASE_STATE` | `SYSTEM.ERB` | 全域基地狀態初始化 |
| `@INIT_SKILL_DATA` | `SYSTEM.ERB` | 技能配方陣列清空並登錄（20001 重力斬擊、20002 絕對防禦） |
| `@INIT_CHARA_STATS` | `CHARA_SYSTEM.ERB` | 角色數值基礎初始化 |
| `@GET_RACE_STR` | `CHARA_SYSTEM.ERB` | 種族字串輸出 |
| `@GET_JOB_STR` | `CHARA_SYSTEM.ERB` | 職業字串輸出 |
| `@DEBUG_MAIN_MENU` | `DEBUG_MENU.ERB` | Debug 總選單入口（按鈕 ID 999） |
| `@TEST_ADD_RANDOM_NPC` | `DEBUG_MENU.ERB` | 測試用 NPC 生成 |
| `@DEBUG_RESOURCE_MENU` | `DEBUG_MENU.ERB` | 資源數值編輯器 |
| `@DEBUG_NPC_EDITOR` | `DEBUG_MENU.ERB` | NPC 屬性編輯器 |

### MVP 檢核點
> 能正常顯示標題畫面；選「新遊戲」後角色資料初始化不報錯；`@INIT_SKILL_DATA` 執行後 `SKILL_REQ_SHAPE_PHYS:20001 == 2` 可驗證。

### 架構規範提醒
- `SYSTEM.ERB` 只負責**啟動路由與全域初始化**，禁止在此堆疊基地/戰鬥/角色的領域細節。
- 所有 debug 功能以 `@DEBUG_` 前綴收斂在 `DEBUG_MENU.ERB`，不得在標題或基地主選單散落測試按鈕。
- 新增初始化入口時，優先擴充 `@SYS_ON_NEWGAME` / `@SYS_ON_LOAD` 等生命週期 Hook（見 System_Config.md 第 1 節）。

---

## Phase 2：每日核心循環狀態機 ✅

### 完成狀態
已完成。`CORE_LOOP.ERB` 實作完整的 Day Cycle 狀態機，三大階段均有入口函式，且戰鬥以 CALL/RETURN 可插拔整合。

### 實現方式
→ 詳見 [Core Loop.md](Core%20Loop.md)（全文）

### 已實作函式
| 函式 | 檔案 | 職責 |
|---|---|---|
| `@GAME_LOOP` | `CORE_LOOP.ERB` | 每日主循環，以 `FLAG:DAY_PHASE` SWITCH 分派三階段 |
| `@PHASE_MORNING` | `CORE_LOOP.ERB` | 晨間：呼叫基地 Tick、顯示晨報、推進到日間 |
| `@PHASE_DAY` | `CORE_LOOP.ERB` | 日間：基地主選單入口、戰鬥入口（CALL @BATTLE_PREPARATION） |
| `@PHASE_NIGHT` | `CORE_LOOP.ERB` | 晚間：數值清算、隨機事件、就寢推進換日 |
| `@SYS_RECOVER` | `CORE_LOOP.ERB` | 非法狀態修正（資源負值、房間槽不足等） |
| `@TRIGGER_NIGHT_EVENT` | `CORE_LOOP.ERB` | 晚間隨機事件觸發路由 |

### MVP 檢核點
> 能在晨間 → 日間 → 晚間順利切換，天數 `FLAG:DAY` 正確 +1；戰鬥以 `CALL @BATTLE_PREPARATION` 進入、`RETURN` 後確認回到呼叫階段（日間或晚間）；`TFLAG:MORNING_DONE` 防止重複晨間結算。

### 架構規範提醒
- **主循環唯一推進權**：只有 `@GAME_LOOP` 能改動 `FLAG:DAY_PHASE`，子模組（基地、戰鬥）不得自行推進日夜。
- **生命週期委派**：晨間/晚間 Tick 由 Core Loop 發出 CALL，具體計算在子系統實作（`@BASE_MORNING_TICK`、`@BASE_NIGHT_TICK`）。
- **跨域呼叫只用 CALL/RETURN**：進戰鬥一律 `CALL @BATTLE_PREPARATION`，禁止跨檔 GOTO。
- **首日保護機制**：`DAY == 1` 的晨間不執行設施產出與維護費結算（見 Core Loop.md 晨間段落）。

---

## Phase 3：戰鬥框架 MVP ✅

### 完成狀態
已完成。雙軸卡牌（Attribute + Shape）完整實作，含回合結算、敵方回合、勝敗結算與資源入帳。

### 實現方式
→ 詳見 [Battle_System.md](Battle_System.md)（全文）、[Data_Structure.md](Data_Structure.md)（第 1–4 節）

### 已實作函式
| 函式 | 檔案 | 職責 |
|---|---|---|
| `@BATTLE_PREPARATION` | `BATTLE_MAIN.ERB` | 戰前編組 UI，建立 `PARTY_MEMBERS:0~3` |
| `@BATTLE_ENTER` | `BATTLE_MAIN.ERB` | 戰鬥主入口，初始化敵方實體，回傳勝敗給呼叫方 |
| `@BATTLE_UI_MAIN` | `BATTLE_MAIN.ERB` | 回合主 UI 迴圈，處理 410~450 按鍵分派 |
| `@BATTLE_DRAW_CARDS` | `BATTLE_MAIN.ERB` | 每回合抽 4 張牌、重置 UI 旗標與 Buff |
| `@BATTLE_PROCESS_ACTION` | `BATTLE_MAIN.ERB` | 出招結算：統計 Shape/Attribute → Buff → 技能映射 → 清槽 |
| `@BATTLE_CALC_BOOST` | `BATTLE_MAIN.ERB` | 同屬性卡牌堆疊計算，寫入 `TFLAG:TEMP_BUFF_*` |
| `@BATTLE_MATCH_SKILL` | `BATTLE_MAIN.ERB` | Shape 配方 >= 比對（20000~20010），無命中回傳 0 普攻 |
| `@BATTLE_ENEMY_TURN` | `BATTLE_MAIN.ERB` | 敵方隨機攻擊邏輯（掃描存活 PARTY_MEMBERS） |
| `@BATTLE_SETTLEMENT_REPORT` | `BATTLE_MAIN.ERB` | 戰鬥結算報告：戰利品入帳、Clamp 防呆、清空 LOOT 旗標 |
| `@BATTLE_UI_PRINT_HAND_SLOT` | `BATTLE_MAIN.ERB` | 手牌槽 UI 顯示（勾選狀態） |
| `@BATTLE_UI_PRINT_TEMP_SLOT` | `BATTLE_MAIN.ERB` | Temp 槽 UI 顯示 |

### MVP 檢核點
> 能從日間進入木人樁戰鬥；選卡勾選後 [430] 出招正確計算 Buff 與技能映射；擊破後 `RESULT=1`，資源（金錢 100、材料 20）正確進帳；敗北後 `RESULT=0`，不發放掉落；回到呼叫階段（日間/晚間）不迷路。

### 架構規範提醒
- **戰鬥域完整封裝**：`@BATTLE_SETTLEMENT_REPORT` 負責資源入帳，`CORE_LOOP.ERB` 只做 CALL，不持有任何戰鬥資源計算邏輯。
- **雙軸擴充規則**：新增屬性只改 `@BATTLE_CALC_BOOST`；新增形狀必須同步擴充 `SKILL_REQ_SHAPE_*` 與 `@BATTLE_MATCH_SKILL`（見 Battle_System.md 第 9 節）。
- 戰鬥 TFLAG 全部為 SESSION 層，`@BATTLE_ENTER` 負責初始化，戰鬥結束後清理（見 System_Config.md 第 1 節）。

---

## Phase 4：基地經營框架與 UI ✅

### 當前狀態
已完成。Step 4-1（晨間/晚間 Tick）至 Step 4-5（UI 輸出函式）全數落地。工坊的**裝備製作與元素權重改造**功能明確標注為「開發中」，留待後續迭代。

### 實現方式（參考）
→ 詳見 [Base_Management.md](Base_Management.md)（全文）、[UI_Layout.md](UI_Layout.md)（第 2 節基地 UI）

### 待實作任務清單

#### Step 4-1：基地晨間/晚間 Tick（優先，與 Core Loop 接口）
| 任務 | 函式 / 標籤 | 說明 |
|---|---|---|
| 晨間產出與維護費結算 | `@BASE_MORNING_TICK` | 設施產出入帳、住宿維護費扣除、住宿加成刷新；`TFLAG:MORNING_DONE` 防重入 |
| 晚間疲勞與事件 Tick | `@BASE_NIGHT_TICK` | 疲勞/心情自然變動、設施使用次數清空（`TFLAG:BASE_USES_TRAIN`、`TFLAG:BASE_USES_LOUNGE`） |
| 事件池擲骰 | `@TRIGGER_NIGHT_EVENT`（擴充） | 基地事件、NPC 事件、資源事件，含「權重 + 冷卻」防重複 |

#### Step 4-2：基地主選單路由 ✅
| 任務 | 函式 / 標籤 | 說明 |
|---|---|---|
| 基地總選單入口 | `@BASE_MENU` | 路由訓練室/工坊/休息室/房間管理/出戰/離開（按鈕 100–140） |
| 訓練室主選單 | `@BASE_TRAIN_MENU` | 選強度（基礎/專項）→ 選 NPC → 選屬性 → 結算 BASE 成長 |
| 工坊主選單 | `@BASE_WORK_MENU` | 建設/升級/產線佇列/取消生產（全額退材料） |
| 休息室主選單 | `@BASE_LOUNGE_MENU` | 日常互動入口、狀態顯示（好感/心情/疲勞） |
| 房間管理選單 | `@BASE_ROOM_MENU` | 顯示槽位列表、入住/撤離操作 |

#### Step 4-3：訓練室結算 ✅
| 任務 | 函式 / 標籤 | 說明 |
|---|---|---|
| NPC 選擇器 | `@SELECT_RESIDENT_NPC` | 回傳合法 NPC 索引；**必須明確 RETURN RESULT**，禁止空 RETURN（見 Interaction_System.md 10.2） |
| 訓練套用 | `@SYS_TRAIN_APPLY` | 封裝成長公式：`基礎成長 + 設施LV加成 + 住宿加成 + 狀態修正`；同步 CF_FATIGUE、CF_TRAIN_MOD |

#### Step 4-4：房間槽管理（單一入口）✅
| 任務 | 函式 / 標籤 | 說明 |
|---|---|---|
| 入住 | `@SYS_ROOM_MOVE_IN` | 檢查空槽、條件（好感/資源）→ 寫 `ROOM_OWNER[slot]`、`ROOM_USED++`、`CFLAG:IS_RESIDENT=1` |
| 撤離 | `@SYS_ROOM_MOVE_OUT` | 釋放槽位、設定 `MOVEOUT_COOLDOWN`、降低撤離壓力 |

#### Step 4-5：基地 UI 輸出函式 ✅
| 任務 | 函式 / 標籤 | 說明 |
|---|---|---|
| 頂欄 HUD | `@BASE_PRINT_HUD` | Day/Phase、金錢/材料/補給、設施等級一行顯示 |
| NPC 住宿列表 | `@BASE_PRINT_NPC_LIST` | Slot 編號、NPC 名、好感段位、心情、疲勞（對齊 UI_Layout.md 2.3） |
| 晨間報告 | `@BASE_PRINT_MORNING_REPORT` | 昨日收支、今日新增資源、基地警示 |

### MVP 檢核點
> 能顯示基地主選單（訓練室/工坊/休息室/房間管理）；晨間結算能正確增減資源與疲勞度，且 `TFLAG:MORNING_DONE` 防止同日重複結算；入住/撤離只走 `@SYS_ROOM_MOVE_IN / @SYS_ROOM_MOVE_OUT`，`ROOM_USED` 數值保持同步；訓練室能完成「選強度 → 選 NPC → 選屬性 → BASE+1/+2」完整流程。

### 架構規範提醒
- **禁止跨域操作底層陣列**：Core Loop 不得直接讀寫 `WORK_Q_REMAIN`；入住/撤離操作不得繞過 `@SYS_ROOM_MOVE_IN/OUT`（見 Project_Structure.md 規範 1）。
- **每日防呆旗標**：所有設施使用次數用 `TFLAG:BASE_USES_*` 控管，晚間 Tick 清空；晨間 Tick 用 `TFLAG:MORNING_DONE` 防重入。
- **生命週期委派**：晨間/晚間鉤子名稱規範為 `@BASE_EVENT_MORNING` / `@BASE_EVENT_NIGHT`（API 命名公約，見 Project_Structure.md 規範 3）。
- UI 畫面更新後必須 GOTO 回選單頂端重繪，防止舊資料殘留（見 Developer_Guidelines.md 陷阱 1）。

---

## Phase 5：互動系統核心與數值計算 ✅

### 當前狀態
設計完整，ERB 已完成實作。`PALAM.csv` 欄位需求已定義，數值公式已在文件中確立。

### 實現方式（參考）
→ 詳見 [Interaction_System.md](Interaction_System.md)（全文）、[Base_Management.md](Base_Management.md)（休息室段落）

### 待實作任務清單

#### Step 5-1：互動入口整合（休息室掛鉤）
| 任務 | 函式 / 標籤 | 說明 |
|---|---|---|
| 互動選單路由 | `@BASE_LOUNGE_MENU`（擴充） | 日常互動 vs 成人向互動分頁，對應按鈕區間 300–399 |
| 互動指令分派 | `@INTERACT_DISPATCH` | 由 CMD_ID 路由到對應 handler |

#### Step 5-2：Gate 判斷（可執行性檢查）
| 任務 | 函式 / 標籤 | 說明 |
|---|---|---|
| Gate 判斷 | `@INTERACT_GATE_CHECK` | 檢查反感 >= MAX_DISGUST、疲勞 >= 90，輸出拒絕訊息；回傳 0（禁止）/ 1（允許） |

#### Step 5-3：核心數值計算
| 任務 | 函式 / 標籤 | 說明 |
|---|---|---|
| 成功率計算（日常） | `@CALC_INTERACT_SUCCESS_DAILY` | `AffinityScore = BOND_LV*8 + MOOD*0.4 - FATIGUE*0.3 - 反感*0.6`；`SuccessRate = CLAMP(10,95, 60+AffinityScore)` |
| 成功率計算（成人向） | `@CALC_INTERACT_SUCCESS_ADULT` | `ConsentScore = BOND_LV*10 + MOOD*0.2 + 興奮*0.4 - 反感*1.2 - FATIGUE*0.4`；`SuccessRate = CLAMP(5,90, 40+ConsentScore)` |
| 效果倍率計算 | `@CALC_INTERACT_EFFECT_SCALE` | `EffectScale = 1.0 + (SuccessRate-50)/100`；反感懲罰/獎勵套用 |
| PALAM 變動套用 | `@INTERACT_APPLY_PALAM` | 依 EffectScale 套用快樂/痛苦/反感/興奮增減 |

#### Step 5-4：經驗、刻印與事件
| 任務 | 函式 / 標籤 | 說明 |
|---|---|---|
| 刻印點數結算 | `@INTERACT_APPLY_MARK` | 苦痛刻印：`max(0, Δ痛苦 - Δ快樂) + (反感>=60 ? 2 : 0)`；快樂刻印：`max(0, Δ快樂 + Δ興奮 - Δ反感)` |
| 刻印升級檢查 | `@INTERACT_CHECK_MARK_LV` | 門檻 20/60/120，升級後更新 `CFLAG:CHARA:MARK_*_PT` |
| 絕頂事件觸發 | `@INTERACT_CHECK_CLIMAX` | 條件：興奮>=90、快樂>=70、反感<=40、EffectScale>=1.0 |
| 處女旗標處理 | `@INTERACT_VIRGIN_CHECK` | 首次帶 `VIRGIN_CHECK` 指令命中，設 `CFLAG:CHARA:VIRGIN=0`，寫入 log |

#### Step 5-5：回饋到基地
| 任務 | 函式 / 標籤 | 說明 |
|---|---|---|
| 訓練效率回饋 | `@INTERACT_FEEDBACK_BASE` | 依刻印/好感調整 `CFLAG:CHARA:TRAIN_MOD`；依疲勞調整休息室回復量 |
| 撤離壓力更新 | `@INTERACT_UPDATE_MOVEOUT` | 高反感/大失敗增加 `MOVEOUT_COOLDOWN`；善後類指令降低壓力 |

#### Step 5-6：失敗處理
| 任務 | 函式 / 標籤 | 說明 |
|---|---|---|
| 三段失敗分派 | `@INTERACT_APPLY_FAIL` | 小失敗（快樂-、反感+）→ 大失敗（觸發撤離壓力）→ 災難級（強制善後路徑） |

### MVP 檢核點
> 日常互動能正確計算 AffinityScore 並擲骰；成功時 PALAM 變動符合公式；失敗時有明確的三段回饋；互動結果能更新 `TRAIN_MOD`，次日訓練結算時效果可見；`@SELECT_RESIDENT_NPC` 回傳合法值後才扣設施次數（防 TARGET 錯綁 MASTER）。

### 架構規範提醒
- **角色選擇函式契約**：`@SELECT_RESIDENT_NPC` 成功路徑必須明確 `RETURN RESULT`，禁止空 RETURN（Emuera 空 RETURN 回傳 0 = MASTER，導致 TARGET 錯綁）。
- **UI 文案敘述性**：選單不顯示硬數字「心情 +10」，改為「有機會改善心情」，結算後輸出實際結果（見 Interaction_System.md 10.1）。
- **呼叫方防呆**：`CALL SELECT_RESIDENT_NPC` 後先判斷 `RESULT < 0` 再允許扣款（Interaction_System.md 10.3）。

---

## Phase 6：口上系統（KOJO） ✅

### 當前狀態
**全部完成。** Step 6-1～6-3 均已實作並整合至既有系統。

| Step | 狀態 | 主要產出 |
|---|---|---|
| 6-1 核心分派框架 | ✅ 完成 | `ERB/Kojo/KOJO_CORE.ERB`（`@KOJO_MESSAGE` / `@KOJO_TONE_FILTER` / `@KOJO_DEFAULT` / `@SYS_KOJO_INIT`） |
| 6-2 觸發點掛鉤 | ✅ 完成 | `BASE_LIFECYCLE.ERB` 晨間 / `BASE_ROOM.ERB` 入住撤離 / `BASE_INTERACT.ERB` 互動結算 / `BATTLE_MAIN.ERB` 戰鬥事件 |
| 6-3 NPC 個別口上 | ✅ 完成 | `ERB/Kojo/KOJO_NPC.ERB`（`@KOJO_0010` / `@KOJO_0011` / `@KOJO_0012`） |

### 實現方式（參考）
→ 詳見 [Kojo_Standard.md](Kojo_Standard.md)（全文）

### 已完成任務清單

#### Step 6-1：核心分派框架
| 任務 | 函式 / 標籤 | 說明 |
|---|---|---|
| 統一入口 | `@KOJO_MESSAGE` | ARG:0=CHARA_ID, ARG:1=EVT, ARG:2=P0, ARG:3=P1；防重入（`TFLAG:KOJO_LAST_EVT`、`KOJO_LAST_TURN`） |
| 語氣濾鏡 | `@KOJO_TONE_FILTER` | 依疲勞/心情輸出 `TFLAG:KOJO_TONE`（0 正常/1 煩躁/2 低落） |
| 預設口上 | `@KOJO_DEFAULT` | 覆蓋所有事件的通用 fallback，避免沉默或報錯 |
| 稱呼初始化 | `@SYS_KOJO_INIT` | 入住時設定 `CSTR:TARGET:CALL_PLAYER`、`CALL_SELF` |

#### Step 6-2：觸發點掛鉤（插入既有函式）
| 觸發位置 | 事件 ID | 插入函式 |
|---|---:|---|
| 晨間問候（每位住宿 NPC） | 110 | `@BASE_EVENT_MORNING` 住宿迴圈內 |
| 入住 | 120 | `@SYS_ROOM_MOVE_IN` 末尾 |
| 撤離 | 121 | `@SYS_ROOM_MOVE_OUT` 末尾 |
| 互動結算後 | 210 | `@INTERACT_DISPATCH` 末尾 |
| 戰鬥回合抽牌後 | 310 | `@BATTLE_DRAW_CARDS` 末尾 |
| 技能映射成功後 | 320 | `@BATTLE_MATCH_SKILL` 命中分支後 |
| Temp 存牌成功後 | 330 | `@BATTLE_UI_MAIN` 450 成功分支 |
| 戰鬥結束前 | 350 | `@BATTLE_ENTER` 回傳前 |

#### Step 6-3：NPC 個別口上（逐一建立）
| 任務 | 函式 | 說明 |
|---|---|---|
| NPC 00 口上 | `@KOJO_0010` | Chara ID 1（冷靜沉穩型）；BOND_TIER / LUST_TIER 段位計算，SELECTCASE EVT，覆蓋 110/120/121/210/311/320/330/331/350 |
| NPC 01 口上 | `@KOJO_0011` | Chara ID 2（開朗活潑型）；同上結構，各事件台詞風格差異顯著 |
| NPC 02 口上 | `@KOJO_0012` | 保留欄位（尚未分配角色）；框架已建，暫轉發至 `@KOJO_DEFAULT` |

### MVP 檢核點驗證
| 檢核項目 | 狀態 |
|---|---|
| `@KOJO_MESSAGE` 能正確分派到對應 NPC 口上 | ✅ CASE 1→KOJO_0010 / CASE 2→KOJO_0011 |
| 同一回合同一事件不重複刷屏（300-399） | ✅ TFLAG:KOJO_LAST_EVT + KOJO_LAST_TURN 防重入 |
| 每個 EVT ID 都有 default fallback | ✅ @KOJO_DEFAULT 覆蓋所有事件 + 個別函式 CASEELSE 靜默 |
| 晨間問候在 DEBUG 模式下可強制觸發 | ✅ DEBUG_MENU [4] 口上測試器（@DEBUG_KOJO_MENU / @DEBUG_KOJO_TRIGGER） |

### 架構規範提醒
- `CSTR` 只存稱呼與短標籤，禁止塞長文本；劇情段落寫在 ERB 台詞區（Kojo_Standard.md 第 3 節）。
- 每個 NPC 口上的段位計算（BOND_TIER/LUST_TIER）集中在函式頂端，避免分支裡重複寫比較式。

---

## Phase 7：進階戰鬥機制擴充 🎨

### 當前狀態
設計文件完整（Battle_System.md 第 11 節），現有戰鬥 MVP 架構不需改動即可容納這兩個系統。待 MVP 穩定後實作。

### 實現方式（參考）
→ 詳見 [Battle_System.md](Battle_System.md)（第 11 節 Advanced Battle Mechanics）

### 待實作任務清單

#### 7-A：敵方意圖預警系統（Enemy Intent System）
| 任務 | 函式 / 標籤 | 說明 |
|---|---|---|
| 意圖決策 | `@BATTLE_ENEMY_INTENT_BUILD` | 每回合起始，計算並快取敵方下一行動（類型/範圍/預估傷害） |
| 意圖 UI 顯示 | `@BATTLE_ENEMY_INTENT_PRINT` | 玩家回合期間常駐顯示；`BATTLE_TYPE == TRAINING` 時關閉 |
| 意圖更新 | `@BATTLE_ENEMY_INTENT_REFRESH` | 敵方行動結束後呼叫，供下回合顯示 |

#### 7-B：EN 保留生息與極限突破（EN Accumulation & Limit Break）
| 任務 | 函式 / 標籤 | 說明 |
|---|---|---|
| EN 累積結算 | `@BATTLE_EN_ACCUMULATE` | 出招後統計未使用卡牌數，累加至 `BASE:BTL_EN`（各角色獨立） |
| 極限突破觸發判定 | `@BATTLE_CHECK_LIMIT_BREAK` | 玩家手動觸發；判斷 `BASE:BTL_EN >= 門檻值` |
| 極限突破執行 | `@BATTLE_EXEC_LIMIT_BREAK` | 扣除 EN、執行突破效果（Shape 條件放寬/傷害倍率強化） |
| 戰鬥結束 EN 重置 | `@BATTLE_ENTER`（擴充） | 戰鬥結束後清空 `BASE:BTL_EN`（戰鬥內個體資源） |

### MVP 檢核點
> 意圖系統：玩家回合期間畫面顯示敵方意圖，`TRAINING` 標記時自動關閉；EN 系統：保留 2 張牌後 EN+2 正確計算，達門檻後玩家可手動觸發突破，觸發後 EN 清零。

### 架構規範提醒
- EN 為戰鬥內個體資源，`BASE:BTL_EN` 屬 SESSION 層，`@BATTLE_ENTER` 初始化清零。
- 意圖系統只影響 UI 顯示，不改動核心出招流程（`@BATTLE_PROCESS_ACTION` 保持不變）。
- 新增 Shape 效果時只動 `@BATTLE_CALC_BOOST` 與技能配方表，不破壞現有雙軸語意（Battle_System.md 第 9 節）。

---

## Phase 8：系統配置、存檔生命週期與畫廊 📋

### 當前狀態
設計規範完整（System_Config.md），三層資料生命週期（SAVEDATA / SESSION / GLOBAL）已定義，尚未實作對應初始化 Hook 與畫廊 UI。

### 實現方式（參考）
→ 詳見 [System_Config.md](System_Config.md)（全文）

### 待實作任務清單

#### Step 8-1：生命週期 Hook
| 任務 | 函式 / 標籤 | 說明 |
|---|---|---|
| 新遊戲初始化 | `@SYS_ON_NEWGAME` | 清空 TFLAG、設定 CFG 預設值、初始化資源與房間槽 |
| 讀檔後清理 | `@SYS_ON_LOAD` | 清空所有 SESSION 層 TFLAG（防讀檔後戰鬥暫存殘留） |
| 換日初始化 | `@SYS_ON_DAY_START` | `FLAG:DAY_PHASE=1`（晨間）、清除當日防呆旗標 |
| 戰鬥進入初始化 | `@SYS_ON_BATTLE_ENTER` | 清空手牌、TEMP、BUFF、LOOT 旗標（已部分實作於 `@BATTLE_ENTER`，整合至此 Hook） |

#### Step 8-2：難度係數表
| 任務 | 函式 / 標籤 | 說明 |
|---|---|---|
| 難度初始化 | `@SYS_DIFFICULTY_INIT` | 依 `FLAG:CFG_DIFFICULTY`（0~3）設定 HP/ATK/掉落係數；係數存入 FLAG 或直接在計算時引用 |

#### Step 8-3：玩家設定 UI
| 任務 | 函式 / 標籤 | 說明 |
|---|---|---|
| 設定選單 | `@SYS_CONFIG_MENU` | `FLAG:CFG_AUTO_BATTLE`、`CFG_AUTO_STRATEGY`、`CFG_TEXT_SPEED`、`CFG_SHOW_KOJO` 等 |

#### Step 8-4：回想與畫廊
| 任務 | 函式 / 標籤 | 說明 |
|---|---|---|
| 畫廊選單 | `@SYS_GALLERY_MENU` | 依 `GLOBAL:CG_UNLOCK[id]` 顯示可播項目 |
| 回想播放（只讀模式） | `@SYS_RECALL_PLAY` | 進入前 `TFLAG:RECALL_MODE=1`；結算段落加 Gate：`IF TFLAG:RECALL_MODE==1` 跳過寫入 |
| 事件解鎖雙寫 | 各事件結算點 | 首次完成時同時寫 `FLAG:EVT_SEEN[id]` 與 `GLOBAL:CG_UNLOCK[id]` |

### MVP 檢核點
> 讀檔後不殘留上一局戰鬥 TFLAG；`FLAG:CFG_*` 設定能跨存檔生效；畫廊回想播放時資源/PALAM 不被修改；新遊戲時全部 SESSION 層旗標確認為 0。

### 架構規範提醒
- 三層命名嚴格區分：`FLAG/VAR/CFLAG` 存檔層、`TFLAG` 暫存層、`GLOBAL:*` 跨存檔層（System_Config.md 第 1 節）。
- Ironman 模式只允許晨間或晚間存檔，避免戰鬥刷讀檔；存檔時機由 Core Loop 控管。

---

## Phase 9：數值平衡校準與 Debug 精煉 📋

### 當前狀態
平衡公式已在文件中確立（Progression_Balance.md），Debug 框架已實作（Developer_Guidelines.md）。待前置系統穩定後進行迭代調參。

### 實現方式（參考）
→ 詳見 [Progression_Balance.md](Progression_Balance.md)（全文）、[Developer_Guidelines.md](Developer_Guidelines.md)

### 待實作任務清單

#### Step 9-1：敵方曲線落地
| 任務 | 函式 / 標籤 | 說明 |
|---|---|---|
| 敵方等級計算 | `@BATTLE_CALC_ENEMY_LEVEL` | `E = FLOOR((D-1)/3) + 2*L`；D 從 `FLAG:DAY`，L 從關卡資料 |
| 敵方 HP/ATK 生成 | `@BATTLE_INIT_ENEMY_STATS` | 線性版 MVP：`HP = HP0 + E*HP_LIN`、`ATK = ATK0 + E*ATK_LIN`；難度係數乘入 |

#### Step 9-2：設施升級成本與產出計算
| 任務 | 函式 / 標籤 | 說明 |
|---|---|---|
| 設施效率計算 | `@BASE_CALC_FAC_EFF` | `FinalEff = (1 + 0.10*(LV-1)) * (TRAIN_MOD/100)`；分設施層與角色層 |
| 升級成本查詢 | `@BASE_CALC_UPGRADE_COST` | 金錢 `M0 * 1.55^(LV-1)`、材料 `T0 * 1.45^(LV-1)` |
| 回本天數計算（Debug 用） | `@DEBUG_CALC_PAYBACK` | 輸出升級回本天數，驗證是否在 3–15 天目標區間 |

#### Step 9-3：Debug 功能補全
| 任務 | 函式 / 標籤 | 說明 |
|---|---|---|
| 強制戰鬥勝利 | `@DEBUG_FORCE_BATTLE_WIN` | 直接設 `RESULT=1`、寫入測試掉落 |
| 強制晚間事件 | `@DEBUG_FORCE_NIGHT_EVENT` | 直接呼叫 `@TRIGGER_NIGHT_EVENT` 並傳入指定 ID |
| 口上測試器 | `@DEBUG_KOJO_TESTER` | 選 NPC 與 EVT_ID，強制呼叫 `@KOJO_MESSAGE` |
| 平衡指標輸出 | `@DEBUG_BALANCE_REPORT` | 每日輸出：維護費/產出/使用率/平均疲勞 |

### MVP 檢核點
> Day 1 戰鬥回合數落在 6–10；Day 15 敵方 HP 仍可在 8–12 回合打完；設施升級回本天數在 3–6 天（前期）；訓練至「戰鬥主力」門檻（TRAIN_RANK >= 3）約需 10–15 天。

### 架構規範提醒
- **每個版本只調一件事**：先調敵方曲線，再調經濟回本，再調刻印速度，最後才調門檻（見 Progression_Balance.md）。
- Debug 模組可快速迭代，但**不得破壞主流程邏輯**；正式功能完成後，對應 debug 工具整併或移除（Developer_Guidelines.md）。

---

## 快速參照：函式命名索引

### 跨模組初始化入口（`@<MODULE>_INIT`）
| 函式 | 模組 | 狀態 |
|---|---|:---:|
| `@BASE_INIT` | Base | 📋 |
| `@BATTLE_INIT` | Battle | ✅（含於 `@BATTLE_ENTER`） |
| `@CHARA_INIT` → `@INIT_CHARA_STATS` | Chara | ✅ |

### 生命週期事件 Hook（`@<MODULE>_EVENT_<TIMING>`）
| 函式 | 模組 | 狀態 |
|---|---|:---:|
| `@BASE_EVENT_MORNING` → `@BASE_MORNING_TICK` | Base | 🚧 |
| `@BASE_EVENT_NIGHT` → `@BASE_NIGHT_TICK` | Base | 🚧 |
| `@CHARA_EVENT_NIGHT` | Chara | 📋 |

### 戰鬥完整函式列表
| 函式 | 狀態 |
|---|:---:|
| `@BATTLE_PREPARATION` | ✅ |
| `@BATTLE_ENTER` | ✅ |
| `@BATTLE_UI_MAIN` | ✅ |
| `@BATTLE_DRAW_CARDS` | ✅ |
| `@BATTLE_PROCESS_ACTION` | ✅ |
| `@BATTLE_CALC_BOOST` | ✅ |
| `@BATTLE_MATCH_SKILL` | ✅ |
| `@BATTLE_ENEMY_TURN` | ✅ |
| `@BATTLE_SETTLEMENT_REPORT` | ✅ |
| `@BATTLE_ENEMY_INTENT_BUILD` | 🎨 |
| `@BATTLE_EN_ACCUMULATE` | 🎨 |
| `@BATTLE_EXEC_LIMIT_BREAK` | 🎨 |
| `@BATTLE_CALC_ENEMY_LEVEL` | 📋 |
| `@BATTLE_INIT_ENEMY_STATS` | 📋 |

---

## 架構規範速查（開發時永遠適用）

| 規範 | 要點 | 來源文件 |
|---|---|---|
| 模組邊界 SRP | System/Core Loop → Domain（Base/Battle/Chara），禁止反向依賴 | [Project_Structure.md](Project_Structure.md) |
| 禁止跨域操作底層陣列 | Core Loop 不直接讀寫 `WORK_Q_REMAIN`；入住/撤離只走單一入口 | [Project_Structure.md](Project_Structure.md) |
| CALL/RETURN 跳轉安全 | 跨模組跳轉一律 CALL/RETURN；同模組選單可用 GOTO | [Core Loop.md](Core%20Loop.md) |
| TARGET 校驗契約 | `@SELECT_RESIDENT_NPC` 必須明確 RETURN RESULT，禁止空 RETURN | [Interaction_System.md](Interaction_System.md) |
| 畫面重繪原則 | 任何狀態變動後 GOTO 回選單頂端重新 PRINT | [Developer_Guidelines.md](Developer_Guidelines.md) |
| UI 文案敘述性 | 不顯示硬數字增減，改用結果回報 | [Interaction_System.md](Interaction_System.md) |
| Debug 命名 | 所有 debug 函式以 `@DEBUG_` 前綴，收斂在 `DEBUG_MENU.ERB` | [Developer_Guidelines.md](Developer_Guidelines.md) |
| 百分比符號 | `PRINTFORM` 中使用 `\%` 或全形 `％`，禁用 `%%` | [UI_Layout.md](UI_Layout.md) |
| CSV 表頭 | 包含文字表頭的 CSV 行首必須加 `;` 註解 | [UI_Layout.md](UI_Layout.md) |
