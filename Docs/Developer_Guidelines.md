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
