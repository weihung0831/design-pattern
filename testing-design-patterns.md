# 測試設計原則與設計模式

> 本文件是測試程式碼的上位規範。所有產出的測試程式碼（人寫或 AI 產生）都必須遵循本文件。
> 本文件不描述特定 codebase，而是定義通用的測試工程準則。
> 後端規範請參閱 [backend-design-patterns.md](./backend-design-patterns.md)。
> 前端規範請參閱 [frontend(blade)-design-patterns.md](./frontend(blade)-design-patterns.md)。

---

## 一、設計原則

### 1.1 測試金字塔

**規則：測試數量由下而上遞減——Unit 最多、Feature 次之、Browser（E2E）最少。**

```
        ╱ ╲          Browser / E2E（少量，驗證關鍵使用者流程）
       ╱───╲
      ╱     ╲        Feature / Integration（中量，驗證跨層協作）
     ╱───────╲
    ╱         ╲      Unit（大量，驗證單一 class 的邏輯）
   ╱───────────╲
```

| 層級 | 驗證什麼 | 速度 | 數量 |
|------|---------|------|------|
| Unit | 單一 class / method 的邏輯正確性 | 快 | 最多 |
| Feature | HTTP 請求 → 回應的完整流程（含資料庫） | 中 | 中 |
| Browser / E2E | 使用者透過瀏覽器操作的關鍵流程 | 慢 | 最少 |

#### 你必須

- 業務邏輯（Service）優先寫 Unit Test
- API 端點優先寫 Feature Test
- 只對關鍵使用者流程（登入、結帳、簽約等）寫 Browser Test

#### 你不可以

- 只寫 Feature Test 取代所有 Unit Test（速度慢、定位問題困難）
- 對每個頁面都寫 Browser Test（維護成本過高）

---

### 1.2 Arrange-Act-Assert（AAA 模式）

**規則：每個測試方法嚴格分為三段——準備、執行、驗證。**

```php
public function test_complete_order_updates_status_and_fires_event(): void
{
    // Arrange — 準備測試資料與環境
    Event::fake([OrderCompleted::class]);
    $order = Order::factory()->create(['status' => 'pending']);

    // Act — 執行被測行為（只有一個動作）
    $this->service->complete($order->id);

    // Assert — 驗證結果
    $this->assertDatabaseHas('orders', [
        'id' => $order->id,
        'status' => 'completed',
    ]);
    Event::assertDispatched(OrderCompleted::class);
}
```

#### 你必須

- 每個測試只有**一個 Act**——一次只測一個行為
- Arrange 只準備這個測試需要的東西，不多不少
- Assert 驗證行為的結果，不驗證實作細節

#### 你不可以

- 在一個測試中執行多個不相關的動作再一起驗證
- Assert 驗證 method 被呼叫了幾次（那是測實作，不是測行為）——除非那個呼叫本身就是行為（如寄信）

---

### 1.3 測試隔離

**規則：每個測試獨立運行，不依賴其他測試的執行結果或順序。**

#### 你必須

- 使用 `RefreshDatabase` 或 `DatabaseTransactions` trait 確保每個測試的資料庫狀態乾淨
- 測試需要的資料在自己的 Arrange 階段建立，不依賴 seeder 或其他測試殘留的資料
- 使用 `Event::fake()`、`Queue::fake()`、`Mail::fake()` 隔離副作用

#### 你不可以

- 測試 A 建立資料，測試 B 拿來用
- 用 `static` 變數在測試之間共享狀態
- 依賴測試的執行順序

---

### 1.4 測試行為，不測實作

**規則：測試驗證「做了什麼」，不驗證「怎麼做的」。**

```php
// 錯誤：測試 Service 內部呼叫了哪些 Repository 方法（實作細節）
$mockRepo = Mockery::mock(OrderRepositoryInterface::class);
$mockRepo->shouldReceive('find')->once()->with(1);
$mockRepo->shouldReceive('update')->once();

// 正確：測試 Service 完成操作後的結果（行為）
$this->service->complete($order->id);
$this->assertDatabaseHas('orders', ['id' => $order->id, 'status' => 'completed']);
```

#### 判斷基準

問自己：「如果重構內部實作但行為不變，這個測試會壞嗎？」如果會，你測的是實作，不是行為。

---

## 二、各層測試策略

### 2.1 層級 × 測試類型對照表

| 層級 | 測試類型 | 測試替身策略 | 驗證重點 |
|------|---------|-------------|---------|
| **Controller** | Feature Test | 真實資料庫（RefreshDatabase） | HTTP status、回應結構、認證、授權 |
| **FormRequest** | Unit Test | 不需替身 | 驗證規則通過 / 拒絕、錯誤訊息 |
| **Service** | Unit Test | Mock RepositoryInterface | 業務流程、條件分支、transaction、事件觸發 |
| **Repository** | Unit Test（Integration） | 真實資料庫（RefreshDatabase） | 查詢結果正確性、資料寫入 |
| **Model** | Unit Test | 不需替身 | 關聯定義、scope、accessor / mutator、$casts |
| **Event / Listener** | Unit Test | Mock 依賴的 Service | Listener 觸發正確行為 |
| **Job** | Unit Test | Mock 依賴的 Service | Job 執行正確行為、失敗重試邏輯 |
| **Policy** | Unit Test | Model Factory 建立使用者 | 各角色 / 狀態的授權結果 |
| **Middleware** | Feature Test | 真實 HTTP 請求 | 請求被允許或攔截 |
| **Adapter** | Unit Test | Mock 被轉接的 Interface | 輸入轉換正確性 |
| **Strategy 實作** | Unit Test | 不需替身（或 Mock 外部依賴） | 各策略的計算 / 行為正確 |
| **Null Object** | Unit Test | 不需替身 | 所有方法回傳安全預設值、不拋例外 |
| **Blade 元件** | Feature Test | 真實 render | 渲染結果包含預期的 HTML 結構 |
| **Alpine 元件** | Browser Test（Dusk） | 真實瀏覽器 | 使用者互動後的 UI 狀態變化 |

---

### 2.2 Controller — Feature Test

**驗證重點：HTTP 請求 → 回應的完整流程。**

Controller 不單獨做 Unit Test。它的價值在於串接所有層級，所以用 Feature Test 打真實 HTTP 請求。

```php
class OrderControllerTest extends TestCase
{
    use RefreshDatabase;

    public function test_store_creates_order_and_returns_201(): void
    {
        // Arrange
        $user = User::factory()->create();
        $payload = [
            'product_id' => Product::factory()->create()->id,
            'quantity' => 2,
        ];

        // Act
        $response = $this->actingAs($user)
            ->postJson('/api/orders', $payload);

        // Assert
        $response->assertStatus(201)
            ->assertJsonStructure(['data' => ['id', 'serial_no', 'status']]);

        $this->assertDatabaseHas('orders', [
            'user_id' => $user->id,
            'status' => 'pending',
        ]);
    }

    public function test_store_returns_422_when_validation_fails(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)
            ->postJson('/api/orders', []);

        $response->assertStatus(422)
            ->assertJsonValidationErrors(['product_id', 'quantity']);
    }

    public function test_store_returns_403_when_unauthorized(): void
    {
        $user = User::factory()->create(['role' => 'viewer']);
        $payload = ['product_id' => 1, 'quantity' => 1];

        $response = $this->actingAs($user)
            ->postJson('/api/orders', $payload);

        $response->assertStatus(403);
    }
}
```

#### 必須涵蓋的情境

- 成功路徑（正確 status code + 回應結構）
- 驗證失敗（422 + 錯誤欄位）
- 認證失敗（401）
- 授權失敗（403）
- 資源不存在（404）

---

### 2.3 FormRequest — Unit Test

**驗證重點：驗證規則正確地接受合法輸入、拒絕非法輸入。**

```php
class StoreOrderRequestTest extends TestCase
{
    use RefreshDatabase;

    private function rules(): array
    {
        return (new StoreOrderRequest())->rules();
    }

    public function test_product_id_is_required(): void
    {
        $validator = Validator::make(
            ['quantity' => 2],
            $this->rules(),
        );

        $this->assertTrue($validator->fails());
        $this->assertArrayHasKey('product_id', $validator->errors()->toArray());
    }

    public function test_quantity_must_be_positive_integer(): void
    {
        $product = Product::factory()->create();

        $validator = Validator::make(
            ['product_id' => $product->id, 'quantity' => -1],
            $this->rules(),
        );

        $this->assertTrue($validator->fails());
        $this->assertArrayHasKey('quantity', $validator->errors()->toArray());
    }

    public function test_valid_payload_passes(): void
    {
        $product = Product::factory()->create();

        $validator = Validator::make(
            ['product_id' => $product->id, 'quantity' => 2],
            $this->rules(),
        );

        $this->assertTrue($validator->passes());
    }
}
```

#### 關鍵規則

- 每個驗證規則至少一個通過、一個拒絕的測試案例
- 測試邊界值（0、負數、超長字串、空值）
- 若有條件式驗證規則（`required_if` 等），測試各條件組合

---

### 2.4 Service — Unit Test

**驗證重點：業務流程、條件分支、事件觸發。**

Service 是業務邏輯的核心，Mock 掉 RepositoryInterface 做純邏輯測試。

```php
class OrderServiceTest extends TestCase
{
    private OrderRepositoryInterface|MockInterface $orderRepository;
    private OrderService $service;

    protected function setUp(): void
    {
        parent::setUp();

        $this->orderRepository = Mockery::mock(OrderRepositoryInterface::class);
        $this->service = new OrderService($this->orderRepository);
    }

    public function test_complete_updates_order_status_and_dispatches_event(): void
    {
        // Arrange
        Event::fake([OrderCompleted::class]);
        $order = new Order(['id' => 1, 'status' => 'pending']);

        $this->orderRepository
            ->shouldReceive('find')->with(1)->andReturn($order);
        $this->orderRepository
            ->shouldReceive('update')
            ->with($order, Mockery::on(fn ($data) => $data['status'] === 'completed'))
            ->andReturn(new Order(['id' => 1, 'status' => 'completed']));

        // Act
        $result = $this->service->complete(1);

        // Assert
        $this->assertEquals('completed', $result->status);
        Event::assertDispatched(OrderCompleted::class);
    }

    public function test_complete_throws_exception_when_order_not_found(): void
    {
        $this->orderRepository
            ->shouldReceive('find')->with(999)->andReturnNull();

        $this->expectException(OrderNotFoundException::class);

        $this->service->complete(999);
    }
}
```

#### 關鍵規則

- Mock RepositoryInterface，不 Mock 具體的 Repository class
- 測試所有業務分支（成功、失敗、邊界條件）
- 用 `Event::fake()` 驗證事件是否觸發，不驗證 Listener 的行為（那是 Listener 自己的測試）

---

### 2.5 Repository — Unit Test（Integration）

**驗證重點：查詢邏輯的正確性、資料寫入的完整性。**

Repository 直接操作資料庫，需要用真實資料庫（`RefreshDatabase`）做整合測試。

```php
class OrderRepositoryTest extends TestCase
{
    use RefreshDatabase;

    private OrderRepository $repository;

    protected function setUp(): void
    {
        parent::setUp();
        $this->repository = new OrderRepository();
    }

    public function test_get_by_status_returns_only_matching_orders(): void
    {
        // Arrange
        Order::factory()->count(3)->create(['status' => 'pending']);
        Order::factory()->count(2)->create(['status' => 'completed']);

        // Act
        $pending = $this->repository->getByStatus('pending');

        // Assert
        $this->assertCount(3, $pending);
        $pending->each(fn ($order) => $this->assertEquals('pending', $order->status));
    }

    public function test_get_by_status_eager_loads_items(): void
    {
        $order = Order::factory()->create(['status' => 'pending']);
        OrderItem::factory()->count(2)->create(['order_id' => $order->id]);

        $results = $this->repository->getByStatus('pending');

        $this->assertTrue($results->first()->relationLoaded('items'));
        $this->assertCount(2, $results->first()->items);
    }

    public function test_find_by_serial_no_returns_null_when_not_found(): void
    {
        $result = $this->repository->findBySerialNo('NONEXISTENT');

        $this->assertNull($result);
    }
}
```

#### 關鍵規則

- 一定用 `RefreshDatabase`，不 Mock Model 或 Query Builder
- 驗證查詢結果的正確性（數量、內容、排序）
- 驗證 eager loading 是否正確載入
- 測試「找不到」的情境（回傳 null 或空 Collection）

---

### 2.6 Model — Unit Test

**驗證重點：關聯定義、scope、accessor / mutator、$casts。**

```php
class OrderTest extends TestCase
{
    use RefreshDatabase;

    public function test_has_many_items(): void
    {
        $order = Order::factory()->create();
        OrderItem::factory()->count(3)->create(['order_id' => $order->id]);

        $this->assertCount(3, $order->items);
        $this->assertInstanceOf(OrderItem::class, $order->items->first());
    }

    public function test_scope_active_excludes_cancelled(): void
    {
        Order::factory()->create(['status' => 'pending']);
        Order::factory()->create(['status' => 'completed']);
        Order::factory()->create(['status' => 'cancelled']);

        $active = Order::active()->get();

        $this->assertCount(2, $active);
        $active->each(fn ($order) => $this->assertNotEquals('cancelled', $order->status));
    }

    public function test_total_amount_accessor_sums_items(): void
    {
        $order = Order::factory()->create();
        OrderItem::factory()->create(['order_id' => $order->id, 'amount' => 100.50]);
        OrderItem::factory()->create(['order_id' => $order->id, 'amount' => 200.00]);

        $this->assertEquals(300.50, $order->fresh()->totalAmount);
    }

    public function test_metadata_cast_to_array(): void
    {
        $order = Order::factory()->create(['metadata' => ['key' => 'value']]);

        $this->assertIsArray($order->fresh()->metadata);
        $this->assertEquals('value', $order->fresh()->metadata['key']);
    }
}
```

---

### 2.7 Event / Listener — Unit Test

```php
class CreateProjectOnContractSignedTest extends TestCase
{
    use RefreshDatabase;

    public function test_creates_project_when_contract_signed(): void
    {
        // Arrange
        $contract = Contract::factory()->create();
        $event = new ContractSigned($contract);
        $listener = app(CreateProjectOnContractSigned::class);

        // Act
        $listener->handle($event);

        // Assert
        $this->assertDatabaseHas('projects', [
            'contract_id' => $contract->id,
        ]);
    }
}
```

---

### 2.8 Job — Unit Test

```php
class SendPaymentReminderTest extends TestCase
{
    use RefreshDatabase;

    public function test_sends_reminder_to_overdue_payments(): void
    {
        // Arrange
        Mail::fake();
        $payment = Payment::factory()->create([
            'due_date' => now()->subDays(3),
            'status' => 'unpaid',
        ]);

        // Act
        (new SendPaymentReminder($payment))->handle();

        // Assert
        Mail::assertSent(PaymentReminderMail::class, function ($mail) use ($payment) {
            return $mail->hasTo($payment->user->email);
        });
    }
}
```

---

### 2.9 Policy — Unit Test

```php
class OrderPolicyTest extends TestCase
{
    use RefreshDatabase;

    public function test_owner_can_view_own_order(): void
    {
        $user = User::factory()->create();
        $order = Order::factory()->create(['user_id' => $user->id]);

        $this->assertTrue((new OrderPolicy())->view($user, $order));
    }

    public function test_non_owner_cannot_view_order(): void
    {
        $user = User::factory()->create();
        $order = Order::factory()->create(); // 別人的訂單

        $this->assertFalse((new OrderPolicy())->view($user, $order));
    }

    public function test_admin_can_view_any_order(): void
    {
        $admin = User::factory()->create(['role' => 'admin']);
        $order = Order::factory()->create();

        $this->assertTrue((new OrderPolicy())->view($admin, $order));
    }
}
```

---

### 2.10 Adapter — Unit Test

```php
class CloudStorageAdapterTest extends TestCase
{
    public function test_store_delegates_to_cloud_provider_with_correct_parameters(): void
    {
        // Arrange
        $cloud = Mockery::mock(CloudProviderInterface::class);
        $cloud->shouldReceive('upload')
            ->with('my-bucket', 'documents/file.pdf', 'content')
            ->andReturn('https://cdn.example.com/file.pdf');

        $adapter = new CloudStorageAdapter($cloud, 'my-bucket');

        // Act
        $url = $adapter->store('documents/file.pdf', 'content');

        // Assert
        $this->assertEquals('https://cdn.example.com/file.pdf', $url);
    }
}
```

#### 關鍵規則

- 驗證 Adapter 正確地將 Interface A 的呼叫轉換為 Interface B 的呼叫
- 不測業務邏輯（Adapter 不應有業務邏輯）

---

### 2.11 Strategy 實作 — Unit Test

```php
class ExpressShippingTest extends TestCase
{
    public function test_calculates_correct_shipping_fee(): void
    {
        $strategy = new ExpressShipping();

        $fee = $strategy->calculate(weight: 2.5, destination: 'Taipei');

        $this->assertEquals(2.5 * 6.0 + 100, $fee);
    }
}

class StandardShippingTest extends TestCase
{
    public function test_calculates_correct_shipping_fee(): void
    {
        $strategy = new StandardShipping();

        $fee = $strategy->calculate(weight: 2.5, destination: 'Taipei');

        $this->assertEquals(2.5 * 2.5, $fee);
    }
}
```

---

### 2.12 Null Object — Unit Test

```php
class NullExternalSystemTest extends TestCase
{
    private NullExternalSystem $nullSystem;

    protected function setUp(): void
    {
        parent::setUp();
        $this->nullSystem = new NullExternalSystem();
    }

    public function test_push_returns_null(): void
    {
        $result = $this->nullSystem->push(['key' => 'value']);

        $this->assertNull($result);
    }

    public function test_push_does_not_throw_exception(): void
    {
        // 不需要 expectException——能走到最後一行就代表沒拋例外
        $this->nullSystem->push([]);

        $this->assertTrue(true); // 明確表達「沒有拋例外」就是通過條件
    }
}
```

#### 關鍵規則

- 驗證所有方法都回傳安全的預設值（null、空陣列、空字串、void）
- 驗證不會拋出例外
- 驗證不會寫 log 或產生副作用

---

### 2.13 Blade 元件 — Feature Test

```php
class UserCardComponentTest extends TestCase
{
    public function test_renders_user_name_and_email(): void
    {
        $user = new User(['name' => 'Alice', 'email' => 'alice@example.com']);

        $view = $this->component('x-core::user-card', ['user' => $user]);

        $view->assertSee('Alice');
        $view->assertSee('alice@example.com');
    }

    public function test_renders_empty_state_when_no_user(): void
    {
        $view = $this->component('x-core::user-card', ['user' => null]);

        $view->assertSee('尚無資料');
    }
}
```

---

### 2.14 Alpine 元件 — Browser Test（Dusk）

```php
class OrderListBrowserTest extends DuskTestCase
{
    public function test_search_filters_orders_in_real_time(): void
    {
        Order::factory()->create(['serial_no' => 'ORD001']);
        Order::factory()->create(['serial_no' => 'ORD002']);

        $this->browse(function (Browser $browser) {
            $browser->loginAs(User::factory()->create())
                ->visit('/orders')
                ->waitForText('ORD001')
                ->type('@search-input', 'ORD001')
                ->waitUntilMissing('ORD002')
                ->assertSee('ORD001')
                ->assertDontSee('ORD002');
        });
    }
}
```

#### 關鍵規則

- 只對關鍵使用者流程撰寫
- 使用 `dusk` selector（`@search-input`）而非 CSS selector，避免樣式變更導致測試壞掉
- 善用 `waitForText` / `waitUntilMissing` 處理非同步行為

---

## 三、測試替身（Test Doubles）

### 3.1 類型速查表

| 替身類型 | 說明 | 使用時機 |
|---------|------|---------|
| **Mock** | 預設期望的呼叫，驗證互動是否發生 | 需要驗證「是否呼叫了某個方法」（如寄信、觸發事件） |
| **Stub** | 回傳固定值，不驗證互動 | 只需要提供依賴的回傳值，不關心怎麼被呼叫 |
| **Fake** | 輕量級的替代實作（如 in-memory Repository） | 需要一個可運作的替代品，但不想碰真實資源 |
| **Spy** | 事後驗證呼叫紀錄 | 需要先執行再驗證（而非預設期望） |

### 3.2 替身使用原則

#### 你必須

- Mock / Stub 的對象是 **Interface**，不是具體 class
- 優先使用 Laravel 內建的 Fake（`Event::fake()`、`Queue::fake()`、`Mail::fake()`、`Storage::fake()`）
- Stub 用於 Arrange（提供環境），Mock 用於 Assert（驗證行為）

#### 你不可以

- Mock 被測試的 class 本身（那等於沒測）
- Mock Model 的 Eloquent 方法（用真實資料庫 + Factory）
- 過度 Mock 導致測試與實作高度耦合

#### 正確 vs 錯誤

```php
// 錯誤：Mock 具體 class
$mock = Mockery::mock(OrderRepository::class);

// 正確：Mock Interface
$mock = Mockery::mock(OrderRepositoryInterface::class);

// 錯誤：Mock Model 的 query builder
$mock = Mockery::mock(Order::class);
$mock->shouldReceive('where->first')->andReturn($order);

// 正確：用真實資料庫 + Factory
$order = Order::factory()->create(['status' => 'pending']);
$result = $this->repository->findByStatus('pending');
```

---

## 四、Model Factory 設計模式

### 4.1 Factory 結構

**規則：每個 Model 都有對應的 Factory，定義合理的預設值和常用的狀態變體。**

```php
class OrderFactory extends Factory
{
    protected $model = Order::class;

    public function definition(): array
    {
        return [
            'serial_no' => 'ORD' . fake()->unique()->numerify('####'),
            'user_id' => User::factory(),
            'status' => 'pending',
            'amount' => fake()->randomFloat(2, 100, 10000),
        ];
    }

    // 狀態變體——用語意化的方法名表達測試情境
    public function completed(): self
    {
        return $this->state(fn () => [
            'status' => 'completed',
            'completed_at' => now(),
        ]);
    }

    public function cancelled(): self
    {
        return $this->state(fn () => [
            'status' => 'cancelled',
            'cancelled_at' => now(),
        ]);
    }

    public function overdue(): self
    {
        return $this->state(fn () => [
            'due_date' => now()->subDays(7),
            'status' => 'pending',
        ]);
    }
}
```

### 4.2 使用方式

```php
// 預設狀態
$order = Order::factory()->create();

// 指定狀態
$order = Order::factory()->completed()->create();

// 組合狀態
$order = Order::factory()->completed()->overdue()->create();

// 批次建立
$orders = Order::factory()->count(10)->create();

// 覆寫特定欄位
$order = Order::factory()->create(['amount' => 500.00]);

// 建立關聯資料
$order = Order::factory()
    ->has(OrderItem::factory()->count(3))
    ->create();
```

### 4.3 關鍵規則

- Factory 的 `definition()` 回傳合理的預設值，能直接 `create()` 不報錯
- 狀態變體用語意化的方法名（`completed()`、`overdue()`），不用 `withStatus('completed')`
- 關聯的 FK 用 `User::factory()` 自動建立，不寫死 ID
- 不在 Factory 中放業務邏輯——Factory 只負責建立資料

---

## 五、命名規範

### 5.1 測試 Class 命名

| 類別 | 格式 | 範例 |
|------|------|------|
| Unit Test | 被測 Class + `Test` | `OrderServiceTest`, `StoreOrderRequestTest` |
| Feature Test | 被測 Controller + `Test` | `OrderControllerTest` |
| Browser Test | 功能描述 + `BrowserTest` | `OrderListBrowserTest`, `CheckoutFlowBrowserTest` |

### 5.2 測試 Method 命名

**規則：`test_` + 行為描述，用底線分隔，讀起來像一句話。**

```php
// 格式：test_{行為}_{條件}_{預期結果}
public function test_complete_order_updates_status_to_completed(): void { }
public function test_complete_order_fires_order_completed_event(): void { }
public function test_complete_throws_exception_when_order_not_found(): void { }
public function test_store_returns_422_when_validation_fails(): void { }
public function test_owner_can_view_own_order(): void { }
public function test_non_owner_cannot_delete_order(): void { }
```

#### 你不可以

- 用 `testA`、`testB`、`test1` 這種無意義命名
- 用 camelCase 命名測試方法（`testCompleteOrderUpdatesStatus` → `test_complete_order_updates_status`）
- 方法名太長超過 80 字元——精簡描述，不是寫文章

### 5.3 檔案與目錄結構

```
tests/
├── Unit/
│   ├── Services/
│   │   └── OrderServiceTest.php
│   ├── Repositories/
│   │   └── OrderRepositoryTest.php
│   ├── Models/
│   │   └── OrderTest.php
│   ├── Requests/
│   │   └── StoreOrderRequestTest.php
│   ├── Policies/
│   │   └── OrderPolicyTest.php
│   ├── Listeners/
│   │   └── CreateProjectOnContractSignedTest.php
│   ├── Jobs/
│   │   └── SendPaymentReminderTest.php
│   └── Adapters/
│       └── CloudStorageAdapterTest.php
├── Feature/
│   └── Controllers/
│       └── OrderControllerTest.php
└── Browser/
    └── OrderListBrowserTest.php
```

#### 規則

- 目錄結構鏡射 `app/` 的結構
- 測試檔案放在對應類型的目錄下（Unit / Feature / Browser）
- 一個被測 class 對應一個測試 class

---

## 六、速查表

### 6.1 我在測哪一層？

| 我在測… | 測試類型 | 用真實 DB？ | 替身策略 |
|---------|---------|-----------|---------|
| Controller | Feature | 是 | 無（打真實 HTTP） |
| FormRequest | Unit | 視規則而定 | 無 |
| Service | Unit | 否 | Mock RepositoryInterface |
| Repository | Unit（Integration） | 是 | 無 |
| Model | Unit | 是 | 無 |
| Event / Listener | Unit | 視情況 | Mock 依賴的 Service |
| Job | Unit | 視情況 | Mock 依賴的 Service |
| Policy | Unit | 是（建立使用者） | 無 |
| Middleware | Feature | 是 | 無（打真實 HTTP） |
| Adapter | Unit | 否 | Mock 被轉接的 Interface |
| Strategy | Unit | 否 | Mock 外部依賴（若有） |
| Null Object | Unit | 否 | 無 |
| Blade 元件 | Feature | 否 | 無 |
| Alpine 元件 | Browser | 是 | 無 |

### 6.2 常見錯誤

| 錯誤做法 | 正確做法 |
|---------|---------|
| Mock 具體 class | Mock Interface |
| Mock Model / Query Builder | 用真實 DB + Factory |
| 一個測試驗證多個不相關行為 | 一個測試一個行為 |
| 測試之間共享狀態 | 每個測試獨立建立資料 |
| 驗證 method 呼叫次數（測實作） | 驗證結果狀態（測行為） |
| 測試名稱無意義（`test1`） | 用 `test_{行為}_{條件}_{結果}` |
| 只寫 happy path | 涵蓋失敗、邊界、例外情境 |
| 所有測試都是 Feature Test | 遵循測試金字塔，Unit 為主 |
