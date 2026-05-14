# 萬用命盤產生器 (研究專用版) - 開發與架構記錄

這份文件記錄了本專案（純 HTML 萬用命盤產生器）的核心架構、技術決策以及重要踩坑紀錄。強烈建議未來若需要交由 AI 進行二次開發或維護時，將此文件作為 System Prompt 或 Context 提供給 AI，以避免破壞現有穩定架構。

## 1. 核心技術與排版架構
*   **技術棧**：純 HTML5 + CSS3 + Vanilla JavaScript。完全無依賴、無框架、單一檔案（Single File），確保永久離線可用性。
*   **命盤佈局 (CSS Grid)**：
    *   使用 `display: grid; grid-template-columns: repeat(4, 1fr); grid-template-rows: repeat(4, 1fr);` 建立 4x4 的網格。
    *   中宮（Center Palace）利用 `grid-column: 2 / 4; grid-row: 2 / 4;` 跨越中央四格。
    *   12 宮位的實際順序是順時針排列，對應陣列的渲染順序在程式碼中使用 `divOrder` 二維陣列進行映射（5~8, 4 & 9, 3 & 10, 2~0 & 11）。

## 2. 列印與版面優化 (Print Media Queries)
**這是本專案最容易踩坑的技術點，修改時務必謹慎：**
*   **列印空白框問題 (Chrome Grid Collapse)**：
    若在 `@media print` 中將 `.chart-container` 設為 `height: 100%`，部分 Chromium 瀏覽器列印引擎會無法計算出高度，導致列印出「只有外框沒有內容的空白頁」。
    *   **解法**：放棄相對高度，改採絕對高度。直式列印強制使用 `height: 25cm;`，橫式列印使用 `height: 17cm;`。
*   **動態切換版面 (Portrait vs Landscape)**：
    瀏覽器無法單純靠 CSS 變數動態更改 `@page { size: A4 ... }`。
    *   **解法**：在 `<head>` 中設立獨立的 `<style id="pageStyle">` 標籤，透過 JS 直接抽換其 `innerHTML` 來達成直橫式的強制宣告，並同步替 `.page` 加上 `.landscape` class 以更改螢幕預覽長寬。
*   **備註欄文字溢出 (Text Overflow)**：
    利用 `white-space: pre-wrap; word-wrap: break-word; max-width: 85%;` 強制長字串在觸碰中宮邊界時自動斷行，防止 Flex 容器被撐破。

## 3. 核心演算法與資料結構
*   **四段式圖層 (Layer 1~4)**：
    透過 `layer` 變數控制陣列寫入與渲染。Layer 1 僅渲染主星；Layer 4 則啟用吉煞星的 push 與渲染。
*   **0-Indexed 計算**：
    12 地支（子~亥）在陣列映射中均轉化為 `0~11` 進行模數計算（Modulo Arithmetic）。例如 `(12 + offset) % 12` 以確保順行逆行不會產生負數索引。
*   **天馬星流派**：
    採用「年馬」標準（依出生年支計算：申子辰馬在寅...），並無實作「月馬」。

## 4. 13 碼生成碼系統 (Generation Code System)
為了兼顧穩定與易解析性，捨棄了 JSON 或 Base64，採用固定長度 `13 位數` 編碼系統：
*   **編碼邏輯**：`T(2碼) + S(2碼) + D(2碼) + Y(2碼) + M(2碼) + H(2碼) + L(1碼)`
*   **佔位符**：下拉選單的數值為 1~12。當使用者選擇低圖層（例如 Layer 1）時，不需要的參數會被強制覆寫為 `00`。
*   **解析優勢**：在 `loadFromCode()` 中，只需使用固定的 `substring` 即可安全解析，不會有長度不一導致解析偏移的問題。

## 5. 重大 Bug 修復紀錄
*   **變數遮蔽 (Variable Shadowing) 導致的 ReferenceError**：
    曾發生外層宣告 `const pName` 讀取使用者輸入，而內部 `for` 迴圈中又宣告 `let pName` 來表示宮位名稱。在 `for` 迴圈內引用 `${pName}` 時觸發了「暫時性死區 (TDZ)」，導致整個 JS 崩潰無反應。
    *   **修復方式**：將迴圈內的宮位名稱變數嚴格更名為 `pNameStr`。未來二次開發時，務必注意 JS 區塊作用域的變數命名。
