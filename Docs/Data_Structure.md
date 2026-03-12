# Data_Structure.md
> 目的：把「基地經營」與「抽牌戰鬥」需要的資料欄位一次定義清楚，方便直接寫入 ERB  
> 原則：永久保存用 CFLAG/CSTR 或 SAVEDATA，單場戰鬥流程用 TFLAG，角色戰鬥資源用 BASE

---

## 0. ID 區間建議（避免撞號）
| 類型 | 建議區間 | 說明 |
|---|---:|---|
| 卡牌（ITEM.csv） | 10000-19999 | 牌組抽到的「卡」本體 |
| 映射技能（Skill ID） | 20000-29999 | A+B 這類配方輸出的技能 |
| 一般道具（非卡） | 30000-39999 | 工坊產物、禮物、素材等 |
| BASE 自訂參數（BASE.csv） | 400-499 | 戰鬥用 HP、能量、暫存槽狀態等 |

> 若未來專案擴充需其他區間，以現有規範為準，這只是防撞建議。

---

## 1. NPC：CFLAG / CSTR 定義（角色檔案長期保存）
### 1.1 CFLAG（數值型）
> 用途：住宿位置、戰鬥傾向、養成進度（可用於解鎖、效率加成、事件門檻）

#### A. 住宿與房間槽（Room Slot）
| Key（建議命名） | 型態 | 範圍 | 說明 | 設定時機 |
|---|---|---|---|---|
| CFLAG:CHARA:IS_RESIDENT | SAVEDATA | 0/1 | 是否住宿 | 入住=1，撤離=0 |
| CFLAG:CHARA:ROOM_SLOT | SAVEDATA | 0..ROOM_CAP-1 或 9999 | 住宿槽位索引，無則用 9999 | 入住時寫入，撤離清空 |
| CFLAG:CHARA:MOVEOUT_COOLDOWN | SAVEDATA | 0.. | 撤離冷卻天數，避免反覆入住刷效果 | 撤離時設定，晨間遞減 |
| CFLAG:CHARA:MAINT_COST_MOD | SAVEDATA | -50..+200 | 該 NPC 的維護費修正百分比 | 由特性或事件改變 |

> 若你希望「空槽」也能用 -1，請確認你那份 BASE/FLAG 的 min 值允許負數。這份規劃預設用 9999 作為空值，最穩。

#### B. 戰鬥傾向（AI Profile）
| Key（建議命名） | 型態 | 範圍 | 說明 | 用途 |
|---|---|---|---|---|
| CFLAG:CHARA:BATTLE_AI_LOCK | SAVEDATA | 0/1 | 是否鎖定策略（玩家手動鎖定） | 1 時忽略自動切換 |
| CFLAG:CHARA:BATTLE_AI_DEFAULT | SAVEDATA | 0..3 | 預設策略：0 均衡，1 進攻，2 防禦，3 節省 | 戰鬥開始初始化策略 |
| CFLAG:CHARA:AI_W_DAMAGE | SAVEDATA | 0..300 | 傷害權重（評分器用） | 進攻傾向更高 |
| CFLAG:CHARA:AI_W_GUARD | SAVEDATA | 0..300 | 防禦權重 | 防禦傾向更高 |
| CFLAG:CHARA:AI_W_RESOURCE | SAVEDATA | 0..300 | 資源權重 | 節省傾向更高 |
| CFLAG:CHARA:AI_RISK_TOL | SAVEDATA | 0..300 | 風險容忍（越高越敢賭命中） | 高爆發流 |

> 實作方式：戰鬥開始時，把 `CFLAG:CHARA:BATTLE_AI_DEFAULT` 寫入 `TFLAG:BATTLE_STRATEGY`。若 `BATTLE_AI_LOCK=1`，就不跑自動切換，只用這個策略到戰鬥結束。

#### C. 養成進度（Training Progress）
| Key（建議命名） | 型態 | 範圍 | 說明 | 連動系統 |
|---|---|---|---|---|
| CFLAG:CHARA:GROWTH_STAGE | SAVEDATA | 0..10 | 養成階段或段位，做解鎖門檻 | 訓練室、休息室事件 |
| CFLAG:CHARA:TRAIN_EXP | SAVEDATA | 0..999999 | 累積訓練經驗值 | 訓練室指令累積 |
| CFLAG:CHARA:TRAIN_RANK_STR | SAVEDATA | 0..5 | 力量系訓練段位 | 訓練室加成 |
| CFLAG:CHARA:TRAIN_RANK_DEX | SAVEDATA | 0..5 | 敏捷系訓練段位 | 訓練室加成 |
| CFLAG:CHARA:WORK_SPEC | SAVEDATA | 0..5 | 工坊專精段位（製作加成或解鎖） | 工坊製作與配方 |
| CFLAG:CHARA:BOND_LV | SAVEDATA | 0..10 | 休息室好感段位 | 互動解鎖與事件權重 |
| CFLAG:CHARA:TRAIN_MOD | SAVEDATA | 50..200 | 訓練倍率百分比（100=正常） | 訓練室成長公式 |
| CFLAG:CHARA:MOOD | SAVEDATA | 0..100 | 心情 | 休息室與事件結果 |
| CFLAG:CHARA:FATIGUE | SAVEDATA | 0..100 | 疲勞 | 晚間清算與訓練成本 |

> `TRAIN_MOD / MOOD / FATIGUE` 建議直接參照目前的基地養成框架沿用，避免同一個概念出現兩套欄位。

---

### 1.2 CSTR（字串型）
> 用途：除錯、UI 顯示、可讀性，不參與核心運算也沒關係

| Key（建議命名） | 型態 | 範例 | 說明 |
|---|---|---|---|
| CSTR:CHARA:ROOM_LABEL | SAVEDATA | "A-03" | 房間標籤，顯示用 |
| CSTR:CHARA:AI_PROFILE_NAME | SAVEDATA | "Aggressive" | AI 配置名稱，顯示與除錯 |
| CSTR:CHARA:GROWTH_ROUTE | SAVEDATA | "STR_FOCUS" | 養成路線備註，用於 UI 或事件分支 |
| CSTR:CHARA:NOTE | SAVEDATA | "喜歡重擊" | 設計備註或玩家筆記 |

---

## 2. ITEM.csv 規劃：卡牌定義為道具 ID，映射技能用 Skill ID
> ⚠️ **Warning**：注意：受限於 Emuera 引擎原生設計，Item.csv 僅能儲存 ID 與名稱。卡牌的戰鬥數值（能量、攻擊力等）絕對不可依賴 CSV，必須統一透過 SYSTEM.ERB 中的 @INIT_CARD_DATA 初始化 CARD_ENERGY_COST 等全域陣列來管理與調用。

### 2.1 欄位建議（最小可用版）
| 欄位 | 型態 | 必填 | 說明 |
|---|---|---|---|
| ITEM_ID | int | 是 | 卡牌 Item ID |
| NAME | str | 是 | 顯示名稱 |
| CATEGORY | str | 是 | 固定填 `CARD` |
| SUBTYPE | str | 是 | A/B/D/S/P 等主類別 |
| TAG1 | str | 否 | 例如 ATK、DEF、SUP |
| TAG2 | str | 否 | 例如 HEAVY、COUNTER |
| RARITY | int | 否 | 0 普通，1 稀有，2 史詩 |
| ENERGY_COST | int | 否 | 單卡行動或技能參考消耗 |
| BASE_POWER | int | 否 | 供評分器與顯示用 |
| BASE_ACC | int | 否 | 供評分器與顯示用 |
| SOLO_SKILL_ID | int | 否 | 若允許單卡也能出招，填技能 ID，否則填 0 |
| SCRIPT_HOOK | str | 否 | ERB 段落鉤子，例如 `CARD_10001` |
| UNLOCK_FLAG | int | 否 | 解鎖旗標或工坊配方門檻 |
| DESC | str | 否 | 描述 |

> 映射技能本體建議走「Skill ID」，由你的映射表（建議另做 CARD_MAP.csv）輸出 Skill ID，再由 ERB 去執行對應技能段落。

### 2.2 範例資料（示意）
| ITEM_ID | NAME | CATEGORY | SUBTYPE | TAG1 | TAG2 | ENERGY_COST | BASE_POWER | BASE_ACC | SOLO_SKILL_ID | SCRIPT_HOOK |
|---:|---|---|---|---|---|---:|---:|---:|---:|---|
| 10001 | 斬擊 | CARD | A | ATK |  | 1 | 10 | 80 | 20001 | CARD_10001 |
| 10002 | 重擊 | CARD | B | ATK | HEAVY | 2 | 18 | 65 | 0 | CARD_10002 |
| 10003 | 格擋 | CARD | D | DEF |  | 1 | 0 | 100 | 20003 | CARD_10003 |
| 10004 | 破甲 | CARD | S | TECH | ARMORBREAK | 2 | 8 | 75 | 20004 | CARD_10004 |
| 10005 | 專注 | CARD | P | SUP | FOCUS | 1 | 0 | 100 | 20005 | CARD_10005 |

---

## 3. BASE.csv 規劃：戰鬥 HP、能量、暫存槽狀態
> BASE 是「每個戰鬥角色」的參數欄位。若戰鬥只有玩家側使用 Temp Slot，也可以只對玩家角色初始化與更新這些欄位。

### 3.1 欄位建議（BASE.csv 常見結構）
| 欄位 | 型態 | 說明 |
|---|---|---|
| BASE_ID | int | 參數索引 |
| NAME | str | 顯示名稱 |
| MIN | int | 最小值 |
| MAX | int | 最大值 |
| DEFAULT | int | 預設值 |
| NOTE | str | 實作備註 |

### 3.2 參數分配表（建議）
| BASE_ID | NAME | MIN | MAX | DEFAULT | NOTE |
|---:|---|---:|---:|---:|---|
| 400 | BTL_HP | 0 | 999999 | 100 | 戰鬥中 HP（可作為持久傷害或單場結算） |
| 401 | BTL_MAXHP | 1 | 999999 | 100 | HP 上限（由養成或裝備影響） |
| 402 | BTL_EN | 0 | 999999 | 10 | 能量或資源（技能消耗用） |
| 403 | BTL_MAXEN | 0 | 999999 | 10 | 能量上限 |
| 404 | BTL_TEMP_CARD | 0 | 19999 | 0 | 暫存槽卡牌 Item ID，0 代表空 |
| 405 | BTL_TEMP_USED | 0 | 1 | 0 | 本回合是否已用 Temp 存取（回合開始重置） |
| 406 | BTL_STRATEGY | 0 | 3 | 0 | 本場策略（可由 CFLAG 初始化） |

---

## 4. 戰鬥流程用的 TFLAG（給你寫 ERB 時直接對照）
> 這些是單場戰鬥的流程狀態，建議在 `@BATTLE_ENTER` 初始化，戰鬥結束後清掉或讓它自然被覆寫。

| 變數 | 用途 | 建議初始值 |
|---|---|---|
| TFLAG:BATTLE_TURN | 回合數 | 1 |
| TFLAG:BATTLE_STRATEGY | 策略 ID（0..3） | 由 `CFLAG:CHARA:BATTLE_AI_DEFAULT` 或玩家選擇 |
| TFLAG:HAND0..HAND3 | 手牌 4 格 Item ID（空用 -1 或 0） | 抽牌後填滿 |
| TFLAG:TEMP_CARD | 暫存槽 Item ID（空用 -1 或 0） | 0 |
| TFLAG:TEMP_USED_THIS_TURN | 本回合是否已使用 Temp 存取 | 0 |
| TFLAG:DECK_SIZE / DISCARD_SIZE | 牌堆計數 | 依牌組配置 |

---