# Part 61: Domain-Driven Design (DDD) บน C#

> Steps 601-610 | ระดับ: Advanced | เวลาประมาณ: 3-4 ชั่วโมง

---

## สารบัญ

- [Step 601: DDD Overview](#step-601-ddd-overview)
- [Step 602: Aggregates & Aggregate Roots](#step-602-aggregates--aggregate-roots)
- [Step 603: Value Objects](#step-603-value-objects)
- [Step 604: Domain Services](#step-604-domain-services)
- [Step 605: Domain Events](#step-605-domain-events)
- [Step 606: Repositories แบบ DDD](#step-606-repositories-แบบ-ddd)
- [Step 607-610: การ Implement DDD จริง (E-Commerce Order Context)](#step-607-610-การ-implement-ddd-จริง-e-commerce-order-context)
- [สรุป](#สรุป)

---

## Step 601: DDD Overview

### DDD คืออะไร?

**Domain-Driven Design (DDD)** คือแนวคิดในการออกแบบซอฟต์แวร์ที่เน้นการสร้างโมเดลให้สอดคล้องกับ "โดเมน" หรือขอบเขตของปัญหาทางธุรกิจ  
แนวคิดนี้ริเริ่มโดย **Eric Evans** ในหนังสือ "Domain-Driven Design: Tackling Complexity in the Heart of Software" (2003)

หัวใจของ DDD คือการที่นักพัฒนาและผู้เชี่ยวชาญด้านธุรกิจ (Domain Expert) ต้องทำงานร่วมกันอย่างใกล้ชิด เพื่อให้โค้ดสะท้อนความจริงของธุรกิจออกมาอย่างชัดเจน

---

### Strategic DDD vs Tactical DDD

DDD แบ่งออกเป็น 2 ระดับ:

#### Strategic DDD (ระดับสูง - ภาพรวมทั้งระบบ)
เน้นการแบ่งระบบขนาดใหญ่ให้เป็นส่วนย่อยๆ ที่จัดการได้ง่าย ประกอบด้วย:

| แนวคิด | คำอธิบาย |
|--------|----------|
| **Bounded Context** | ขอบเขตที่ชัดเจนของโมเดลและภาษา |
| **Ubiquitous Language** | ภาษากลางที่ทุกคนในทีมใช้ร่วมกัน |
| **Context Map** | แผนที่แสดงความสัมพันธ์ระหว่าง Bounded Context |

#### Tactical DDD (ระดับล่าง - การ Implement จริง)
เน้นรูปแบบการเขียนโค้ดภายใน Bounded Context:

| แนวคิด | คำอธิบาย |
|--------|----------|
| **Entities** | Object ที่มี Identity ไม่เปลี่ยนแปลง |
| **Value Objects** | Object ที่ Immutable เปรียบเทียบด้วยค่า |
| **Aggregates** | กลุ่มของ Object ที่จัดการร่วมกัน |
| **Domain Events** | เหตุการณ์ที่เกิดขึ้นในโดเมน |
| **Repositories** | ต้นแบบสำหรับเข้าถึงข้อมูล |
| **Domain Services** | Logic ที่ไม่เหมาะกับ Entity ใดๆ |

---

### Bounded Contexts คืออะไร?

**Bounded Context** คือขอบเขตที่ชัดเจนซึ่ง "โมเดล" และ "ภาษา" มีความหมายเฉพาะของมันเอง

ตัวอย่าง: คำว่า "Product" อาจมีความหมายต่างกันใน Context ที่ต่างกัน:
- ใน **Catalog Context**: Product มี Name, Description, Images, Category
- ใน **Inventory Context**: Product มี StockQuantity, ReorderPoint, Supplier
- ใน **Pricing Context**: Product มี BasePrice, DiscountRules, TaxCategory

ทั้งหมดนี้คือ "Product" แต่คนละ Context คนละโมเดล!

---

### Ubiquitous Language

**Ubiquitous Language** คือภาษาร่วมที่นักพัฒนาและ Domain Expert ใช้คุยกัน และภาษานี้ต้องปรากฏในโค้ดด้วย

ตัวอย่างในธุรกิจ E-Commerce:
- ❌ ไม่ควรใช้: `AddItemToCart()`, `UpdateOrderStatus(3)`, `ProcessPayment()`
- ✓ ควรใช้: `PlaceOrder()`, `ConfirmOrder()`, `ShipOrder()`, `CancelOrder()`

```csharp
// ไม่ดี - ใช้ภาษาทางเทคนิค ไม่ใช่ภาษาธุรกิจ
public void UpdateStatus(int statusCode) { ... }
public void AddRow(ProductId id, int qty, decimal price) { ... }

// ดี - ใช้ Ubiquitous Language
public void ConfirmOrder() { ... }
public void AddItem(ProductId productId, Quantity quantity, Money unitPrice) { ... }
```

---

### Context Map: Upstream/Downstream, ACL, Conformist

**Context Map** แสดงความสัมพันธ์ระหว่าง Bounded Contexts ในระบบ

#### รูปแบบความสัมพันธ์:

**Upstream/Downstream**: Context หนึ่งพึ่งพาอีก Context หนึ่ง
- **Upstream (U)**: ผู้ส่ง ไม่ขึ้นกับ Downstream
- **Downstream (D)**: ผู้รับ ต้องปรับตัวตาม Upstream

**Anti-Corruption Layer (ACL)**: Layer กันชนที่แปลงโมเดลจาก Context อื่น ไม่ให้ "ปนเปื้อน" โมเดลของเรา

**Conformist**: Downstream ยอมรับโมเดลของ Upstream ทั้งหมด โดยไม่มี ACL

```
┌──────────────────────────────────────────────────────────┐
│                    E-Commerce System                      │
│                                                           │
│  ┌─────────────┐   U/D    ┌──────────────────────────┐   │
│  │   Catalog   │ ──────>  │        Orders            │   │
│  │   Context   │          │        Context           │   │
│  └─────────────┘          └──────────┬───────────────┘   │
│                                      │ U/D                │
│  ┌─────────────┐           ┌─────────▼──────────────┐    │
│  │  Payments   │ <──ACL──  │       Shipping         │    │
│  │   Context   │           │       Context          │    │
│  └─────────────┘           └────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

---

### ตัวอย่าง: E-Commerce กับ Bounded Contexts หลายอัน

ระบบ E-Commerce ขนาดใหญ่แบ่งเป็น 4 Bounded Contexts หลัก:

```
1. Catalog Context
   - Product, Category, Brand
   - Search, Browse, Filter
   - ดูแลโดยทีม Product

2. Orders Context
   - Order, OrderLine, Customer
   - PlaceOrder, ConfirmOrder, CancelOrder
   - ดูแลโดยทีม Commerce

3. Shipping Context
   - Shipment, Delivery, Address
   - PrepareShipment, TrackShipment
   - ดูแลโดยทีม Logistics

4. Payments Context
   - Payment, Refund, Invoice
   - ProcessPayment, IssueRefund
   - ดูแลโดยทีม Finance
```

แต่ละ Context มีฐานข้อมูลของตัวเอง มี Team ของตัวเอง และ Deploy แยกกัน (Microservices)

---

## Step 602: Aggregates & Aggregate Roots

### Aggregate คืออะไร?

**Aggregate** คือกลุ่มของ Domain Object (Entities + Value Objects) ที่จัดการร่วมกันในฐานะหน่วยเดียว ทุก Operation ที่เปลี่ยนแปลงสถานะต้องเป็น Consistent เสมอ

### Aggregate Root คืออะไร?

**Aggregate Root** คือ Entity หลักที่เป็น "ประตูเข้า" ของ Aggregate ทุกการเปลี่ยนแปลงใน Aggregate ต้องผ่าน Aggregate Root เท่านั้น

#### กฎสำคัญของ Aggregate:

1. **มีแค่ Aggregate Root เดียว** ต่อ Aggregate
2. **ภายนอกอ้างอิง** Aggregate อื่นด้วย ID เท่านั้น (ไม่อ้าง Object โดยตรง)
3. **Invariant** (กฎธุรกิจ) ต้องถูก Enforce ภายใน Aggregate เสมอ
4. **Save/Load ทั้ง Aggregate** ในคราวเดียว

```csharp
// ❌ ผิด - เข้าถึง OrderLine โดยตรงจากภายนอก
var line = orderLineRepository.GetById(lineId); // ไม่ควรมี!
line.UpdateQuantity(5);

// ✓ ถูก - ทำผ่าน Aggregate Root (Order)
var order = orderRepository.GetById(orderId);
order.UpdateItemQuantity(lineId, new Quantity(5));
orderRepository.Save(order);
```

---

### Invariant Enforcement

**Invariant** คือกฎธุรกิจที่ต้องเป็นจริงเสมอตลอดอายุของ Aggregate

ตัวอย่าง Invariants ของ Order:
- Order ที่ Cancelled แล้วจะ Add Item เพิ่มไม่ได้
- Order ต้องมี Order Line อย่างน้อย 1 รายการจึงจะ Confirm ได้
- ยอดรวมของ Order ต้องเท่ากับผลรวมของ OrderLine ทั้งหมด

```csharp
public class Order : AggregateRoot
{
    private readonly List<OrderLine> _lines = new();
    private OrderStatus _status;

    public void AddItem(ProductId productId, Quantity quantity, Money unitPrice)
    {
        // Enforce Invariant: ห้าม Add ถ้า Order ถูก Cancel แล้ว
        if (_status == OrderStatus.Cancelled)
            throw new DomainException("Cannot add items to a cancelled order.");

        // Enforce Invariant: ห้าม Add ถ้า Order ถูก Confirm แล้ว
        if (_status == OrderStatus.Confirmed)
            throw new DomainException("Cannot add items to a confirmed order.");

        var existingLine = _lines.FirstOrDefault(l => l.ProductId == productId);
        if (existingLine != null)
        {
            existingLine.IncreaseQuantity(quantity);
        }
        else
        {
            _lines.Add(new OrderLine(productId, quantity, unitPrice));
        }
    }

    public void Confirm()
    {
        // Enforce Invariant: ต้องมี OrderLine อย่างน้อย 1
        if (!_lines.Any())
            throw new DomainException("Cannot confirm an empty order.");

        // Enforce Invariant: ต้องอยู่ใน Status Pending เท่านั้น
        if (_status != OrderStatus.Pending)
            throw new DomainException($"Cannot confirm order in {_status} status.");

        _status = OrderStatus.Confirmed;
    }
}
```

---

### ตัวอย่าง: Order Aggregate

```csharp
// Aggregate Root
public class Order : AggregateRoot
{
    public OrderId Id { get; private set; }
    public CustomerId CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public Money TotalAmount { get; private set; }
    
    // Child Entities - เก็บเป็น private list ไม่ expose ออกไปตรงๆ
    private readonly List<OrderLine> _orderLines = new();
    public IReadOnlyList<OrderLine> OrderLines => _orderLines.AsReadOnly();

    // Private constructor - สร้างผ่าน Factory Method เท่านั้น
    private Order() { }

    // Factory Method
    public static Order Create(CustomerId customerId)
    {
        var order = new Order
        {
            Id = new OrderId(Guid.NewGuid()),
            CustomerId = customerId,
            Status = OrderStatus.Pending,
            TotalAmount = Money.Zero("THB")
        };
        
        order.AddDomainEvent(new OrderCreatedEvent(order.Id, customerId));
        return order;
    }

    public void AddItem(ProductId productId, Quantity quantity, Money unitPrice)
    {
        EnsureOrderIsEditable();
        
        var line = _orderLines.FirstOrDefault(l => l.ProductId == productId);
        if (line is not null)
        {
            line.IncreaseQuantity(quantity);
        }
        else
        {
            _orderLines.Add(OrderLine.Create(Id, productId, quantity, unitPrice));
        }

        RecalculateTotal();
        AddDomainEvent(new OrderItemAddedEvent(Id, productId, quantity));
    }

    public void RemoveItem(ProductId productId)
    {
        EnsureOrderIsEditable();
        
        var line = _orderLines.FirstOrDefault(l => l.ProductId == productId)
            ?? throw new DomainException($"Product {productId} not found in order.");
        
        _orderLines.Remove(line);
        RecalculateTotal();
    }

    public void Confirm()
    {
        if (!_orderLines.Any())
            throw new DomainException("Cannot confirm an empty order.");
        
        if (Status != OrderStatus.Pending)
            throw new DomainException("Order can only be confirmed when pending.");
        
        Status = OrderStatus.Confirmed;
        AddDomainEvent(new OrderConfirmedEvent(Id, CustomerId, TotalAmount));
    }

    private void EnsureOrderIsEditable()
    {
        if (Status is OrderStatus.Confirmed or OrderStatus.Cancelled or OrderStatus.Shipped)
            throw new DomainException($"Order cannot be modified in {Status} status.");
    }

    private void RecalculateTotal()
    {
        TotalAmount = _orderLines
            .Aggregate(Money.Zero("THB"), (sum, line) => sum + line.TotalPrice);
    }
}

// Child Entity (ไม่ใช่ Aggregate Root)
public class OrderLine : Entity
{
    public OrderLineId Id { get; private set; }
    public OrderId OrderId { get; private set; }
    public ProductId ProductId { get; private set; }  // อ้างอิงด้วย ID เท่านั้น!
    public Quantity Quantity { get; private set; }
    public Money UnitPrice { get; private set; }
    public Money TotalPrice => UnitPrice * Quantity.Value;

    private OrderLine() { }

    public static OrderLine Create(OrderId orderId, ProductId productId, 
                                   Quantity quantity, Money unitPrice)
    {
        return new OrderLine
        {
            Id = new OrderLineId(Guid.NewGuid()),
            OrderId = orderId,
            ProductId = productId,
            Quantity = quantity,
            UnitPrice = unitPrice
        };
    }

    internal void IncreaseQuantity(Quantity additionalQuantity)
    {
        Quantity = new Quantity(Quantity.Value + additionalQuantity.Value);
    }
}
```

---

## Step 603: Value Objects

### Value Object คืออะไร?

**Value Object** คือ Object ที่:
1. **Immutable** - ไม่สามารถเปลี่ยนค่าได้หลังสร้าง (ต้องสร้างใหม่)
2. **Equality by Value** - เปรียบเทียบด้วยค่า ไม่ใช่ Reference
3. **ไม่มี Identity** - ไม่มี ID ของตัวเอง
4. **Self-Validating** - Validate ตัวเองใน Constructor

ตัวอย่างง่ายๆ: `Money`, `Email`, `Address`, `Quantity`, `DateRange`

---

### Money Value Object

```csharp
// ใช้ record struct สำหรับ Value Object ใน C# 10+
public record struct Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        if (amount < 0)
            throw new ArgumentException("Amount cannot be negative.", nameof(amount));
        if (string.IsNullOrWhiteSpace(currency))
            throw new ArgumentException("Currency cannot be empty.", nameof(currency));
        if (currency.Length != 3)
            throw new ArgumentException("Currency must be a 3-letter ISO code.", nameof(currency));

        Amount = Math.Round(amount, 2);
        Currency = currency.ToUpperInvariant();
    }

    // Factory Methods
    public static Money Zero(string currency) => new(0, currency);
    public static Money Of(decimal amount, string currency) => new(amount, currency);

    // Arithmetic Operators
    public static Money operator +(Money left, Money right)
    {
        EnsureSameCurrency(left, right);
        return new Money(left.Amount + right.Amount, left.Currency);
    }

    public static Money operator -(Money left, Money right)
    {
        EnsureSameCurrency(left, right);
        if (left.Amount < right.Amount)
            throw new InvalidOperationException("Result would be negative.");
        return new Money(left.Amount - right.Amount, left.Currency);
    }

    public static Money operator *(Money money, decimal multiplier)
    {
        return new Money(money.Amount * multiplier, money.Currency);
    }

    public static bool operator >(Money left, Money right)
    {
        EnsureSameCurrency(left, right);
        return left.Amount > right.Amount;
    }

    public static bool operator <(Money left, Money right)
    {
        EnsureSameCurrency(left, right);
        return left.Amount < right.Amount;
    }

    private static void EnsureSameCurrency(Money left, Money right)
    {
        if (left.Currency != right.Currency)
            throw new InvalidOperationException(
                $"Cannot operate on different currencies: {left.Currency} and {right.Currency}");
    }

    public override string ToString() => $"{Amount:N2} {Currency}";
}

// การใช้งาน
var price = new Money(99.99m, "THB");
var tax = new Money(7.00m, "THB");
var total = price + tax;  // 106.99 THB

// Value equality - record struct ทำให้ใช้ == ได้เลย
var money1 = new Money(100m, "THB");
var money2 = new Money(100m, "THB");
Console.WriteLine(money1 == money2); // true ✓

// Immutable - ไม่มี Setter
// money1.Amount = 200; // ❌ Compile Error!
```

---

### Email Value Object

```csharp
public record struct Email
{
    public string Value { get; }

    public Email(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new ArgumentException("Email cannot be empty.", nameof(value));

        var normalized = value.Trim().ToLowerInvariant();
        
        if (!IsValidEmail(normalized))
            throw new ArgumentException($"'{value}' is not a valid email address.", nameof(value));

        Value = normalized;
    }

    private static bool IsValidEmail(string email)
    {
        try
        {
            var addr = new System.Net.Mail.MailAddress(email);
            return addr.Address == email;
        }
        catch
        {
            return false;
        }
    }

    public static implicit operator string(Email email) => email.Value;
    public override string ToString() => Value;
}

// การใช้งาน
var email = new Email("  User@Example.COM  "); // normalize อัตโนมัติ
Console.WriteLine(email.Value);  // user@example.com

var email1 = new Email("test@example.com");
var email2 = new Email("TEST@EXAMPLE.COM");
Console.WriteLine(email1 == email2); // true ✓ (เพราะ normalize แล้ว)
```

---

### Address Value Object

```csharp
public record struct Address
{
    public string Street { get; }
    public string City { get; }
    public string Province { get; }
    public string PostalCode { get; }
    public string Country { get; }

    public Address(string street, string city, string province, 
                   string postalCode, string country)
    {
        Street = ValidateNotEmpty(street, nameof(street));
        City = ValidateNotEmpty(city, nameof(city));
        Province = ValidateNotEmpty(province, nameof(province));
        PostalCode = ValidatePostalCode(postalCode);
        Country = ValidateNotEmpty(country, nameof(country));
    }

    private static string ValidateNotEmpty(string value, string paramName)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new ArgumentException($"{paramName} cannot be empty.", paramName);
        return value.Trim();
    }

    private static string ValidatePostalCode(string postalCode)
    {
        if (string.IsNullOrWhiteSpace(postalCode))
            throw new ArgumentException("Postal code cannot be empty.");
        if (!postalCode.All(char.IsDigit) || postalCode.Length != 5)
            throw new ArgumentException("Thai postal code must be 5 digits.");
        return postalCode;
    }

    public string FullAddress => $"{Street}, {City}, {Province} {PostalCode}, {Country}";
    public override string ToString() => FullAddress;
}

// การใช้งาน
var address = new Address(
    street: "123 สุขุมวิท 11",
    city: "กรุงเทพมหานคร",
    province: "กรุงเทพมหานคร",
    postalCode: "10110",
    country: "Thailand"
);

// Value equality อัตโนมัติจาก record struct
var addr1 = new Address("123 Main St", "BKK", "BKK", "10110", "TH");
var addr2 = new Address("123 Main St", "BKK", "BKK", "10110", "TH");
Console.WriteLine(addr1 == addr2); // true ✓
```

---

### ทำไมต้องใช้ `record struct` สำหรับ Value Object?

ใน C# 10+ `record struct` เหมาะสำหรับ Value Object เพราะ:

| คุณสมบัติ | record struct | class |
|-----------|---------------|-------|
| Equality by value | ✓ อัตโนมัติ | ต้อง Override เอง |
| Immutable | ✓ ง่ายกว่า | ต้อง `init` ทุก property |
| Stack allocation | ✓ (สำหรับ struct เล็กๆ) | Heap เสมอ |
| `with` expression | ✓ | ✓ (record class) |

```csharp
// ง่ายมาก! ทุก property เป็น readonly โดย default
public record struct Quantity(int Value)
{
    // Validate ใน constructor
    public Quantity(int value) : this()
    {
        if (value <= 0)
            throw new ArgumentException("Quantity must be positive.");
        Value = value;
    }
}

// ใช้ with expression เพื่อสร้าง "modified copy"
var qty = new Quantity(5);
// var newQty = qty with { Value = 10 }; // ได้ copy ใหม่ ของเดิมไม่เปลี่ยน
```

---

## Step 604: Domain Services

### เมื่อไหรที่ Logic ไม่เหมาะกับ Entity?

บางครั้งมี Business Logic ที่:
- เกี่ยวข้องกับหลาย Entity/Aggregate พร้อมกัน
- ดูแปลกถ้าใส่ใน Entity ใด Entity หนึ่ง
- ต้องการข้อมูลจาก External Service (Exchange Rate, Tax Rate)

ในกรณีเหล่านี้ ให้ใช้ **Domain Service**

---

### PriceCalculationService

```csharp
// Domain Service Interface (อยู่ใน Domain Layer)
public interface IPriceCalculationService
{
    Money CalculateOrderTotal(Order order, IEnumerable<Discount> applicableDiscounts);
    Money CalculateTax(Money subtotal, TaxCategory taxCategory, Address deliveryAddress);
}

// Domain Service Implementation
public class PriceCalculationService : IPriceCalculationService
{
    private readonly ITaxRateRepository _taxRateRepository;

    public PriceCalculationService(ITaxRateRepository taxRateRepository)
    {
        _taxRateRepository = taxRateRepository;
    }

    public Money CalculateOrderTotal(Order order, IEnumerable<Discount> applicableDiscounts)
    {
        var subtotal = order.OrderLines
            .Aggregate(Money.Zero("THB"), (sum, line) => sum + line.TotalPrice);

        // Apply Discounts
        var totalDiscount = applicableDiscounts
            .Where(d => d.IsApplicableTo(order))
            .Aggregate(Money.Zero("THB"), (sum, d) => sum + d.CalculateDiscount(subtotal));

        // Ensure discount doesn't exceed subtotal
        if (totalDiscount > subtotal)
            totalDiscount = subtotal;

        return subtotal - totalDiscount;
    }

    public Money CalculateTax(Money subtotal, TaxCategory taxCategory, Address deliveryAddress)
    {
        var taxRate = _taxRateRepository.GetRate(taxCategory, deliveryAddress.Province);
        return subtotal * taxRate.Percentage / 100;
    }
}
```

---

### OrderConfirmationService

```csharp
// Domain Service ที่ Orchestrate หลาย Aggregate
public interface IOrderConfirmationService
{
    ConfirmationResult ConfirmOrder(Order order, Customer customer, StockReservation stockReservation);
}

public class OrderConfirmationService : IOrderConfirmationService
{
    public ConfirmationResult ConfirmOrder(
        Order order, 
        Customer customer, 
        StockReservation stockReservation)
    {
        // ตรวจสอบว่า Customer ยังมีสิทธิ์สั่งซื้อ
        if (customer.IsSuspended)
            return ConfirmationResult.Failed("Customer account is suspended.");

        // ตรวจสอบ Credit Limit
        if (customer.HasExceededCreditLimit(order.TotalAmount))
            return ConfirmationResult.Failed("Order exceeds customer credit limit.");

        // ตรวจสอบ Stock
        if (!stockReservation.IsFullyReserved)
            return ConfirmationResult.Failed("Not all items are available in stock.");

        // ยืนยัน Order
        order.Confirm();

        return ConfirmationResult.Success();
    }
}

public record ConfirmationResult(bool IsSuccess, string? ErrorMessage)
{
    public static ConfirmationResult Success() => new(true, null);
    public static ConfirmationResult Failed(string reason) => new(false, reason);
}
```

---

### Domain Service vs Application Service

สิ่งที่แตกต่างกัน:

| | Domain Service | Application Service |
|---|----------------|---------------------|
| **Layer** | Domain Layer | Application Layer |
| **รู้จัก Infrastructure?** | ไม่รู้จัก | รู้จัก (DI) |
| **มี Business Logic?** | ✓ มี | ❌ ไม่ควรมี |
| **ทำงานกับอะไร?** | Domain Objects | Use Cases |
| **ตัวอย่าง** | `PriceCalculationService` | `PlaceOrderCommandHandler` |

```csharp
// Application Service - ประสาน Infrastructure กับ Domain
public class PlaceOrderCommandHandler
{
    private readonly IOrderRepository _orderRepository;
    private readonly ICustomerRepository _customerRepository;
    private readonly IOrderConfirmationService _confirmationService; // Domain Service
    private readonly IEventPublisher _eventPublisher; // Infrastructure

    public async Task<OrderId> Handle(PlaceOrderCommand command)
    {
        // 1. โหลดข้อมูลจาก Repository (Infrastructure)
        var customer = await _customerRepository.GetByIdAsync(command.CustomerId);
        
        // 2. สร้าง Order Aggregate (Domain)
        var order = Order.Create(customer.Id);
        
        foreach (var item in command.Items)
        {
            order.AddItem(item.ProductId, item.Quantity, item.UnitPrice);
        }

        // 3. บันทึกผ่าน Repository (Infrastructure)
        await _orderRepository.SaveAsync(order);

        // 4. Publish Domain Events (Infrastructure)
        await _eventPublisher.PublishAsync(order.DomainEvents);
        
        return order.Id;
    }
}
```

---

## Step 605: Domain Events

### Domain Event คืออะไร?

**Domain Event** คือ Object ที่แทน "สิ่งที่เกิดขึ้นแล้วในโดเมน" มีลักษณะ:
- เป็น Past Tense เสมอ: `OrderPlaced`, `PaymentReceived`, `ItemShipped`
- **Immutable** - เปลี่ยนแปลงไม่ได้ (มันเกิดขึ้นแล้ว)
- มีข้อมูลเพียงพอสำหรับ Handler ในการทำงาน
- ไม่ Return ค่าอะไร

---

### Domain Events ตัวอย่าง

```csharp
// Base class สำหรับทุก Domain Event
public abstract class DomainEvent
{
    public Guid EventId { get; } = Guid.NewGuid();
    public DateTime OccurredAt { get; } = DateTime.UtcNow;
}

// เหตุการณ์: Order ถูกสร้าง
public class OrderCreatedEvent : DomainEvent
{
    public OrderId OrderId { get; }
    public CustomerId CustomerId { get; }
    public DateTime CreatedAt { get; }

    public OrderCreatedEvent(OrderId orderId, CustomerId customerId)
    {
        OrderId = orderId;
        CustomerId = customerId;
        CreatedAt = DateTime.UtcNow;
    }
}

// เหตุการณ์: เพิ่ม Item เข้า Order
public class OrderItemAddedEvent : DomainEvent
{
    public OrderId OrderId { get; }
    public ProductId ProductId { get; }
    public Quantity Quantity { get; }

    public OrderItemAddedEvent(OrderId orderId, ProductId productId, Quantity quantity)
    {
        OrderId = orderId;
        ProductId = productId;
        Quantity = quantity;
    }
}

// เหตุการณ์: Order ถูก Confirm
public class OrderConfirmedEvent : DomainEvent
{
    public OrderId OrderId { get; }
    public CustomerId CustomerId { get; }
    public Money TotalAmount { get; }

    public OrderConfirmedEvent(OrderId orderId, CustomerId customerId, Money totalAmount)
    {
        OrderId = orderId;
        CustomerId = customerId;
        TotalAmount = totalAmount;
    }
}

// เหตุการณ์: Order ถูกส่ง
public class OrderShippedEvent : DomainEvent
{
    public OrderId OrderId { get; }
    public TrackingNumber TrackingNumber { get; }
    public DateTime ShippedAt { get; }

    public OrderShippedEvent(OrderId orderId, TrackingNumber trackingNumber)
    {
        OrderId = orderId;
        TrackingNumber = trackingNumber;
        ShippedAt = DateTime.UtcNow;
    }
}

// เหตุการณ์: ได้รับการชำระเงิน
public class PaymentReceivedEvent : DomainEvent
{
    public OrderId OrderId { get; }
    public Money AmountPaid { get; }
    public PaymentMethod Method { get; }

    public PaymentReceivedEvent(OrderId orderId, Money amountPaid, PaymentMethod method)
    {
        OrderId = orderId;
        AmountPaid = amountPaid;
        Method = method;
    }
}
```

---

### AddEvent Pattern: Raise Events Inside Aggregate

```csharp
// Base class สำหรับทุก Aggregate Root
public abstract class AggregateRoot : Entity
{
    private readonly List<DomainEvent> _domainEvents = new();
    
    // อ่านได้อย่างเดียว
    public IReadOnlyList<DomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    // เรียกใช้ภายใน Aggregate เท่านั้น
    protected void AddDomainEvent(DomainEvent domainEvent)
    {
        _domainEvents.Add(domainEvent);
    }

    // Application Service เรียกหลังจาก Save เสร็จ
    public void ClearDomainEvents()
    {
        _domainEvents.Clear();
    }
}

// Entity Base
public abstract class Entity
{
    // Common entity properties
}

// ใน Order Aggregate - Raise events ตรงที่เกิดเหตุการณ์
public class Order : AggregateRoot
{
    public static Order Create(CustomerId customerId)
    {
        var order = new Order { /* init properties */ };
        
        // Raise event ตรงนี้!
        order.AddDomainEvent(new OrderCreatedEvent(order.Id, customerId));
        
        return order;
    }

    public void Confirm()
    {
        // ... validation logic ...
        Status = OrderStatus.Confirmed;
        
        // Raise event ตรงนี้!
        AddDomainEvent(new OrderConfirmedEvent(Id, CustomerId, TotalAmount));
    }
}
```

---

### Process Events in Handlers (Eventually Consistent)

```csharp
// Event Handler Interface
public interface IDomainEventHandler<TEvent> where TEvent : DomainEvent
{
    Task HandleAsync(TEvent domainEvent, CancellationToken cancellationToken = default);
}

// Handler: ส่ง Email เมื่อ Order ถูก Confirm
public class SendOrderConfirmationEmailHandler 
    : IDomainEventHandler<OrderConfirmedEvent>
{
    private readonly IEmailService _emailService;
    private readonly ICustomerRepository _customerRepository;

    public SendOrderConfirmationEmailHandler(
        IEmailService emailService, 
        ICustomerRepository customerRepository)
    {
        _emailService = emailService;
        _customerRepository = customerRepository;
    }

    public async Task HandleAsync(
        OrderConfirmedEvent domainEvent, 
        CancellationToken cancellationToken = default)
    {
        var customer = await _customerRepository
            .GetByIdAsync(domainEvent.CustomerId, cancellationToken);
        
        await _emailService.SendAsync(new OrderConfirmationEmail
        {
            To = customer.Email,
            OrderId = domainEvent.OrderId.ToString(),
            TotalAmount = domainEvent.TotalAmount.ToString(),
            ConfirmedAt = domainEvent.OccurredAt
        }, cancellationToken);
    }
}

// Handler: Reserve Stock เมื่อ Order ถูก Confirm
public class ReserveStockOnOrderConfirmedHandler 
    : IDomainEventHandler<OrderConfirmedEvent>
{
    private readonly IInventoryService _inventoryService;
    private readonly IOrderRepository _orderRepository;

    public async Task HandleAsync(
        OrderConfirmedEvent domainEvent, 
        CancellationToken cancellationToken = default)
    {
        var order = await _orderRepository
            .GetByIdAsync(domainEvent.OrderId, cancellationToken);

        foreach (var line in order.OrderLines)
        {
            await _inventoryService.ReserveAsync(
                line.ProductId, 
                line.Quantity, 
                cancellationToken);
        }
    }
}
```

---

## Step 606: Repositories แบบ DDD

### Repository Pattern ใน DDD

**Repository** ใน DDD มีหน้าที่:
1. **โหลด Aggregate ทั้งอัน** (ไม่ใช่ส่วนหนึ่ง)
2. **บันทึก Aggregate ทั้งอัน** (ไม่ใช่ส่วนหนึ่ง)
3. **มี Repository แค่สำหรับ Aggregate Root เท่านั้น**

กฎสำคัญ: **ไม่มี IOrderLineRepository!** เพราะ OrderLine ไม่ใช่ Aggregate Root

---

### IOrderRepository

```csharp
// Repository Interface อยู่ใน Domain Layer
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(OrderId orderId, CancellationToken cancellationToken = default);
    Task<IEnumerable<Order>> GetByCustomerIdAsync(CustomerId customerId, CancellationToken cancellationToken = default);
    Task SaveAsync(Order order, CancellationToken cancellationToken = default);
    Task DeleteAsync(OrderId orderId, CancellationToken cancellationToken = default);
}

// Repository Implementation อยู่ใน Infrastructure Layer
public class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _dbContext;

    public OrderRepository(AppDbContext dbContext)
    {
        _dbContext = dbContext;
    }

    public async Task<Order?> GetByIdAsync(
        OrderId orderId, 
        CancellationToken cancellationToken = default)
    {
        // Load COMPLETE aggregate (ทั้ง Order + OrderLines)
        return await _dbContext.Orders
            .Include(o => o.OrderLines) // Load children ด้วย
            .FirstOrDefaultAsync(o => o.Id == orderId, cancellationToken);
    }

    public async Task<IEnumerable<Order>> GetByCustomerIdAsync(
        CustomerId customerId, 
        CancellationToken cancellationToken = default)
    {
        return await _dbContext.Orders
            .Include(o => o.OrderLines)
            .Where(o => o.CustomerId == customerId)
            .ToListAsync(cancellationToken);
    }

    public async Task SaveAsync(
        Order order, 
        CancellationToken cancellationToken = default)
    {
        // EF Core track changes ให้อัตโนมัติ
        if (_dbContext.Entry(order).State == EntityState.Detached)
        {
            await _dbContext.Orders.AddAsync(order, cancellationToken);
        }
        
        await _dbContext.SaveChangesAsync(cancellationToken);
    }

    public async Task DeleteAsync(
        OrderId orderId, 
        CancellationToken cancellationToken = default)
    {
        var order = await GetByIdAsync(orderId, cancellationToken);
        if (order is not null)
        {
            _dbContext.Orders.Remove(order);
            await _dbContext.SaveChangesAsync(cancellationToken);
        }
    }
}
```

---

### Specification Pattern สำหรับ Domain Queries

**Specification Pattern** ช่วยให้ Query Logic อยู่ใน Domain Layer แทนที่จะกระจายใน Repository

```csharp
// Base Specification
public abstract class Specification<T>
{
    public abstract Expression<Func<T, bool>> Criteria { get; }
    
    public Specification<T> And(Specification<T> other)
        => new AndSpecification<T>(this, other);
    
    public Specification<T> Or(Specification<T> other)
        => new OrSpecification<T>(this, other);
}

// Specification: Orders ที่ Pending
public class PendingOrdersSpecification : Specification<Order>
{
    public override Expression<Func<Order, bool>> Criteria
        => order => order.Status == OrderStatus.Pending;
}

// Specification: Orders ของ Customer
public class OrdersByCustomerSpecification : Specification<Order>
{
    private readonly CustomerId _customerId;
    
    public OrdersByCustomerSpecification(CustomerId customerId)
    {
        _customerId = customerId;
    }
    
    public override Expression<Func<Order, bool>> Criteria
        => order => order.CustomerId == _customerId;
}

// Specification: Orders ที่ Total มากกว่า X
public class OrdersAboveAmountSpecification : Specification<Order>
{
    private readonly Money _minimumAmount;
    
    public OrdersAboveAmountSpecification(Money minimumAmount)
    {
        _minimumAmount = minimumAmount;
    }
    
    public override Expression<Func<Order, bool>> Criteria
        => order => order.TotalAmount.Amount >= _minimumAmount.Amount 
                 && order.TotalAmount.Currency == _minimumAmount.Currency;
}

// Repository ที่รับ Specification
public interface IOrderRepository
{
    Task<IEnumerable<Order>> FindAsync(
        Specification<Order> specification, 
        CancellationToken cancellationToken = default);
}

// การใช้งาน
var pendingOrders = await _orderRepository.FindAsync(
    new PendingOrdersSpecification()
        .And(new OrdersByCustomerSpecification(customerId))
);

var bigOrders = await _orderRepository.FindAsync(
    new OrdersAboveAmountSpecification(new Money(5000, "THB"))
);
```

---

## Step 607-610: การ Implement DDD จริง (E-Commerce Order Bounded Context)

ในส่วนนี้เราจะสร้าง **Order Bounded Context** ที่สมบูรณ์ด้วย DDD Patterns ทั้งหมด

### โครงสร้างโปรเจกต์

```
OrderContext/
├── Domain/
│   ├── Aggregates/
│   │   └── Orders/
│   │       ├── Order.cs                  ← Aggregate Root
│   │       ├── OrderLine.cs              ← Child Entity
│   │       └── OrderStatus.cs            ← Enum
│   ├── ValueObjects/
│   │   ├── Money.cs
│   │   ├── Quantity.cs
│   │   ├── Address.cs
│   │   └── Ids/
│   │       ├── OrderId.cs
│   │       ├── OrderLineId.cs
│   │       ├── CustomerId.cs
│   │       └── ProductId.cs
│   ├── Events/
│   │   ├── OrderCreatedEvent.cs
│   │   ├── OrderItemAddedEvent.cs
│   │   └── OrderConfirmedEvent.cs
│   ├── Repositories/
│   │   └── IOrderRepository.cs
│   └── Services/
│       └── IOrderConfirmationService.cs
├── Application/
│   ├── Commands/
│   │   ├── PlaceOrderCommand.cs
│   │   └── PlaceOrderCommandHandler.cs
│   └── Queries/
│       └── GetOrderQueryHandler.cs
└── Infrastructure/
    ├── Persistence/
    │   ├── AppDbContext.cs
    │   └── OrderRepository.cs
    └── EventHandlers/
        └── SendConfirmationEmailHandler.cs
```

---

### Strong-Typed IDs

```csharp
// ป้องกัน Primitive Obsession - ไม่ใช้ Guid ตรงๆ
public record struct OrderId(Guid Value)
{
    public static OrderId New() => new(Guid.NewGuid());
    public static OrderId From(Guid value) => new(value);
    public override string ToString() => Value.ToString();
}

public record struct OrderLineId(Guid Value)
{
    public static OrderLineId New() => new(Guid.NewGuid());
}

public record struct CustomerId(Guid Value)
{
    public static CustomerId New() => new(Guid.NewGuid());
}

public record struct ProductId(Guid Value)
{
    public static ProductId New() => new(Guid.NewGuid());
}

// ตอนนี้ method signature ชัดเจน ไม่สับสน!
// Order.AddItem(ProductId, Quantity, Money)  ← ชัดมาก
// vs
// Order.AddItem(Guid, int, decimal)          ← สับสนง่าย
```

---

### Domain Base Classes

```csharp
// Entity Base
public abstract class Entity<TId>
{
    public TId Id { get; protected set; } = default!;

    protected Entity() { }

    public override bool Equals(object? obj)
    {
        if (obj is not Entity<TId> other) return false;
        if (ReferenceEquals(this, other)) return true;
        if (GetType() != other.GetType()) return false;
        return Id!.Equals(other.Id);
    }

    public override int GetHashCode() => Id!.GetHashCode();
    
    public static bool operator ==(Entity<TId>? left, Entity<TId>? right)
        => Equals(left, right);
    
    public static bool operator !=(Entity<TId>? left, Entity<TId>? right)
        => !Equals(left, right);
}

// Aggregate Root Base
public abstract class AggregateRoot<TId> : Entity<TId>
{
    private readonly List<DomainEvent> _domainEvents = new();
    
    public IReadOnlyList<DomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected void AddDomainEvent(DomainEvent domainEvent)
    {
        _domainEvents.Add(domainEvent);
    }

    public void ClearDomainEvents()
    {
        _domainEvents.Clear();
    }
}

// Domain Exception
public class DomainException : Exception
{
    public string DomainError { get; }

    public DomainException(string message) : base(message)
    {
        DomainError = message;
    }
}
```

---

### OrderStatus Enum

```csharp
public enum OrderStatus
{
    Pending = 1,      // สร้างแล้ว รอดำเนินการ
    Confirmed = 2,    // ยืนยันแล้ว รอ Payment
    Paid = 3,         // ชำระเงินแล้ว
    Shipped = 4,      // ส่งแล้ว
    Delivered = 5,    // ได้รับสินค้าแล้ว
    Cancelled = 6     // ยกเลิก
}
```

---

### Quantity Value Object

```csharp
public record struct Quantity
{
    public int Value { get; }

    public Quantity(int value)
    {
        if (value <= 0)
            throw new ArgumentException("Quantity must be greater than zero.", nameof(value));
        Value = value;
    }

    public static Quantity Of(int value) => new(value);
    
    public static Quantity operator +(Quantity left, Quantity right)
        => new(left.Value + right.Value);

    public static bool operator >(Quantity left, Quantity right)
        => left.Value > right.Value;

    public static bool operator <(Quantity left, Quantity right)
        => left.Value < right.Value;

    public override string ToString() => Value.ToString();
}
```

---

### Complete Order Aggregate (Step 607)

```csharp
public class Order : AggregateRoot<OrderId>
{
    // Properties
    public CustomerId CustomerId { get; private set; }
    public Address? ShippingAddress { get; private set; }
    public OrderStatus Status { get; private set; }
    public Money TotalAmount { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? ConfirmedAt { get; private set; }
    public string? CancellationReason { get; private set; }

    // Private collection - encapsulated!
    private readonly List<OrderLine> _orderLines = new();
    public IReadOnlyList<OrderLine> OrderLines => _orderLines.AsReadOnly();

    // EF Core needs parameterless constructor
    private Order() { }

    // ============================================================
    // Factory Method - ทางเดียวที่จะสร้าง Order
    // ============================================================
    public static Order Create(CustomerId customerId, Address shippingAddress)
    {
        ArgumentNullException.ThrowIfNull(shippingAddress);

        var order = new Order
        {
            Id = OrderId.New(),
            CustomerId = customerId,
            ShippingAddress = shippingAddress,
            Status = OrderStatus.Pending,
            TotalAmount = Money.Zero("THB"),
            CreatedAt = DateTime.UtcNow
        };

        order.AddDomainEvent(new OrderCreatedEvent(
            order.Id, 
            customerId, 
            shippingAddress));

        return order;
    }

    // ============================================================
    // Business Operations
    // ============================================================

    public void AddItem(ProductId productId, Quantity quantity, Money unitPrice)
    {
        EnsureOrderIsEditable("add items to");

        // ตรวจสอบ currency ต้องตรงกัน
        if (unitPrice.Currency != TotalAmount.Currency)
            throw new DomainException(
                $"Item currency '{unitPrice.Currency}' doesn't match order currency '{TotalAmount.Currency}'.");

        var existingLine = _orderLines
            .FirstOrDefault(l => l.ProductId == productId);

        if (existingLine is not null)
        {
            existingLine.IncreaseQuantity(quantity);
        }
        else
        {
            _orderLines.Add(OrderLine.Create(Id, productId, quantity, unitPrice));
        }

        RecalculateTotal();

        AddDomainEvent(new OrderItemAddedEvent(Id, productId, quantity, unitPrice));
    }

    public void RemoveItem(ProductId productId)
    {
        EnsureOrderIsEditable("remove items from");

        var line = _orderLines.FirstOrDefault(l => l.ProductId == productId)
            ?? throw new DomainException(
                $"Product {productId} is not in this order.");

        _orderLines.Remove(line);
        RecalculateTotal();
    }

    public void UpdateItemQuantity(ProductId productId, Quantity newQuantity)
    {
        EnsureOrderIsEditable("update items in");

        var line = _orderLines.FirstOrDefault(l => l.ProductId == productId)
            ?? throw new DomainException(
                $"Product {productId} is not in this order.");

        line.SetQuantity(newQuantity);
        RecalculateTotal();
    }

    public void UpdateShippingAddress(Address newAddress)
    {
        EnsureOrderIsEditable("update shipping address of");
        ShippingAddress = newAddress ?? throw new ArgumentNullException(nameof(newAddress));
    }

    public void Confirm()
    {
        if (Status != OrderStatus.Pending)
            throw new DomainException(
                $"Order can only be confirmed when Pending. Current status: {Status}");

        if (!_orderLines.Any())
            throw new DomainException("Cannot confirm an order with no items.");

        if (ShippingAddress is null)
            throw new DomainException("Cannot confirm an order without a shipping address.");

        Status = OrderStatus.Confirmed;
        ConfirmedAt = DateTime.UtcNow;

        AddDomainEvent(new OrderConfirmedEvent(
            Id, 
            CustomerId, 
            TotalAmount, 
            ConfirmedAt.Value));
    }

    public void MarkAsPaid(Money amountPaid, PaymentMethod paymentMethod)
    {
        if (Status != OrderStatus.Confirmed)
            throw new DomainException(
                "Order must be confirmed before marking as paid.");

        if (amountPaid.Amount < TotalAmount.Amount)
            throw new DomainException(
                $"Paid amount {amountPaid} is less than order total {TotalAmount}.");

        Status = OrderStatus.Paid;

        AddDomainEvent(new OrderPaidEvent(Id, amountPaid, paymentMethod));
    }

    public void Ship(TrackingNumber trackingNumber)
    {
        if (Status != OrderStatus.Paid)
            throw new DomainException("Order must be paid before shipping.");

        Status = OrderStatus.Shipped;

        AddDomainEvent(new OrderShippedEvent(Id, trackingNumber, DateTime.UtcNow));
    }

    public void MarkAsDelivered()
    {
        if (Status != OrderStatus.Shipped)
            throw new DomainException("Order must be shipped before marking as delivered.");

        Status = OrderStatus.Delivered;
    }

    public void Cancel(string reason)
    {
        if (Status is OrderStatus.Shipped or OrderStatus.Delivered)
            throw new DomainException(
                $"Cannot cancel an order that is already {Status}.");

        if (string.IsNullOrWhiteSpace(reason))
            throw new ArgumentException("Cancellation reason is required.", nameof(reason));

        Status = OrderStatus.Cancelled;
        CancellationReason = reason;

        AddDomainEvent(new OrderCancelledEvent(Id, CustomerId, reason));
    }

    // ============================================================
    // Private Helpers
    // ============================================================

    private void EnsureOrderIsEditable(string operation)
    {
        if (Status is not OrderStatus.Pending)
            throw new DomainException(
                $"Cannot {operation} an order in {Status} status.");
    }

    private void RecalculateTotal()
    {
        TotalAmount = _orderLines.Count == 0
            ? Money.Zero("THB")
            : _orderLines.Aggregate(
                Money.Zero("THB"), 
                (sum, line) => sum + line.TotalPrice);
    }
}
```

---

### OrderLine Entity (Step 608)

```csharp
public class OrderLine : Entity<OrderLineId>
{
    public OrderId OrderId { get; private set; }
    public ProductId ProductId { get; private set; }  // Reference by ID only!
    public string ProductName { get; private set; } = string.Empty; // Snapshot ตอนสั่ง
    public Quantity Quantity { get; private set; }
    public Money UnitPrice { get; private set; }
    public Money TotalPrice => UnitPrice * Quantity.Value;

    private OrderLine() { }

    public static OrderLine Create(
        OrderId orderId,
        ProductId productId,
        Quantity quantity,
        Money unitPrice,
        string productName = "")
    {
        return new OrderLine
        {
            Id = OrderLineId.New(),
            OrderId = orderId,
            ProductId = productId,
            Quantity = quantity,
            UnitPrice = unitPrice,
            ProductName = productName
        };
    }

    // Internal methods - เรียกได้แค่จาก Order (same assembly)
    internal void IncreaseQuantity(Quantity additionalQuantity)
    {
        Quantity = Quantity + additionalQuantity;
    }

    internal void SetQuantity(Quantity newQuantity)
    {
        Quantity = newQuantity;
    }
}
```

---

### Domain Events ครบชุด (Step 609)

```csharp
// Base
public abstract record DomainEvent
{
    public Guid EventId { get; } = Guid.NewGuid();
    public DateTime OccurredAt { get; } = DateTime.UtcNow;
    public string EventType => GetType().Name;
}

// Order Created
public record OrderCreatedEvent(
    OrderId OrderId,
    CustomerId CustomerId,
    Address ShippingAddress
) : DomainEvent;

// Item Added
public record OrderItemAddedEvent(
    OrderId OrderId,
    ProductId ProductId,
    Quantity Quantity,
    Money UnitPrice
) : DomainEvent;

// Order Confirmed
public record OrderConfirmedEvent(
    OrderId OrderId,
    CustomerId CustomerId,
    Money TotalAmount,
    DateTime ConfirmedAt
) : DomainEvent;

// Order Paid
public record OrderPaidEvent(
    OrderId OrderId,
    Money AmountPaid,
    PaymentMethod PaymentMethod
) : DomainEvent;

// Order Shipped
public record OrderShippedEvent(
    OrderId OrderId,
    TrackingNumber TrackingNumber,
    DateTime ShippedAt
) : DomainEvent;

// Order Cancelled
public record OrderCancelledEvent(
    OrderId OrderId,
    CustomerId CustomerId,
    string Reason
) : DomainEvent;
```

---

### IOrderRepository Interface (Step 610)

```csharp
// Domain Layer - ไม่รู้จัก EF Core หรือ Database!
public interface IOrderRepository
{
    // Get single aggregate
    Task<Order?> GetByIdAsync(
        OrderId orderId, 
        CancellationToken cancellationToken = default);

    // Get multiple aggregates
    Task<IReadOnlyList<Order>> GetByCustomerIdAsync(
        CustomerId customerId,
        CancellationToken cancellationToken = default);

    // Specification pattern
    Task<IReadOnlyList<Order>> FindAsync(
        Specification<Order> specification,
        int pageNumber = 1,
        int pageSize = 20,
        CancellationToken cancellationToken = default);

    // Count (without loading all aggregates)
    Task<int> CountAsync(
        Specification<Order> specification,
        CancellationToken cancellationToken = default);

    // Save (Add or Update)
    Task SaveAsync(
        Order order, 
        CancellationToken cancellationToken = default);

    // Check existence
    Task<bool> ExistsAsync(
        OrderId orderId,
        CancellationToken cancellationToken = default);
}

// ============================================================
// Application Layer - PlaceOrder Use Case
// ============================================================
public class PlaceOrderCommandHandler
{
    private readonly IOrderRepository _orderRepository;
    private readonly IEventDispatcher _eventDispatcher;

    public PlaceOrderCommandHandler(
        IOrderRepository orderRepository,
        IEventDispatcher eventDispatcher)
    {
        _orderRepository = orderRepository;
        _eventDispatcher = eventDispatcher;
    }

    public async Task<OrderId> HandleAsync(
        PlaceOrderCommand command,
        CancellationToken cancellationToken = default)
    {
        // 1. สร้าง Shipping Address Value Object
        var shippingAddress = new Address(
            command.Street,
            command.City,
            command.Province,
            command.PostalCode,
            command.Country
        );

        // 2. สร้าง Order Aggregate
        var order = Order.Create(
            new CustomerId(command.CustomerId),
            shippingAddress
        );

        // 3. Add Items
        foreach (var item in command.Items)
        {
            order.AddItem(
                new ProductId(item.ProductId),
                new Quantity(item.Quantity),
                new Money(item.UnitPrice, "THB")
            );
        }

        // 4. บันทึก (Save Aggregate ทั้งอัน)
        await _orderRepository.SaveAsync(order, cancellationToken);

        // 5. Dispatch Domain Events
        await _eventDispatcher.DispatchAsync(
            order.DomainEvents, 
            cancellationToken);
        
        order.ClearDomainEvents();

        return order.Id;
    }
}

// Command DTO
public record PlaceOrderCommand(
    Guid CustomerId,
    string Street,
    string City,
    string Province,
    string PostalCode,
    string Country,
    IReadOnlyList<OrderItemDto> Items
);

public record OrderItemDto(
    Guid ProductId,
    int Quantity,
    decimal UnitPrice
);
```

---

### ตัวอย่างการใช้งานจริง (Integration Test Style)

```csharp
// ทดสอบ Order Domain Logic โดยไม่ต้องมี Database
[Test]
public void Order_AddItem_Then_Confirm_ShouldWork()
{
    // Arrange
    var customerId = CustomerId.New();
    var address = new Address("123 Main St", "Bangkok", "Bangkok", "10110", "TH");
    var productId = ProductId.New();
    
    // Act
    var order = Order.Create(customerId, address);
    order.AddItem(productId, new Quantity(2), new Money(500m, "THB"));
    order.Confirm();
    
    // Assert
    Assert.That(order.Status, Is.EqualTo(OrderStatus.Confirmed));
    Assert.That(order.TotalAmount, Is.EqualTo(new Money(1000m, "THB")));
    Assert.That(order.OrderLines.Count, Is.EqualTo(1));
    
    // Check Domain Events
    var events = order.DomainEvents;
    Assert.That(events.Count, Is.EqualTo(3)); // Created + ItemAdded + Confirmed
    Assert.That(events[0], Is.TypeOf<OrderCreatedEvent>());
    Assert.That(events[1], Is.TypeOf<OrderItemAddedEvent>());
    Assert.That(events[2], Is.TypeOf<OrderConfirmedEvent>());
}

[Test]
public void Order_Confirm_WithNoItems_ShouldThrowDomainException()
{
    // Arrange
    var order = Order.Create(
        CustomerId.New(), 
        new Address("123", "BKK", "BKK", "10110", "TH"));
    
    // Act & Assert
    var ex = Assert.Throws<DomainException>(() => order.Confirm());
    Assert.That(ex.Message, Does.Contain("Cannot confirm an empty order"));
}

[Test]
public void Order_Cancel_AfterShipped_ShouldThrowDomainException()
{
    // Arrange
    var order = CreateConfirmedAndPaidOrder();
    order.Ship(new TrackingNumber("TH123456789"));
    
    // Act & Assert
    Assert.Throws<DomainException>(() => order.Cancel("Changed mind"));
}
```

---

## สรุป

### สรุปแนวคิด DDD ในแต่ละ Step

| Step | แนวคิด | สิ่งสำคัญที่ต้องจำ |
|------|---------|---------------------|
| **601** | DDD Overview | Strategic (Bounded Context, Context Map) vs Tactical (Aggregate, Entity, VO) |
| **602** | Aggregates | มี Root เดียว, Enforce Invariants, อ้าง Aggregate อื่นด้วย ID เท่านั้น |
| **603** | Value Objects | Immutable, Equality by Value, Self-Validating, ใช้ `record struct` |
| **604** | Domain Services | เมื่อ Logic ไม่เป็นของ Entity ใดเดียว, ไม่รู้จัก Infrastructure |
| **605** | Domain Events | Past Tense, Immutable, Raise inside Aggregate, Process ใน Handlers |
| **606** | Repositories | แค่ Aggregate Root, Load/Save ทั้ง Aggregate, Specification Pattern |
| **607-610** | Implementation | Strong-typed IDs, Base Classes, Complete Order Aggregate, Application Layer |

---

### DDD ดีอย่างไร?

**ข้อดี:**
- โค้ดสะท้อน Business Logic ได้ชัดเจน
- ป้องกัน Invalid State ด้วย Invariants
- ง่ายต่อการ Test (ทดสอบ Domain Logic ได้โดยไม่ต้อง Database)
- ทีมนักพัฒนาและ Domain Expert คุยกันรู้เรื่อง
- Scale ได้ดีในระบบใหญ่

**ข้อเสีย:**
- Learning Curve สูง
- Overhead สำหรับระบบเล็กๆ ที่ Logic ไม่ซับซ้อน
- ต้องลงทุนเวลาในการ Modeling เริ่มต้น

**เมื่อไหรที่ควรใช้ DDD:**
- ระบบที่มี Business Logic ซับซ้อน
- หลาย Teams ทำงานบน Codebase เดียวกัน
- Domain Experts เป็นส่วนสำคัญของทีม
- ระบบที่ต้องการ Maintainability ระยะยาว

---

### Quick Reference: DDD Building Blocks

```
Domain Layer (ไม่รู้จัก Infrastructure!)
├── Aggregates (Cluster of Objects + Business Rules)
│   ├── Aggregate Root (Entry Point, raises Events)
│   └── Child Entities (accessed through Root only)
├── Value Objects (Immutable, Equality by Value)
│   ├── Money, Quantity, Address, Email
│   └── Strong-Typed IDs (OrderId, CustomerId)
├── Domain Events (Something happened, Past Tense)
│   └── OrderCreated, OrderConfirmed, OrderShipped
├── Domain Services (Logic spanning multiple Aggregates)
│   └── PriceCalculationService, OrderConfirmationService
└── Repository Interfaces (Define contract, no implementation)
    └── IOrderRepository (Aggregate Root only!)

Application Layer (Orchestrates Domain + Infrastructure)
├── Command Handlers (Write operations)
│   └── PlaceOrderCommandHandler
└── Query Handlers (Read operations)
    └── GetOrderQueryHandler

Infrastructure Layer (Implements interfaces)
├── Repositories (EF Core, MongoDB, etc.)
│   └── OrderRepository : IOrderRepository
└── Event Handlers (Email, SMS, other Contexts)
    └── SendConfirmationEmailHandler
```

---

**การนำทาง:**

- ← บทที่แล้ว: [Part 60: Docker Deployment](../part51-60/part60-docker-deployment.md)
- → บทถัดไป: [Part 62: CQRS Pattern](part62-cqrs.md)
