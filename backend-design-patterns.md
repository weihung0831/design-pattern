# 後端設計原則與設計模式

> 本文件是後端程式碼的上位規範。所有產出的後端程式碼（人寫或 AI 產生）都必須遵循本文件。
> 本文件不描述特定 codebase，而是定義通用的軟體工程準則。
> 前端規範請參閱 [frontend(blade)-design-patterns.md](./frontend(blade)-design-patterns.md)。

---

## 一、設計原則

### 1.1 DRY — Don't Repeat Yourself

**規則：每一份知識在系統中只能有一個權威來源。**

當同一段邏輯出現在兩個以上的地方，就必須抽取到單一來源，讓所有使用方引用它。

#### 你必須

- 新增邏輯前，先搜尋系統中是否已有相同或相似的實作
- 重複出現 2 次以上的邏輯，抽取到適當的位置（Trait、Service、Component、共用函式）
- 常數、設定值、翻譯字串各只定義一次，其餘地方引用

#### 你不可以

- 複製貼上一段程式碼到另一個檔案，只改幾個變數
- 在多個地方各自定義相同的驗證規則、錯誤訊息、商業常數
- 建立「幾乎一樣但又不完全相同」的平行 class，而不是抽取共用部分

#### 正確 vs 錯誤

```php
// 錯誤：兩個 Service 各自寫相同的格式化邏輯
class ReportService
{
    public function export(array $rows): string
    {
        $lines = [];
        foreach ($rows as $row) {
            $lines[] = implode(',', array_map(fn ($v) => '"' . str_replace('"', '""', $v) . '"', $row));
        }
        return implode("\n", $lines);
    }
}

class BackupService
{
    public function dump(array $rows): string
    {
        $lines = [];
        foreach ($rows as $row) {
            // ... 完全相同的 CSV 格式化邏輯，複製貼上
        }
        return implode("\n", $lines);
    }
}

// 正確：抽取共用工具，兩處引用同一來源
class CsvFormatter
{
    public static function format(array $rows): string
    {
        $lines = [];
        foreach ($rows as $row) {
            $lines[] = implode(',', array_map(fn ($v) => '"' . str_replace('"', '""', $v) . '"', $row));
        }
        return implode("\n", $lines);
    }
}

class ReportService
{
    public function export(array $rows): string
    {
        return CsvFormatter::format($rows);
    }
}
```

#### 例外

若兩處邏輯「目前相似但業務語意不同、未來會各自演化」，允許暫時重複。過早抽取反而製造不必要的耦合。

---

### 1.2 SOLID

#### S — Single Responsibility Principle（單一職責）

**規則：一個 class 只有一個改變的理由。**

如果一個 class 因為「驗證規則要改」和「PDF 排版要調」這兩個毫不相關的原因都需要修改，它就承擔了過多職責。

##### 你必須

- 每個 class 只負責一個明確的職責領域
- 需要新職責時建立新的 class，不在現有 class 上堆功能
- 層級間職責嚴格劃分：

目標結構：

```
Controller → ServiceInterface → Service → RepositoryInterface → Repository → Model
                ↑                              ↑
            ServiceProvider 綁定 (Factory Pattern)
```

| 層級 | 唯一職責 | 不該出現的 |
|------|---------|-----------|
| Controller | 接收 HTTP 請求、回傳 HTTP 回應 | 業務邏輯、資料庫 transaction |
| FormRequest | 輸入驗證、錯誤訊息 | 業務邏輯、資料寫入 |
| ServiceInterface | 定義業務操作的契約 | 實作細節 |
| Service | 業務流程、transaction 管理 | HTTP 相關操作（request / response）、直接操作 Model（必須透過 Repository） |
| RepositoryInterface | 定義資料存取的契約 | 實作細節、業務邏輯 |
| Repository | 資料存取、查詢邏輯 | 業務流程、HTTP 操作 |
| Model | 資料結構、關聯定義、scope、型別轉換 | 業務流程、HTTP 操作、複雜查詢邏輯（joins、subquery 等屬 Repository） |
| ServiceProvider | 綁定 Interface → 實作（Factory Pattern） | 業務邏輯 |

##### 正確 vs 錯誤

```php
// 錯誤：Controller 同時做驗證、業務邏輯、寄信
class ProductController
{
    public function store(Request $request)
    {
        // 驗證（應在 FormRequest）
        $request->validate(['name' => 'required', 'price' => 'numeric']);

        // 業務邏輯（應在 Service）
        $product = new Product($request->all());
        $product->sku = 'SKU-' . Str::random(8);
        $product->save();
        $product->categories()->sync($request->category_ids);

        // 寄通知信（應在 NotificationService 或 Event）
        Mail::to($request->user())->send(new ProductCreated($product));

        return response()->json($product);
    }
}

// 正確：各司其職，依賴 Interface 而非具體 class
class ProductController
{
    public function __construct(private ProductServiceInterface $service) {}

    public function store(StoreProductRequest $request): JsonResponse
    {
        $product = $this->service->create($request->validated());

        return response()->json(['data' => $product], 201);
    }
}

class ProductService implements ProductServiceInterface
{
    public function __construct(private ProductRepositoryInterface $repository) {}

    public function create(array $data): Product
    {
        return DB::transaction(function () use ($data) {
            $data['sku'] = 'SKU-' . Str::random(8);
            $product = $this->repository->create($data);
            $this->repository->syncCategories($product, $data['category_ids'] ?? []);

            event(new ProductCreated($product));

            return $product;
        });
    }
}

class ProductRepository implements ProductRepositoryInterface
{
    public function create(array $data): Product
    {
        return Product::create($data);
    }

    public function syncCategories(Product $product, array $categoryIds): void
    {
        $product->categories()->sync($categoryIds);
    }
}
```

---

#### O — Open/Closed Principle（開放封閉）

**規則：對擴充開放，對修改封閉。新增行為不需要改動既有程式碼。**

##### 你必須

- 可預見會有多種實作的行為，定義成 Interface
- 新增實作時，建立新的 class 來實作 Interface，不修改原本的 class
- 用依賴注入容器切換實作，不用 if/else 判斷

##### 正確 vs 錯誤

```php
// 錯誤：每新增一種通知方式就要改這個方法
class NotificationService
{
    public function send(string $channel, string $message): void
    {
        if ($channel === 'email') {
            // 寄 email
        } elseif ($channel === 'sms') {
            // 發簡訊
        } elseif ($channel === 'slack') {
            // 發 Slack — 每次都要改這裡
        }
    }
}

// 正確：新增通知方式只需新增一個 class
interface NotificationChannelInterface
{
    public function send(string $message): void;
}

class EmailChannel implements NotificationChannelInterface
{
    public function send(string $message): void { /* ... */ }
}

class SlackChannel implements NotificationChannelInterface
{
    public function send(string $message): void { /* ... */ }
}

// 新增 LINE 通知？建立 LineChannel implements NotificationChannelInterface 即可
// NotificationService 不需要任何修改
```

---

#### L — Liskov Substitution Principle（里氏替換）

**規則：任何使用父類別 / Interface 的地方，替換成其子類別 / 實作，行為必須正確且不意外。**

##### 你必須

- Interface 的所有實作都遵守相同的契約（參數型別、回傳型別、例外行為）
- 空實作（Null Object）也必須是安全的——回傳合理的預設值，不拋出例外

##### 正確 vs 錯誤

```php
interface PaymentGatewayInterface
{
    /** @return string Transaction ID */
    public function charge(float $amount): string;
}

// 正確：所有實作都回傳 Transaction ID
class StripeGateway implements PaymentGatewayInterface
{
    public function charge(float $amount): string
    {
        return $this->stripe->charges->create($amount)->id;
    }
}

class NullGateway implements PaymentGatewayInterface
{
    public function charge(float $amount): string
    {
        return 'null-' . uniqid(); // 安全的空操作，回傳符合契約的值
    }
}

// 錯誤：替換後行為不一致
class BrokenGateway implements PaymentGatewayInterface
{
    public function charge(float $amount): string
    {
        throw new \Exception('Not implemented'); // 呼叫者沒預期會拋例外
    }
}
```

---

#### I — Interface Segregation Principle（介面隔離）

**規則：不強迫實作者依賴它不需要的方法。Interface 要小而專注。**

##### 你必須

- 一個 Interface 只定義一組高度相關的方法
- 不同關注點拆成不同 Interface，讓實作者只實作需要的

##### 正確 vs 錯誤

```php
// 錯誤：一個肥大的 Interface，實作者被迫實作不相關的方法
interface DocumentInterface
{
    public function create(array $data): Model;
    public function generatePdf(int $id): string;
    public function sendEmail(int $id, string $to): void;
    public function archive(int $id): void;
}

// 正確：拆成獨立的小 Interface
interface DocumentCreatorInterface
{
    public function create(array $data): Model;
}

interface PdfGeneratorInterface
{
    public function generate(int $id): string;
}

interface DocumentArchiverInterface
{
    public function archive(int $id): void;
}
```

##### 判斷基準

一個 Interface 超過 3 個方法時，審視這些方法是否真的屬於同一個職責。如果不同的實作者只會用到其中一部分，就該拆。

---

#### D — Dependency Inversion Principle（依賴反轉）

**規則：高層模組不依賴低層模組，兩者都依賴抽象（Interface）。**

##### 你必須

- Service 的外部依賴透過建構子注入 Interface，不直接 `new` 具體 class
- 由依賴注入容器（ServiceProvider）負責綁定 Interface → 實作
- 跨模組依賴必須透過 Interface，不直接引用其他模組的具體 class

##### 正確 vs 錯誤

```php
// 錯誤：直接依賴具體 class
class OrderService
{
    public function complete(Order $order): void
    {
        $notifier = new SlackNotifier(); // 寫死了具體實作
        $notifier->notify("Order {$order->id} completed");
    }
}

// 正確：依賴 Interface，由容器注入
class OrderService implements OrderServiceInterface
{
    public function __construct(
        private NotificationChannelInterface $notifier,
    ) {}

    public function complete(Order $order): void
    {
        $this->notifier->send("Order {$order->id} completed");
    }
}

// ServiceProvider 負責接線
$this->app->bind(NotificationChannelInterface::class, SlackNotifier::class);
```

---

## 二、設計模式

以下列出後端開發中應採用的設計模式。每個模式說明「什麼時候用」和「怎麼用」。

### 2.1 Null Object Pattern（空物件模式）

**用途：當某個依賴「可能不存在」時，提供安全的空實作取代 null 檢查。**

#### 何時使用

- 模組 A 定義了一個擴充點，但擴充方（模組 B）不一定存在
- 某個 Service 的依賴在特定環境下不可用（例如第三方 API）

#### 結構

```php
// 1. 定義 Interface
interface ExternalSystemInterface
{
    public function push(array $data): ?int;
}

// 2. 正式實作
class ActualExternalSystem implements ExternalSystemInterface
{
    public function push(array $data): ?int
    {
        // 真正呼叫外部系統
        return $externalId;
    }
}

// 3. Null Object — 安全的空操作
class NullExternalSystem implements ExternalSystemInterface
{
    public function push(array $data): ?int
    {
        return null; // 不做任何事，不拋例外
    }
}

// 4. ServiceProvider 預設綁定 Null，有擴充時覆寫
if (! $this->app->bound(ExternalSystemInterface::class)) {
    $this->app->bind(ExternalSystemInterface::class, NullExternalSystem::class);
}
```

#### 關鍵規則

- Null Object 的每個方法都必須是 **安全的無操作**（回傳 null、空陣列、void）
- 不可在 Null Object 中拋出例外或寫 log 警告
- 消費端的程式碼不需要檢查 `if ($service !== null)`——Null Object 就是為了消除這種檢查

---

### 2.2 Strategy Pattern（策略模式）

**用途：同一個行為有多種演算法 / 實作方式，在執行時期切換。**

#### 何時使用

- 同一個操作依據條件有不同的處理方式（例如：不同國家的稅金計算、不同格式的匯出）
- 未來可能新增更多處理方式

#### 結構

```php
// 定義策略 Interface
interface ShippingCalculatorInterface
{
    public function calculate(float $weight, string $destination): float;
}

// 各種策略實作
class StandardShipping implements ShippingCalculatorInterface
{
    public function calculate(float $weight, string $destination): float
    {
        return $weight * 2.5;
    }
}

class ExpressShipping implements ShippingCalculatorInterface
{
    public function calculate(float $weight, string $destination): float
    {
        return $weight * 6.0 + 100;
    }
}

// 消費端只依賴 Interface
class CheckoutService
{
    public function __construct(private ShippingCalculatorInterface $shipping) {}

    public function calculateTotal(float $subtotal, float $weight, string $dest): float
    {
        return $subtotal + $this->shipping->calculate($weight, $dest);
    }
}
```

#### 與 if/else 的區別

- if/else 把所有策略塞在同一個方法裡 → 違反 OCP
- Strategy 讓每個策略是獨立的 class → 新增策略不改既有程式碼

---

### 2.3 Adapter Pattern（轉接器模式）

**用途：讓兩個不相容的 Interface 能夠協作，中間加一層轉接。**

#### 何時使用

- 模組 A 定義了 Interface X，模組 B 提供了 Interface Y 的能力，兩者簽名不同但語意相通
- 整合第三方套件，其 API 與系統內部 Interface 不一致

#### 結構

```php
// 模組 A 的 Interface（不知道模組 B 的存在）
interface FileStorageInterface
{
    public function store(string $path, string $content): string;
}

// 模組 B 提供的能力（自己的 Interface）
interface CloudProviderInterface
{
    public function upload(string $bucket, string $key, string $body): string;
}

// Adapter：把 A 的呼叫轉接到 B
class CloudStorageAdapter implements FileStorageInterface
{
    public function __construct(
        private CloudProviderInterface $cloud,
        private string $bucket,
    ) {}

    public function store(string $path, string $content): string
    {
        return $this->cloud->upload($this->bucket, $path, $content);
    }
}
```

#### 關鍵規則

- Adapter 只做「轉換」，不加業務邏輯
- Adapter 是唯一同時引用兩邊 Interface 的地方

---

### 2.4 Factory Pattern（工廠模式）

**用途：將物件的建構邏輯集中管理，消費端不需要知道建構細節。**

#### 何時使用

- 物件的建構需要複雜的初始化邏輯
- 根據條件建構不同的實作
- 測試時需要快速產生帶有特定狀態的物件

#### 結構一：ServiceProvider 作為工廠

```php
// ServiceProvider 集中管理建構邏輯
class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->bind(PaymentGatewayInterface::class, function ($app) {
            return new StripeGateway(
                config('services.stripe.key'),
                config('services.stripe.secret'),
            );
        });
    }
}

// 消費端不需要知道建構細節
class PaymentService
{
    public function __construct(private PaymentGatewayInterface $gateway) {}
}
```

#### 結構二：Model Factory（測試用）

```php
class OrderFactory extends Factory
{
    protected $model = Order::class;

    public function definition(): array
    {
        return [
            'serial_no' => 'ORD' . fake()->unique()->numerify('####'),
            'status' => 'pending',
            'amount' => fake()->randomFloat(2, 100, 10000),
        ];
    }

    // 狀態變體——測試不同情境
    public function completed(): self
    {
        return $this->state(fn () => [
            'status' => 'completed',
            'completed_at' => now(),
        ]);
    }
}

// 使用
Order::factory()->create();                    // 預設狀態
Order::factory()->completed()->create();       // completed 狀態
Order::factory()->count(10)->create();         // 批次建立
```

---

### 2.5 Template Method Pattern（模板方法模式）

**用途：定義演算法的骨架，讓子類別（或實作方）填入具體步驟。**

#### 何時使用

- 多個 class 有相同的流程骨架，但個別步驟不同
- 用 Trait 定義流程，讓使用方實作抽象方法

#### 結構（以 Trait 實現）

```php
trait HasAuditLog
{
    // 模板方法：固定的流程骨架
    public function writeAuditLog(string $action): AuditLog
    {
        $snapshot = $this->toAuditSnapshot(); // 抽象步驟——由使用方實作

        return AuditLog::create([
            'auditable_type' => static::class,
            'auditable_id' => $this->id,
            'action' => $action,
            'snapshot' => $snapshot,
            'performed_by' => auth()->id(),
        ]);
    }

    // 抽象步驟——使用方必須實作
    abstract public function toAuditSnapshot(): array;
}

// 使用方只負責定義「要記錄哪些欄位」
class Account extends Model
{
    use HasAuditLog;

    public function toAuditSnapshot(): array
    {
        return [
            'balance' => $this->balance,
            'status' => $this->status,
            'owner' => $this->owner_name,
        ];
    }
}

class Permission extends Model
{
    use HasAuditLog;

    public function toAuditSnapshot(): array
    {
        return [
            'role' => $this->role_name,
            'level' => $this->access_level,
        ];
    }
}
```

---

### 2.6 Repository Pattern（儲存庫模式）

**用途：將資料存取邏輯從 Service 中抽離，讓 Service 專注於業務流程，Repository 專注於資料操作。**

#### 何時使用

- 所有對 Model 的 CRUD 和查詢操作（這是架構的強制要求，不是可選模式）
- Service 需要存取資料時，必須透過 RepositoryInterface，不直接操作 Model

#### 結構

```php
// 1. 定義 RepositoryInterface
interface OrderRepositoryInterface
{
    public function find(int $id): ?Order;
    public function findBySerialNo(string $serialNo): ?Order;
    public function getByStatus(string $status): Collection;
    public function create(array $data): Order;
    public function update(Order $order, array $data): Order;
    public function delete(Order $order): void;
}

// 2. 實作 Repository
class OrderRepository implements OrderRepositoryInterface
{
    public function find(int $id): ?Order
    {
        return Order::find($id);
    }

    public function findBySerialNo(string $serialNo): ?Order
    {
        return Order::where('serial_no', $serialNo)->first();
    }

    public function getByStatus(string $status): Collection
    {
        return Order::where('status', $status)
            ->with('items')
            ->orderByDesc('created_at')
            ->get();
    }

    public function create(array $data): Order
    {
        return Order::create($data);
    }

    public function update(Order $order, array $data): Order
    {
        $order->update($data);
        return $order->refresh();
    }

    public function delete(Order $order): void
    {
        $order->delete();
    }
}

// 3. ServiceProvider 綁定
$this->app->bind(OrderRepositoryInterface::class, OrderRepository::class);

// 4. Service 透過 Interface 使用
class OrderService implements OrderServiceInterface
{
    public function __construct(private OrderRepositoryInterface $orderRepository) {}

    public function complete(int $orderId): Order
    {
        $order = $this->orderRepository->find($orderId);

        return DB::transaction(function () use ($order) {
            $order = $this->orderRepository->update($order, [
                'status' => 'completed',
                'completed_at' => now(),
            ]);

            event(new OrderCompleted($order));

            return $order;
        });
    }
}
```

#### 關鍵規則

- **一個 Repository 對應一個 Model**——`OrderRepository` 只操作 `Order`，不直接操作 `Payment`。若業務流程需要操作多個 Model，由 Service 協調多個 Repository
- **只有 Repository 可以直接操作 Model**——Service、Controller 都不可以
- **Repository 不包含業務邏輯**——判斷、流程控制、transaction 都屬於 Service

#### Repository vs Model 的職責分界

| 該放在 Model | 該放在 Repository |
|-------------|------------------|
| 關聯定義（`hasMany`, `belongsTo`） | 帶條件的查詢（`where`, `orderBy`） |
| 簡單 scope（`scopeActive`） | 複雜查詢（joins、subquery） |
| Accessor / Mutator | Eager loading 組合（`with`） |
| 型別轉換（`$casts`） | CRUD 操作 |

#### Method 命名慣例

| 操作 | 命名格式 | 範例 |
|------|---------|------|
| 依主鍵查詢 | `find` | `find($id)` |
| 依條件查詢單筆 | `findBy` + 欄位 | `findBySerialNo($serialNo)` |
| 依條件查詢多筆 | `getBy` + 條件 / `getAll` | `getByStatus($status)`, `getAll()` |
| 新增 | `create` | `create(array $data)` |
| 更新 | `update` | `update(Order $order, array $data)` |
| 刪除 | `delete` | `delete(Order $order)` |
| 檢查存在 | `exists` + 條件 | `existsByEmail(string $email)` |

---

### 2.7 模式選用速查表

| 你遇到的情境 | 使用的模式 |
|-------------|-----------|
| 資料存取需要從 Service 抽離 | **Repository**（架構強制） |
| 依賴可能不存在，需要安全的預設行為 | **Null Object** |
| 同一操作有多種實作方式，需要切換 | **Strategy** |
| 兩個模組的 Interface 不相容，需要橋接 | **Adapter** |
| 物件建構邏輯複雜或需要集中管理 | **Factory** |
| 多個 class 有相同流程但個別步驟不同 | **Template Method** |
| 新增行為不想改既有 class | **Strategy** 或 **Null Object**（搭配 Interface + DI） |

---

## 三、命名規範

> 命名是程式碼可讀性的基礎。好的名稱讓程式碼自解釋，壞的名稱讓所有人猜。

### 3.1 通用原則

#### 語意優先，長度其次

名稱的首要目標是**傳達意圖**。短但模糊的名稱不如長但清楚的名稱。

```php
// 錯誤：短但無意義
$d = Carbon::now()->diffInDays($contract->end_date);
$flag = $user->role === 'admin';
$tmp = $this->calculate($items);

// 正確：看名稱就知道是什麼
$daysUntilExpiry = Carbon::now()->diffInDays($contract->end_date);
$isAdmin = $user->role === 'admin';
$totalAmount = $this->calculate($items);
```

#### 一致性勝過個人偏好

同一個概念在整個系統中只能有一個名稱。不要在 A 檔案叫 `user`、B 檔案叫 `member`、C 檔案叫 `account` 指的都是同一件事。

```php
// 錯誤：同一概念多種叫法
$client = $this->userRepository->find($id);   // Service A
$member = $this->userRepository->find($id);   // Service B
$account = $this->userRepository->find($id);  // Service C

// 正確：統一用一個名稱
$user = $this->userRepository->find($id);     // 全系統一致
```

#### 不縮寫，除非是廣泛公認的縮寫

```php
// 可接受的縮寫
$id, $url, $api, $html, $css, $pdf, $ip, $db

// 不可接受的縮寫
$usr, $mgr, $btn, $msg, $qty, $amt, $addr, $idx, $cnt, $tmp, $cfg
// 應該寫成
$user, $manager, $button, $message, $quantity, $amount, $address, $index, $count, $temporary, $config
```

**例外**：迴圈中的迭代變數（`$i`、`$j`）、Lambda 中的短暫參數（`fn ($v) => ...`）在作用域極小時可接受。

---

### 3.2 PHP

#### 總覽表

| 類別 | 格式 | 範例 |
|------|------|------|
| Class | PascalCase | `QuotationService`, `StoreLeadRequest` |
| Method | camelCase | `calculateTotal()`, `sendReminder()` |
| Variable | camelCase | `$totalAmount`, `$isActive` |
| Property | camelCase | `$this->createdBy`, `$this->itemCount` |
| Constant | UPPER_SNAKE_CASE | `self::MAX_RETRY_COUNT`, `self::STATUS_ACTIVE` |
| Enum Case | PascalCase | `DocumentStatus::Approved`, `PaymentType::CreditCard` |
| Trait | PascalCase，形容詞或 Has 開頭 | `HasCalendarEvents`, `Searchable`, `Auditable` |
| Interface | PascalCase，以 Interface 結尾 | `ContractLifecycleInterface`, `PaymentGatewayInterface` |

#### Class 命名

遵循「名詞 + 職責後綴」的結構：

| 職責 | 後綴 | 範例 |
|------|------|------|
| 業務邏輯契約 | `ServiceInterface` | `QuotationServiceInterface`, `ContractServiceInterface` |
| 業務邏輯 | `Service` | `QuotationService`, `ContractService` |
| 資料存取契約 | `RepositoryInterface` | `QuotationRepositoryInterface`, `LeadRepositoryInterface` |
| 資料存取 | `Repository` | `QuotationRepository`, `LeadRepository` |
| HTTP 控制器 | `Controller` | `LeadController`, `PaymentController` |
| 表單驗證 | `Request`，動詞開頭 | `StoreLeadRequest`, `UpdateContractRequest` |
| Eloquent 模型 | 無後綴，單數 | `User`, `Quotation`, `Contract` |
| 資料庫工廠 | `Factory` | `UserFactory`, `LeadFactory` |
| 資料庫播種 | `Seeder` | `RoleSeeder`, `UserSeeder` |
| 事件 | 過去式 | `ContractSigned`, `PaymentReceived` |
| 監聽器 | 動作描述 | `CreateProjectOnContractSigned` |
| Job | 動詞開頭 | `SendPaymentReminder`, `ImportFacebookLeads` |
| Policy | 模型名 + `Policy` | `LeadPolicy`, `ContractPolicy` |
| Enum | 名詞 | `DocumentStatus`, `PaymentType`, `CalendarEventType` |
| Middleware | 動詞或形容詞 | `AuthMiddleware`, `EnsureLocale` |

#### Method 命名

動詞開頭，表達這個方法「做什麼」：

```php
// CRUD 操作
public function create(array $data): Model;
public function update(Model $model, array $data): Model;
public function delete(Model $model): void;

// 查詢
public function findBySerialNo(string $serialNo): ?Quotation;
public function getActiveContracts(): Collection;

// 業務操作
public function approve(Contract $contract): void;
public function calculateTotal(Quotation $quotation): float;
public function sendReminder(Payment $payment): void;

// 布林查詢——is / has / can / should 開頭
public function isExpired(): bool;
public function hasPermission(string $permission): bool;
public function canBeEdited(): bool;
public function shouldSendNotification(): bool;
```

#### 你不可以

- Method 用名詞命名（`$service->total()` → 應該是 `$service->calculateTotal()` 或 getter `$service->getTotal()`）
- 用 `get` 前綴包裝單純的屬性存取（Eloquent accessor 用 attribute 定義，不加 `get`）
- Boolean method 不加 `is`/`has`/`can`/`should` 前綴（`$user->admin()` → `$user->isAdmin()`）

---

### 3.3 資料庫

#### 總覽表

| 類別 | 格式 | 範例 |
|------|------|------|
| 資料表 | snake_case，**複數** | `users`, `quotation_items`, `calendar_events` |
| 欄位 | snake_case | `created_at`, `serial_no`, `total_amount` |
| 外鍵 | 關聯模型單數 + `_id` | `user_id`, `contract_id`, `project_id` |
| Polymorphic | `*able_type` + `*able_id` | `eventable_type`, `eventable_id` |
| 布林欄位 | `is_` 開頭或形容詞 | `is_active`, `is_archived`, `approved` |
| 日期欄位 | `*_at` 或 `*_date` | `signed_at`, `due_date`, `expired_at` |
| Pivot 表 | 兩個模型單數，字母序，底線連接 | `contract_user`, `permission_role` |
| 索引 | 自動命名即可（Laravel convention） | — |

#### 你不可以

- 資料表用單數（`user` → `users`）
- 欄位用 camelCase（`createdAt` → `created_at`）
- 外鍵不加 `_id` 後綴（`user` → `user_id`）
- 布林欄位用動詞（`activate` → `is_active`）
- 資料表名含模組前綴（`bm_leads` → `leads`），除非確實有跨模組衝突

---

### 3.4 路由

#### URL 路徑

```
# 格式：kebab-case，名詞複數
GET    /business-management/quotation-items
POST   /business-management/quotation-items
GET    /business-management/quotation-items/{id}
PUT    /business-management/quotation-items/{id}
DELETE /business-management/quotation-items/{id}
```

#### Route Name

```php
// 格式：dot.separated，模組前綴
Route::name('business-management.')->group(function () {
    Route::resource('leads', LeadController::class);
    // 產生：business-management.leads.index, .create, .store, .show, .edit, .update, .destroy
});
```

#### 你不可以

- URL 用 camelCase 或 snake_case（`/quotationItems` → `/quotation-items`）
- URL 用動詞（`/getUsers`、`/createOrder` → 用 HTTP method + 名詞）
- Route name 用 kebab-case（`business-management.quotation-items.index` → `business-management.quotation_items.index` 或遵循 resource 自動產生的格式）

---

### 3.5 檔案命名

| 類別 | 格式 | 範例 |
|------|------|------|
| Class 檔案 | PascalCase，與 class 同名 | `QuotationService.php`, `StoreLeadRequest.php` |
| Migration | Laravel 預設格式 | `2026_01_15_000000_create_leads_table.php` |
| Config | kebab-case 或 snake_case | `business.php`, `pdf.php` |
| 翻譯檔 | snake_case | `common.php`, `quotation.php` |

#### 你不可以

- PHP 檔案用 kebab-case 或 snake_case（`quotation-service.php` → `QuotationService.php`）
- 目錄名用複數（`Controllers/` 除外，那是 Laravel convention）

---

### 3.6 命名速查表

| 我在寫… | 格式 | 前綴/後綴規則 |
|---------|------|--------------|
| PHP Class | PascalCase | 加職責後綴（Service, Repository, Controller, Request...） |
| PHP Interface | PascalCase | 以 Interface 結尾（ServiceInterface, RepositoryInterface...） |
| PHP Method | camelCase | 動詞開頭 |
| PHP Variable | camelCase | 布林加 `$is`/`$has`/`$can` |
| PHP Constant | UPPER_SNAKE_CASE | — |
| Enum Case | PascalCase | — |
| DB Table | snake_case 複數 | — |
| DB Column | snake_case | 外鍵加 `_id`，日期加 `_at`/`_date` |
| URL Path | kebab-case | 名詞複數 |
| Route Name | dot.separated | 模組前綴 |
| PHP 檔案 | PascalCase | 與 class 同名 |
