# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 專案概述

本倉庫是一份**設計原則與設計模式的上位規範文件集**，適用於 Laravel（PHP）後端 + Blade / Alpine.js 前端的技術棧。所有產出的程式碼（人寫或 AI 產生）都必須遵循這些文件。

本倉庫不包含可執行的程式碼，沒有 build、lint 或 test 指令。

## 文件結構

- `backend-design-patterns.md` — 後端規範（PHP / Laravel）
- `frontend(blade)-design-patterns.md` — 前端規範（Blade / Alpine.js）

兩份文件互相引用，後端文件的命名通用原則同時適用於前端。

## 核心架構約束

### 後端分層架構（不可違反）

```
Controller → ServiceInterface → Service → RepositoryInterface → Repository → Model
                ↑                              ↑
            ServiceProvider 綁定 (Factory Pattern)
```

| 層級 | 唯一職責 | 禁止出現 |
|------|---------|---------|
| Controller | 接收 / 回傳 HTTP | 業務邏輯、DB transaction |
| FormRequest | 輸入驗證 | 業務邏輯、資料寫入 |
| ServiceInterface | 定義業務操作的契約 | 實作細節 |
| Service | 業務流程、transaction | HTTP 相關操作、直接操作 Model（必須透過 Repository） |
| RepositoryInterface | 定義資料存取的契約 | 實作細節、業務邏輯 |
| Repository | 資料存取、查詢邏輯 | 業務流程、HTTP 操作 |
| Model | 資料結構、關聯定義、scope、型別轉換 | 業務流程、HTTP 操作、複雜查詢邏輯（屬 Repository） |
| ServiceProvider | 綁定 Interface → 實作 | 業務邏輯 |

### 前端分層職責

- **Blade 元件**：純展示層，不呼叫 API、不修改全域狀態
- **Alpine 元件**：扮演容器角色，負責資料取得與狀態管理
- 元件超過 150 行須審視是否拆分

### 設計模式選用

依情境選用對應模式，不可用 if/else 堆疊替代：

- 資料存取從 Service 抽離 → **Repository**（架構強制，一個 Repository 對應一個 Model）
- 依賴可能不存在 → **Null Object**
- 同操作多種實作 → **Strategy**
- Interface 不相容需橋接 → **Adapter**
- 物件建構複雜 → **Factory**（含 ServiceProvider 綁定）
- 相同流程不同步驟 → **Template Method**（Trait 實現）
- 動態列表事件 → **Event Delegation**
- 跨元件通訊 → **Observer**（CustomEvent，`namespace:action` 格式）
- 多互斥 UI 狀態 → **State Machine**（單一字串變數，非多個 boolean）
- 高頻事件 → **Debounce / Throttle**

## 命名規範摘要

### PHP

- Class: `PascalCase` + 職責後綴（`Service`, `Repository`, `Controller`, `Request`...）
- Interface: `PascalCase` + `Interface` 結尾（`ServiceInterface`, `RepositoryInterface`...）
- Method: `camelCase` 動詞開頭；布林用 `is`/`has`/`can`/`should`
- 不縮寫（`$usr` → `$user`），除 `$id`, `$url`, `$api` 等公認縮寫

### 資料庫

- 資料表: `snake_case` **複數**（`users`, `quotation_items`）
- 外鍵: 單數模型名 + `_id`；布林欄位 `is_` 開頭
- Pivot 表: 兩模型單數、字母序（`contract_user`）

### 路由

- URL: `kebab-case` 名詞複數（`/quotation-items`），不用動詞
- Route name: `dot.separated`，模組前綴

### JavaScript / 前端

- 變數 / 函式 / Alpine 元件: `camelCase`
- Custom Event: `kebab-case`，`namespace:action`（如 `cart:item-added`）
- CSS class / HTML id / data-*: `kebab-case`
- Blade 元件: `kebab-case` + 模組前綴（`<x-core::modal>`）
- JS / CSS 檔案: `kebab-case`；Blade partial 加 `_` 前綴

## 關鍵規則

- 外部依賴透過建構子注入 Interface，不直接 `new` 具體 class
- 同一概念全系統統一命名，不可 A 檔叫 `user`、B 檔叫 `member`
- 核心功能不依賴 JavaScript（漸進增強）
- 衍生狀態用 getter / computed 計算，不另存變數手動同步
- Adapter 只做轉換，不加業務邏輯
- Null Object 不可拋例外或寫 log 警告
