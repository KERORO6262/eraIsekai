# 開發者選項與調試規範 (Developer Guidelines & Debug Mode)

## 目的
為避免測試/作弊邏輯污染主遊戲 Core Loop，本專案將開發者功能集中管理於獨立 Debug 選單，確保一般遊玩流程（標題、基地、戰鬥）維持穩定與可預期的玩家體驗。

## 選單入口規範
- 所有測試功能統一收斂在 `@DEBUG_MAIN_MENU`。
- 一般 UI（目前為基地管理）只提供單一 Debug 入口，按鈕 ID 固定為 `[999]`。
- 不得在一般選單（如 `@TITLE_SCREEN`、`@BASE_MENU`、設施子選單）散落一次性測試按鈕。

## 命名慣例
- 所有開發者專用函式必須以 `@DEBUG_` 前綴命名。
  - 範例：`@DEBUG_MAIN_MENU`、`@DEBUG_RESOURCE_MENU`、`@DEBUG_NPC_EDITOR`。
- 若需調用正式系統函式（例如 `@INIT_CHARA_STATS`），應由 `@DEBUG_` 函式作為入口，避免在主流程中混入 debug 分支。

## 擴充指南
當要新增新調試功能（例如「戰鬥直接勝利」、「強制觸發晚間事件」）時，請遵守以下流程：
1. 先新增對應 `@DEBUG_` 函式（例如 `@DEBUG_FORCE_BATTLE_WIN`）。
2. 在 `@DEBUG_MAIN_MENU` 註冊新按鈕與路由。
3. 在函式內輸出清楚的調試提示，避免與正常遊戲訊息混淆。
4. 若會改動保存資料（FLAG/CFLAG/BASE），需在函式中做最小必要防呆（例如 clamp、下限歸零）。

## 維護原則
- Debug 模組可快速迭代，但不得破壞主流程邏輯。
- 正式功能完成後，若相關 debug 功能已不需要，應優先整併或移除。
- 文件與實作需同步更新，保持單一事實來源（Single Source of Truth）。

## 長期維護架構規範 (Long-term Architecture)

### 1) 模組邊界與單一職責 (Module Boundaries & SRP)
- System/Core Loop 僅能負責流程控制，不可承載 Base/Battle/Chara 的領域細節。
- Base/Battle/Chara 各自管理自身狀態與資料結構，不可要求 Core 層直接操控其底層陣列。
- 依賴方向固定為「核心層 -> 子系統層」，不得讓子系統回頭控制核心流程。
- 禁止跨域直接寫入他模組底層資料（例：Core Loop 不可直接讀寫 `WORK_Q_REMAIN`）。

### 2) 生命週期委派設計 (Lifecycle Delegation Pattern)
- 全域生命周期事件（初始化、換日、晨間、晚間等）必須由核心層發送呼叫。
- 子系統透過 Hook 介面自行處理狀態更新，核心層不包含業務規則計算。
- 若需擴充事件，先新增/擴充對應 Hook，再將邏輯放回對應子系統模組。

### 3) 介面命名公約 (API Naming Conventions)
- 初始化 Hook：`@模組名_INIT`（如 `@BASE_INIT`, `@BATTLE_INIT`）。
- 生命週期 Hook：`@模組名_EVENT_時機`（如 `@BASE_EVENT_MORNING`, `@CHARA_EVENT_NIGHT`）。
- 對外公開 API 需穩定、語意明確；內部工具函式避免濫用跨模組 CALL。

## 開發與除錯常見陷阱 (Common Pitfalls)
1. **UI 畫面重繪原則**：執行任何會改變狀態的動作（例如產生 NPC、切換頁面、修改屬性）後，必須 `GOTO` 回對應選單頂部標籤重新跑 `PRINT/PRINTFORM`，否則畫面不會立即反映最新資料。
2. **輸入與賦值原則**：`INPUT` 後的數值在 `RESULT`。若要修改全域狀態，必須明確把 `RESULT` 賦值到目標變數（如 `FLAG`、`CFLAG`、`BASE`），不可只停留在 `LOCAL` 暫存而未寫回。
