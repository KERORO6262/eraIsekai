# Battle_System.md

> 系統：四象元素編織回合戰鬥（Element Weaving System）
>
> 入口：`CALL BATTLE_PREPARATION`（於 `ERB/Battle/BATTLE_MAIN.ERB`）
>
> 核心：
> 1) 以元素槽位（非道具卡）組成回合資源；
> 2) 玩家透過 UI 勾選控制「本回合要消耗的元素」；
> 3) 先做同色堆疊 Buff，再做技能配方映射，最後清空已選元素。

---


## 1. 戰前準備模組（Pre-Battle Phase）

`@BATTLE_PREPARATION` 為日間/晚間流程銜接戰鬥的前置 UI 入口。

本模組負責編組出戰隊伍。系統以 `PARTY_MEMBERS:0~3` 陣列儲存最多 4 名出戰角色的索引，`-1` 代表空槽位。玩家於介面中勾選本次參戰名單，並以該陣列作為後續戰鬥流程的參戰依據。

---

## 2. 參戰實體與敵人生成結構（Combat Entities）

戰鬥中的敵方單位以動態生成的角色實體（Character）存在。

戰鬥初始化流程為：
1. 建立敵方角色實體（如木人樁）。
2. 寫入該實體的 `BASE` 戰鬥數值（`BTL_HP`、`BTL_PATK`、`BTL_PDEF` 等）。
3. 將敵方索引儲存為本場戰鬥目標參照。

戰鬥結束流程為：
1. 判定勝敗。
2. 釋放敵方角色實體（`DELCHARA`）。
3. 清理戰鬥暫存旗標，回歸基地/階段流程。

---

## 3. 戰鬥資料模型總覽

### 1.1 四象元素定義（抽象資源，不是道具）

| 元素 | 常數 | 數值 | 說明 |
|---|---:|---:|---|
| 物理 | `#DEFINE ELEM_PHYS` | `1` | 偏向物理輸出與壓制資源。 |
| 魔法 | `#DEFINE ELEM_MAG` | `2` | 偏向魔法輸出與法術鏈結資源。 |
| 防禦 | `#DEFINE ELEM_DEF` | `3` | 偏向護甲/減傷等生存資源。 |
| 特殊 | `#DEFINE ELEM_SPEC` | `4` | 偏向機制觸發與功能性資源。 |

> 設計原則：戰鬥中抽到的是元素編碼（1~4），不再是 `ITEM.csv` 的卡牌實體。這使技能判定回歸「元素配方」而非「單卡 ID」。

### 1.2 回合槽位模型

| 類型 | 變數 | 說明 |
|---|---|---|
| 手牌 H0~H3 | `TFLAG:HAND0` ~ `TFLAG:HAND3` | 當回合主要元素池（4 槽）。 |
| 暫存槽 Temp | `TFLAG:TEMP_CARD` | 可跨流程操作的額外 1 槽。 |
| Temp 本回合使用鎖 | `TFLAG:TEMP_USED_THIS_TURN` | `0`=可用，`1`=本回合已使用。 |

---

## 4. 戰鬥 UI 操作規範（Toggle + Temp）

### 2.1 勾選機制（Toggle）

玩家透過以下指令切換選取狀態（0/1）：

| 指令 | 槽位 | 旗標 |
|---:|---|---|
| `410` | H0 | `TFLAG:UI_HAND0_SEL` |
| `411` | H1 | `TFLAG:UI_HAND1_SEL` |
| `412` | H2 | `TFLAG:UI_HAND2_SEL` |
| `413` | H3 | `TFLAG:UI_HAND3_SEL` |
| `420` | Temp | `TFLAG:UI_TEMP_SEL` |

- `0`：未選取（UI 顯示 `[ ]`）
- `1`：已選取（UI 顯示 `[X]`）

> UI 顯示邏輯由 `@BATTLE_UI_PRINT_HAND_SLOT` / `@BATTLE_UI_PRINT_TEMP_SLOT` 控制。

### 2.2 暫存槽（Temp Slot）硬性規則

執行按鍵：`[450] 執行暫存槽存/取 (每回合限1次)`

#### 規則 A：每回合限 1 次
- 若 `TFLAG:TEMP_USED_THIS_TURN == 1`，直接拒絕並提示：
  - `本回合已操作過暫存槽。`

#### 規則 B：必須「精確選取 1 個手牌元素」
- 系統只統計 H0~H3 中「非空且被勾選」的槽位。
- 若選取數量 `!= 1`，拒絕並提示：
  - `必須精確選取 1 個手牌元素才能進行暫存操作。`

#### 規則 C：存入 / 交換
- **Temp 為空 (`TFLAG:TEMP_CARD == 0`)**：
  - 把被選手牌元素移入 Temp。
  - 原手牌槽位清空為 `0`。
- **Temp 非空**：
  - 與被選手牌元素做雙向交換。

#### 規則 D：操作成功後清理
- 清空 `UI_HAND0_SEL~UI_HAND3_SEL` 與 `UI_TEMP_SEL`。
- 設定 `TFLAG:TEMP_USED_THIS_TURN = 1`。

---

## 5. 回合結算：Two-Stage Resolution

確認出招按鍵：`[430]`

`@BATTLE_UI_MAIN` 在 `RESULT == 430` 時流程如下：

```erb
CALL BATTLE_PROCESS_ACTION
IF RESULT == 0
    GOTO BATTLE_UI_MAIN_LOOP
ELSEIF RESULT == 1
    RETURN
ENDIF

CALL BATTLE_ENEMY_TURN
IF RESULT == 1
    RETURN
ENDIF

TFLAG:BATTLE_TURN += 1
CALL BATTLE_DRAW_CARDS
GOTO BATTLE_UI_MAIN_LOOP
```

- `@BATTLE_PROCESS_ACTION` 回傳 `0`：本次出招失敗（通常是沒選任何元素），不推進回合。
- 回傳 `1`：玩家已直接結束戰鬥（例如擊破敵人），UI 立即返回基地流程。
- 回傳 `2`：玩家出招成功且敵人仍存活，系統接著進入敵方回合，只有在敵方行動後戰鬥仍持續時才推進回合並抽牌。

### 3.1 階段一：同色堆疊加成（Stat Boosting）

`@BATTLE_PROCESS_ACTION` 會先統計被選元素數量：
- `LOCAL:50` = 物理數量
- `LOCAL:51` = 魔法數量
- `LOCAL:52` = 防禦數量
- `LOCAL:53` = 特殊數量
- `LOCAL:54` = 總選取數

若 `LOCAL:54 == 0`：
- `PRINTW 請至少選取一個元素來發動攻擊！`
- `RETURN 0`

接著呼叫 `@BATTLE_CALC_BOOST, ARG:0..ARG:3`，計算公式：

| Buff 變數 | 公式 | 說明 |
|---|---|---|
| `TFLAG:TEMP_BUFF_PATK` | `ARG:0 * 10` | 每 1 個物理元素給 +10% 臨時物攻。 |
| `TFLAG:TEMP_BUFF_MATK` | `ARG:1 * 10` | 每 1 個魔法元素給 +10% 臨時魔攻。 |
| `TFLAG:TEMP_BUFF_PDEF` | `ARG:2 * 10` | 每 1 個防禦元素給 +10% 臨時物防。 |
| `TFLAG:TEMP_BUFF_MDEF` | `ARG:3 * 10` | 每 1 個特殊元素給 +10% 臨時魔防（現行設計）。 |

> 註：目前 Buff 先存入 `TFLAG:TEMP_BUFF_*`，作為後續傷害公式接入點；`@BATTLE_DRAW_CARDS` 會在下一回合開始將這些值重置為 0。

#### 計算範例
若本回合選取：物理 2、魔法 1、防禦 0、特殊 1，則
- `TEMP_BUFF_PATK = 20`
- `TEMP_BUFF_MATK = 10`
- `TEMP_BUFF_PDEF = 0`
- `TEMP_BUFF_MDEF = 10`

### 3.2 階段二：技能映射比對（Skill Resolution）

呼叫：`CALL BATTLE_MATCH_SKILL, LOCAL:50, LOCAL:51, LOCAL:52, LOCAL:53`

比對範圍：`FOR LOCAL, 20000, 20010`

比對條件（全部成立才命中）：
1. 技能需求總和 > 0  
   `SKILL_REQ_PHYS:ID + SKILL_REQ_MAG:ID + SKILL_REQ_DEF:ID + SKILL_REQ_SPEC:ID > 0`
2. 玩家元素供給皆達標（特徵比對）  
   - `選取物理 >= SKILL_REQ_PHYS:ID`
   - `選取魔法 >= SKILL_REQ_MAG:ID`
   - `選取防禦 >= SKILL_REQ_DEF:ID`
   - `選取特殊 >= SKILL_REQ_SPEC:ID`

命中後：`RETURN 技能ID`  
未命中任何技能：`RETURN 0`

#### 普攻保底（Fallback）
- 當 `RESULT == 0`，視為未觸發特殊技能，輸出：
  - `【元素共鳴】未觸發特殊技能。發動基礎攻擊！`

#### 技能命中
- 當 `RESULT > 0`，輸出：
  - `【元素共鳴】條件達成！發動技能 ID：{RESULT}！`

### 3.3 敵方行動階段（Enemy Turn）

當玩家出招結算完成且敵人未死亡時，系統會進入敵方回合。

敵方行動邏輯：掃描 `PARTY_MEMBERS:0~3` 中有效且 `BASE:(該角色):BTL_HP > 0` 的目標，建立可被攻擊名單，並以 `RAND` 隨機選取其中一名進行基礎攻擊。若名單為空，視為隊伍全滅並直接結束戰鬥。

敵方基礎攻擊的傷害公式為：`敵方攻擊力 - 防禦方對應屬性DEF`。木人樁反擊採用物理傷害，實作為 `MAX(0, BASE:敵人:BTL_PATK - BASE:目標:BTL_PDEF)`。

---

## 6. 戰鬥結算與戰報（Battle Resolution）

`@BATTLE_ENTER` 的回傳值採固定契約：`RESULT = 1` 代表本次戰鬥以勝利結束，`RESULT = 0` 代表撤退、敗北或未完成勝利條件即離場。`@BATTLE_PREPARATION` 僅負責前置編組與戰鬥啟動，因此必須將 `@BATTLE_ENTER` 的回傳值原樣向上層流程 `RETURN`，讓基地階段可以用單一介面接收戰果。

戰利品採用戰鬥期間的暫存旗標記錄：

| 變數 | 定義 | 說明 |
|---|---|---|
| `TFLAG:BATTLE_LOOT_MONEY` | 本場戰鬥獲得金錢 | 由戰鬥模組在勝利瞬間寫入。 |
| `TFLAG:BATTLE_LOOT_MAT` | 本場戰鬥獲得材料 | 由戰鬥模組在勝利瞬間寫入。 |

目前木人樁戰鬥的基準掉落為金錢 `100`、材料 `20`。此值屬於戰鬥模組輸出的結算資料，而非基地即時計算結果；基地流程只負責讀取並消費這些暫存值。

日間與晚間流程在 `CALL BATTLE_PREPARATION` 返回後，會根據 `RESULT` 決定是否進入結算發放：

1. 若 `RESULT == 1`，基地流程將 `TFLAG:BATTLE_LOOT_MONEY` 與 `TFLAG:BATTLE_LOOT_MAT` 併入基地資源池，並印出戰報摘要。
2. 若 `RESULT == 0`，基地流程視為撤退或敗北，只印出失利摘要，不發放掉落。
3. 結算完成後，基地流程必須清空 `TFLAG:BATTLE_LOOT_*`，避免前一次戰鬥結果滲入後續流程。

此設計使戰鬥模組只需保證「回傳勝敗 + 暫存掉落」，而基地模組則專責「資源入帳 + 戰報輸出」，兩者邊界穩定且可長期擴充。

## 7. 技能庫與特殊效果

技能庫以 `SKILL_REQ_*` 陣列維護元素配方，並由 `@BATTLE_MATCH_SKILL` 以「供給量大於等於需求量」的宣告式規則映射至技能 ID。技能效果可分為以下三類：

| 類型 | 範例技能 | 結算原則 |
|---|---|---|
| 傷害型 | `20001 重力斬擊` | 依攻擊屬性、元素 Buff 與技能基礎威力扣減目標 `BASE:BTL_HP`。 |
| 恢復型 | `20003 治癒之風` | 依技能 ID 進入恢復分支，直接回復施術者 `BASE:BTL_HP`，且不可超過 `BASE:BTL_MAXHP`。 |
| 增益型 | `20002 絕對防禦`、`20004 堅壁陣型` | 依技能 ID 對回合 Buff 寫入額外加成，例如提升 `TFLAG:TEMP_BUFF_PDEF`。 |

當結算函式收到技能 ID 時，系統會透過 `SELECTCASE ARG` 進入對應效果分支：
- `20003 治癒之風`：執行回復邏輯，將回復量加至 `BASE:MASTER:BTL_HP`，並以 `BTL_MAXHP` 做上限鉗制。
- `20004 堅壁陣型`：為本回合額外寫入 `TFLAG:TEMP_BUFF_PDEF`，使後續敵方回合承受的物理傷害下降。
- 其他技能 / 未命中技能：沿用傷害型結算，對敵方目標做 HP 扣減。

此設計讓技能擴充維持「新增配方資料 + 新增技能 ID 分支」的單向工作流，不需重寫整體戰鬥主循環。

## 8. 自動戰鬥與 AI 邏輯 (Auto-Battle & AI)

玩家可在戰鬥 UI 中啟用 `5`、`10` 或 `20` 回合的自動戰鬥。系統以 `TFLAG:AUTO_TURNS_LEFT` 記錄剩餘回合數：

| 指令 | 行為 |
|---:|---|
| `505` | 啟用自動戰鬥 5 回合。 |
| `510` | 啟用自動戰鬥 10 回合。 |
| `520` | 啟用自動戰鬥 20 回合。 |
| `500` | 提前停止自動戰鬥。 |

當 `TFLAG:AUTO_TURNS_LEFT > 0` 時，戰鬥 UI 會：
1. 顯示 `【自動戰鬥中... 剩餘 X 回合】`。
2. 跳過手動 `INPUT RESULT`。
3. 先呼叫 `@BATTLE_AUTO_PROCESS`。
4. 再以既有 `@BATTLE_PROCESS_ACTION` / `@BATTLE_ENEMY_TURN` 推進回合。
5. 回合成功結束後將 `TFLAG:AUTO_TURNS_LEFT -= 1`。

`@BATTLE_AUTO_PROCESS` 的 AI 思考流程如下：
- 讀取目前出戰者 `PARTY_MEMBERS:0` 的 `CFLAG:CF_AI_W_DAMAGE` 與 `CFLAG:CF_AI_W_GUARD`；若未設定，預設視為 `50 / 50`。
- 檢查自身 HP 比例；當防禦傾向較高，或 HP 低於 50% 時，AI 轉入保守模式。
- 掃描 `HAND0 ~ HAND3` 與 `TEMP_CARD`：
  - 激進模式優先勾選物理 / 魔法 / 特殊元素。
  - 保守模式優先勾選防禦 / 魔法元素，以提高生存與補血技能觸發率。
- 若 Temp 內存在更符合當前策略的元素，且本回合尚未使用暫存槽，AI 會嘗試與不理想手牌交換，以改善本回合可用配方。
- 選牌完成後不直接結算技能，而是將結果寫入 `TFLAG:UI_HANDx_SEL` / `TFLAG:UI_TEMP_SEL`，交由主戰鬥流程統一處理。

此機制使玩家、NPC 與未來腳本事件都能共享同一套「選牌 → 暫存槽 → 技能結算 → 回合推進」管線，只需替換決策來源即可。

## 9. 防呆與清理機制

### 9.1 防呆清單

| 防呆條件 | 觸發位置 | 行為 |
|---|---|---|
| 出招未選元素（總數=0） | `@BATTLE_PROCESS_ACTION` | `PRINTW` 並 `RETURN 0`，不推進回合。 |
| Temp 重複操作 | `@BATTLE_UI_MAIN` 的 450 分支 | 提示並退回 UI。 |
| Temp 操作非單選 | `@BATTLE_UI_MAIN` 的 450 分支 | 提示並退回 UI。 |

### 9.2 清理順序（成功出招）
1. 依勾選狀態清空 `HAND0~HAND3`、`TEMP_CARD` 對應槽位。
2. 若敵人已被擊破則回傳 `1` 結束戰鬥；若敵人存活則回傳 `2` 並進入敵方回合。
3. 僅當敵方行動後戰鬥仍持續時，`@BATTLE_DRAW_CARDS` 才重置：
   - `TFLAG:TEMP_USED_THIS_TURN = 0`
   - `TFLAG:UI_HAND0_SEL~UI_TEMP_SEL = 0`
   - `TFLAG:TEMP_BUFF_PATK~TEMP_BUFF_MDEF = 0`

---

## 10. 當前已實作技能配方（初始化）

初始化來源：`@INIT_SKILL_DATA`

| 技能 ID | 名稱 | 配方需求 |
|---:|---|---|
| 20001 | 重力斬擊 | 物理 2 + 魔法 1 |
| 20002 | 絕對防禦 | 防禦 3 |
| 20003 | 治癒之風 | 魔法 1 + 防禦 1 |
| 20004 | 堅壁陣型 | 防禦 2 |

> 以上需求由 `SKILL_REQ_*` 陣列定義，並由 `@BATTLE_MATCH_SKILL` 做 `>=` 比對。

---

## 11. 傷害公式與結算（Damage Calculation）

單次傷害採用下列基礎公式：

`最終傷害 = MAX(0, (攻擊方對應屬性ATK * (1 + 元素Buff百分比) + 技能基礎威力) - 防禦方對應屬性DEF)`

其中：
- 「攻擊方對應屬性 ATK」依技能型態取物攻或魔攻。
- 「元素 Buff 百分比」由本回合元素堆疊計算後轉為倍率。
- 「技能基礎威力」可由技能 ID 對映；未命中技能時視為 0（普攻保底）。
- 防禦方對應屬性 DEF 依攻擊型態取物防或魔防。

勝敗條件如下：
- 敵人 `BTL_HP <= 0`：玩家勝利。
- 玩家出招後若敵人仍存活，系統進入敵方回合。
- 敵方回合結束後若 `MASTER`（角色 0）的 `BTL_HP <= 0`：玩家敗北。
