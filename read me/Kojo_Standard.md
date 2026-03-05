# Kojo_Standard.md
> 目的：提供 NPC 專屬口上（KOJO）的編寫規範與主程式調用流程  
> 適用：基地晨間結算、房間槽入住、互動結果、戰鬥抽牌與映射技能等事件  
> 注意：本文件只規範系統與分支，不包含任何露骨內容

口上的定位是「回饋與辨識度」，讓玩家在同一套系統事件下感覺到 NPC 的差異。主程式應把事件轉成統一的事件 ID 與參數，再交給 NPC 口上處理，避免到處散落 `PRINT`。口上作者只要會寫分支，就能針對好感、心情、疲勞、淫慾度等狀態切換內容。戰鬥與基地的觸發點要穩定且可除錯，避免同一回合刷屏。

---

## 行動建議（先照做就能穩定跑）
- **建立事件字典**：先把 KOJO 事件 ID 與參數定義好，再開始寫台詞。
- **統一入口函數**：主程式只呼叫 `@KOJO_MESSAGE`，口上只處理 EVT，不直接依賴外部流程。
- **加上防重入**：用 `TFLAG:KOJO_LAST_EVT`、`TFLAG:KOJO_LAST_TURN` 避免重複觸發刷屏。
- **把分支集中在狀態段位**：好感段位、淫慾段位、疲勞段位先算出來，台詞分支會更乾淨。
- **規範 CSTR 用途**：只存稱呼與短記憶標籤，不要塞大段文本。
- **提供預設口上**：每個事件都要有 default fallback，避免漏寫導致沉默或錯誤。
- **留 debug 開關**：用系統按鈕區間放「口上測試器」，方便作者自測。

---

## 1. 觸發點定義（Event Triggers）
### 1.1 KOJO 事件 ID 區間
| 區間 | 類別 | 說明 |
|---:|---|---|
| 0-99 | 系統通用 | 啟動、返回、測試、通用提示 |
| 100-199 | 基地與住宿 | 晨間問候、入住、撤離、維護費結算 |
| 200-299 | 互動結果 | 日常互動、成人向互動的成功或失敗結果 |
| 300-399 | 戰鬥流程 | 抽牌、命中關鍵牌、映射技能成功、Temp 存取、策略切換、戰鬥結束 |
| 900-999 | Debug | 強制觸發、列印參數、覆蓋狀態 |

### 1.2 常見觸發時機表
| EVT 名稱 | EVT_ID | 觸發點（建議插入位置） | 參數 P0/P1（建議） | 頻率建議 |
|---|---:|---|---|---|
| MORNING_GREETING | 110 | `@BASE_MORNING_TICK` 對每位住宿 NPC | P0=DAY, P1=0 | 每日每人一次 |
| ROOM_MOVE_IN | 120 | `@SYS_ROOM_MOVE_IN` 完成槽位寫入後 | P0=ROOM_SLOT, P1=0 | 事件型 |
| ROOM_MOVE_OUT | 121 | `@SYS_ROOM_MOVE_OUT` 釋放槽位後 | P0=OLD_SLOT, P1=REASON | 事件型 |
| INTERACT_RESULT | 210 | 互動指令結算後 | P0=CMD_ID, P1=RESULT(0/1/2) | 每次互動 |
| BATTLE_TURN_DRAW | 310 | 回合抽滿 4 張牌後 | P0=TURN, P1=0 | 每回合一次 |
| BATTLE_KEYCARD_HIT | 311 | 抽牌後檢查到關鍵牌 | P0=CARD_ID, P1=SLOT(0..3) | 有命中才觸發 |
| BATTLE_MAP_SUCCESS | 320 | 映射技能確定後，結算前或結算後 | P0=SKILL_ID, P1=COMBO_CODE | 有技能才觸發 |
| BATTLE_TEMP_STORE | 330 | Temp 存牌成功 | P0=CARD_ID, P1=FROM_SLOT | 存取成功才觸發 |
| BATTLE_TEMP_RETRIEVE | 331 | Temp 取牌或交換成功 | P0=CARD_ID, P1=TO_SLOT | 存取成功才觸發 |
| BATTLE_STRATEGY_SWITCH | 340 | 策略被手動或自動切換 | P0=OLD, P1=NEW | 有切換才觸發 |
| BATTLE_END | 350 | 戰鬥結束，回傳前 | P0=WINLOSE, P1=TURNS | 每場一次 |

> COMBO_CODE 建議用排序後的卡牌類別字串或 hash，例如 `A+B`、`D+D`，方便除錯。

---

## 2. 條件分支範例（依 CFLAG 切換內容）
### 2.1 段位切換建議
- 好感段位（BOND_LV）：
  - 0-2：疏離
  - 3-6：普通
  - 7+：親密
- 淫慾度（CFLAG 建議新增或同步）：
  - 0-30：低
  - 31-70：中
  - 71+：高
- 疲勞（FATIGUE）：
  - 0-40：良好
  - 41-80：疲憊
  - 81+：過勞

> 淫慾度來源建議：若已用 PALAM 的「興奮（或淫欲）」跑互動系統，可在互動結算或晚間清算時把它同步進 `CFLAG:CHARA:LUST`，口上就能用同一把尺做分支。

### 2.2 口上分支代碼示例（示意）
~~~erb
; =========================
; NPC 個別口上入口（示意）
; @KOJO_0010: NPC ID 10
; ARG:0 = EVT, ARG:1 = P0, ARG:2 = P1
; =========================
@KOJO_0010
#DIM EVT
#DIM P0
#DIM P1
EVT = ARG:0
P0  = ARG:1
P1  = ARG:2

; 段位整理，避免到處寫比較式
#DIM BOND_TIER
#DIM LUST_TIER
BOND_TIER = (CFLAG:TARGET:BOND_LV >= 7) ? 2 : ((CFLAG:TARGET:BOND_LV >= 3) ? 1 : 0)
LUST_TIER = (CFLAG:TARGET:LUST >= 71) ? 2 : ((CFLAG:TARGET:LUST >= 31) ? 1 : 0)

SELECTCASE EVT
    CASE 110 ; MORNING_GREETING
        IF BOND_TIER == 2
            PRINTFORM {CSTR:TARGET:CALL_PLAYER}，早安。今天也一起加油吧。
        ELSEIF BOND_TIER == 1
            PRINTFORM 早。今天要做什麼安排？
        ELSE
            PRINTFORM ……早。
        ENDIF

    CASE 320 ; BATTLE_MAP_SUCCESS
        ; P0=SKILL_ID, P1=COMBO_CODE
        IF LUST_TIER == 2 && BOND_TIER >= 1
            PRINTFORM 好，節奏抓到了。{P1} 這一手很漂亮。
        ELSE
            PRINTFORM {P1}，就用這招收尾。
        ENDIF

    CASE 330 ; BATTLE_TEMP_STORE
        ; P0=CARD_ID, P1=FROM_SLOT
        IF BOND_TIER >= 1
            PRINTFORM 先留著，下一回合再用。
        ELSE
            PRINTFORM ……保留。
        ENDIF

    CASEELSE
        ; 沒寫到的事件至少要安靜返回，避免報錯或刷屏
ENDSELECT

RETURN
~~~

### 2.3 用心情與疲勞控制語氣（示意）
~~~erb
@KOJO_TONE_FILTER
; 輸入：TARGET 既定
; 輸出：TFLAG:KOJO_TONE (0 正常 1 煩躁 2 低落)
IF CFLAG:TARGET:FATIGUE >= 81
    TFLAG:KOJO_TONE = 2
ELSEIF CFLAG:TARGET:MOOD <= 30
    TFLAG:KOJO_TONE = 1
ELSE
    TFLAG:KOJO_TONE = 0
ENDIF
RETURN
~~~

---

## 3. 自訂變數規範（CSTR：稱呼與記憶）
### 3.1 CSTR 使用原則
- 只存「短字串」：稱呼、稱謂、簡短標籤。
- 不存「長文本」：劇情段落應寫在 ERB 的台詞區，CSTR 只拿來選分支。
- 不把 CSTR 當資料庫：需要計數與判定時，改用 CFLAG 或 FLAG。

### 3.2 建議保留鍵（作者可用）
| Key | 用途 | 範例 |
|---|---|---|
| CSTR:CHARA:CALL_PLAYER | NPC 對玩家稱呼 | "主人"、"隊長"、"你" |
| CSTR:CHARA:CALL_SELF | NPC 自稱 | "我"、"本小姐" |
| CSTR:CHARA:ROOM_LABEL | 房間標籤顯示 | "A-03" |
| CSTR:CHARA:MEM_TAG1 | 記憶標籤 1 | "PROMISE" |
| CSTR:CHARA:MEM_TAG2 | 記憶標籤 2 | "RIVAL" |
| CSTR:CHARA:NOTE | 作者備註 | "偏毒舌" |

### 3.3 初始化與更新規範
- 初始化：建議在 NPC 初次加入或初次入住時設好 `CALL_PLAYER`、`CALL_SELF`。
- 更新：只允許透過封裝函數更新，避免不同檔案寫法不一致。

~~~erb
@SYS_KOJO_INIT
; 初次建立 NPC 的稱呼
IF CSTR:TARGET:CALL_PLAYER == ""
    CSTR:TARGET:CALL_PLAYER = "你"
ENDIF
IF CSTR:TARGET:CALL_SELF == ""
    CSTR:TARGET:CALL_SELF = "我"
ENDIF
RETURN
~~~

---

## 4. 調用流程（主程式如何呼叫 @KOJO_MESSAGE）
### 4.1 呼叫點插入建議
- 基地晨間：`@BASE_MORNING_TICK` 內對每位住宿 NPC 呼叫 `MORNING_GREETING`。
- 入住撤離：`@SYS_ROOM_MOVE_IN / @SYS_ROOM_MOVE_OUT` 的最後呼叫對應事件。
- 互動結果：互動指令結算後呼叫 `INTERACT_RESULT`，P0 傳 CMD_ID，P1 傳結果碼。
- 戰鬥回合：抽牌後呼叫 `BATTLE_TURN_DRAW`，檢查關鍵牌命中再呼叫 `BATTLE_KEYCARD_HIT`。
- 映射成功：確定技能後呼叫 `BATTLE_MAP_SUCCESS`。
- Temp 存取：成功後呼叫 `BATTLE_TEMP_STORE / RETRIEVE`。

### 4.2 統一入口：@KOJO_MESSAGE（示意）
~~~erb
; =========================
; @KOJO_MESSAGE
; ARG:0 = CHARA_ID
; ARG:1 = EVT
; ARG:2 = P0
; ARG:3 = P1
; =========================
@KOJO_MESSAGE
#DIM CH
#DIM EVT
#DIM P0
#DIM P1
CH  = ARG:0
EVT = ARG:1
P0  = ARG:2
P1  = ARG:3

; 防重入：同一回合同一事件只允許一次
IF TFLAG:KOJO_LAST_EVT == EVT && TFLAG:KOJO_LAST_TURN == TFLAG:BATTLE_TURN
    RETURN
ENDIF
TFLAG:KOJO_LAST_EVT = EVT
TFLAG:KOJO_LAST_TURN = TFLAG:BATTLE_TURN

; 切換 TARGET，讓口上用 TARGET 取值
TARGET = CH

; NPC 分派
SELECTCASE CH
    CASE 10
        CALL @KOJO_0010, EVT, P0, P1
    CASE 11
        CALL @KOJO_0011, EVT, P0, P1
    CASEELSE
        CALL @KOJO_DEFAULT, EVT, P0, P1
ENDSELECT

RETURN
~~~

### 4.3 預設口上：@KOJO_DEFAULT（示意）
~~~erb
@KOJO_DEFAULT
#DIM EVT
EVT = ARG:0

; 只做很短的通用回饋，避免搶走 NPC 個性
SELECTCASE EVT
    CASE 110
        PRINTFORM 早安。
    CASE 320
        PRINTFORM 好，就這樣做。
    CASE 350
        PRINTFORM 回來了。
    CASEELSE
ENDSELECT

RETURN
~~~

---

## 5. 關鍵牌定義（抽到關鍵牌的判定規範）
### 5.1 關鍵牌來源建議
- 方案 A：每個 NPC 設定 1 到 3 張關鍵牌 ID
  - `CFLAG:CHARA:KOJO_KEYCARD1..3`
- 方案 B：每個 NPC 設定偏好類別或 Tag
  - 例如偏好重擊、偏好防禦，抽到該類就觸發

### 5.2 判定與參數
- 在抽牌填滿 4 格後，檢查 `HAND0..HAND3`，有命中就觸發 `BATTLE_KEYCARD_HIT`。
- P0 傳卡牌 ID，P1 傳命中的槽位索引。

---

## 追蹤指標（口上品質與手感）
- 每回合口上行數：是否超過 2 行
- 同一事件觸發率：是否過度頻繁
- 口上覆蓋率：每個 EVT 是否都有 NPC 版本或 default
- 分支命中率：高好感與高淫慾段位是否有明顯差異

---