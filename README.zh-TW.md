# 設計模式與設計原則

適用於 Laravel（PHP）後端 + Blade / Alpine.js 前端專案的程式碼規範與設計模式指南。所有產出的程式碼（人寫或 AI 產生）都必須遵循這些文件。

> **For English version, see [README.md](./README.md)**

## 文件

| 文件 | 範圍 |
|------|------|
| [backend-design-patterns.md](./backend-design-patterns.md) | 後端設計原則、設計模式與命名規範（PHP / Laravel） |
| [frontend(blade)-design-patterns.md](./frontend(blade)-design-patterns.md) | 前端設計原則、設計模式與命名規範（Blade / Alpine.js） |
| [testing-design-patterns.md](./testing-design-patterns.md) | 測試策略、測試替身與命名規範（PHPUnit / Dusk） |

## 架構

### 後端分層架構

```
Controller → ServiceInterface → Service → RepositoryInterface → Repository → Model
                ↑                              ↑
            ServiceProvider 綁定 (Factory Pattern)
```

| 層級 | 職責 |
|------|------|
| Controller | 接收 HTTP 請求、回傳 HTTP 回應 |
| FormRequest | 輸入驗證 |
| ServiceInterface | 定義業務操作的契約 |
| Service | 業務流程、transaction 管理 |
| RepositoryInterface | 定義資料存取的契約 |
| Repository | 資料存取、查詢邏輯 |
| Model | 資料結構、關聯定義、scope、型別轉換 |
| ServiceProvider | 綁定 Interface → 實作 |

### 前端分層架構

- **Blade 元件** — 純展示層
- **Alpine 元件** — 扮演容器角色，負責資料取得與狀態管理

## 涵蓋主題

### 設計原則

- **DRY** — 每一份知識在系統中只能有一個權威來源
- **SOLID** — SRP、OCP、LSP、ISP、DIP，搭配 Laravel 專屬範例
- 漸進增強、狀態最小化、關注點分離

### 設計模式

| 模式 | 用途 |
|------|------|
| Repository | 資料存取抽象（架構強制） |
| Null Object | 依賴可能不存在時的安全預設 |
| Strategy | 同一操作有多種實作方式 |
| Adapter | 橋接不相容的 Interface |
| Factory | 集中管理物件建構（ServiceProvider） |
| Template Method | 透過 Trait 實現共用流程骨架 |
| Event Delegation | 動態列表的事件處理 |
| Observer | 跨元件通訊（CustomEvent） |
| State Machine | 管理互斥的 UI 狀態 |
| Debounce / Throttle | 高頻事件控制 |

### 測試策略

| 層級 | 測試類型 | 替身策略 |
|------|---------|---------|
| Controller | Feature Test（真實 HTTP） | 真實 DB |
| FormRequest | Unit Test | 無 |
| Service | Unit Test | Mock RepositoryInterface |
| Repository | Unit Test（Integration） | 真實 DB |
| Model | Unit Test | 真實 DB |
| Policy | Unit Test | Factory 建立使用者 |
| Adapter / Strategy / Null Object | Unit Test | Mock Interface（若有外部依賴） |
| Blade 元件 | Feature Test | 真實 render |
| Alpine 元件 | Browser Test（Dusk） | 真實瀏覽器 |

核心測試模式：測試金字塔、Arrange-Act-Assert、測試隔離、測試替身（Mock / Stub / Fake / Spy）、Model Factory 狀態變體。

### 命名規範

涵蓋 PHP class、method、variable、資料庫表/欄位、路由、JavaScript、HTML/Blade 及檔案命名的完整規則。

## 技術棧

| 類別 | 技術 |
|------|------|
| 後端語言 | PHP |
| 後端框架 | Laravel |
| 前端模板 | Blade |
| 前端 JS | Alpine.js |
