# Battle_System.md

> 系統：雙軸卡牌編織回合戰鬥（Attribute + Shape）
>
> 入口：`CALL BATTLE_PREPARATION`（於 `ERB/Battle/BATTLE_MAIN.ERB`）
>
> 核心：
> 1) 以雙軸卡牌槽位（非道具卡）組成回合資源；
> 2) 玩家透過 UI 勾選控制「本回合要消耗的卡牌」；
> 3) 先用 Attribute 計算 Buff，再用 Shape 進行技能配方映射，最後清空已選卡牌。

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

### 1.1 卡牌雙軸機制（Shape + Attribute）

每張戰鬥卡牌同時具備兩個維度：

1. **Shape（形狀）**：只負責技能配方判定。
   - `SHAPE_PHYS`（物理）
   - `SHAPE_MAG`（魔法）
   - `SHAPE_DEF`（防禦）
   - `SHAPE_SPEC`（特殊）
2. **Attribute（屬性）**：只負責當回合 Buff 計算。
   - `ATTR_FIRE`（火）→ `TEMP_BUFF_PATK`
   - `ATTR_WATER`（水）→ `TEMP_BUFF_MATK`
   - `ATTR_WIND`（風）→ `TEMP_BUFF_PDEF`
   - `ATTR_EARTH`（土）→ `TEMP_BUFF_MDEF`

> 職責分離原則：**Shape 決定能不能放技能，Attribute 決定放技能時拿到哪些能力加成**。

### 1.2 卡牌編碼與解碼（TFLAG）

`HAND0~HAND3` 與 `TEMP_CARD` 使用整數打包：

`CARD = ATTRIBUTE * 10 + SHAPE`

- `0`：空槽。
- `11~44`：有效雙軸卡牌。
- 解碼方式：
  - `ATTRIBUTE = CARD / 10`
  - `SHAPE = CARD - ATTRIBUTE*10`

UI 展示格式為：`[屬性·形狀]`，例如 `[火·物理]`、`[水·防禦]`。

### 1.2 回合槽位模型

| 類型 | 變數 | 說明 |
|---|---|---|
| 手牌 H0~H3 | `TFLAG:HAND0` ~ `TFLAG:HAND3` | 當回合主要卡牌池（4 槽，值為 `ATTR*10+SHAPE`）。 |
| 暫存槽 Temp | `TFLAG:TEMP_CARD` | 可跨流程操作的額外 1 槽（值為 `ATTR*10+SHAPE`）。 |
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

#### 規則 B：必須「精確選取 1 個手牌卡牌」
- 系統只統計 H0~H3 中「非空且被勾選」的槽位。
- 若選取數量 `!= 1`，拒絕並提示：
  - `必須精確選取 1 個手牌元素才能進行暫存操作。`

#### 規則 C：存入 / 交換
- **Temp 為空 (`TFLAG:TEMP_CARD == 0`)**：
  - 把被選手牌卡牌移入 Temp。
  - 原手牌槽位清空為 `0`。
- **Temp 非空**：
  - 與被選手牌卡牌做雙向交換。

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

### 3.1 階段一：Attribute 堆疊加成（Stat Boosting）

`@BATTLE_PROCESS_ACTION` 會先統計被選卡牌數量：
- Shape 計數（技能配方用）：
  - `LOCAL:50` = 物理 Shape
  - `LOCAL:51` = 魔法 Shape
  - `LOCAL:52` = 防禦 Shape
  - `LOCAL:53` = 特殊 Shape
- Attribute 計數（Buff 用）：
  - `LOCAL:55` = 火 Attribute
  - `LOCAL:56` = 水 Attribute
  - `LOCAL:57` = 風 Attribute
  - `LOCAL:58` = 土 Attribute
- `LOCAL:54` = 總選取數

若 `LOCAL:54 == 0`：
- `PRINTW 請至少選取一張卡牌來發動攻擊！`
- `RETURN 0`

接著呼叫 `@BATTLE_CALC_BOOST(火, 水, 風, 土)`，計算公式：

| Buff 變數 | 公式 | 說明 |
|---|---|---|
| `TFLAG:TEMP_BUFF_PATK` | `火 * 10` | 每 1 張火屬性卡給 +10% 臨時物攻。 |
| `TFLAG:TEMP_BUFF_MATK` | `水 * 10` | 每 1 張水屬性卡給 +10% 臨時魔攻。 |
| `TFLAG:TEMP_BUFF_PDEF` | `風 * 10` | 每 1 張風屬性卡給 +10% 臨時物防。 |
| `TFLAG:TEMP_BUFF_MDEF` | `土 * 10` | 每 1 張土屬性卡給 +10% 臨時魔防。 |

> 註：目前 Buff 先存入 `TFLAG:TEMP_BUFF_*`，作為後續傷害公式接入點；`@BATTLE_DRAW_CARDS` 會在下一回合開始將這些值重置為 0。

#### 計算範例
若本回合選取 Attribute：火 2、水 1、風 0、土 1，則
- `TEMP_BUFF_PATK = 20`
- `TEMP_BUFF_MATK = 10`
- `TEMP_BUFF_PDEF = 0`
- `TEMP_BUFF_MDEF = 10`

### 3.2 階段二：Shape 技能映射比對（Skill Resolution）

呼叫：`CALL BATTLE_MATCH_SKILL, LOCAL:50, LOCAL:51, LOCAL:52, LOCAL:53`

比對範圍：`FOR LOCAL, 20000, 20010`

比對條件（全部成立才命中）：
1. 技能需求總和 > 0
   `SKILL_REQ_SHAPE_PHYS:ID + SKILL_REQ_SHAPE_MAG:ID + SKILL_REQ_SHAPE_DEF:ID + SKILL_REQ_SHAPE_SPEC:ID > 0`
2. 玩家 Shape 供給皆達標（`>=` 比對）
   - `選取物理 Shape >= SKILL_REQ_SHAPE_PHYS:ID`
   - `選取魔法 Shape >= SKILL_REQ_SHAPE_MAG:ID`
   - `選取防禦 Shape >= SKILL_REQ_SHAPE_DEF:ID`
   - `選取特殊 Shape >= SKILL_REQ_SHAPE_SPEC:ID`

命中後：`RETURN 技能ID`
未命中任何技能：`RETURN 0`

#### 普攻保底（Fallback）
- 當 `RESULT == 0`，視為未觸發特殊技能，輸出：
  - `【形狀連鎖】未觸發特殊技能。發動基礎攻擊！`

#### 技能命中
- 當 `RESULT > 0`，輸出：
  - `【形狀連鎖】條件達成！發動技能 ID：{RESULT}！`

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

## 7. 防呆與清理機制

### 7.1 防呆清單

| 防呆條件 | 觸發位置 | 行為 |
|---|---|---|
| 出招未選元素（總數=0） | `@BATTLE_PROCESS_ACTION` | `PRINTW` 並 `RETURN 0`，不推進回合。 |
| Temp 重複操作 | `@BATTLE_UI_MAIN` 的 450 分支 | 提示並退回 UI。 |
| Temp 操作非單選 | `@BATTLE_UI_MAIN` 的 450 分支 | 提示並退回 UI。 |

### 7.2 清理順序（成功出招）
1. 依勾選狀態清空 `HAND0~HAND3`、`TEMP_CARD` 對應槽位。
2. 若敵人已被擊破則回傳 `1` 結束戰鬥；若敵人存活則回傳 `2` 並進入敵方回合。
3. 僅當敵方行動後戰鬥仍持續時，`@BATTLE_DRAW_CARDS` 才重置：
   - `TFLAG:TEMP_USED_THIS_TURN = 0`
   - `TFLAG:UI_HAND0_SEL~UI_TEMP_SEL = 0`
   - `TFLAG:TEMP_BUFF_PATK~TEMP_BUFF_MDEF = 0`

---

## 8. 當前已實作技能配方（初始化）

初始化來源：`@INIT_SKILL_DATA`

| 技能 ID | 名稱 | 配方需求 |
|---:|---|---|
| 20001 | 重力斬擊 | 物理 2 + 魔法 1 |
| 20002 | 絕對防禦 | 防禦 3 |

> 以上需求由 `SKILL_REQ_SHAPE_*` 陣列定義，並由 `@BATTLE_MATCH_SKILL` 做 `>=` 比對。

## 9. 開發者注意事項（雙軸擴充）

1. 新增屬性時，只能影響 `@BATTLE_CALC_BOOST` 與 UI 名稱映射；不得動到 `@BATTLE_MATCH_SKILL` 的 Shape 配方語意。
2. 新增形狀時，必須同步擴充 `SKILL_REQ_SHAPE_*`、`@BATTLE_MATCH_SKILL`、UI 形狀文字映射。
3. 若調整編碼基底（`CARD_PACK_BASE`），需同步檢查所有 `CARD / 10` 與 `CARD - ATTR*10` 的解碼邏輯。
4. `UI_HAND_SEL` 與 `TEMP` 交換邏輯只搬運整數卡牌編碼，屬性/形狀解析應維持在出招與顯示階段，避免 Temp 功能與雙軸耦合。

---

## 10. 傷害公式與結算（Damage Calculation）

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

---

## 11. 進階戰鬥機制 (Advanced Battle Mechanics)

> 本章節定義戰鬥層擴充機制，目標是在不破壞現行「屬性（Attribute）＋形狀（Shape）」雙軸核心的前提下，提升玩家回合決策密度與戰鬥節奏層次。

### A. 敵方意圖預警機制（Enemy Intent System）

#### A-1. 系統定義
在實戰中（不含木人樁、測試沙包或教學模擬戰），敵方於玩家回合期間公開下一回合的行動意圖（Intent）與預估傷害值（Forecast Damage）。

- **意圖資訊最小集合**：
  1. 行為類型（單體攻擊 / 群體攻擊 / 防禦 / 強化 / 特殊）
  2. 目標範圍（單體、隨機、全體）
  3. 預估傷害數值（或區間）
- **顯示時機**：玩家回合 UI 可操作期間常駐顯示，並在敵方行動後刷新下一筆意圖。
- **顯示例外**：戰鬥標記為 `TRAINING` / `DUMMY` 類型時，預警 UI 關閉，避免干擾基礎測試。

#### A-2. 觸發時機與流程掛點
- **回合起始**：進入玩家操作階段前，先計算並快取敵方下一筆意圖。
- **玩家決策中**：UI 持續可見，供玩家在勾選卡牌時動態調整輸出/防禦比。
- **敵方回合結束**：重新生成下一筆意圖，供下個玩家回合使用。

流程概念：
1. `Enemy Intent Build`（敵人 AI 先決策）
2. `Intent UI Render`（玩家可見）
3. `Player Action`（雙軸卡牌出招）
4. `Enemy Action Resolve`
5. `Next Intent Build`

#### A-3. 玩家決策誘因（為何好玩）
- 預警讓「防禦選擇」由被動補救改為主動規劃：玩家看見高額致死預告時，必須犧牲部分輸出牌。
- 玩家需在「現在最大輸出」與「下一回合生存」間取捨，降低無腦全梭哈最優解。
- 預警資訊可形成心理博弈：低風險回合偏進攻，高風險回合轉防守或蓄力。

#### A-4. 與雙軸卡牌系統連動
- **Shape 連動**：玩家可優先打出 `SHAPE_DEF`（防禦形狀）觸發防禦向技能配方（如護盾、格擋、減傷姿態）。
- **Attribute 連動**：玩家可依敵方預告傷害型態，選擇提供 `TEMP_BUFF_PDEF` / `TEMP_BUFF_MDEF` 的屬性組合，提前建立減傷 Buff。
- **策略結果**：輸出 Shape 與防禦 Shape 的配比，不再僅受當回合斬殺線影響，也受「已知下一回合風險」驅動。

### B. EN 保留生息與極限突破（EN Accumulation & Limit Break）

#### B-1. 系統定義
玩家每回合未打出（放棄使用）的可用卡牌，於回合結算時轉化為 EN（Energy）資源。EN 為戰鬥內個體資源，按參戰角色分開記錄與消耗。

- **資源對應**：每名參戰者使用自身 `BASE:BTL_EN` 累積。
- **累積來源**：該角色當回合可出但未出的卡牌數（含策略性保留）。
- **戰鬥邊界**：EN 僅於單場戰鬥內生效；戰鬥結束後依規格重置（預設歸零）。

#### B-2. 觸發時機與流程掛點
- **回合結算階段**：在玩家出招解析完成後，統計未使用卡牌並轉換為 EN。
- **可觸發條件**：當 `BASE:BTL_EN` 達到角色技能門檻值（Threshold）時，該角色獲得「可爆發」狀態。
- **觸發主體**：由玩家手動決定是否於當前回合啟動該角色的極限突破（不可強制自動觸發）。
- **消耗規範**：成功啟動後扣除對應 EN（可為一次性清空或固定成本，依技能設計表）。

#### B-3. 玩家決策誘因（為何好玩）
- 放棄當回合部分輸出可換取後續回合更高上限，形成「短期損失換長期收益」決策。
- 與敵方預警機制配合時，玩家可在高壓回合偏防守蓄 EN，於安全窗口或破防窗口一次性爆發。
- 玩家可自行掌握爆發時點，戰鬥節奏由「固定線性輸出」轉為「蓄力 -> 轉折 -> 傾瀉」的波段節奏。

#### B-4. 與雙軸卡牌系統連動
- **與 Shape 的關係**：
  - 保留某些高價值 Shape 卡不立即打出，可在爆發回合集中滿足高階技能配方。
  - 極限突破技能可定義為「Shape 條件放寬」或「特定 Shape 倍率強化」。
- **與 Attribute 的關係**：
  - 蓄力回合可優先堆疊防禦向 Attribute Buff 保命。
  - 爆發回合再改堆攻擊向 Attribute Buff，最大化突破技能收益（如傷害加倍、無視防禦）。
- **多角色資源分離**：
  - 因 EN 為角色獨立池，玩家需安排隊伍中「誰防守蓄力、誰先爆發」，強化隊伍編成與輪轉價值。
