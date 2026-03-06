# System_Config.md
> 主題：全域機制規範（存檔、難度、回想畫廊、自動功能）  
> 適用：主循環以 `CALL/RETURN` 進出戰鬥並回到原階段，戰鬥只回傳結果與消耗。

本文件定義三層資料生命週期：**存檔層（SAVEDATA）**、**本局暫存層（TFLAG/臨時）**、**跨存檔層（GLOBAL）**。存檔層負責基地建設與角色養成等長期進度，暫存層負責單場戰鬥手牌、暫存槽與回合狀態，避免被讀檔後污染。跨存檔層則用於畫廊解鎖與玩家偏好設定，讓你換存檔也能保留收藏。難度系統以係數方式影響敵方曲線、掉落率與 AI 策略傾向，避免你每個模組都要重寫一次平衡。

---

## 行動建議（先落地這 6 件）
- **建立三層命名規則**：`FLAG/VAR/CFLAG/CSTR` 當存檔層，`TFLAG` 當本局暫存層，`GLOBAL:*` 當跨存檔層，並在讀檔後做一次清理。
- **集中初始化入口**：加上 `@SYS_ON_NEWGAME`、`@SYS_ON_LOAD`、`@SYS_ON_DAY_START`、`@SYS_ON_BATTLE_ENTER`，避免初始化散落導致狀態漏設。
- **把難度做成係數表**：HP/ATK、掉落率、AI 風險容忍都走同一張表，改難度只改表不改程式。
- **把回想當「只讀模式」**：回想播放時設 `TFLAG:RECALL_MODE=1`，禁用資源變動與養成推進，播完就回主選單。
- **用 GLOBAL 解鎖畫廊**：事件完成時同時寫入 `FLAG:EVT_SEEN[id]` 與 `GLOBAL:CG_UNLOCK[id]`，畫廊以 GLOBAL 為準。
- **把自動功能都收斂到 CFG**：自動戰鬥策略、訊息速度、顯示細節，都用 `FLAG:CFG_*` 統一管理，UI 也能用固定按鈕區間呼叫。

---

## 1. 存檔（SAVEDATA）規範
### 1.1 三層資料生命週期
- **SAVEDATA（存檔層）**：會被寫入存檔槽，讀檔後必須一致重現。
- **SESSION（本局暫存層）**：只在「這次遊戲進程」有效，讀檔或回到標題應清空。
- **GLOBAL（跨存檔層）**：獨立於存檔槽的全域保存，用於畫廊解鎖與玩家偏好。

> 你先前已把「永久保存」與「當日暫存」分開，這裡只是把它再往上加一層 GLOBAL。

### 1.2 必須保存的內容（SAVEDATA）
基地與角色屬於長期進度，必須保存，例如基地天數、資源、設施等級與房間槽映射等。

### 1.3 每局重置的內容（SESSION）
戰鬥流程狀態一律使用 `TFLAG` 或同等暫存欄位，例如手牌 4 格、暫存槽卡、回合數、策略等。

---

## 2. 難度與模式（Difficulty & Modes）
### 2.1 難度等級與係數
| 難度 | HP/ATK 係數 | 掉落率係數 | 敵方 AI 風險容忍 | 玩家資源收益 | 建議用途 |
|---|---:|---:|---:|---:|---|
| Story | 0.80 | 1.20 | 0.80 | 1.15 | 以劇情與收集為主 |
| Normal | 1.00 | 1.00 | 1.00 | 1.00 | 預設 |
| Hard | 1.15 | 0.95 | 1.10 | 0.95 | 追求策略與壓力 |
| Brutal | 1.30 | 0.90 | 1.20 | 0.90 | 長線挑戰 |

- HP/ATK 係數直接乘在你 Progression 的敵方曲線輸出上。
- 掉落率係數乘在戰鬥結束回傳的 `TFLAG:BATTLE_LOOT_*` 生成流程上，因為戰鬥回傳結構化掉落與資源變動是既定規格。
- AI 風險容忍影響「策略切換」的門檻與評分器權重，策略本身已定義為權重差異與條件切換。

### 2.2 模式（不只是難度）
| 模式 | 設定 | 效果 |
|---|---|---|
| Casual | 允許隨時存檔 | 容錯高 |
| Ironman | 限制存檔頻率 | 只允許晨間或晚間存檔，避免戰鬥刷讀檔 |
| FastRun | 加速 UI 與動畫 | 預設訊息速度快、跳過已讀口上 |
| Sandbox | 解鎖測試工具 | 允許開啟 debug 掉落與事件解鎖 |

---

## 3. 回想與畫廊（Replay & Gallery）
### 3.1 解鎖資料結構（GLOBAL + 存檔雙寫）
- **存檔內見過**：`FLAG:EVT_SEEN[id] = 1`
- **跨存檔解鎖**：`GLOBAL:CG_UNLOCK[id] = 1`

規則：
1. 事件首次完成時，先寫入存檔的 `EVT_SEEN`，再同步寫入 GLOBAL 的 `CG_UNLOCK`。
2. 畫廊清單預設使用 `GLOBAL:CG_UNLOCK` 決定可播放項目。
3. 若你想要「本存檔限定」模式，加入 `FLAG:CFG_GALLERY_SCOPE`：
   - 0：使用 GLOBAL（預設）
   - 1：只用本存檔 `EVT_SEEN`

### 3.2 回想播放的「只讀模式」
回想播放要做到兩件事：
- 不改變當前世界狀態與資源
- 播放結束能回到主選單或回想目錄

建議流程：
1. 進入回想前：`TFLAG:RECALL_MODE = 1`，備份必要狀態（若有）
2. 播放事件：呼叫對應事件段落（只輸出，不結算）
3. 播放結束：清空 `TFLAG:RECALL_MODE`，回到回想目錄

> 回想內所有會改動資源或 CFLAG 的結算段落，都加一個 Gate：若 `TFLAG:RECALL_MODE == 1` 就跳過寫入，只保留輸出與演出。

### 3.3 模擬案例（假設情境）
（假設）你把畫廊解鎖只寫在存檔內，玩家開新存檔想回看收藏卻發現全都鎖回去，心情直接歸零。你改成事件結算時同步寫入 GLOBAL，畫廊用 GLOBAL 讀取，玩家就能放心開新周目試不同路線。接著你又加了「本存檔限定」的選項給偏好無劇透的人，兩邊都照顧到。

---

## 4. 自動功能設定（Player Config）
### 4.1 自動戰鬥傾向
自動戰鬥策略與切換條件已存在，且支援鎖定策略。 
建議提供下列設定（存檔層保存，讓玩家每個周目可不同）：

| 設定鍵（FLAG:CFG_*） | 值域 | 預設 | 說明 |
|---|---:|---:|---|
| CFG_AUTO_BATTLE | 0/1 | 0 | 是否預設開啟自動戰鬥 |
| CFG_AUTO_STRATEGY | 0..3 | 0 | 預設策略：0 均衡，1 進攻，2 防禦，3 節省（戰鬥開始寫入 `TFLAG:BATTLE_STRATEGY`） |
| CFG_AUTO_STRATEGY_LOCK | 0/1 | 0 | 是否鎖定策略，不跑自動切換 |
| CFG_AUTO_TEMP | 0..2 | 1 | Temp 使用傾向：0 不用，1 只在壞手牌用，2 積極用（每回合一次存取仍是硬限制） |
| CFG_AUTO_CONFIRM | 0/1 | 0 | 自動選出招後是否自動確認 |

### 4.2 訊息顯示與 UI 開關
按鈕 ID 區間已規範 0-99 為系統設定與模式切換，適合放置選項頁入口。  

| 設定鍵（FLAG:CFG_*） | 值域 | 預設 | 說明 |
|---|---:|---:|---|
| CFG_TEXT_SPEED | 0..3 | 1 | 訊息速度：0 慢，1 標準，2 快，3 立即 |
| CFG_SKIP_SEEN | 0/1 | 1 | 跳過已看過的回想與事件提示（回想模式仍可強制顯示） |
| CFG_SHOW_BATTLE_HINT | 0/1 | 1 | 顯示映射技能提示區（UI 層） |
| CFG_SHOW_KOJO | 0/1 | 1 | 是否顯示口上（KOJO）輸出 |
| CFG_LOG_LINES | 6..20 | 10 | 戰鬥 log 保留行數（UI 內容區） |

---

## 5. 變數儲存表（你可以直接照這張表寫 ERB）
| 類別 | 例子變數 | 儲存層級 | 重置時機 | 來源對齊 |
|---|---|---|---|---|
| 天數與階段 | `FLAG:BASE_DAY`、`FLAG:DAY_PHASE` | SAVEDATA | 不重置 | Day Cycle 主流程:contentReference[oaicite:16]{index=16} |
| 基地資源 | `VAR:BASE_RES_MONEY/MAT/SUP` | SAVEDATA | 不重置 | 基地資源結算:contentReference[oaicite:17]{index=17} |
| 設施等級 | `FLAG:BASE_FAC_LV_*` | SAVEDATA | 不重置 | 基地設施等級:contentReference[oaicite:18]{index=18} |
| 每日使用次數 | `TFLAG:BASE_USES_*` | SESSION | 就寢或晚間 Tick | 每日上限:contentReference[oaicite:19]{index=19} |
| 房間槽 | `FLAG:ROOM_CAP/USED`、`FLAG:ROOM_OWNER[*]` | SAVEDATA | 不重置 | 槽位映射:contentReference[oaicite:20]{index=20} |
| NPC 狀態 | `CFLAG:CHARA:BOND_LV/MOOD/FATIGUE` | SAVEDATA | 不重置 | 養成狀態:contentReference[oaicite:21]{index=21} |
| 戰鬥資源 | `BASE:BTL_HP/EN/MAX*` | SESSION 或 SAVEDATA | 戰鬥開始初始化 | BASE 規劃:contentReference[oaicite:22]{index=22} |
| 戰鬥流程 | `TFLAG:HAND0..3`、`TFLAG:TEMP_*`、`TFLAG:BATTLE_TURN` | SESSION | `@BATTLE_ENTER` 初始化，戰鬥結束清理 | 戰鬥流程:contentReference[oaicite:23]{index=23} |
| 戰鬥策略 | `TFLAG:BATTLE_STRATEGY` | SESSION | 戰鬥開始從 CFG 或 NPC 預設載入 | 策略切換規則:contentReference[oaicite:24]{index=24} |
| 掉落與回傳 | `TFLAG:BATTLE_LOOT_*`、`RESULT:WINLOSE` | SESSION | 戰鬥結束後由日間/晚間結算消耗 | 回傳格式:contentReference[oaicite:25]{index=25} |
| 畫廊解鎖 | `GLOBAL:CG_UNLOCK[*]` | GLOBAL | 永久 | 跨存檔收藏 |
| 存檔內事件見過 | `FLAG:EVT_SEEN[*]` | SAVEDATA | 不重置 | 本周目記錄 |
| 玩家設定 | `FLAG:CFG_*` | SAVEDATA 或 GLOBAL | 視需求 | 自動與 UI |

---