# Base_Management.md
> 系統：基地經營（設施、房間槽、住宿 NPC）  
> 適用流程：日間階段主操作，晨間產出與晚間清算（由主循環呼叫）

基地經營的核心是三件事：設施提供「可見的操作選項」，房間槽提供「住宿容量與維護成本」，住宿 NPC 提供「成長、加成與事件」。日間讓玩家做選擇與投入資源，晨間把設施與住宿帶來的被動效果一次結算，晚間做疲勞與狀態清算並觸發事件。
把基地系統當成一個可插拔模組，主循環只需要在晨間與晚間各呼叫一次 Tick，在日間提供入口選單即可。

---

## 模組目錄結構（ERB/Base/）

重構後基地系統拆分為 8 個獨立 ERB 模組，各自負責單一職責：

```text
ERB/Base/
├── BASE_MANAGEMENT.ERB   協調入口：初始化、主選單路由、共用工具函式
├── BASE_LIFECYCLE.ERB    生命週期 Tick：晨間/晚間結算、晨報輸出
├── BASE_UI.ERB           UI 輸出：HUD 與住宿 NPC 列表
├── BASE_TRAIN.ERB        訓練室：選單 UI + @SYS_TRAIN_APPLY（成長公式唯一入口）
├── BASE_WORK.ERB         工坊：選單 UI + 生產佇列操作
├── BASE_LOUNGE.ERB       休息室：選單 UI + 互動前置檢查
├── BASE_INTERACT.ERB     互動系統：分派、指令、成功率、PALAM、刻印、回饋
└── BASE_ROOM.ERB         房間管理：選單 UI + @SYS_ROOM_MOVE_IN/OUT（入住/撤離唯一入口）
```

Emuera 引擎載入同目錄所有 ERB 時，所有函式共享命名空間，無需明確 import。

---

## 系統邊界與呼叫點

### 主循環需要呼叫的入口（`CORE_LOOP.ERB`）

| Hook 函式 | 呼叫時機 | 職責 |
|---|---|---|
| `CALL @BASE_INIT` | 遊戲初始化 | 初始化房間槽與工坊佇列 |
| `CALL @BASE_EVENT_MORNING` | 晨間階段（`@PHASE_MORNING`） | 設施產出、住宿加成刷新、維護費、工坊佇列推進 |
| `CALL @BASE_MENU` | 日間階段（`@PHASE_DAY`） | 玩家操作基地 |
| `CALL @BASE_EVENT_NIGHT` | 晚間階段（`@PHASE_NIGHT`） | 使用次數清空、事件冷卻遞減、撤離冷卻遞減 |

> 戰鬥入口由基地選單呼叫：`CALL INIT_PARTY_STATE` + `CALL BATTLE_PREPARATION`，戰鬥結束後 `RETURN` 回基地選單原位置。

---

## 生命週期 Tick 詳細說明（BASE_LIFECYCLE.ERB）

### 晨間 Tick：`@BASE_EVENT_MORNING`

**防重入機制：**
- 函式開頭檢查 `TFLAG:MORNING_DONE != 0`，若已執行則立即 `RETURN`。
- 通過後設 `TFLAG:MORNING_DONE = 1`，同時重設 `TFLAG:NIGHT_DONE = 0`（確保今晚可結算）。

**執行順序：**
1. **住宿加成刷新**：依 `FLAG:BASE_FAC_LV_ROOM` 上限，對所有住宿 NPC 執行 `CF_MOOD += 3`（上限 100）。
2. **首日保護**：`FLAG:BASE_DAY > 1` 才執行設施產出與維護費（Day 1 跳過）。
3. **設施產出**：
   - 金錢收入 = `40 + BASE_FAC_LV_WORK * 60`
   - 材料收入 = `8 + BASE_FAC_LV_WORK * 6`
   - 補給收入 = `BASE_FAC_LV_TRAIN * 2 + BASE_FAC_LV_LOUNGE * 3`
4. **維護費結算**：逐槽計算，基礎 3 點，房間等級每級折扣 5%（上限 50% 折扣）。優先扣補給，不足再以 12:1 換算扣金錢，仍不足則計入欠費警告。
5. **工坊佇列推進**：所有佇列槽 `WORK_Q_REMAIN -= 1`，歸零時物品入庫 `ITEM:(TYPE) += 1`。
6. **防呆歸零**：資源三項不可低於 0。
7. **暫存晨報至 TFLAG**，呼叫 `@BASE_PRINT_MORNING_REPORT`。

### 晚間 Tick：`@BASE_EVENT_NIGHT`

**防重入機制：**
- 函式開頭檢查 `TFLAG:NIGHT_DONE != 0`，若已執行則立即 `RETURN`。
- 通過後設 `TFLAG:NIGHT_DONE = 1`，同時重設 `TFLAG:MORNING_DONE = 0`（確保明天早上可結算）。

**執行順序：**
1. 清空今日設施使用次數（`TFLAG:BASE_USES_TRAIN = 0`、`TFLAG:BASE_USES_LOUNGE = 0`）。
2. 事件冷卻遞減（`EVT_COOLDOWN_RES / BASE / NPC` 各 -1，下限 0）。
3. 撤離冷卻遞減（所有 NPC 的 `CF_MOVEOUT_COOLDOWN -= 1`，下限 0）。

### 晨報輸出：`@BASE_PRINT_MORNING_REPORT`

讀取 TFLAG 暫存數據並格式化輸出：

| TFLAG 鍵 | 內容 |
|---|---|
| `MORNING_INCOME_MONEY / MAT / SUP` | 設施收益（金錢/材料/補給） |
| `MORNING_COST_SUP / MONEY` | 維護費扣除（補給/金錢） |
| `MORNING_WARN_NPC` | 欠費 NPC 數量（>0 時輸出警告） |
| `MORNING_CRAFT_COUNT / CRAFT_LAST` | 工坊完工數量與最後一項物品 ID |

---

## 跳轉規範（BASE_MANAGEMENT.ERB）

### `@BASE_MENU` 主選單路由

| 按鈕 | 行為 | 跳轉方式 |
|---|---|---|
| [100] 訓練室 | `CALL BASE_TRAIN_MENU` | CALL/RETURN（設施子選單） |
| [110] 工坊 | `CALL BASE_WORK_MENU` | CALL/RETURN（設施子選單） |
| [120] 休息室 | `CALL BASE_LOUNGE_MENU` | CALL/RETURN（設施子選單） |
| [130] 房間管理 | `CALL BASE_ROOM_MENU` | CALL/RETURN（設施子選單） |
| [140] 出戰 | `CALL INIT_PARTY_STATE` + `CALL BATTLE_PREPARATION` | CALL/RETURN（跨域） |
| [999] Debug | `CALL DEBUG_MAIN_MENU` | CALL/RETURN（跨域） |
| [9] 返回 | `RETURN` | 回上層呼叫方 |

CALL 返回後一律 `GOTO BASE_MENU_TOP` 刷新 UI，符合「同模組內部 UI 更新才可使用 GOTO」規範。

---

## 設施系統（Facilities）

### 共通規格

每個設施都遵循下列欄位與流程：

- 等級（LV）：決定可用指令、效率、每日次數上限
- 每日使用次數（Daily Uses）：日內上限，晚間 Tick 清空
- 消耗（Cost）：金錢、材料、補給、NPC 疲勞等
- 效果（Effect）：立即生效或延遲到晨間結算

### 1) 訓練室（BASE_TRAIN.ERB）

**每日次數上限** = `1 + FLOOR(訓練室LV / 2)`（最低 1 次）

#### 訓練流程

1. **選訓練強度**：
   - 基礎訓練：消耗補給 5、疲勞 +20
   - 專項訓練（Lv.3+）：消耗補給 10、疲勞 +30
2. **選目標 NPC**：`CALL SELECT_RESIDENT_NPC`，疲勞 >= 80 不可訓練
3. **選側重屬性**：物理 / 魔法 / 特攻 / 物防 / 魔防 / 速度
4. **`CALL SYS_TRAIN_APPLY`**（唯一成長寫入點）

#### `@SYS_TRAIN_APPLY` 參數

| ARG | 說明 |
|---|---|
| ARG | NPC ID |
| ARG:1 | 目標能力值索引（BTL_PATK 等） |
| ARG:2 | 疲勞增加量 |
| ARG:3 | 基礎成長量（基礎=1 / 專項=2） |
| ARG:4 | 訓練倍率加成量 |
| RETURN | 實際套用的成長量 |

**成長公式：**
```
成長量 = ARG:3 + FLOOR(BASE_FAC_LV_TRAIN / 3) + 住宿加成(好感≥5:+1) + 狀態修正(心情≥70:+1, 疲勞≥60:-1)
最低保底 = 1
```

---

### 2) 工坊（BASE_WORK.ERB）

**生產槽位上限** = `1 + FLOOR(工坊LV / 2)`，不超過 `WORK_Q_MAX`

#### 主要功能

| 操作 | 按鈕 | 說明 |
|---|---|---|
| 材料轉補給 | [1] | 消耗材料 10，補給 += `5 + BASE_FAC_LV_WORK * 2` |
| 製作卡牌 | [2] | 待實裝（`@FACILITY_WORK_CRAFT_CARD`） |
| 取消生產 | [160+槽位] | 清空佇列，100% 退還材料成本 |
| 升級工坊 | [8] | `CALL FACILITY_UPGRADE_MENU` |

**工坊佇列推進由 `@BASE_EVENT_MORNING` 統一處理，`WORK_Q_REMAIN` 與 `WORK_Q_TYPE` 的寫入僅在 `BASE_WORK.ERB` 與 `BASE_LIFECYCLE.ERB` 中發生。**

---

### 3) 休息室（BASE_LOUNGE.ERB + BASE_INTERACT.ERB）

**每日次數上限** = `1 + FLOOR(休息室LV / 2)`（最低 1 次）

#### 互動流程（BASE_LOUNGE.ERB 負責前置）

1. 次數限制檢查（`TFLAG:BASE_USES_LOUNGE >= 上限`）
2. 成人向門檻：任一住宿 NPC 好感 >= 3 才顯示成人選項
3. `CALL SELECT_RESIDENT_NPC` 選目標
4. 個別好感門檻確認（CMD 321 需段位 >= 5；CMD 320+ 需段位 >= 3）
5. `CALL INTERACT_DISPATCH, CMD_ID` 進入互動系統

#### 互動指令表

| CMD_ID | 名稱 | 類型 | 主要效果 |
|---|---|---|---|
| 300 | 日常交流 | 日常 | 心情+8，疲勞-10~，好感機率提升 |
| 301 | 安撫 | 日常 | 疲勞大幅緩解，輕微降低撤離壓力 |
| 302 | 娛樂活動 | 日常 | 心情大幅改善，輕微疲勞代價 |
| 303 | 贈送物資 | 日常 | 消耗補給 5，好感確定+1 |
| 320 | 試探 | 成人 | Gate(反感<75) → 成功率計算 → PALAM/刻印/絕頂 |
| 321 | 開發指令 | 成人 | Gate(反感<65) + 好感≥5 → 含處女旗標 |
| 329 | 善後安撫 | 成人 | 反感-5，心情+5，撤離壓力-2（不走成功率） |

---

## 互動系統詳細說明（BASE_INTERACT.ERB）

### 呼叫流程圖

```
@INTERACT_DISPATCH (CMD_ID)
    └─ CALL INTERACT_GATE_CHECK          Step 5-2：Gate 判斷
    └─ CALL CALC_INTERACT_SUCCESS_ADULT  Step 5-3：成功率計算
    └─ [若成功]
        ├─ CALL CALC_INTERACT_EFFECT_SCALE   效果倍率
        ├─ CALL INTERACT_APPLY_PALAM         PALAM 套用
        ├─ CALL INTERACT_APPLY_MARK          刻印點數
        ├─ CALL INTERACT_CHECK_MARK_LV       刻印升級
        ├─ CALL INTERACT_CHECK_CLIMAX        絕頂觸發
        ├─ CALL INTERACT_VIRGIN_CHECK        處女旗標（CMD 321 限定）
        ├─ CALL INTERACT_FEEDBACK_BASE       回饋 TRAIN_MOD
        └─ CALL INTERACT_UPDATE_MOVEOUT, 0   撤離壓力更新
    └─ [若失敗]
        └─ CALL INTERACT_APPLY_FAIL, 失敗等級
            └─ CALL INTERACT_UPDATE_MOVEOUT, 1/2/3
```

### 輔助函式職責說明

| 函式 | 職責 | TFLAG 鍵 |
|---|---|---|
| `@INTERACT_GATE_CHECK` | 反感門檻 + 疲勞 >= 90 拒絕 | — |
| `@CALC_INTERACT_SUCCESS_DAILY` | 日常互動成功率（CLAMP 10–95） | `INTERACT_SUCCESS_RATE` |
| `@CALC_INTERACT_SUCCESS_ADULT` | 成人向成功率（CLAMP 5–90） | `INTERACT_SUCCESS_RATE` |
| `@CALC_INTERACT_EFFECT_SCALE` | 效果倍率（x100 整數，含反感加成/懲罰） | `INTERACT_EFFECT_SCALE_X100` |
| `@INTERACT_APPLY_PALAM` | 將 DELTA * Scale / 100 套用到 PALAM | `INTERACT_ACTUAL_*` |
| `@INTERACT_APPLY_MARK` | 依實際 DELTA 計算刻印點數並呼叫升級檢查 | — |
| `@INTERACT_CHECK_MARK_LV` | 刻印升級門檻 20/60/120，GOTO 迴圈直到穩定 | — |
| `@INTERACT_CHECK_CLIMAX` | 絕頂條件：興奮≥90 + 快樂≥70 + 反感≤40 + Scale≥100 | — |
| `@INTERACT_VIRGIN_CHECK` | 首次體驗旗標：CF_VIRGIN=0、EXP+10、刻印+5 | — |
| `@INTERACT_FEEDBACK_BASE` | 依刻印/好感/疲勞/反感調整 CF_TRAIN_MOD | — |
| `@INTERACT_UPDATE_MOVEOUT` | 0=成功/1=小失敗/2=大失敗/3=災難/4=善後 | — |
| `@INTERACT_APPLY_FAIL` | 0=小失敗/1=大失敗/2=災難級 數值惡化 | — |

---

## 房間槽（Room Slot）機制（BASE_ROOM.ERB）

### 定義
- 可用槽位數 = `FLAG:BASE_FAC_LV_ROOM`（上限 `ROOM_OWNER_MAX`）
- `ROOM_OWNER:slot` = NPC ID，空槽為 -1

### 入住（`@SYS_ROOM_MOVE_IN`）

**前置條件（任一不滿足則 RETURN 0）：**
- 槽位索引在有效範圍內
- `ROOM_OWNER:slot == -1`（空槽）
- `CFLAG:NPC:CF_IS_RESIDENT == 0`（未住宿）
- `FLAG:RES_SUPPLY >= 5`（資源門檻）

**成功後寫入：**
- `ROOM_OWNER:slot = NPC_ID`
- `FLAG:ROOM_USED += 1`
- `CF_IS_RESIDENT = 1`，`CF_ROOM_SLOT = slot`

### 撤離（`@SYS_ROOM_MOVE_OUT`）

**成功後寫入：**
- `ROOM_OWNER:slot = -1`
- `FLAG:ROOM_USED -= 1`（下限 0）
- `CF_IS_RESIDENT = 0`，`CF_ROOM_SLOT = 9999`
- `CF_MOVEOUT_COOLDOWN = 3`（冷卻 3 日不可再入住）

> **架構規範**：所有 `ROOM_OWNER` 陣列的寫入必須透過 `@SYS_ROOM_MOVE_IN` / `@SYS_ROOM_MOVE_OUT`，禁止在其他模組直接操作。

---

## 共用工具函式（BASE_MANAGEMENT.ERB）

| 函式 | 職責 |
|---|---|
| `@BASE_INIT` | 初始化所有槽位（`ROOM_OWNER = -1`）與工坊佇列 |
| `@BASE_SYNC_DAILY_USES` | 換日時清空 `TFLAG:BASE_USES_TRAIN / LOUNGE`（以 `FLAG:BASE_DAY` 比對） |
| `@SELECT_RESIDENT_NPC` | 輸出住宿 NPC 列表，INPUT 後驗證並回傳 NPC ID；取消 RETURN -1 |
| `@FACILITY_UPGRADE_MENU` | 通用升級子選單；ARG=目前等級、ARGS=設施名稱；RETURN 1=成功 |

---

## 防呆與防重入規範

| 風險 | 防呆機制 |
|---|---|
| 晨間重複結算 | `TFLAG:MORNING_DONE != 0` 在 `@BASE_EVENT_MORNING` 開頭攔截 |
| 晚間重複結算 | `TFLAG:NIGHT_DONE != 0` 在 `@BASE_EVENT_NIGHT` 開頭攔截 |
| 跨日旗標殘留 | 晨間執行後重設 `NIGHT_DONE = 0`；晚間執行後重設 `MORNING_DONE = 0` |
| 房間槽狀態不同步 | 入住/撤離只走 `@SYS_ROOM_MOVE_IN / OUT`，禁止直接改 `ROOM_USED` |
| 資源負值 | 所有資源操作後執行 `< 0 → 0` 防呆 |
| 訓練指令跳過成長公式 | 能力值成長只允許透過 `@SYS_TRAIN_APPLY` |

---

## 玩家如何用設施培養 NPC

### 養成路徑 A：訓練室直升數值
1. 日間進入訓練室，先選訓練強度（基礎 / 專項）。
2. 選擇住宿 NPC，確認可訓練狀態後扣除補給。
3. 選擇側重屬性（物理/魔法/特攻/物防/魔防/速度）。
4. 即時提升目標屬性對應 BASE（+1 或 +2）。
5. 同步套用疲勞與訓練倍率變化，晚間再用休息室做狀態管理。

### 養成路徑 B：工坊做資源，拉長線
1. 日間在工坊升級，提高產線槽位與金錢/材料被動收益
2. 利用材料轉補給確保訓練室每日不斷糧
3. 透過宿舍擴建（升級房間管理）增加槽位，讓更多 NPC 進入日常循環

### 養成路徑 C：休息室養關係，解鎖互動深度
1. 日間進入休息室互動
2. 提升好感段位，解鎖成人向互動（段位 3）與開發指令（段位 5）
3. 維持心情與疲勞在健康區間，提高次日訓練效率（`CF_TRAIN_MOD` 加成）
4. 使用善後安撫穩定關係壓力，防止撤離

---

## SAVEDATA 數據結構規劃（全域變數）

### 變數表

| 類別 | 變數 | 型態 | 說明 | 重置時機 |
|---|---|---|---|---|
| 基地 | `FLAG:BASE_DAY` | SAVEDATA | 經營天數 | 不重置 |
| 資源 | `FLAG:RES_MONEY / RES_MATERIAL / RES_SUPPLY` | SAVEDATA | 金錢、材料、補給 | 不重置 |
| 設施等級 | `FLAG:BASE_FAC_LV_TRAIN / WORK / LOUNGE / ROOM` | SAVEDATA | 四設施等級 | 不重置 |
| 設施使用次數 | `TFLAG:BASE_USES_TRAIN / LOUNGE` | TFLAG | 今日已使用次數 | 晚間 Tick |
| 日期同步 | `TFLAG:BASE_DAY_SYNC` | TFLAG | 用於 `@BASE_SYNC_DAILY_USES` 換日偵測 | 換日時更新 |
| 晨間防重入 | `TFLAG:MORNING_DONE` | TFLAG | 1=已結算 | 晚間 Tick 重設 |
| 晚間防重入 | `TFLAG:NIGHT_DONE` | TFLAG | 1=已結算 | 晨間 Tick 重設 |
| 工坊佇列 | `WORK_Q_TYPE:n / WORK_Q_REMAIN:n` | SAVEDATA | 產線類型與剩餘天數 | 不重置 |
| 房間容量 | `FLAG:ROOM_USED` | SAVEDATA | 已住宿人數 | 透過 SYS 更新 |
| 槽位歸屬 | `ROOM_OWNER:slot` | SAVEDATA | 槽位對應 NPC 編號（-1=空） | 不重置 |
| NPC 住宿狀態 | `CFLAG:NPC:CF_IS_RESIDENT / CF_ROOM_SLOT` | SAVEDATA | 是否住宿與槽位編號 | 不重置 |
| NPC 成長相關 | `CFLAG:NPC:CF_TRAIN_MOD / CF_MOOD / CF_FATIGUE` | SAVEDATA | 訓練倍率、心情、疲勞 | 不重置 |
| 互動暫存 | `TFLAG:INTERACT_*` | TFLAG | 成功率、DELTA、SCALE 等中間值 | 每次互動覆蓋 |

---

## 追蹤指標
- 每日晨間：維護費總額、產出總額、住宿加成總額
- 日間：三設施使用率，出戰前平均操作步數
- NPC：平均成長速度、疲勞與心情分布、撤離觸發原因
- 房間槽：入住率、平均空槽天數、擴建後回本天數
