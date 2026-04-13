# 前端設計原則與設計模式

> 本文件是前端程式碼的上位規範。所有產出的前端程式碼（人寫或 AI 產生）都必須遵循本文件。
> 本文件不描述特定 codebase，而是定義通用的前端軟體工程準則。
> 後端規範請參閱 [backend-design-patterns.md](./backend-design-patterns.md)。

---

## 一、設計原則

### 1.1 關注點分離 — Structure / Style / Behavior

**規則：HTML 負責結構與語意，CSS 負責外觀，JavaScript 負責行為。三者不混為一談。**

#### 你必須

- HTML 使用語意化標籤（`<nav>`、`<main>`、`<section>`、`<button>`），不全部用 `<div>`
- 樣式透過 CSS class 控制，不用 JS 直接操作 `element.style`
- 行為邏輯寫在 JS 中，不在 HTML 屬性裡塞大段程式碼

#### 你不可以

- 在 JS 中拼接大段 HTML 字串（應由模板引擎或元件處理）
- 用 `display: none` + JS toggle 取代語意化的條件渲染
- 在 CSS 中寫 hardcoded 的內容文字（`content: "提交"` 之類的 i18n 文字）

#### 正確 vs 錯誤

```html
<!-- 錯誤：HTML 裡塞滿行為邏輯和樣式 -->
<div onclick="document.getElementById('modal').style.display='block';
              document.getElementById('modal').style.backgroundColor='rgba(0,0,0,0.5)';
              fetch('/api/data').then(r => r.json()).then(d => {
                  document.getElementById('content').innerHTML = d.html;
              })">
    打開
</div>

<!-- 正確：結構、樣式、行為各司其職 -->
<button type="button" class="btn-primary" data-action="open-modal">
    打開
</button>
```

---

### 1.2 元件單一職責

**規則：一個 UI 元件只解決一個視覺或互動問題。**

此原則是後端 SRP 在前端的延伸。一個元件如果同時負責「資料載入」、「表單驗證」、「結果渲染」，就承擔了過多職責。

#### 你必須

- 每個元件有明確的職責邊界，從命名就能看出它做什麼
- 元件只接受它需要的資料，不傳入整個 page-level 物件
- 超過 150 行的元件，審視是否應拆分

#### 你不可以

- 建立「萬能元件」，靠大量 props/flags 切換行為
- 在同一個元件中混合完全不相關的 UI 區塊
- 讓元件直接存取全域狀態或直接呼叫 API（應由外部傳入或透過事件通知）

#### 正確 vs 錯誤

```javascript
// 錯誤：一個元件做太多事
Alpine.data('orderPage', () => ({
    orders: [],
    searchQuery: '',
    selectedOrder: null,
    showModal: false,
    formData: {},
    validationErrors: {},
    async loadOrders() { /* ... */ },
    async submitForm() { /* ... */ },
    validateEmail(email) { /* ... */ },
    formatCurrency(amount) { /* ... */ },
    exportToCsv() { /* ... */ },
    printOrder() { /* ... */ },
    // ... 300 行
}))

// 正確：拆成職責明確的小元件
Alpine.data('orderList', () => ({
    orders: [],
    searchQuery: '',
    async loadOrders() { /* ... */ },
}))

Alpine.data('orderForm', () => ({
    formData: {},
    validationErrors: {},
    async submit() { /* ... */ },
}))
```

---

### 1.3 漸進增強 — Progressive Enhancement

**規則：核心功能不依賴 JavaScript。JS 用來增強體驗，不是提供基本功能。**

#### 你必須

- 表單用原生 `<form>` + `action`，JS 再加上非同步提交
- 連結用 `<a href>`，不用 `<span onclick>` 模擬
- 按鈕用 `<button type="button">`，不用 `<div>` 加 click handler

#### 你不可以

- 用 `<a href="#">` 或 `<a href="javascript:void(0)">`——沒有真實目的地就用 `<button>`
- 把關鍵的導覽或提交行為完全建構在 JS 之上，讓 JS 失敗時頁面無法使用
- 忽略 `<noscript>` 情境下的可用性

---

### 1.4 狀態最小化

**規則：只儲存必要的原始狀態，衍生資料用計算得出，不另存一份。**

這是 DRY 原則在前端狀態管理的延伸。

#### 你必須

- 能從既有狀態計算出來的值，用 getter / computed 取得，不另存變數
- 狀態只存在一個地方（single source of truth），不在多個元件各自維護副本
- 清楚區分「UI 狀態」（modal 是否打開）和「資料狀態」（使用者清單），不混在一起

#### 你不可以

- 同一份資料在多個元件各存一份，靠手動同步保持一致
- 把「篩選後的結果」另存為狀態——它是從「原始資料 + 篩選條件」計算出來的
- 在 DOM 上用 `data-*` 屬性儲存應該在 JS 狀態中管理的業務資料

#### 正確 vs 錯誤

```javascript
// 錯誤：衍生值另存為狀態，需要手動同步
Alpine.data('cart', () => ({
    items: [],
    itemCount: 0,          // 多餘，是 items.length
    totalPrice: 0,         // 多餘，是 items 的 price 加總
    hasItems: false,        // 多餘，是 items.length > 0

    addItem(item) {
        this.items.push(item);
        this.itemCount = this.items.length;        // 手動同步
        this.totalPrice = this.items.reduce(...);  // 手動同步
        this.hasItems = this.items.length > 0;     // 手動同步
    },
}))

// 正確：只存原始狀態，其餘計算得出
Alpine.data('cart', () => ({
    items: [],

    get itemCount() { return this.items.length; },
    get totalPrice() { return this.items.reduce((sum, i) => sum + i.price, 0); },
    get hasItems() { return this.items.length > 0; },

    addItem(item) {
        this.items.push(item);
        // 衍生值自動更新，不需要手動同步
    },
}))
```

---

## 二、設計模式

以下列出前端開發中應採用的設計模式。每個模式說明「什麼時候用」和「怎麼用」。

### 2.1 Event Delegation Pattern（事件委派模式）

**用途：在父元素上監聽事件，處理動態子元素的互動。**

#### 何時使用

- 列表項目數量不固定、會動態新增移除
- 大量同類型子元素需要相同的事件處理

#### 結構

```javascript
// 錯誤：對每個按鈕個別綁定事件
document.querySelectorAll('.delete-btn').forEach(btn => {
    btn.addEventListener('click', () => deleteItem(btn.dataset.id));
});
// 動態新增的按鈕不會被綁定，且效能差

// 正確：在父容器上委派
document.querySelector('.item-list').addEventListener('click', (e) => {
    const btn = e.target.closest('[data-action="delete"]');
    if (btn) {
        deleteItem(btn.dataset.id);
    }
});
// 動態新增的元素自動生效，只有一個 listener
```

#### 關鍵規則

- 用 `closest()` 尋找觸發目標，不要用 `e.target` 直接比對（可能點到子元素）
- 委派層級不要太高（不要全掛在 `document` 上），掛在最近的穩定父容器

---

### 2.2 Observer Pattern（觀察者模式）

**用途：元件之間透過事件通訊，避免直接引用造成耦合。**

#### 何時使用

- 兩個無父子關係的元件需要通訊
- 一個操作需要通知多個不相關的元件更新
- 想讓元件之間保持鬆耦合

#### 結構

```javascript
// 錯誤：元件 A 直接操作元件 B 的內部狀態
function addToCart(item) {
    cartComponent.items.push(item);       // 直接耦合
    headerComponent.cartCount++;          // 直接耦合
    notificationComponent.show('已加入'); // 直接耦合
}

// 正確：透過事件通知，各元件自行決定如何反應
function addToCart(item) {
    window.dispatchEvent(new CustomEvent('cart:item-added', {
        detail: { item },
    }));
}

// 購物車元件——自己監聽、自己更新
window.addEventListener('cart:item-added', (e) => {
    this.items.push(e.detail.item);
});

// Header 元件——自己監聽、自己更新
window.addEventListener('cart:item-added', () => {
    this.cartCount++;
});
```

#### 關鍵規則

- 事件名稱用 `namespace:action` 格式（如 `cart:item-added`），避免衝突
- 事件只攜帶資料（`detail`），不攜帶行為指令
- 發送端不需要知道有誰在監聯

---

### 2.3 State Machine Pattern（狀態機模式）

**用途：用有限狀態和明確的轉換規則管理 UI 狀態，取代散落的布林旗標。**

#### 何時使用

- UI 有多個互斥的狀態（idle / loading / success / error）
- 用多個 boolean（`isLoading`、`isError`、`isSuccess`）管理時出現不合法的組合
- 狀態轉換有明確的規則（loading 只能轉到 success 或 error，不能直接轉到 idle）

#### 結構

```javascript
// 錯誤：多個 boolean 可能產生不合法狀態
Alpine.data('form', () => ({
    isLoading: false,
    isSuccess: false,
    isError: false,
    errorMessage: '',

    async submit() {
        this.isLoading = true;
        this.isError = false;   // 忘了重設 isSuccess？
        try {
            await api.post('/submit', this.formData);
            this.isSuccess = true;
            this.isLoading = false; // 忘了這行就同時是 loading + success
        } catch (e) {
            this.isError = true;
            this.errorMessage = e.message;
            // 忘了設 isLoading = false
        }
    },
}))

// 正確：單一狀態變數，不可能出現不合法組合
Alpine.data('form', () => ({
    state: 'idle', // idle | loading | success | error
    errorMessage: '',

    get isLoading() { return this.state === 'loading'; },
    get isSuccess() { return this.state === 'success'; },
    get isError() { return this.state === 'error'; },

    async submit() {
        this.state = 'loading';
        try {
            await api.post('/submit', this.formData);
            this.state = 'success';
        } catch (e) {
            this.state = 'error';
            this.errorMessage = e.message;
        }
    },
}))
```

#### 關鍵規則

- 狀態用單一字串變數管理，不用多個 boolean
- 在模板中用 `x-show="state === 'loading'"` 而非 `x-show="isLoading && !isError"`
- 如果需要在模板中頻繁使用，用 getter 提供語意化的存取

---

### 2.4 Container / Presenter Pattern（容器 / 展示分離模式）

**用途：分離「資料取得與業務邏輯」和「UI 渲染」，讓展示層可以獨立測試和複用。**

#### 何時使用

- 同一份資料需要在不同頁面以不同形式呈現
- 元件中混雜了 API 呼叫和 DOM 渲染邏輯

#### 結構

```html
<!-- 容器：負責資料取得和業務邏輯 -->
<div x-data="userListContainer()">
    <!-- 展示：只負責渲染，資料全由外部傳入 -->
    <template x-for="user in users" :key="user.id">
        <x-user-card :user="user" />
    </template>
</div>

<!-- 展示元件：純粹接收 props 並渲染，不呼叫 API -->
<!-- user-card.blade.php -->
<div class="card">
    <h3>{{ $user->name }}</h3>
    <p>{{ $user->email }}</p>
</div>
```

```javascript
// 容器邏輯
Alpine.data('userListContainer', () => ({
    users: [],
    async init() {
        const res = await window.api.get('/users');
        this.users = res.data;
    },
}))
```

#### 關鍵規則

- 展示元件不呼叫 API、不修改全域狀態，只接收資料、發出事件
- 容器負責資料取得、錯誤處理、狀態管理
- Server-rendered 架構下，Blade 元件天然是展示層，Alpine 元件扮演容器

---

### 2.5 Debounce / Throttle Pattern（防抖 / 節流模式）

**用途：控制高頻事件的觸發頻率，避免不必要的計算或請求。**

#### 何時使用

| 情境 | 使用 |
|------|------|
| 搜尋框輸入 → 呼叫 API | **Debounce**（等使用者停止輸入後才發送） |
| 視窗 resize → 重新計算布局 | **Debounce**（等 resize 結束後才計算） |
| 捲動 → 載入更多 | **Throttle**（固定間隔執行，不要每個 scroll 事件都觸發） |
| 按鈕點擊 → 提交表單 | **不適用**——用 disabled 狀態防止重複提交 |

#### 結構

```javascript
// 錯誤：每次按鍵都呼叫 API
async searchUsers() {
    const res = await window.api.get(`/users?q=${this.query}`);
    this.results = res.data;
}

// 正確：Debounce — 使用者停止輸入 300ms 後才呼叫
searchUsers: Alpine.debounce(async function () {
    const res = await window.api.get(`/users?q=${this.query}`);
    this.results = res.data;
}, 300)
```

#### 關鍵規則

- Debounce 用於「只需要最終結果」的場景（搜尋、resize）
- Throttle 用於「過程中也需要間歇性反應」的場景（捲動、拖曳）
- 提交按鈕不用 debounce，用 loading 狀態 + disabled 防止重複提交

---

### 2.6 模式選用速查表

| 你遇到的情境 | 使用的模式 |
|-------------|-----------|
| 動態列表需要事件處理 | **Event Delegation** |
| 無關係的元件需要通訊 | **Observer**（Custom Event） |
| UI 有多個互斥狀態，boolean 管不住 | **State Machine** |
| 元件同時做資料取得和渲染，太胖了 | **Container / Presenter** |
| 高頻事件造成效能問題或多餘請求 | **Debounce / Throttle** |
| 元件做太多事、超過 150 行 | 回頭看 **1.2 元件單一職責**，拆分它 |

---

## 三、命名規範

> 命名通用原則（語意優先、一致性、不縮寫）請參閱 [backend-design-patterns.md § 三、命名規範](./backend-design-patterns.md#三命名規範)。

### 3.1 JavaScript

#### 總覽表

| 類別 | 格式 | 範例 |
|------|------|------|
| 變數 | camelCase | `userName`, `totalPrice`, `isVisible` |
| 函式 | camelCase，動詞開頭 | `fetchUsers()`, `handleSubmit()`, `formatCurrency()` |
| 常數 | UPPER_SNAKE_CASE | `MAX_FILE_SIZE`, `API_BASE_URL` |
| Alpine 元件 | camelCase | `Alpine.data('orderList', ...)`, `Alpine.data('searchFilter', ...)` |
| Custom Event | kebab-case，`namespace:action` | `cart:item-added`, `modal:closed`, `form:submitted` |
| CSS class | kebab-case | `btn-primary`, `card-header`, `is-active` |

#### 函式命名慣例

```javascript
// 事件處理器——handle + 事件名
function handleClick() { }
function handleFormSubmit() { }
function handleModalClose() { }

// 資料操作——動詞 + 名詞
function fetchUsers() { }
function createOrder(data) { }
function deleteItem(id) { }
function updateProfile(data) { }

// 格式化 / 轉換——format / convert / parse / to
function formatCurrency(amount) { }
function parseDate(dateString) { }
function toPercentage(value) { }

// 布林——is / has / can / should
function isValid() { }
function hasPermission(name) { }
function canSubmit() { }
```

#### 你不可以

- 函式用名詞命名（`data()` → `fetchData()`、`validation()` → `validateForm()`）
- Alpine 元件用 PascalCase 或 kebab-case（`Alpine.data('OrderList', ...)` → `Alpine.data('orderList', ...)`）
- 事件名用 camelCase（`itemAdded` → `item-added`）
- CSS class 用 camelCase 或 snake_case（`btnPrimary` → `btn-primary`）

---

### 3.2 HTML / Blade

#### 總覽表

| 類別 | 格式 | 範例 |
|------|------|------|
| HTML attribute | kebab-case | `data-user-id`, `aria-label` |
| Blade 元件 | kebab-case，模組前綴 | `<x-core::modal>`, `<x-core::search-input>` |
| Blade 元件 prop | kebab-case | `<x-core::modal show="showModal">` |
| `id` attribute | kebab-case | `id="user-profile-form"` |
| `data-*` attribute | kebab-case | `data-action="delete"`, `data-item-id="42"` |
| Alpine directive 值 | camelCase（因為是 JS） | `x-data="orderList()"`, `x-show="isVisible"` |

#### 你不可以

- `id` 用 camelCase 或 snake_case（`id="userForm"` → `id="user-form"`）
- `data-*` 用 camelCase（`data-userId` → `data-user-id`）
- Blade 元件命名不加模組前綴（`<x-modal>` → `<x-core::modal>`）

---

### 3.3 檔案命名

| 類別 | 格式 | 範例 |
|------|------|------|
| JS 檔案 | kebab-case | `signature-pad.js`, `flatpickr-datepicker.js` |
| CSS 檔案 | kebab-case | `app.css`, `dashboard-charts.css` |
| 圖片 / 靜態資源 | kebab-case | `logo-dark.png`, `watermark.png` |
| Blade view | kebab-case | `index.blade.php`, `create.blade.php` |
| Blade partial | `_` 前綴 | `_sidebar.blade.php`, `_form-fields.blade.php` |

#### 你不可以

- JS / CSS 檔案用 PascalCase 或 camelCase（`SignaturePad.js` → `signature-pad.js`）
- Blade view 用 PascalCase（`UserProfile.blade.php` → `user-profile.blade.php`）

---

### 3.4 命名速查表

| 我在寫… | 格式 | 前綴/後綴規則 |
|---------|------|--------------|
| JS Variable / Function | camelCase | 函式動詞開頭 |
| JS Constant | UPPER_SNAKE_CASE | — |
| Alpine Component | camelCase | — |
| Custom Event | kebab-case | `namespace:action` |
| CSS Class | kebab-case | — |
| HTML id / data-* | kebab-case | — |
| JS / CSS 檔案 | kebab-case | — |
| Blade view | kebab-case | partial 加 `_` 前綴 |
