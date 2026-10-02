# Part 76: Functional Programming Patterns in C#
## Steps 751-760: เทคนิค Functional Programming ใน C#

---

## บทนำ: Functional Programming คืออะไร?

Functional Programming (FP) เป็นแนวทางการเขียนโปรแกรมที่เน้น:
- **การทำงานด้วย Functions บริสุทธิ์** (Pure Functions) ที่ไม่มี side effects
- **ความไม่เปลี่ยนแปลง** (Immutability) ของข้อมูล
- **การประกอบฟังก์ชัน** (Function Composition) แทนการเปลี่ยนแปลง state

C# เป็นภาษา multi-paradigm ที่รองรับ FP patterns ได้ดีผ่าน LINQ, lambda expressions, records, และ extension methods

---

## Step 751: แนวคิดพื้นฐาน Functional Programming ใน C#

### 751.1 Pure Functions (ฟังก์ชันบริสุทธิ์)

ฟังก์ชันบริสุทธิ์คือฟังก์ชันที่:
1. ให้ผลลัพธ์เดิมเสมอเมื่อรับ input เดิม
2. ไม่มี side effects (ไม่แก้ไขข้อมูลภายนอก)

```csharp
// ตัวอย่าง: Pure Function vs Impure Function

// ❌ Impure Function - มี side effect
public class OrderProcessor
{
    private List<string> _log = new();
    
    public decimal CalculateTotal(List<decimal> prices)
    {
        _log.Add($"Calculating total at {DateTime.Now}"); // Side effect!
        return prices.Sum();
    }
}

// ✅ Pure Function - ไม่มี side effect
public static class PricingCalculator
{
    // ผลลัพธ์เดิมเสมอสำหรับ input เดิม
    public static decimal CalculateTotal(IEnumerable<decimal> prices)
        => prices.Sum();
    
    public static decimal ApplyDiscount(decimal total, decimal discountRate)
        => total * (1 - discountRate);
    
    public static decimal CalculateTax(decimal total, decimal taxRate)
        => total * taxRate;
    
    public static decimal FinalPrice(decimal total, decimal discount, decimal tax)
        => total - ApplyDiscount(total, discount) + CalculateTax(total, tax);
}

// การทดสอบ Pure Functions ง่ายมาก
var prices = new[] { 100m, 200m, 300m };
var total = PricingCalculator.CalculateTotal(prices); // เสมอได้ 600
```

### 751.2 Higher-Order Functions (ฟังก์ชันอันดับสูง)

ฟังก์ชันที่รับหรือส่งคืนฟังก์ชันเป็น argument

```csharp
// Higher-Order Functions
public static class FunctionalHelpers
{
    // รับ function เป็น parameter
    public static IEnumerable<TResult> Transform<T, TResult>(
        IEnumerable<T> source, 
        Func<T, TResult> transform)
        => source.Select(transform);
    
    // ส่งคืน function
    public static Func<T, bool> Not<T>(Func<T, bool> predicate)
        => x => !predicate(x);
    
    // Function composition
    public static Func<T, TResult> Compose<T, TMiddle, TResult>(
        Func<T, TMiddle> first,
        Func<TMiddle, TResult> second)
        => x => second(first(x));
    
    // Memoization - cache ผลลัพธ์ของ pure function
    public static Func<T, TResult> Memoize<T, TResult>(Func<T, TResult> func)
        where T : notnull
    {
        var cache = new Dictionary<T, TResult>();
        return key =>
        {
            if (!cache.TryGetValue(key, out var result))
            {
                result = func(key);
                cache[key] = result;
            }
            return result;
        };
    }
}

// การใช้งาน
var numbers = new[] { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

// Transform ด้วย higher-order function
var doubled = FunctionalHelpers.Transform(numbers, x => x * 2);

// Not predicate
var isEven = (int n) => n % 2 == 0;
var isOdd = FunctionalHelpers.Not(isEven);
var oddNumbers = numbers.Where(isOdd);

// Compose functions
var doubleAndSquare = FunctionalHelpers.Compose<int, int, int>(
    x => x * 2,
    x => x * x
);
var result = doubleAndSquare(3); // (3*2)^2 = 36

// Memoize expensive calculation
var expensiveCalc = FunctionalHelpers.Memoize<int, long>(n =>
{
    Thread.Sleep(100); // simulate expensive work
    return Fibonacci(n);
});

static long Fibonacci(int n) => n <= 1 ? n : Fibonacci(n - 1) + Fibonacci(n - 2);
```

### 751.3 Immutability (ความไม่เปลี่ยนแปลง)

```csharp
// ❌ Mutable approach
public class MutableOrder
{
    public int Id { get; set; }
    public string CustomerName { get; set; } = "";
    public decimal Total { get; set; }
    public List<string> Items { get; set; } = new();
}

// ✅ Immutable approach ด้วย C# records
public record ImmutableOrder(
    int Id,
    string CustomerName,
    decimal Total,
    IReadOnlyList<string> Items);

// การ "แก้ไข" สร้าง instance ใหม่แทน
var order = new ImmutableOrder(1, "สมชาย", 500m, new[] { "A", "B" });
var updatedOrder = order with { Total = 600m }; // สร้าง order ใหม่

Console.WriteLine(order.Total);        // 500 - ยังคงเดิม
Console.WriteLine(updatedOrder.Total); // 600 - instance ใหม่
```

---

## Step 752: Option\<T\> / Maybe Monad สำหรับ Null Safety

### 752.1 ปัญหาของ null

```csharp
// ❌ Traditional null handling - เสี่ยง NullReferenceException
public class UserRepository
{
    public User? FindUser(int id) => /* ... */ null;
}

var user = repository.FindUser(id);
var email = user.Email; // NullReferenceException ถ้า user เป็น null!
```

### 752.2 สร้าง Option\<T\>

```csharp
// Option<T> - บรรจุค่าที่อาจมีหรือไม่มี
public abstract record Option<T>
{
    public static Option<T> Some(T value) => new Some<T>(value);
    public static Option<T> None() => new None<T>();
    
    public abstract bool HasValue { get; }
    
    // Map: แปลงค่าภายใน Option
    public abstract Option<TResult> Map<TResult>(Func<T, TResult> mapper);
    
    // Bind (FlatMap): chain operations ที่ส่งคืน Option
    public abstract Option<TResult> Bind<TResult>(Func<T, Option<TResult>> binder);
    
    // GetOrElse: ดึงค่าหรือใช้ค่า default
    public abstract T GetOrElse(T defaultValue);
    public abstract T GetOrElse(Func<T> defaultFactory);
    
    // Match: pattern matching
    public abstract TResult Match<TResult>(
        Func<T, TResult> some, 
        Func<TResult> none);
    
    // Filter: กรองตามเงื่อนไข
    public Option<T> Filter(Func<T, bool> predicate)
        => Match(
            some: value => predicate(value) ? this : None(),
            none: () => None()
        );
}

public sealed record Some<T>(T Value) : Option<T>
{
    public override bool HasValue => true;
    
    public override Option<TResult> Map<TResult>(Func<T, TResult> mapper)
        => Option<TResult>.Some(mapper(Value));
    
    public override Option<TResult> Bind<TResult>(Func<T, Option<TResult>> binder)
        => binder(Value);
    
    public override T GetOrElse(T defaultValue) => Value;
    public override T GetOrElse(Func<T> defaultFactory) => Value;
    
    public override TResult Match<TResult>(Func<T, TResult> some, Func<TResult> none)
        => some(Value);
}

public sealed record None<T> : Option<T>
{
    public override bool HasValue => false;
    
    public override Option<TResult> Map<TResult>(Func<T, TResult> mapper)
        => Option<TResult>.None();
    
    public override Option<TResult> Bind<TResult>(Func<T, Option<TResult>> binder)
        => Option<TResult>.None();
    
    public override T GetOrElse(T defaultValue) => defaultValue;
    public override T GetOrElse(Func<T> defaultFactory) => defaultFactory();
    
    public override TResult Match<TResult>(Func<T, TResult> some, Func<TResult> none)
        => none();
}
```

### 752.3 Extension Methods สำหรับ Option\<T\>

```csharp
public static class OptionExtensions
{
    // แปลง nullable เป็น Option
    public static Option<T> ToOption<T>(this T? value) where T : class
        => value is not null ? Option<T>.Some(value) : Option<T>.None();
    
    public static Option<T> ToOption<T>(this T? value) where T : struct
        => value.HasValue ? Option<T>.Some(value.Value) : Option<T>.None();
    
    // Tap - ดู value โดยไม่แก้ไข (สำหรับ debugging/logging)
    public static Option<T> Tap<T>(this Option<T> option, Action<T> action)
    {
        if (option.HasValue)
            option.Match(value => { action(value); return value; }, () => default!);
        return option;
    }
    
    // OrElse - ลอง alternative Option
    public static Option<T> OrElse<T>(this Option<T> option, Option<T> alternative)
        => option.HasValue ? option : alternative;
    
    public static Option<T> OrElse<T>(this Option<T> option, Func<Option<T>> alternativeFactory)
        => option.HasValue ? option : alternativeFactory();
}
```

### 752.4 การใช้งาน Option\<T\>

```csharp
public record User(int Id, string Name, string? Email);
public record Order(int Id, int UserId, decimal Total);

public class UserService
{
    private readonly Dictionary<int, User> _users = new()
    {
        [1] = new User(1, "สมชาย", "somchai@example.com"),
        [2] = new User(2, "สมหญิง", null) // ไม่มี email
    };
    
    public Option<User> FindUser(int id)
        => _users.TryGetValue(id, out var user) 
            ? Option<User>.Some(user) 
            : Option<User>.None();
    
    public Option<string> GetUserEmail(int userId)
        => FindUser(userId)
            .Bind(user => user.Email.ToOption()); // null-safe
    
    // ตัวอย่างการ chain operations
    public string GetEmailOrDefault(int userId)
        => GetUserEmail(userId)
            .Map(email => email.ToUpper())
            .Filter(email => email.Contains('@'))
            .GetOrElse("no-email@default.com");
}

// การใช้งาน
var service = new UserService();

// User ที่มี email
var email1 = service.GetEmailOrDefault(1);
Console.WriteLine(email1); // SOMCHAI@EXAMPLE.COM

// User ที่ไม่มี email
var email2 = service.GetEmailOrDefault(2);
Console.WriteLine(email2); // no-email@default.com

// User ที่ไม่มีอยู่จริง
var email3 = service.GetEmailOrDefault(99);
Console.WriteLine(email3); // no-email@default.com

// Pattern matching กับ Option
var result = service.FindUser(1).Match(
    some: user => $"พบผู้ใช้: {user.Name}",
    none: () => "ไม่พบผู้ใช้"
);
```

---

## Step 753: Result\<T, E\> สำหรับ Error Handling โดยไม่ใช้ Exceptions

### 753.1 ปัญหาของ Exception-based Error Handling

```csharp
// ❌ Exception-based - ซ่อน error flow
public decimal Divide(decimal a, decimal b)
{
    if (b == 0) throw new DivideByZeroException("ไม่สามารถหารด้วยศูนย์");
    return a / b;
}

// ผู้เรียกต้องจำ try/catch
try 
{
    var result = Divide(10, 0);
}
catch (DivideByZeroException ex)
{
    // handle error
}
```

### 753.2 สร้าง Result\<T, E\>

```csharp
// Result<T, E> - บรรจุทั้ง success value หรือ error
public abstract record Result<T, E>
{
    public static Result<T, E> Ok(T value) => new Ok<T, E>(value);
    public static Result<T, E> Err(E error) => new Err<T, E>(error);
    
    public abstract bool IsSuccess { get; }
    public abstract bool IsError { get; }
    
    // Map: แปลง success value
    public abstract Result<TResult, E> Map<TResult>(Func<T, TResult> mapper);
    
    // MapError: แปลง error value
    public abstract Result<T, EResult> MapError<EResult>(Func<E, EResult> mapper);
    
    // Bind: chain operations ที่ส่งคืน Result
    public abstract Result<TResult, E> Bind<TResult>(Func<T, Result<TResult, E>> binder);
    
    // Match: pattern matching
    public abstract TResult Match<TResult>(
        Func<T, TResult> ok,
        Func<E, TResult> err);
    
    // Tap
    public Result<T, E> Tap(Action<T> action)
    {
        Match(value => { action(value); return value; }, _ => default!);
        return this;
    }
    
    public Result<T, E> TapError(Action<E> action)
    {
        Match(_ => default!, error => { action(error); return default!; });
        return this;
    }
    
    // GetOrThrow
    public abstract T GetOrThrow();
    
    // Implicit conversions
    public static implicit operator Result<T, E>(T value) => Ok(value);
    public static implicit operator Result<T, E>(E error) => Err(error);
}

public sealed record Ok<T, E>(T Value) : Result<T, E>
{
    public override bool IsSuccess => true;
    public override bool IsError => false;
    
    public override Result<TResult, E> Map<TResult>(Func<T, TResult> mapper)
        => Result<TResult, E>.Ok(mapper(Value));
    
    public override Result<T, EResult> MapError<EResult>(Func<E, EResult> mapper)
        => Result<T, EResult>.Ok(Value);
    
    public override Result<TResult, E> Bind<TResult>(Func<T, Result<TResult, E>> binder)
        => binder(Value);
    
    public override TResult Match<TResult>(Func<T, TResult> ok, Func<E, TResult> err)
        => ok(Value);
    
    public override T GetOrThrow() => Value;
}

public sealed record Err<T, E>(E Error) : Result<T, E>
{
    public override bool IsSuccess => false;
    public override bool IsError => true;
    
    public override Result<TResult, E> Map<TResult>(Func<T, TResult> mapper)
        => Result<TResult, E>.Err(Error);
    
    public override Result<T, EResult> MapError<EResult>(Func<E, EResult> mapper)
        => Result<T, EResult>.Err(mapper(Error));
    
    public override Result<TResult, E> Bind<TResult>(Func<T, Result<TResult, E>> binder)
        => Result<TResult, E>.Err(Error);
    
    public override TResult Match<TResult>(Func<T, TResult> ok, Func<E, TResult> err)
        => err(Error);
    
    public override T GetOrThrow() 
        => throw new InvalidOperationException($"Result contains error: {Error}");
}
```

### 753.3 ตัวอย่างการใช้งาน Result\<T, E\>

```csharp
// กำหนด Error types
public abstract record ValidationError;
public record FieldRequired(string FieldName) : ValidationError;
public record InvalidFormat(string FieldName, string Message) : ValidationError;
public record OutOfRange(string FieldName, decimal Min, decimal Max) : ValidationError;

// Service ที่ใช้ Result pattern
public class ProductService
{
    public Result<decimal, ValidationError> ParsePrice(string input)
    {
        if (string.IsNullOrWhiteSpace(input))
            return Result<decimal, ValidationError>.Err(new FieldRequired("Price"));
        
        if (!decimal.TryParse(input, out var price))
            return Result<decimal, ValidationError>.Err(
                new InvalidFormat("Price", $"'{input}' ไม่ใช่ตัวเลข"));
        
        if (price < 0 || price > 1_000_000)
            return Result<decimal, ValidationError>.Err(
                new OutOfRange("Price", 0, 1_000_000));
        
        return Result<decimal, ValidationError>.Ok(price);
    }
    
    public Result<string, ValidationError> ValidateProductName(string name)
    {
        if (string.IsNullOrWhiteSpace(name))
            return Result<string, ValidationError>.Err(new FieldRequired("Name"));
        
        if (name.Length > 100)
            return Result<string, ValidationError>.Err(
                new InvalidFormat("Name", "ชื่อสินค้าต้องไม่เกิน 100 ตัวอักษร"));
        
        return Result<string, ValidationError>.Ok(name.Trim());
    }
}

// การใช้งาน
var service = new ProductService();

var priceResult = service.ParsePrice("299.99");
var message = priceResult.Match(
    ok: price => $"ราคา: {price:C2}",
    err: error => error switch
    {
        FieldRequired f => $"กรุณากรอก {f.FieldName}",
        InvalidFormat f => $"{f.FieldName}: {f.Message}",
        OutOfRange o => $"{o.FieldName} ต้องอยู่ระหว่าง {o.Min} - {o.Max}",
        _ => "เกิดข้อผิดพลาด"
    }
);
Console.WriteLine(message); // ราคา: ฿299.99
```

---

## Step 754: Railway-Oriented Programming (การ chain Result operations)

### 754.1 แนวคิด Railway-Oriented Programming

ลองนึกภาพ "ราง 2 ราง":
- **ราง Success**: ดำเนินการต่อเมื่อทุกอย่างถูกต้อง
- **ราง Error**: เมื่อเกิดข้อผิดพลาด ข้ามขั้นตอนที่เหลือทั้งหมด

```csharp
// Extension methods สำหรับ Railway-oriented programming
public static class ResultExtensions
{
    // Ensure: เพิ่มการตรวจสอบเงื่อนไข
    public static Result<T, E> Ensure<T, E>(
        this Result<T, E> result,
        Func<T, bool> predicate,
        E error)
        => result.Bind(value => 
            predicate(value) 
                ? result 
                : Result<T, E>.Err(error));
    
    // Combine: รวม results หลายตัว
    public static Result<(T1, T2), E> Combine<T1, T2, E>(
        Result<T1, E> result1,
        Result<T2, E> result2)
        => (result1, result2) switch
        {
            ({ IsSuccess: true } r1, { IsSuccess: true } r2) 
                => Result<(T1, T2), E>.Ok((r1.GetOrThrow(), r2.GetOrThrow())),
            ({ IsError: true } r1, _) 
                => Result<(T1, T2), E>.Err(r1.Match(_ => default!, e => e)),
            (_, { IsError: true } r2) 
                => Result<(T1, T2), E>.Err(r2.Match(_ => default!, e => e)),
            _ => throw new InvalidOperationException()
        };
    
    // ToOption
    public static Option<T> ToOption<T, E>(this Result<T, E> result)
        => result.Match(
            ok: value => Option<T>.Some(value),
            err: _ => Option<T>.None()
        );
}

// ตัวอย่าง Order Processing Pipeline
public record OrderRequest(
    string CustomerId,
    string ProductId,
    int Quantity,
    string VoucherCode);

public record Customer(string Id, string Name, bool IsActive);
public record Product(string Id, string Name, decimal Price, int Stock);
public record Voucher(string Code, decimal DiscountPercent, bool IsValid);
public record ProcessedOrder(Customer Customer, Product Product, int Qty, decimal FinalPrice);

public abstract record OrderError;
public record CustomerNotFound(string Id) : OrderError;
public record CustomerInactive(string Id) : OrderError;
public record ProductNotFound(string Id) : OrderError;
public record InsufficientStock(string Id, int Available, int Requested) : OrderError;
public record InvalidVoucher(string Code) : OrderError;

public class OrderPipeline
{
    private readonly Dictionary<string, Customer> _customers = new()
    {
        ["C001"] = new Customer("C001", "สมชาย", true),
        ["C002"] = new Customer("C002", "วิชา", false) // inactive
    };
    
    private readonly Dictionary<string, Product> _products = new()
    {
        ["P001"] = new Product("P001", "แล็ปท็อป", 30000m, 10),
        ["P002"] = new Product("P002", "เมาส์", 500m, 0) // out of stock
    };
    
    private readonly Dictionary<string, Voucher> _vouchers = new()
    {
        ["SAVE10"] = new Voucher("SAVE10", 0.10m, true),
        ["EXPIRED"] = new Voucher("EXPIRED", 0.20m, false)
    };
    
    public Result<Customer, OrderError> GetCustomer(string customerId)
        => _customers.TryGetValue(customerId, out var customer)
            ? Result<Customer, OrderError>.Ok(customer)
            : Result<Customer, OrderError>.Err(new CustomerNotFound(customerId));
    
    public Result<Customer, OrderError> ValidateCustomerActive(Customer customer)
        => customer.IsActive
            ? Result<Customer, OrderError>.Ok(customer)
            : Result<Customer, OrderError>.Err(new CustomerInactive(customer.Id));
    
    public Result<Product, OrderError> GetProduct(string productId)
        => _products.TryGetValue(productId, out var product)
            ? Result<Product, OrderError>.Ok(product)
            : Result<Product, OrderError>.Err(new ProductNotFound(productId));
    
    public Result<Product, OrderError> ValidateStock(Product product, int quantity)
        => product.Stock >= quantity
            ? Result<Product, OrderError>.Ok(product)
            : Result<Product, OrderError>.Err(
                new InsufficientStock(product.Id, product.Stock, quantity));
    
    public Option<Voucher> GetVoucher(string? code)
        => code is null
            ? Option<Voucher>.None()
            : _vouchers.TryGetValue(code, out var v) && v.IsValid
                ? Option<Voucher>.Some(v)
                : Option<Voucher>.None();
    
    // Railway Pipeline
    public Result<ProcessedOrder, OrderError> ProcessOrder(OrderRequest request)
    {
        // Chain operations - หากขั้นตอนใดผิดพลาด จะข้ามขั้นต่อไปทั้งหมด
        return GetCustomer(request.CustomerId)
            .Bind(ValidateCustomerActive)
            .Bind(customer => 
                GetProduct(request.ProductId)
                    .Bind(product => ValidateStock(product, request.Quantity))
                    .Map(product =>
                    {
                        var basePrice = product.Price * request.Quantity;
                        var voucher = GetVoucher(request.VoucherCode);
                        var discount = voucher.Match(
                            some: v => basePrice * v.DiscountPercent,
                            none: () => 0m
                        );
                        var finalPrice = basePrice - discount;
                        return new ProcessedOrder(customer, product, request.Quantity, finalPrice);
                    }));
    }
}

// การใช้งาน
var pipeline = new OrderPipeline();

// ✅ Order ที่สำเร็จ
var order1 = pipeline.ProcessOrder(new OrderRequest("C001", "P001", 2, "SAVE10"));
order1.Match(
    ok: o => Console.WriteLine($"สำเร็จ! {o.Customer.Name} สั่ง {o.Product.Name} x{o.Qty} = {o.FinalPrice:C0}"),
    err: e => Console.WriteLine($"ผิดพลาด: {e}")
);
// สำเร็จ! สมชาย สั่ง แล็ปท็อป x2 = ฿54,000

// ❌ Customer inactive
var order2 = pipeline.ProcessOrder(new OrderRequest("C002", "P001", 1, null));
order2.Match(
    ok: o => Console.WriteLine("สำเร็จ"),
    err: e => Console.WriteLine(e switch
    {
        CustomerInactive c => $"บัญชีถูกระงับ: {c.Id}",
        _ => $"ผิดพลาด: {e}"
    })
);
// บัญชีถูกระงับ: C002
```

---

## Step 755: Functional LINQ Pipelines

### 755.1 LINQ Pipeline พื้นฐาน

```csharp
// Domain
public record Product(string Id, string Name, string Category, decimal Price, int SoldCount);

var products = new List<Product>
{
    new("P01", "แล็ปท็อป", "Electronics", 35000m, 150),
    new("P02", "เมาส์", "Electronics", 450m, 890),
    new("P03", "โต๊ะ", "Furniture", 5500m, 45),
    new("P04", "เก้าอี้", "Furniture", 3200m, 78),
    new("P05", "คีย์บอร์ด", "Electronics", 1200m, 320),
    new("P06", "จอมอนิเตอร์", "Electronics", 8500m, 200),
    new("P07", "ชั้นวาง", "Furniture", 1800m, 120),
};

// LINQ Pipeline ที่ใช้ functional style
var report = products
    .Where(p => p.Price > 1000)                          // กรองราคา > 1000
    .GroupBy(p => p.Category)                             // จัดกลุ่มตาม Category
    .Select(g => new                                      // แปลงเป็น summary
    {
        Category = g.Key,
        ProductCount = g.Count(),
        TotalRevenue = g.Sum(p => p.Price * p.SoldCount),
        AveragePrice = g.Average(p => p.Price),
        TopSeller = g.MaxBy(p => p.SoldCount)?.Name ?? "N/A"
    })
    .OrderByDescending(r => r.TotalRevenue)               // เรียงตาม Revenue
    .ToList();

foreach (var r in report)
    Console.WriteLine($"{r.Category}: {r.ProductCount} สินค้า, รายได้ {r.TotalRevenue:C0}, ขายดีที่สุด: {r.TopSeller}");
```

### 755.2 Aggregate และ Fold

```csharp
// Aggregate (fold) - accumulate values
var numbers = Enumerable.Range(1, 10);

// Sum ด้วย Aggregate
var sum = numbers.Aggregate(0, (acc, n) => acc + n); // 55

// ประวัติการสะสม (running total)
var runningTotals = numbers
    .Aggregate(
        new List<int> { 0 },
        (acc, n) =>
        {
            acc.Add(acc[^1] + n);
            return acc;
        }
    )
    .Skip(1)
    .ToList();
// [1, 3, 6, 10, 15, 21, 28, 36, 45, 55]

// สร้าง Dictionary ด้วย Aggregate
var productIndex = products.Aggregate(
    new Dictionary<string, Product>(),
    (dict, p) =>
    {
        dict[p.Id] = p;
        return dict;
    }
);
```

### 755.3 Advanced LINQ Patterns

```csharp
// Functional data transformation pipeline
public static class DataPipeline
{
    // Chunk - แบ่งข้อมูลเป็นกลุ่ม
    public static IEnumerable<IEnumerable<T>> Batch<T>(
        this IEnumerable<T> source, int size)
        => source
            .Select((item, index) => (item, index))
            .GroupBy(x => x.index / size)
            .Select(g => g.Select(x => x.item));
    
    // Scan - คล้าย Aggregate แต่ให้ intermediate values
    public static IEnumerable<TAccumulate> Scan<T, TAccumulate>(
        this IEnumerable<T> source,
        TAccumulate seed,
        Func<TAccumulate, T, TAccumulate> func)
    {
        var current = seed;
        yield return current;
        foreach (var item in source)
        {
            current = func(current, item);
            yield return current;
        }
    }
    
    // ZipWithIndex
    public static IEnumerable<(T Item, int Index)> WithIndex<T>(
        this IEnumerable<T> source)
        => source.Select((item, i) => (item, i));
    
    // Partition - แบ่งเป็น 2 กลุ่มตามเงื่อนไข
    public static (List<T> Matches, List<T> NonMatches) Partition<T>(
        this IEnumerable<T> source, Func<T, bool> predicate)
    {
        var matches = new List<T>();
        var nonMatches = new List<T>();
        foreach (var item in source)
            (predicate(item) ? matches : nonMatches).Add(item);
        return (matches, nonMatches);
    }
}

// การใช้งาน
var prices = new[] { 100m, 200m, 300m, 400m, 500m };

// Running total
var runningBalance = prices
    .Scan(0m, (acc, price) => acc + price)
    .ToList();
// [0, 100, 300, 600, 1000, 1500]

// Partition expensive and cheap products
var (expensive, cheap) = products.Partition(p => p.Price > 5000);
Console.WriteLine($"สินค้าแพง: {expensive.Count} ชิ้น, สินค้าถูก: {cheap.Count} ชิ้น");

// Batch processing
var batches = products.Batch(3).ToList();
Console.WriteLine($"จำนวน batch: {batches.Count()}");
```

---

## Step 756: Partial Application และ Currying

### 756.1 Partial Application

Partial Application คือการล็อค argument บางตัวของฟังก์ชัน สร้างฟังก์ชันใหม่ที่ต้องการ argument น้อยกว่า

```csharp
// Partial Application helper
public static class PartialApplication
{
    // Partial apply argument แรก
    public static Func<T2, TResult> Partial<T1, T2, TResult>(
        Func<T1, T2, TResult> func, T1 arg1)
        => arg2 => func(arg1, arg2);
    
    // Partial apply arguments แรกสอง
    public static Func<T3, TResult> Partial<T1, T2, T3, TResult>(
        Func<T1, T2, T3, TResult> func, T1 arg1, T2 arg2)
        => arg3 => func(arg1, arg2, arg3);
    
    // Flip - สลับ argument แรกและสอง
    public static Func<T2, T1, TResult> Flip<T1, T2, TResult>(
        Func<T1, T2, TResult> func)
        => (arg2, arg1) => func(arg1, arg2);
}

// ตัวอย่าง
Func<decimal, decimal, decimal> multiply = (a, b) => a * b;
Func<decimal, decimal, decimal> add = (a, b) => a + b;

// Partial apply - ล็อค multiplier = 1.07 (VAT 7%)
var addVat = PartialApplication.Partial(multiply, 1.07m);
var withTax = PartialApplication.Partial(add, 0m); // identity

Console.WriteLine(addVat(100m));  // 107
Console.WriteLine(addVat(200m));  // 214

// สร้างชุดของ discount functions
Func<decimal, decimal, decimal> applyDiscount = (rate, price) => price * (1 - rate);
var apply10Percent = PartialApplication.Partial(applyDiscount, 0.10m);
var apply20Percent = PartialApplication.Partial(applyDiscount, 0.20m);
var apply30Percent = PartialApplication.Partial(applyDiscount, 0.30m);

Console.WriteLine(apply10Percent(1000m)); // 900
Console.WriteLine(apply20Percent(1000m)); // 800
Console.WriteLine(apply30Percent(1000m)); // 700
```

### 756.2 Currying

Currying คือการแปลงฟังก์ชันที่รับหลาย argument เป็นฟังก์ชันที่รับทีละหนึ่ง

```csharp
public static class Curry
{
    // Curry 2 arguments
    public static Func<T1, Func<T2, TResult>> Curried<T1, T2, TResult>(
        Func<T1, T2, TResult> func)
        => arg1 => arg2 => func(arg1, arg2);
    
    // Curry 3 arguments
    public static Func<T1, Func<T2, Func<T3, TResult>>> Curried<T1, T2, T3, TResult>(
        Func<T1, T2, T3, TResult> func)
        => arg1 => arg2 => arg3 => func(arg1, arg2, arg3);
    
    // Uncurry - แปลงกลับ
    public static Func<T1, T2, TResult> Uncurried<T1, T2, TResult>(
        Func<T1, Func<T2, TResult>> curriedFunc)
        => (arg1, arg2) => curriedFunc(arg1)(arg2);
}

// ตัวอย่าง Currying
Func<string, string, string> format = (template, value) 
    => template.Replace("{0}", value);

var curriedFormat = Curry.Curried(format);

// สร้าง specialized formatters
var greet = curriedFormat("สวัสดี, {0}!");
var error = curriedFormat("ข้อผิดพลาด: {0}");
var success = curriedFormat("สำเร็จ: {0}");

Console.WriteLine(greet("สมชาย"));    // สวัสดี, สมชาย!
Console.WriteLine(error("ไม่พบไฟล์")); // ข้อผิดพลาด: ไม่พบไฟล์
Console.WriteLine(success("บันทึก"));  // สำเร็จ: บันทึก

// Currying กับ validation
Func<int, int, int, bool> inRange = (min, max, value) => value >= min && value <= max;
var curriedInRange = Curry.Curried(inRange);

var isValidAge = curriedInRange(0)(150);
var isValidScore = curriedInRange(0)(100);
var isValidPrice = curriedInRange(1)(10_000_000);

Console.WriteLine(isValidAge(25));     // true
Console.WriteLine(isValidScore(150));  // false
Console.WriteLine(isValidPrice(500m.ToInt())); // true (แปลงให้ถูกต้องก่อนใช้งาน)
```

### 756.3 Function Pipeline Composition

```csharp
public static class FunctionPipeline
{
    // Pipe - ส่ง value ผ่าน chain ของ functions
    public static TResult Pipe<T, TResult>(T value, Func<T, TResult> func)
        => func(value);
    
    public static T3 Pipe<T1, T2, T3>(T1 value, Func<T1, T2> f1, Func<T2, T3> f2)
        => f2(f1(value));
    
    public static T4 Pipe<T1, T2, T3, T4>(
        T1 value, Func<T1, T2> f1, Func<T2, T3> f2, Func<T3, T4> f3)
        => f3(f2(f1(value)));
    
    // Extension method pipe
    public static TResult PipeTo<T, TResult>(this T value, Func<T, TResult> func)
        => func(value);
}

// การใช้งาน pipeline
var result = "  Hello, World!  "
    .PipeTo(s => s.Trim())
    .PipeTo(s => s.ToUpper())
    .PipeTo(s => s.Replace(",", ""))
    .PipeTo(s => $"[{s}]");
// [HELLO WORLD!]

// Pipeline สำหรับ price calculation
decimal basePrice = 1000m;
var finalPrice = basePrice
    .PipeTo(p => p * 1.07m)    // เพิ่ม VAT
    .PipeTo(p => p * 0.9m)     // ลด 10%
    .PipeTo(p => Math.Round(p, 2));
Console.WriteLine(finalPrice); // 963
```

---

## Step 757: Discriminated Unions ด้วย C# Records/Sealed Classes

### 757.1 Discriminated Union Pattern

```csharp
// Shape hierarchy - Discriminated Union
public abstract record Shape
{
    public abstract double Area { get; }
    public abstract double Perimeter { get; }
    public abstract string Description { get; }
}

public sealed record Circle(double Radius) : Shape
{
    public override double Area => Math.PI * Radius * Radius;
    public override double Perimeter => 2 * Math.PI * Radius;
    public override string Description => $"วงกลม รัศมี {Radius}";
}

public sealed record Rectangle(double Width, double Height) : Shape
{
    public override double Area => Width * Height;
    public override double Perimeter => 2 * (Width + Height);
    public override string Description => $"สี่เหลี่ยม {Width}x{Height}";
}

public sealed record Triangle(double Base, double Height, double Side1, double Side2, double Side3) : Shape
{
    public override double Area => 0.5 * Base * Height;
    public override double Perimeter => Side1 + Side2 + Side3;
    public override string Description => $"สามเหลี่ยม ฐาน {Base}, สูง {Height}";
}

// Pattern matching กับ Shape
public static class ShapeCalculator
{
    public static string GetScaledDescription(Shape shape, double factor)
        => shape switch
        {
            Circle c => new Circle(c.Radius * factor).Description,
            Rectangle r => new Rectangle(r.Width * factor, r.Height * factor).Description,
            Triangle t => new Triangle(
                t.Base * factor, t.Height * factor,
                t.Side1 * factor, t.Side2 * factor, t.Side3 * factor
            ).Description,
            _ => throw new ArgumentException($"Unknown shape: {shape.GetType().Name}")
        };
    
    public static bool IsLargerThan(Shape shape, double minArea)
        => shape.Area > minArea;
}

// Payment state machine ด้วย Discriminated Union
public abstract record PaymentState;
public sealed record Pending(decimal Amount) : PaymentState;
public sealed record Processing(decimal Amount, string TransactionId) : PaymentState;
public sealed record Completed(decimal Amount, string TransactionId, DateTime CompletedAt) : PaymentState;
public sealed record Failed(decimal Amount, string Reason) : PaymentState;
public sealed record Refunded(decimal Amount, string TransactionId, DateTime RefundedAt) : PaymentState;

public class PaymentProcessor
{
    public string GetStatusMessage(PaymentState state)
        => state switch
        {
            Pending p => $"รอดำเนินการ: ฿{p.Amount:N0}",
            Processing p => $"กำลังประมวลผล [{p.TransactionId}]: ฿{p.Amount:N0}",
            Completed c => $"สำเร็จ [{c.TransactionId}] เมื่อ {c.CompletedAt:dd/MM/yyyy HH:mm}",
            Failed f => $"ล้มเหลว: {f.Reason}",
            Refunded r => $"คืนเงินแล้ว [{r.TransactionId}] เมื่อ {r.RefundedAt:dd/MM/yyyy HH:mm}",
            _ => "ไม่ทราบสถานะ"
        };
    
    public PaymentState Transition(PaymentState current, string action)
        => (current, action) switch
        {
            (Pending p, "process") => new Processing(p.Amount, Guid.NewGuid().ToString()[..8]),
            (Processing p, "complete") => new Completed(p.Amount, p.TransactionId, DateTime.Now),
            (Processing p, "fail") => new Failed(p.Amount, "การชำระเงินถูกปฏิเสธ"),
            (Completed c, "refund") => new Refunded(c.Amount, c.TransactionId, DateTime.Now),
            _ => current // Invalid transition - ไม่เปลี่ยนแปลง
        };
}
```

---

## Step 758: Immutable Data Structures และ Persistent Data

### 758.1 System.Collections.Immutable

```csharp
using System.Collections.Immutable;

// ImmutableList<T>
var originalList = ImmutableList.Create(1, 2, 3, 4, 5);

// การ "แก้ไข" สร้าง list ใหม่ ไม่แก้ original
var withSix = originalList.Add(6);
var withoutFirst = originalList.RemoveAt(0);
var modified = originalList.SetItem(2, 99);

Console.WriteLine(string.Join(", ", originalList));   // 1, 2, 3, 4, 5
Console.WriteLine(string.Join(", ", withSix));        // 1, 2, 3, 4, 5, 6
Console.WriteLine(string.Join(", ", withoutFirst));   // 2, 3, 4, 5
Console.WriteLine(string.Join(", ", modified));       // 1, 2, 99, 4, 5

// ImmutableDictionary<K, V>
var catalog = ImmutableDictionary<string, decimal>.Empty
    .Add("Apple", 20m)
    .Add("Banana", 15m)
    .Add("Cherry", 45m);

var updatedCatalog = catalog.SetItem("Apple", 25m); // ราคาใหม่
var expandedCatalog = catalog.Add("Durian", 150m);

Console.WriteLine(catalog["Apple"]);        // 20 - original ไม่เปลี่ยน
Console.WriteLine(updatedCatalog["Apple"]); // 25

// ImmutableStack / ImmutableQueue
var stack = ImmutableStack<int>.Empty
    .Push(1).Push(2).Push(3);

var (top, remaining) = (stack.Peek(), stack.Pop());
Console.WriteLine(top);                                    // 3
Console.WriteLine(string.Join(", ", remaining));           // 2, 1
Console.WriteLine(string.Join(", ", stack));               // 3, 2, 1 - original ไม่เปลี่ยน
```

### 758.2 Builder Pattern สำหรับ Immutable Objects

```csharp
// Immutable record ที่ซับซ้อน
public record OrderConfiguration
{
    public string Currency { get; init; } = "THB";
    public decimal TaxRate { get; init; } = 0.07m;
    public decimal MinimumOrderAmount { get; init; } = 100m;
    public ImmutableList<string> AllowedPaymentMethods { get; init; } 
        = ImmutableList.Create("Credit Card", "PromptPay");
    public bool RequireEmailConfirmation { get; init; } = true;
    
    // Fluent builder methods
    public OrderConfiguration WithCurrency(string currency)
        => this with { Currency = currency };
    
    public OrderConfiguration WithTaxRate(decimal rate)
        => this with { TaxRate = rate };
    
    public OrderConfiguration WithPaymentMethod(string method)
        => this with { AllowedPaymentMethods = AllowedPaymentMethods.Add(method) };
    
    public OrderConfiguration WithoutPaymentMethod(string method)
        => this with { AllowedPaymentMethods = AllowedPaymentMethods.Remove(method) };
}

// การใช้งาน
var defaultConfig = new OrderConfiguration();

var thaiConfig = defaultConfig
    .WithCurrency("THB")
    .WithTaxRate(0.07m)
    .WithPaymentMethod("TrueMoney Wallet")
    .WithPaymentMethod("LINE Pay");

var usdConfig = defaultConfig
    .WithCurrency("USD")
    .WithTaxRate(0.0825m);

// ทั้งสอง config เป็น immutable และ independent
Console.WriteLine(thaiConfig.AllowedPaymentMethods.Count);  // 4
Console.WriteLine(usdConfig.AllowedPaymentMethods.Count);   // 2
Console.WriteLine(defaultConfig.AllowedPaymentMethods.Count); // 2 - ไม่เปลี่ยน
```

---

## Step 759: Lens Pattern สำหรับ Deep Object Updates

### 759.1 แนวคิด Lens

Lens เป็น abstraction สำหรับ focus เข้าไปใน nested object และทำการ get/set โดยไม่เปลี่ยนแปลง original

```csharp
// Lens<TObject, TProperty>
public record Lens<TObject, TProperty>(
    Func<TObject, TProperty> Get,
    Func<TObject, TProperty, TObject> Set)
{
    // Map - แก้ไขค่าโดยใช้ transformation function
    public TObject Modify(TObject obj, Func<TProperty, TProperty> modify)
        => Set(obj, modify(Get(obj)));
    
    // Compose - เชื่อม lens สองตัว
    public Lens<TObject, TNested> Compose<TNested>(
        Lens<TProperty, TNested> inner)
        => new(
            Get: obj => inner.Get(Get(obj)),
            Set: (obj, nested) => Set(obj, inner.Set(Get(obj), nested))
        );
}

// Domain objects
public record Address(string Street, string City, string PostCode, string Country);
public record ContactInfo(string Phone, string Email, Address? Address);
public record Employee(
    string Id,
    string Name, 
    decimal Salary,
    ContactInfo Contact,
    ImmutableList<string> Skills);

// Lens definitions
public static class Lenses
{
    // Employee lenses
    public static readonly Lens<Employee, string> EmployeeName = new(
        Get: e => e.Name,
        Set: (e, v) => e with { Name = v }
    );
    
    public static readonly Lens<Employee, decimal> EmployeeSalary = new(
        Get: e => e.Salary,
        Set: (e, v) => e with { Salary = v }
    );
    
    public static readonly Lens<Employee, ContactInfo> EmployeeContact = new(
        Get: e => e.Contact,
        Set: (e, v) => e with { Contact = v }
    );
    
    // ContactInfo lenses
    public static readonly Lens<ContactInfo, string> ContactEmail = new(
        Get: c => c.Email,
        Set: (c, v) => c with { Email = v }
    );
    
    public static readonly Lens<ContactInfo, Address?> ContactAddress = new(
        Get: c => c.Address,
        Set: (c, v) => c with { Address = v }
    );
    
    // Address lenses
    public static readonly Lens<Address, string> AddressCity = new(
        Get: a => a.City,
        Set: (a, v) => a with { City = v }
    );
    
    // Composed lenses
    public static readonly Lens<Employee, string> EmployeeEmail 
        = EmployeeContact.Compose(ContactEmail);
    
    public static readonly Lens<Employee, Address?> EmployeeAddress
        = EmployeeContact.Compose(ContactAddress);
}

// การใช้งาน Lens
var emp = new Employee(
    "E001", "สมชาย",
    Salary: 50000m,
    Contact: new ContactInfo("02-123-4567", "somchai@company.com",
        new Address("99 ถ.สุขุมวิท", "กรุงเทพ", "10110", "TH")),
    Skills: ImmutableList.Create("C#", "SQL")
);

// Deep update โดยไม่ต้องเขียน with { Contact = emp.Contact with { Email = ... } }
var updatedEmail = Lenses.EmployeeEmail.Set(emp, "somchai.new@company.com");
var raisedSalary = Lenses.EmployeeSalary.Modify(emp, salary => salary * 1.10m);

// ตรวจสอบว่า original ไม่เปลี่ยน
Console.WriteLine(emp.Contact.Email);        // somchai@company.com
Console.WriteLine(updatedEmail.Contact.Email); // somchai.new@company.com
Console.WriteLine($"{emp.Salary:N0}");       // 50,000
Console.WriteLine($"{raisedSalary.Salary:N0}"); // 55,000

// Chain multiple updates
var fullyUpdated = emp
    |> (e => Lenses.EmployeeSalary.Modify(e, s => s * 1.15m))
    |> (e => Lenses.EmployeeEmail.Set(e, "new@company.com"));
```

> **หมายเหตุ:** C# ยังไม่มี native pipe operator `|>` แต่สามารถใช้ extension methods แทนได้:

```csharp
// Extension method แทน pipe operator
public static class PipeExtensions
{
    public static TResult Pipe<T, TResult>(this T value, Func<T, TResult> func)
        => func(value);
}

var fullyUpdated = emp
    .Pipe(e => Lenses.EmployeeSalary.Modify(e, s => s * 1.15m))
    .Pipe(e => Lenses.EmployeeEmail.Set(e, "new@company.com"));
```

---

## Step 760: Real-World Example: Order Validation Pipeline

### 760.1 โครงสร้างระบบ Order Validation

```csharp
using System.Collections.Immutable;

// Domain models
public record ProductId(string Value)
{
    public static Result<ProductId, ValidationError> Create(string value)
        => string.IsNullOrWhiteSpace(value)
            ? Result<ProductId, ValidationError>.Err(new FieldRequired("ProductId"))
            : Result<ProductId, ValidationError>.Ok(new ProductId(value.Trim()));
}

public record Quantity(int Value)
{
    public static Result<Quantity, ValidationError> Create(int value)
        => value <= 0
            ? Result<Quantity, ValidationError>.Err(
                new OutOfRange("Quantity", 1, int.MaxValue))
            : Result<Quantity, ValidationError>.Ok(new Quantity(value));
}

public record Money(decimal Amount, string Currency = "THB")
{
    public static readonly Money Zero = new(0m);
    
    public static Result<Money, ValidationError> Create(decimal amount, string currency = "THB")
    {
        if (amount < 0)
            return Result<Money, ValidationError>.Err(
                new OutOfRange("Amount", 0, decimal.MaxValue));
        
        if (string.IsNullOrWhiteSpace(currency))
            return Result<Money, ValidationError>.Err(new FieldRequired("Currency"));
        
        return Result<Money, ValidationError>.Ok(new Money(amount, currency));
    }
    
    public Money Add(Money other) => this with { Amount = Amount + other.Amount };
    public Money Multiply(decimal factor) => this with { Amount = Amount * factor };
    public Money ApplyDiscount(decimal percent) => this with { Amount = Amount * (1 - percent) };
    
    public override string ToString() => $"{Amount:N2} {Currency}";
}

public record OrderItem(ProductId ProductId, Quantity Quantity, Money UnitPrice)
{
    public Money TotalPrice => UnitPrice.Multiply(Quantity.Value);
}

public record ShippingAddress(string RecipientName, string Street, string City, string PostCode)
{
    public static Result<ShippingAddress, ValidationError> Create(
        string name, string street, string city, string postCode)
    {
        if (string.IsNullOrWhiteSpace(name))
            return Result<ShippingAddress, ValidationError>.Err(new FieldRequired("RecipientName"));
        
        if (string.IsNullOrWhiteSpace(street))
            return Result<ShippingAddress, ValidationError>.Err(new FieldRequired("Street"));
        
        if (postCode.Length != 5 || !postCode.All(char.IsDigit))
            return Result<ShippingAddress, ValidationError>.Err(
                new InvalidFormat("PostCode", "รหัสไปรษณีย์ต้องเป็นตัวเลข 5 หลัก"));
        
        return Result<ShippingAddress, ValidationError>.Ok(
            new ShippingAddress(name.Trim(), street.Trim(), city.Trim(), postCode));
    }
}

public record ValidatedOrder(
    string OrderId,
    string CustomerId,
    ImmutableList<OrderItem> Items,
    ShippingAddress ShippingAddress,
    Money SubTotal,
    Money Discount,
    Money Tax,
    Money Total,
    DateTime CreatedAt);
```

### 760.2 Validation Rules

```csharp
public abstract record OrderValidationError : ValidationError;
public record EmptyOrderItems : OrderValidationError;
public record OrderAmountTooLow(Money Minimum, Money Actual) : OrderValidationError;
public record ProductUnavailable(string ProductId) : OrderValidationError;

// Validation rules เป็น pure functions
public static class OrderValidationRules
{
    public static Result<ImmutableList<OrderItem>, ValidationError> ValidateItems(
        ImmutableList<OrderItem> items)
        => items.IsEmpty
            ? Result<ImmutableList<OrderItem>, ValidationError>.Err(new EmptyOrderItems())
            : Result<ImmutableList<OrderItem>, ValidationError>.Ok(items);
    
    public static Result<ImmutableList<OrderItem>, ValidationError> ValidateMinimumOrder(
        ImmutableList<OrderItem> items, Money minimumAmount)
    {
        var total = items.Aggregate(Money.Zero, (acc, item) => acc.Add(item.TotalPrice));
        return total.Amount < minimumAmount.Amount
            ? Result<ImmutableList<OrderItem>, ValidationError>.Err(
                new OrderAmountTooLow(minimumAmount, total))
            : Result<ImmutableList<OrderItem>, ValidationError>.Ok(items);
    }
    
    public static Result<ImmutableList<OrderItem>, ValidationError> ValidateProductAvailability(
        ImmutableList<OrderItem> items, IReadOnlySet<string> availableProducts)
    {
        var unavailable = items
            .Where(item => !availableProducts.Contains(item.ProductId.Value))
            .Select(item => item.ProductId.Value)
            .ToList();
        
        return unavailable.Any()
            ? Result<ImmutableList<OrderItem>, ValidationError>.Err(
                new ProductUnavailable(string.Join(", ", unavailable)))
            : Result<ImmutableList<OrderItem>, ValidationError>.Ok(items);
    }
}
```

### 760.3 Order Processing Pipeline

```csharp
public class OrderValidationPipeline
{
    private readonly IReadOnlySet<string> _availableProducts;
    private readonly Money _minimumOrderAmount;
    private readonly decimal _taxRate;
    
    public OrderValidationPipeline(
        IReadOnlySet<string> availableProducts,
        Money minimumOrderAmount,
        decimal taxRate = 0.07m)
    {
        _availableProducts = availableProducts;
        _minimumOrderAmount = minimumOrderAmount;
        _taxRate = taxRate;
    }
    
    public record OrderRequest(
        string CustomerId,
        List<(string ProductId, int Quantity, decimal UnitPrice)> Items,
        string RecipientName,
        string Street,
        string City,
        string PostCode,
        decimal DiscountPercent = 0m);
    
    public Result<ValidatedOrder, ValidationError> Process(OrderRequest request)
    {
        // Step 1: Validate shipping address
        return ShippingAddress.Create(
                request.RecipientName, request.Street, request.City, request.PostCode)
            // Step 2: Build and validate order items
            .Bind(address => BuildOrderItems(request.Items)
                .Bind(items => OrderValidationRules.ValidateItems(items))
                .Bind(items => OrderValidationRules.ValidateProductAvailability(items, _availableProducts))
                .Bind(items => OrderValidationRules.ValidateMinimumOrder(items, _minimumOrderAmount))
                // Step 3: Calculate pricing
                .Map(items => CalculatePricing(items, request.DiscountPercent))
                // Step 4: Assemble final order
                .Map(pricing => AssembleOrder(
                    request.CustomerId, items: pricing.Items, address,
                    pricing.SubTotal, pricing.Discount, pricing.Tax, pricing.Total)));
    }
    
    private Result<ImmutableList<OrderItem>, ValidationError> BuildOrderItems(
        List<(string ProductId, int Quantity, decimal UnitPrice)> rawItems)
    {
        var results = rawItems.Select(item =>
        {
            var productId = ProductId.Create(item.ProductId);
            var quantity = Quantity.Create(item.Quantity);
            var price = Money.Create(item.UnitPrice);
            
            return (productId, quantity, price) switch
            {
                ({ IsSuccess: true } p, { IsSuccess: true } q, { IsSuccess: true } m)
                    => Result<OrderItem, ValidationError>.Ok(
                        new OrderItem(p.GetOrThrow(), q.GetOrThrow(), m.GetOrThrow())),
                ({ IsError: true } p, _, _) => Result<OrderItem, ValidationError>.Err(
                    p.Match(_ => default!, e => e)),
                (_, { IsError: true } q, _) => Result<OrderItem, ValidationError>.Err(
                    q.Match(_ => default!, e => e)),
                (_, _, { IsError: true } m) => Result<OrderItem, ValidationError>.Err(
                    m.Match(_ => default!, e => e)),
                _ => throw new InvalidOperationException()
            };
        }).ToList();
        
        var firstError = results.FirstOrDefault(r => r.IsError);
        if (firstError is not null)
            return firstError.Map(_ => ImmutableList<OrderItem>.Empty);
        
        var items = results
            .Where(r => r.IsSuccess)
            .Select(r => r.GetOrThrow())
            .ToImmutableList();
        
        return Result<ImmutableList<OrderItem>, ValidationError>.Ok(items);
    }
    
    private record PricingResult(
        ImmutableList<OrderItem> Items,
        Money SubTotal, Money Discount, Money Tax, Money Total);
    
    private PricingResult CalculatePricing(
        ImmutableList<OrderItem> items, decimal discountPercent)
    {
        var subTotal = items.Aggregate(Money.Zero, (acc, item) => acc.Add(item.TotalPrice));
        var discount = subTotal.Multiply(discountPercent);
        var afterDiscount = subTotal.Add(new Money(-discount.Amount));
        var tax = afterDiscount.Multiply(_taxRate);
        var total = afterDiscount.Add(tax);
        
        return new PricingResult(items, subTotal, discount, tax, total);
    }
    
    private ValidatedOrder AssembleOrder(
        string customerId, ImmutableList<OrderItem> items,
        ShippingAddress address, Money subTotal, Money discount,
        Money tax, Money total)
        => new ValidatedOrder(
            OrderId: $"ORD-{DateTime.Now:yyyyMMdd}-{Guid.NewGuid().ToString()[..6].ToUpper()}",
            CustomerId: customerId,
            Items: items,
            ShippingAddress: address,
            SubTotal: subTotal,
            Discount: discount,
            Tax: tax,
            Total: total,
            CreatedAt: DateTime.UtcNow
        );
}
```

### 760.4 ทดสอบ Pipeline

```csharp
// Setup
var availableProducts = new HashSet<string> { "P001", "P002", "P003" };
var minimumOrder = new Money(500m);

var pipeline = new OrderValidationPipeline(availableProducts, minimumOrder);

// ✅ Order ที่ถูกต้อง
var validOrder = pipeline.Process(new OrderValidationPipeline.OrderRequest(
    CustomerId: "C001",
    Items: new()
    {
        ("P001", 2, 350m),
        ("P002", 1, 150m)
    },
    RecipientName: "สมชาย ใจดี",
    Street: "99/5 ถ.สุขุมวิท",
    City: "กรุงเทพมหานคร",
    PostCode: "10110",
    DiscountPercent: 0.10m
));

validOrder.Match(
    ok: order => {
        Console.WriteLine($"=== คำสั่งซื้อสำเร็จ ===");
        Console.WriteLine($"เลขที่: {order.OrderId}");
        Console.WriteLine($"ลูกค้า: {order.CustomerId}");
        Console.WriteLine($"ส่งไปที่: {order.ShippingAddress.RecipientName}, {order.ShippingAddress.City}");
        Console.WriteLine($"สินค้า: {order.Items.Count} รายการ");
        Console.WriteLine($"ยอดรวม: {order.SubTotal}");
        Console.WriteLine($"ส่วนลด: -{order.Discount}");
        Console.WriteLine($"ภาษี: {order.Tax}");
        Console.WriteLine($"ยอดสุทธิ: {order.Total}");
        return 0;
    },
    err: error => { Console.WriteLine($"ผิดพลาด: {error}"); return 1; }
);

// ❌ Order ที่มีข้อผิดพลาด
var invalidOrder = pipeline.Process(new OrderValidationPipeline.OrderRequest(
    CustomerId: "C002",
    Items: new() { ("P099", 1, 100m) }, // P099 ไม่มีในระบบ
    RecipientName: "วิชา",
    Street: "123 ถ.พระราม",
    City: "กรุงเทพ",
    PostCode: "ABC12" // รหัสไปรษณีย์ผิด
));

invalidOrder.Match(
    ok: _ => { Console.WriteLine("สำเร็จ"); return 0; },
    err: error => {
        Console.WriteLine($"=== ข้อผิดพลาด ===");
        var msg = error switch
        {
            InvalidFormat f => $"รูปแบบไม่ถูกต้อง [{f.FieldName}]: {f.Message}",
            ProductUnavailable p => $"สินค้าไม่พร้อมขาย: {p.ProductId}",
            EmptyOrderItems => "กรุณาเพิ่มสินค้า",
            OrderAmountTooLow o => $"ยอดขั้นต่ำ {o.Minimum} (ปัจจุบัน {o.Actual})",
            _ => error.ToString()
        };
        Console.WriteLine(msg);
        return 1;
    }
);
```

### 760.5 เปรียบเทียบ: Traditional try/catch vs Result Pattern

```csharp
// ❌ Traditional try/catch approach
public class TraditionalOrderService
{
    public Order CreateOrder(OrderRequest request)
    {
        ValidateAddress(request.Address);   // throws if invalid
        var items = ParseItems(request.Items); // throws if invalid
        ValidateStock(items);               // throws if out of stock
        return BuildOrder(request, items);
    }
    
    // ปัญหา:
    // 1. ไม่รู้ว่า function throws exception อะไรบ้างจาก signature
    // 2. ต้อง catch หลาย exception types
    // 3. Error handling กระจาย
    // 4. ยากต่อการ compose
}

// ✅ Result pattern approach
public class FunctionalOrderService
{
    // Signature บอกชัดเจนว่า:
    // - Return Type: ValidatedOrder หรือ ValidationError
    // - ไม่ throw exceptions ใน normal flow
    public Result<ValidatedOrder, ValidationError> CreateOrder(OrderRequest request)
        => ValidateAddress(request.Address)        // Result<Address, Error>
            .Bind(addr => ParseItems(request.Items)) // Result<Items, Error>
            .Bind(items => ValidateStock(items))    // Result<Items, Error>
            .Map(items => BuildOrder(request, items)); // ValidatedOrder
    
    // ข้อดี:
    // 1. Type signature บอก error cases ทั้งหมด
    // 2. Error handling อยู่ที่เดียว (Match)
    // 3. Composable - chain ง่าย
    // 4. Testable - pure functions
    // 5. Railway pattern - short-circuit เมื่อ error
}

// Unit Testing Result pattern ง่ายกว่า
[TestClass]
public class OrderValidationTests
{
    [TestMethod]
    public void ValidOrder_ReturnsSuccess()
    {
        var service = CreateService();
        var result = service.CreateOrder(ValidOrderRequest());
        Assert.IsTrue(result.IsSuccess);
    }
    
    [TestMethod]
    public void InvalidPostCode_ReturnsValidationError()
    {
        var service = CreateService();
        var result = service.CreateOrder(RequestWithInvalidPostCode());
        
        result.Match(
            ok: _ => Assert.Fail("Should have failed"),
            err: error => Assert.IsInstanceOfType(error, typeof(InvalidFormat))
        );
    }
}
```

---

## สรุป Steps 751-760

| Step | หัวข้อ | Pattern หลัก |
|------|--------|-------------|
| 751 | Functional Concepts | Pure Functions, HOF, Immutability |
| 752 | Option\<T\> Monad | Map, Bind, Match, GetOrElse |
| 753 | Result\<T, E\> Type | Ok/Err, Map, MapError, Match |
| 754 | Railway-Oriented | Bind chaining, short-circuit |
| 755 | LINQ Pipelines | Select, Where, Aggregate, GroupBy |
| 756 | Partial Application | Curry, Flip, Compose |
| 757 | Discriminated Unions | sealed record, pattern matching |
| 758 | Immutable Structures | ImmutableList, ImmutableDictionary |
| 759 | Lens Pattern | Get/Set/Modify/Compose |
| 760 | Real-world Pipeline | Combined patterns |

### หลักการสำคัญ

1. **Prefer immutability** - ใช้ `record` และ `init` properties
2. **Make errors explicit** - ใช้ `Result<T, E>` แทน exceptions ใน domain logic
3. **Chain operations** - ใช้ `Bind/Map` สร้าง readable pipeline
4. **Pure functions** - ฟังก์ชันไม่มี side effects → ทดสอบง่าย
5. **Types as documentation** - Type signature บอก behavior ชัดเจน

---

**ก่อนหน้า → [Part 75: Background Services](part75-background-services.md)**

**ต่อไป → [Part 77: WPF Advanced](part77-wpf-advanced.md)**
