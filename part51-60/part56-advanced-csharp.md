# Part 56: Advanced C# Language Features
## ขั้นตอนที่ 551-560: ฟีเจอร์ภาษา C# ขั้นสูง

---

## 🎯 เป้าหมายของ Part นี้
- Records และ Immutability สำหรับ Domain Models
- Pattern Matching ขั้นสูง (Property, Positional, List Patterns)
- Nullable Reference Types และ Flow Analysis
- Generic Math ด้วย INumber<T> (.NET 7+)
- Source Generators สำหรับ Code Generation อัตโนมัติ
- ฟีเจอร์ใหม่ใน C# 12/13

---

## ขั้นตอนที่ 551: Records & Immutability

### แนวคิด Records

Records คือ value-based data types ที่ถูกออกแบบมาเพื่อ immutability และ data representation  
แทนที่จะใช้ class ธรรมดา Records มาพร้อม equality comparison, toString, deconstruction ในตัวเอง

```csharp
// === record class (reference type, immutable by default) ===

// Positional record - สั้นที่สุด สร้าง constructor + properties อัตโนมัติ
public record Person(string FirstName, string LastName, int Age);

// ใช้งาน
var alice = new Person("Alice", "Smith", 30);
var bob   = new Person("Bob", "Jones", 25);

Console.WriteLine(alice);          // Person { FirstName = Alice, LastName = Smith, Age = 30 }
Console.WriteLine(alice == bob);   // False
Console.WriteLine(alice == new Person("Alice", "Smith", 30)); // True (value equality)

// Nominal record - ประกาศ properties แบบ explicit
public record Order
{
    public required Guid Id          { get; init; }
    public required string CustomerId { get; init; }
    public required decimal Total    { get; init; }
    public required OrderStatus Status { get; init; }
    public DateTime CreatedAt        { get; init; } = DateTime.UtcNow;
}

// init-only properties - ตั้งค่าได้แค่ตอน object initialization
var order = new Order
{
    Id         = Guid.NewGuid(),
    CustomerId = "CUST-001",
    Total      = 2500.00m,
    Status     = OrderStatus.Pending
};

// order.Total = 3000m;  // ❌ Compile error: init-only property
```

---

### With Expressions — Non-Destructive Mutation

```csharp
// with expression: สร้าง copy ใหม่ที่เปลี่ยนเฉพาะ fields ที่ระบุ
// ตัว original ไม่เปลี่ยน

var alice     = new Person("Alice", "Smith", 30);
var olderAlice = alice with { Age = 31 };

Console.WriteLine(alice.Age);      // 30 (ไม่เปลี่ยน)
Console.WriteLine(olderAlice.Age); // 31

// Order domain example
var pendingOrder  = new Order { Id = id, CustomerId = "C1", Total = 500m, Status = OrderStatus.Pending };
var confirmedOrder = pendingOrder with { Status = OrderStatus.Confirmed };
var shippedOrder   = confirmedOrder with { Status = OrderStatus.Shipped };

// เหมาะมากกับ Event Sourcing pattern
Order Apply(Order state, IOrderEvent evt) => evt switch
{
    OrderConfirmed  => state with { Status = OrderStatus.Confirmed },
    OrderShipped e  => state with { Status = OrderStatus.Shipped },
    OrderCancelled  => state with { Status = OrderStatus.Cancelled },
    _               => state
};
```

---

### record struct vs record class

```csharp
// record struct: value type (stack-allocated), mutable โดย default
public record struct Point(double X, double Y);

// Mutable record struct
public record struct MutablePoint(double X, double Y)
{
    public double X { get; set; } = X;  // set แทน init
    public double Y { get; set; } = Y;
}

// readonly record struct: value type + immutable
public readonly record struct Temperature(double Celsius)
{
    public double Fahrenheit => Celsius * 9 / 5 + 32;
    public double Kelvin     => Celsius + 273.15;
}

// เปรียบเทียบการใช้งาน
var p1 = new Point(3.0, 4.0);
var p2 = new Point(3.0, 4.0);
Console.WriteLine(p1 == p2);  // True (value equality ทั้ง class และ struct)

// Performance: record struct ไม่ allocate heap ทำให้เร็วกว่า record class
// ใช้ record struct เมื่อ: เป็น small value (< 16 bytes), ไม่มี inheritance
// ใช้ record class เมื่อ: ต้องการ inheritance, large object, reference semantics
```

---

### Record Inheritance

```csharp
// record class รองรับ inheritance
public abstract record Shape(string Color);

public record Circle(string Color, double Radius) : Shape(Color)
{
    public double Area => Math.PI * Radius * Radius;
}

public record Rectangle(string Color, double Width, double Height) : Shape(Color)
{
    public double Area => Width * Height;
}

// Deconstruction ทำงานตาม hierarchy
var circle = new Circle("Red", 5.0);
var (color, radius) = circle;   // Deconstruct
Console.WriteLine($"{color}, r={radius}");

// with expression ใช้งานได้ใน derived records
var biggerCircle = circle with { Radius = 10.0 };
```

---

### Immutable Domain Models

```csharp
// Design pattern: immutable aggregate root
public record ProductId(Guid Value)
{
    public static ProductId New() => new(Guid.NewGuid());
    public override string ToString() => Value.ToString();
}

public record Money(decimal Amount, string Currency)
{
    public static readonly Money Zero = new(0m, "THB");

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Cannot add different currencies");
        return this with { Amount = Amount + other.Amount };
    }

    public Money Multiply(decimal factor) => this with { Amount = Amount * factor };
    
    public override string ToString() => $"{Amount:N2} {Currency}";
}

public record Product(
    ProductId Id,
    string Name,
    Money Price,
    int StockQuantity)
{
    public Product AdjustPrice(Money newPrice)    => this with { Price = newPrice };
    public Product AddStock(int quantity)         => this with { StockQuantity = StockQuantity + quantity };
    public Product RemoveStock(int quantity)
    {
        if (quantity > StockQuantity)
            throw new InvalidOperationException("Insufficient stock");
        return this with { StockQuantity = StockQuantity - quantity };
    }
    
    public bool IsAvailable => StockQuantity > 0;
}

// Usage: ทุก operation คืน instance ใหม่ ไม่แก้ไข original
var product = new Product(ProductId.New(), "T-Shirt", new Money(299m, "THB"), 100);
var updated = product.RemoveStock(5).AdjustPrice(new Money(279m, "THB"));

Console.WriteLine(product.StockQuantity);  // 100 (ไม่เปลี่ยน)
Console.WriteLine(updated.StockQuantity);  // 95
```

---

## ขั้นตอนที่ 552: Pattern Matching (Advanced)

### Switch Expression พร้อม Patterns

```csharp
// Switch expression: คืนค่า ไม่ใช่ statement
// ทุก case ต้องครอบคลุม (exhaustive) หรือมี _ wildcard

// Type patterns
string Describe(object obj) => obj switch
{
    int n when n < 0    => $"Negative integer: {n}",
    int n               => $"Non-negative integer: {n}",
    string s            => $"String of length {s.Length}",
    null                => "Null value",
    _                   => $"Something else: {obj.GetType().Name}"
};

// Constant patterns
string GetDiscount(string memberTier) => memberTier switch
{
    "Gold"     => "20%",
    "Silver"   => "10%",
    "Bronze"   => "5%",
    _          => "0%"
};
```

---

### Property Patterns

```csharp
// Property patterns: match บน properties ของ object
// Syntax: { PropertyName: pattern }

public record OrderItem(string ProductId, decimal Price, int Quantity, bool IsActive);
public record Order(string Id, OrderStatus Status, decimal Total, List<OrderItem> Items);

// ตัวอย่าง Property Pattern
string ClassifyOrder(Order order) => order switch
{
    { Status: OrderStatus.Cancelled }         => "Cancelled order",
    { Status: OrderStatus.Pending, Total: < 100 }   => "Small pending order",
    { Status: OrderStatus.Pending, Total: >= 100 }  => "Large pending order",
    { Status: OrderStatus.Active, Total: > 1000 }   => "High-value active order",
    { Status: OrderStatus.Active }            => "Regular active order",
    { Status: OrderStatus.Shipped }           => "Shipped order",
    _                                         => "Unknown state"
};

// Nested property patterns
decimal CalculateShipping(Order order) => order switch
{
    // Cancelled ไม่คิดค่าส่ง
    { Status: OrderStatus.Cancelled } => 0m,

    // Free shipping เมื่อ Total > 1000
    { Total: > 1000, Status: OrderStatus.Active or OrderStatus.Pending } => 0m,

    // Express shipping สำหรับ premium customers
    { Items: { Count: > 5 }, Status: OrderStatus.Active } => 150m,

    // Standard rate
    _ => 50m
};

// Deep nested property patterns
bool IsHighValueActiveCustomer(Customer c) => c switch
{
    {
        Status: CustomerStatus.Active,
        Address: { Country: "TH" },
        Profile: { TotalPurchases: > 50000m }
    } => true,
    _ => false
};
```

---

### Positional Patterns กับ Deconstruct

```csharp
// Positional patterns: ใช้ Deconstruct method
// Syntax: (pattern1, pattern2, ...)

public record Point(double X, double Y);

string ClassifyPoint(Point p) => p switch
{
    (0, 0)      => "Origin",
    (0, _)      => "On Y-axis",
    (_, 0)      => "On X-axis",
    (> 0, > 0)  => "First quadrant",
    (< 0, > 0)  => "Second quadrant",
    (< 0, < 0)  => "Third quadrant",
    _           => "Fourth quadrant"
};

// กับ tuple
string ClassifyTemperature(double celsius, string unit) => (celsius, unit) switch
{
    (< 0, "C")      => "Below freezing",
    (>= 100, "C")   => "At or above boiling",
    (< 32, "F")     => "Below freezing (F)",
    (>= 212, "F")   => "At or above boiling (F)",
    _               => "Normal temperature"
};

// Deconstructing custom types
public class Rectangle
{
    public double Width { get; }
    public double Height { get; }
    
    public Rectangle(double width, double height) => (Width, Height) = (width, height);
    
    // Deconstruct method
    public void Deconstruct(out double width, out double height)
        => (width, height) = (Width, Height);
}

string ClassifyRect(Rectangle r) => r switch
{
    (var w, var h) when w == h   => "Square",
    (var w, var h) when w > h    => "Landscape",
    _                             => "Portrait"
};
```

---

### List Patterns (C# 11+)

```csharp
// List patterns: match บน collections
// [first, .., last] - spread pattern (..) match ส่วนที่เหลือ

string DescribeList(int[] numbers) => numbers switch
{
    []                         => "Empty list",
    [var single]               => $"Single element: {single}",
    [var first, var second]    => $"Two elements: {first}, {second}",
    [var first, .., var last]  => $"Many elements: starts {first}, ends {last}",
};

// Pattern กับ ค่า specific
bool IsValidPrefix(string[] parts) => parts switch
{
    ["https", ..]   => true,
    ["http", ..]    => true,
    _               => false
};

// Nested list patterns
bool IsSorted(int[] arr) => arr switch
{
    []      => true,
    [_]     => true,
    [var a, var b, ..] when a <= b => IsSorted(arr[1..]),
    _       => false
};

// List pattern กับ property pattern รวมกัน
string AnalyzeOrders(Order[] orders) => orders switch
{
    []                                      => "No orders",
    [{ Status: OrderStatus.Pending }]       => "One pending order",
    [.., { Status: OrderStatus.Cancelled }] => "Last order was cancelled",
    [var first, var second, ..]
        when first.Total > second.Total     => "Decreasing order totals",
    _                                       => $"{orders.Length} orders"
};
```

---

### Guard Clauses ด้วย when

```csharp
// when clause: เพิ่มเงื่อนไข boolean เพิ่มเติม
// ทำงานหลัง pattern match สำเร็จ

decimal CalculateDiscount(Customer customer, decimal orderTotal) => customer switch
{
    // VIP + large order
    { Tier: "VIP" } when orderTotal > 5000 => orderTotal * 0.25m,
    
    // VIP regular
    { Tier: "VIP" }                         => orderTotal * 0.20m,
    
    // Gold member
    { Tier: "Gold" } when orderTotal > 2000 => orderTotal * 0.15m,
    { Tier: "Gold" }                        => orderTotal * 0.10m,
    
    // First time buyer
    { PurchaseCount: 0 }                    => orderTotal * 0.05m,
    
    // Default
    _                                       => 0m
};

// when กับ complex expressions
string ValidatePassword(string password) => password switch
{
    null or ""                                  => "Password is required",
    { Length: < 8 }                             => "Too short (min 8 chars)",
    var p when !p.Any(char.IsUpper)             => "Must contain uppercase",
    var p when !p.Any(char.IsLower)             => "Must contain lowercase",
    var p when !p.Any(char.IsDigit)             => "Must contain digit",
    var p when p.All(char.IsLetterOrDigit)      => "Must contain special character",
    _                                           => "Password is valid"
};
```

---

## ขั้นตอนที่ 553: Nullable Reference Types

### Enable Nullable Context

```csharp
// ใน .csproj เปิดใช้ทั้ง project:
// <Nullable>enable</Nullable>

// หรือ per-file:
#nullable enable

// หลัง enable:
// - Reference types (string, MyClass) เป็น non-nullable โดย default
// - ต้องใช้ ? สำหรับ nullable references

string  name    = "Alice";   // Non-nullable: ต้องไม่เป็น null
string? optName = null;      // Nullable: อาจเป็น null

// Compiler warning เมื่อ:
name = null;                  // ⚠️ Warning: null assigned to non-nullable
Console.WriteLine(optName.Length); // ⚠️ Warning: possible null dereference
```

---

### Null-Forgiving Operator (!)

```csharp
// ! บอก compiler ว่า "ฉันรู้ว่าค่านี้ไม่ใช่ null"
// ใช้เมื่อ compiler ไม่สามารถพิสูจน์ได้ แต่เราแน่ใจ

string? maybeNull = GetValueFromConfig("key");

// เราตรวจสอบแล้วว่าไม่ใช่ null ด้วยวิธีอื่น
if (IsKeyValid("key"))
{
    string definitelyNotNull = maybeNull!;  // null-forgiving
    Console.WriteLine(definitelyNotNull.Length);
}

// ใน DI: ถ้า required service ไม่ใช่ null แน่นอน
public class OrderService
{
    private readonly IOrderRepository _repo = null!; // จะถูก inject
    
    public OrderService(IOrderRepository repo)
    {
        _repo = repo;
    }
}
```

---

### Nullable Attributes

```csharp
using System.Diagnostics.CodeAnalysis;

// [MaybeNull] - method อาจคืน null แม้ return type เป็น non-nullable
[return: MaybeNull]
public T Find<T>(IEnumerable<T> items, Func<T, bool> predicate)
    => items.FirstOrDefault(predicate);

// [NotNull] - parameter/return ไม่ใช่ null หลังจากเรียก method นี้
public void EnsureNotNull([NotNull] object? value, string paramName)
{
    if (value is null)
        throw new ArgumentNullException(paramName);
}

// [NotNullWhen] - parameter ไม่ใช่ null เมื่อ method คืน true/false
public bool TryGetUser(string id, [NotNullWhen(true)] out User? user)
{
    user = _users.GetValueOrDefault(id);
    return user is not null;
}

// [MemberNotNull] - property จะไม่ใช่ null หลังจาก method นี้ทำงาน
public class Service
{
    private string? _connectionString;
    
    [MemberNotNull(nameof(_connectionString))]
    public void Initialize(string connStr)
    {
        _connectionString = connStr;
    }
    
    public void DoWork()
    {
        Initialize("Server=localhost;");
        // หลังจากนี้ compiler รู้ว่า _connectionString ไม่ใช่ null
        Console.WriteLine(_connectionString.Length); // No warning
    }
}
```

---

### Nullable Flow Analysis

```csharp
// Compiler ติดตาม null-state ตลอดโปรแกรม

string? GetName() => null;

void Example()
{
    string? name = GetName();
    
    // ก่อนตรวจสอบ: name อาจเป็น null
    // name.Length;  // ⚠️ Warning
    
    if (name != null)
    {
        // ในบล็อคนี้ compiler รู้ว่า name ไม่ใช่ null
        Console.WriteLine(name.Length);  // ✅ No warning
    }
    
    // หลังบล็อค: กลับเป็น nullable อีกครั้ง
    // name.Length;  // ⚠️ Warning
    
    // Pattern matching ก็ช่วย flow analysis ได้
    if (name is string s)
    {
        Console.WriteLine(s.Length);  // ✅ s เป็น non-nullable string
    }
    
    // Null-coalescing
    string definite = name ?? "default";
    Console.WriteLine(definite.Length);  // ✅ No warning
    
    // Null-conditional: คืน nullable
    int? len = name?.Length;  // int? ไม่ใช่ int
}

// Flow analysis ผ่าน early return
string ProcessName(string? name)
{
    if (name is null)
        throw new ArgumentNullException(nameof(name));
    
    // หลัง throw compiler รู้ว่า name ไม่ใช่ null
    return name.ToUpper();  // ✅ No warning
}
```

---

### Migrating Existing Code

```csharp
// กลยุทธ์การ migrate:
// 1. เปิด <Nullable>warnings</Nullable> ก่อน (warnings เท่านั้น ไม่ใช่ errors)
// 2. แก้ทีละ namespace/file
// 3. เปลี่ยนเป็น <Nullable>enable</Nullable>

// Before migration (nullable disabled):
public class CustomerService
{
    public Customer GetCustomer(string id)
    {
        return _db.Find(id); // อาจคืน null
    }
    
    public string GetEmail(Customer customer)
    {
        return customer.Email; // Email อาจเป็น null
    }
}

// After migration (nullable enabled):
public class CustomerService
{
    public Customer? GetCustomer(string id)    // nullable return
    {
        return _db.Find(id);
    }
    
    public string GetEmail(Customer customer)  // non-nullable parameter
    {
        return customer.Email ?? string.Empty; // handle null
    }
    
    // หรือใช้ ThrowIfNull
    public string GetEmailOrThrow(string? id)
    {
        ArgumentNullException.ThrowIfNull(id);
        var customer = GetCustomer(id);
        return customer?.Email
            ?? throw new InvalidOperationException($"Customer {id} not found");
    }
}
```

---

## ขั้นตอนที่ 554: Generic Math (.NET 7+)

### INumber<T> Interface

```csharp
// .NET 7 นำเสนอ Generic Math interfaces ใน System.Numerics
// ทำให้เขียน generic algorithms ที่ทำงานกับ numeric types ได้

using System.Numerics;

// INumber<T> combines: IComparable, IAdditionOperators, ISubtractionOperators,
//                      IMultiplyOperators, IDivisionOperators, etc.

// Generic Sum
public static T Sum<T>(IEnumerable<T> values) where T : INumber<T>
{
    T result = T.Zero;
    foreach (var value in values)
        result += value;
    return result;
}

// Generic Average
public static T Average<T>(IEnumerable<T> values) where T : INumber<T>
{
    var list = values.ToList();
    if (list.Count == 0) throw new InvalidOperationException("Sequence is empty");
    
    T sum  = Sum(list);
    T count = T.CreateChecked(list.Count);
    return sum / count;
}

// ใช้งาน
var ints    = new[] { 1, 2, 3, 4, 5 };
var doubles = new[] { 1.5, 2.5, 3.5 };
var decimals = new[] { 10.1m, 20.2m, 30.3m };

Console.WriteLine(Sum(ints));      // 15
Console.WriteLine(Average(doubles)); // 2.5
Console.WriteLine(Sum(decimals));  // 60.6
```

---

### Arithmetic Operators Interfaces

```csharp
// Interfaces แยกย่อยสำหรับ operators แต่ละอัน
// IAdditionOperators<TSelf, TOther, TResult>
// ISubtractionOperators<TSelf, TOther, TResult>
// IMultiplyOperators<TSelf, TOther, TResult>
// IDivisionOperators<TSelf, TOther, TResult>

// Generic Min/Max
public static T Min<T>(T a, T b) where T : IComparable<T>
    => a.CompareTo(b) <= 0 ? a : b;

public static T Max<T>(T a, T b) where T : IComparable<T>
    => a.CompareTo(b) >= 0 ? a : b;

// Clamp: จำกัดค่าให้อยู่ระหว่าง min-max
public static T Clamp<T>(T value, T min, T max) 
    where T : INumber<T>
    => T.Clamp(value, min, max);

// Generic statistics
public static (T Min, T Max, T Sum, T Average) Statistics<T>(IEnumerable<T> data)
    where T : INumber<T>
{
    var list = data.ToList();
    if (list.Count == 0) throw new InvalidOperationException("Empty sequence");
    
    T min  = list[0];
    T max  = list[0];
    T sum  = T.Zero;
    
    foreach (var item in list)
    {
        if (item < min) min = item;
        if (item > max) max = item;
        sum += item;
    }
    
    T count = T.CreateChecked(list.Count);
    return (min, max, sum, sum / count);
}

// ใช้งาน
var (min, max, sum, avg) = Statistics(new[] { 3.0, 1.5, 4.2, 2.8, 5.1 });
Console.WriteLine($"Min={min}, Max={max}, Sum={sum}, Avg={avg}");
```

---

### IParsable<T> และ Generic Parsing

```csharp
// IParsable<T>: สำหรับ types ที่ parse จาก string ได้
// ทำให้เขียน generic input parsing

public static T ParseOrDefault<T>(string input, T defaultValue)
    where T : IParsable<T>
{
    return T.TryParse(input, null, out T? result) ? result : defaultValue;
}

// Generic CSV parser
public static IEnumerable<T> ParseCsv<T>(string csv)
    where T : INumber<T>
{
    return csv.Split(',')
              .Select(s => s.Trim())
              .Where(s => s.Length > 0)
              .Select(s => T.Parse(s, null));
}

// ใช้งาน
var intValue  = ParseOrDefault("42", 0);           // 42
var dblValue  = ParseOrDefault("3.14", 0.0);       // 3.14
var bad       = ParseOrDefault("not-a-number", -1); // -1 (default)

var intList = ParseCsv<int>("1, 2, 3, 4, 5").ToList();    // [1,2,3,4,5]
var dblList = ParseCsv<double>("1.1, 2.2, 3.3").ToList(); // [1.1,2.2,3.3]
```

---

### Custom Type กับ INumber<T>

```csharp
// สร้าง custom numeric type ที่รองรับ generic math
public readonly struct Fraction 
    : INumber<Fraction>,
      IAdditionOperators<Fraction, Fraction, Fraction>,
      ISubtractionOperators<Fraction, Fraction, Fraction>
{
    public int Numerator   { get; }
    public int Denominator { get; }
    
    public Fraction(int numerator, int denominator)
    {
        if (denominator == 0) throw new DivideByZeroException();
        var gcd  = GCD(Math.Abs(numerator), Math.Abs(denominator));
        Numerator   = numerator / gcd;
        Denominator = denominator / gcd;
    }
    
    private static int GCD(int a, int b) => b == 0 ? a : GCD(b, a % b);
    
    public static Fraction Zero => new(0, 1);
    public static Fraction One  => new(1, 1);
    
    public static Fraction operator +(Fraction left, Fraction right)
        => new(left.Numerator * right.Denominator + right.Numerator * left.Denominator,
               left.Denominator * right.Denominator);
    
    public static Fraction operator -(Fraction left, Fraction right)
        => new(left.Numerator * right.Denominator - right.Numerator * left.Denominator,
               left.Denominator * right.Denominator);
    
    public static Fraction operator *(Fraction left, Fraction right)
        => new(left.Numerator * right.Numerator, left.Denominator * right.Denominator);
    
    public static Fraction operator /(Fraction left, Fraction right)
        => new(left.Numerator * right.Denominator, left.Denominator * right.Numerator);
    
    // ... implement remaining INumber<T> members ...
    
    public override string ToString() => $"{Numerator}/{Denominator}";
}

// ใช้กับ generic Sum
var fractions = new[] { new Fraction(1, 2), new Fraction(1, 3), new Fraction(1, 6) };
var total     = Sum(fractions);  // 1/1 (= 1)
Console.WriteLine(total);        // 1/1
```

---

## ขั้นตอนที่ 555: Source Generators

### Source Generators คืออะไร

Source Generators คือโค้ดที่ทำงานระหว่าง compilation และสร้าง C# source code เพิ่มเติม  
แก้ปัญหาของ reflection-heavy code (เช่น JSON serialization, ORM mapping) ให้เป็น compile-time แทน

```
Build Pipeline:
  Source Code → [Roslyn Compiler] → [Source Generators run] → [Generated Code added] → Assembly
```

---

### IIncrementalGenerator

```csharp
// สร้าง NuGet package แยกต่างหาก (Microsoft.CodeAnalysis.CSharp)
// ใช้ IIncrementalGenerator แทน ISourceGenerator (deprecated)

using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp.Syntax;
using System.Text;

[Generator]
public class ToStringGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        // ค้นหา classes ที่มี [AutoToString] attribute
        var classDeclarations = context.SyntaxProvider
            .CreateSyntaxProvider(
                predicate: static (node, _) => node is ClassDeclarationSyntax c
                    && c.AttributeLists.Count > 0,
                transform: static (ctx, _) => GetClassWithAttribute(ctx))
            .Where(static m => m is not null);

        context.RegisterSourceOutput(classDeclarations,
            static (spc, source) => Execute(source!, spc));
    }

    private static ClassDeclarationSyntax? GetClassWithAttribute(
        GeneratorSyntaxContext context)
    {
        var classDecl = (ClassDeclarationSyntax)context.Node;
        foreach (var attributeList in classDecl.AttributeLists)
        {
            foreach (var attr in attributeList.Attributes)
            {
                if (context.SemanticModel.GetSymbolInfo(attr).Symbol
                    is IMethodSymbol method &&
                    method.ContainingType.ToDisplayString() == "AutoToStringAttribute")
                {
                    return classDecl;
                }
            }
        }
        return null;
    }

    private static void Execute(ClassDeclarationSyntax classDecl,
        SourceProductionContext context)
    {
        var className  = classDecl.Identifier.Text;
        var properties = classDecl.Members
            .OfType<PropertyDeclarationSyntax>()
            .Select(p => p.Identifier.Text)
            .ToList();

        var sb = new StringBuilder();
        sb.AppendLine("// <auto-generated/>");
        sb.AppendLine($"public partial class {className}");
        sb.AppendLine("{");
        sb.AppendLine("    public override string ToString()");
        sb.AppendLine("    {");
        sb.Append($"        return $\"{className} {{ ");
        for (int i = 0; i < properties.Count; i++)
        {
            if (i > 0) sb.Append(", ");
            sb.Append($"{properties[i]} = {{{properties[i]}}}");
        }
        sb.AppendLine(" }}\";");
        sb.AppendLine("    }");
        sb.AppendLine("}");

        context.AddSource($"{className}.g.cs", sb.ToString());
    }
}
```

---

### Auto-generate INotifyPropertyChanged

```csharp
// Attribute สำหรับ trigger generator
[AttributeUsage(AttributeTargets.Class)]
public class NotifyPropertyChangedAttribute : Attribute { }

// User code: แค่ attribute + partial class
[NotifyPropertyChanged]
public partial class PersonViewModel
{
    private string _name    = "";
    private int    _age     = 0;
    private string _email   = "";
}

// Generator จะสร้างโค้ดนี้อัตโนมัติ:
// PersonViewModel.g.cs
public partial class PersonViewModel : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler? PropertyChanged;

    protected void OnPropertyChanged([CallerMemberName] string? name = null)
        => PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name));

    public string Name
    {
        get => _name;
        set { if (_name != value) { _name = value; OnPropertyChanged(); } }
    }

    public int Age
    {
        get => _age;
        set { if (_age != value) { _age = value; OnPropertyChanged(); } }
    }

    public string Email
    {
        get => _email;
        set { if (_email != value) { _email = value; OnPropertyChanged(); } }
    }
}
```

---

### Generate Mapping Code ระหว่าง DTOs

```csharp
// Attribute สำหรับ mapping
[AttributeUsage(AttributeTargets.Class)]
public class MapFromAttribute : Attribute
{
    public Type SourceType { get; }
    public MapFromAttribute(Type sourceType) => SourceType = sourceType;
}

// Domain model
public class OrderEntity
{
    public Guid    Id         { get; set; }
    public string  CustomerId { get; set; } = "";
    public decimal Total      { get; set; }
    public string  Status     { get; set; } = "";
    public DateTime CreatedAt { get; set; }
}

// DTO ที่ต้องการ mapping
[MapFrom(typeof(OrderEntity))]
public partial class OrderDto
{
    public string Id         { get; set; } = "";
    public string CustomerId { get; set; } = "";
    public decimal Total     { get; set; }
    public string Status     { get; set; } = "";
}

// Generator สร้าง extension method:
// OrderDto.g.cs
public partial class OrderDto
{
    public static OrderDto FromEntity(OrderEntity entity)
    {
        return new OrderDto
        {
            Id         = entity.Id.ToString(),
            CustomerId = entity.CustomerId,
            Total      = entity.Total,
            Status     = entity.Status
        };
    }

    public static List<OrderDto> FromEntities(IEnumerable<OrderEntity> entities)
        => entities.Select(FromEntity).ToList();
}

// ใช้งาน (โค้ด generated ใช้ได้ทันที)
var entity = GetOrderFromDatabase(orderId);
var dto    = OrderDto.FromEntity(entity);  // generated code
```

---

### ToString Auto-generation สำหรับ Records

```csharp
// สำหรับ records ธรรมดา C# สร้าง ToString ให้อัตโนมัติ
// แต่ถ้าต้องการ format พิเศษ ใช้ source generator

[AttributeUsage(AttributeTargets.Class | AttributeTargets.Struct)]
public class AutoToStringAttribute : Attribute
{
    public string Format { get; set; } = "default";
}

// User code:
[AutoToString(Format = "json-like")]
public partial record Product(string Name, decimal Price, int Stock);

// Generated:
// Product.g.cs
public partial record Product
{
    public override string ToString()
        => $"{{ \"Name\": \"{Name}\", \"Price\": {Price}, \"Stock\": {Stock} }}";
}

// ใช้งาน
var p = new Product("Widget", 29.99m, 100);
Console.WriteLine(p);
// { "Name": "Widget", "Price": 29.99, "Stock": 100 }
```

---

## ขั้นตอนที่ 556-560: C# 12/13 Features

### Step 556: Primary Constructors สำหรับ Classes (C# 12)

```csharp
// Primary constructors: ประกาศ parameters ใน class declaration
// เดิมมีแค่ใน record แต่ C# 12 นำมาใช้กับ class/struct ด้วย

// Before C# 12 (verbose)
public class EmailService
{
    private readonly ISmtpClient _smtpClient;
    private readonly ILogger<EmailService> _logger;
    private readonly string _fromAddress;
    
    public EmailService(ISmtpClient smtpClient, 
                        ILogger<EmailService> logger, 
                        string fromAddress)
    {
        _smtpClient  = smtpClient;
        _logger      = logger;
        _fromAddress = fromAddress;
    }
}

// C# 12: primary constructor
public class EmailService(ISmtpClient smtpClient, 
                          ILogger<EmailService> logger, 
                          string fromAddress)
{
    // Parameters ใช้ได้ตลอด class body
    public async Task SendAsync(string to, string subject, string body)
    {
        logger.LogInformation("Sending email to {To}", to);
        await smtpClient.SendAsync(fromAddress, to, subject, body);
    }
}

// กับ DI (Dependency Injection)
public class OrderService(
    IOrderRepository orderRepo,
    IEmailService emailService,
    ILogger<OrderService> logger)
{
    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        logger.LogInformation("Creating order for {Customer}", request.CustomerId);
        
        var order = new Order { /* ... */ };
        await orderRepo.SaveAsync(order);
        await emailService.SendOrderConfirmationAsync(order);
        
        return order;
    }
}

// Primary constructor กับ base class
public class CachingOrderService(
    IOrderRepository repo,
    IMemoryCache cache,
    ILogger<CachingOrderService> logger) : OrderService(repo, null!, logger)
{
    public new async Task<Order?> GetByIdAsync(Guid id)
    {
        if (cache.TryGetValue(id, out Order? cached))
            return cached;
        
        var order = await base.GetByIdAsync(id);
        cache.Set(id, order, TimeSpan.FromMinutes(5));
        return order;
    }
}
```

---

### Step 557: Collection Expressions (C# 12)

```csharp
// Collection expressions: syntax ใหม่สำหรับสร้าง collections
// Syntax: [elem1, elem2, ..spread, elem3]

// Array
int[] numbers = [1, 2, 3, 4, 5];

// List<T>
List<string> names = ["Alice", "Bob", "Charlie"];

// Span<T> (stack-allocated)
Span<int> span = [10, 20, 30];

// Spread operator (..) - inline another collection
int[] first  = [1, 2, 3];
int[] second = [4, 5, 6];
int[] merged = [..first, ..second];        // [1,2,3,4,5,6]
int[] combined = [0, ..first, ..second, 7]; // [0,1,2,3,4,5,6,7]

// กับ string collections
string[] admin = ["admin@example.com"];
string[] staff = ["staff1@example.com", "staff2@example.com"];
string[] all   = [..admin, ..staff, "extra@example.com"];

// Immutable collections
ImmutableArray<int> immutable = [1, 2, 3, 4, 5];

// HashSet
HashSet<string> tags = ["C#", "dotnet", "backend"];

// Method ที่รับ collection expressions
void ProcessItems(IEnumerable<int> items) { /* ... */ }
ProcessItems([1, 2, 3, 4, 5]); // ใช้ collection expression ตรงๆ ได้

// กับ pattern matching
bool HasItems(int[] arr) => arr is [_, ..]; // มีอย่างน้อย 1 element

// Practical: config defaults
string[] defaultScopes = ["read", "write"];
string[] adminScopes   = [..defaultScopes, "admin", "delete"];
string[] superScopes   = [..adminScopes, "superadmin"];

// Dictionary - C# 12 supports collection literals for Dictionary
// (using target-typing)
Dictionary<string, int> scores = new()
{
    ["Alice"] = 95,
    ["Bob"]   = 87
};
```

---

### Step 558: Interceptors (C# 12, Preview Feature)

```csharp
// Interceptors: ฟีเจอร์ preview ที่ให้ Source Generators
// intercept (แทนที่) calls ไปยัง method ณ compile time
// ใช้ใน ASP.NET Core minimal API ที่ generate code ล่วงหน้า

// เปิดใช้ใน .csproj:
// <InterceptorsPreviewNamespaces>$(InterceptorsPreviewNamespaces);MyApp.Generated</InterceptorsPreviewNamespaces>

// Original method ใน user code:
public static class Greeter
{
    public static string Greet(string name) => $"Hello, {name}!";
}

// User call:
// string result = Greeter.Greet("World");  // line 10, column 18 of Program.cs

// Interceptor ที่ generator สร้าง:
namespace MyApp.Generated
{
    file static class GreeterInterceptors
    {
        // InterceptsLocation ระบุ file + line + column ของ call ที่จะ intercept
        [System.Runtime.CompilerServices.InterceptsLocation(
            "Program.cs", line: 10, character: 18)]
        public static string Greet_Intercepted(string name)
        {
            // Pre-processing หรือ optimization
            return $"Hi, {name}! (intercepted)";
        }
    }
}

// ASP.NET Core ใช้ Interceptors สำหรับ:
// - Pre-compile minimal API route handlers
// - Avoid reflection at startup
// - Improve startup performance
```

---

### Step 559: Default Lambda Parameters (C# 12)

```csharp
// C# 12 อนุญาตให้ lambda มี default parameter values
// เหมือน regular methods

// Default parameters ใน lambda
var greet = (string name, string greeting = "Hello") 
    => $"{greeting}, {name}!";

Console.WriteLine(greet("Alice"));         // Hello, Alice!
Console.WriteLine(greet("Bob", "Hi"));     // Hi, Bob!

// กับ Func<>
Func<int, int, int> add = (x, y = 1) => x + y;
Console.WriteLine(add(5));    // 6  (5 + 1)
Console.WriteLine(add(5, 3)); // 8  (5 + 3)

// กับ callbacks ใน event handling
button.Click += (sender, e = EventArgs.Empty) => HandleClick(sender);

// กับ generic lambdas (C# 10+)
var identity = <T>(T value, bool verbose = false) =>
{
    if (verbose) Console.WriteLine($"Value: {value}");
    return value;
};

Console.WriteLine(identity(42));           // 42
Console.WriteLine(identity(42, true));     // Prints "Value: 42", returns 42

// Practical: configureable transformations
var transform = (string input, 
                 bool toUpper = false, 
                 bool trim = true,
                 string prefix = "") =>
{
    var result = trim ? input.Trim() : input;
    result     = toUpper ? result.ToUpper() : result;
    return prefix + result;
};

Console.WriteLine(transform("  hello  "));           // "hello"
Console.WriteLine(transform("  hello  ", toUpper: true));  // "HELLO"
Console.WriteLine(transform("world", prefix: ">> ")); // ">> world"
```

---

### Step 560: Ref Readonly Parameters (C# 12)

```csharp
// ref readonly parameters: ส่ง reference ไม่ให้แก้ไข
// ประหยัด memory เมื่อ pass large structs

public readonly struct Matrix4x4
{
    public readonly float M11, M12, M13, M14;
    public readonly float M21, M22, M23, M24;
    public readonly float M31, M32, M33, M34;
    public readonly float M41, M42, M43, M44;
    // ... 16 floats = 64 bytes
}

// Before C# 12: ต้องใช้ 'in' (ก็ทำงานเหมือนกัน แต่ semantics ต่างกัน)
public static Matrix4x4 Multiply(in Matrix4x4 a, in Matrix4x4 b)
{
    // a และ b ไม่สามารถแก้ไขได้ แต่ compiler อนุญาตให้ capture
    return default; // simplified
}

// C# 12: ref readonly ชัดเจนกว่า
// แสดงว่า caller อาจส่ง ref หรือ value
public static float Determinant(ref readonly Matrix4x4 matrix)
{
    // matrix เป็น read-only reference
    // ไม่มี copy, ไม่สามารถ modify ได้
    return matrix.M11 * (matrix.M22 * matrix.M33 - matrix.M23 * matrix.M32)
         - matrix.M12 * (matrix.M21 * matrix.M33 - matrix.M23 * matrix.M31)
         + matrix.M13 * (matrix.M21 * matrix.M32 - matrix.M22 * matrix.M31);
}

// ใช้งาน - caller เลือกได้ว่าจะส่งเป็น ref หรือ value
var m = new Matrix4x4 { /* ... */ };
float det1 = Determinant(ref m);  // ส่งเป็น ref (ไม่มี copy)
float det2 = Determinant(m);       // ส่งเป็น value (compiler สร้าง copy ให้)

// เปรียบเทียบ: in vs ref readonly
// in    : ไม่ต้องใช้ ref ณ call site (compiler implicit)
// ref readonly: ต้องการ ref explicit ณ call site เมื่อไม่ต้องการ copy

// Performance critical code
public static void BatchTransform(
    ReadOnlySpan<Matrix4x4> matrices,
    ref readonly Matrix4x4 transformMatrix,
    Span<Matrix4x4> results)
{
    for (int i = 0; i < matrices.Length; i++)
    {
        results[i] = Multiply(matrices[i], transformMatrix);
    }
}

// C# 13: ref/unsafe ใน async และ iterator methods (partial support)
// C# 13: params collections
public static int Sum(params IEnumerable<int> numbers)
    => numbers.Sum();

int total = Sum(1, 2, 3, 4, 5);        // 15
int total2 = Sum([1, 2, 3]);           // 15 (collection expression)
int total3 = Sum(new List<int> {1,2}); // 3
```

---

## สรุปภาพรวม: Advanced C# Features

### ตารางสรุปฟีเจอร์ทั้งหมด

| ฟีเจอร์ | Version | ประโยชน์หลัก | เมื่อใช้ |
|---------|---------|--------------|---------|
| `record class` | C# 9 | Value equality, immutable data | Domain models, DTOs |
| `record struct` | C# 10 | Value type + value equality | Small immutable structs |
| `with` expression | C# 9 | Non-destructive mutation | Functional updates |
| `init` properties | C# 9 | Initialization-only | Immutable after init |
| Property patterns | C# 8 | Match on object properties | Complex conditionals |
| List patterns | C# 11 | Match on collections | Sequence validation |
| Positional patterns | C# 8 | Match via Deconstruct | Tuple/record matching |
| `#nullable enable` | C# 8 | Null safety at compile time | All new projects |
| `INumber<T>` | .NET 7 | Generic numeric algorithms | Math libraries |
| Source Generators | C# 9 | Compile-time code generation | Boilerplate reduction |
| Primary constructors | C# 12 | Concise DI/constructor | Classes with dependencies |
| Collection expressions | C# 12 | Unified collection syntax | All collections |
| Default lambda params | C# 12 | Flexible callbacks | Event handlers, configs |
| `ref readonly` params | C# 12 | Efficient large struct passing | Performance-critical code |
| Interceptors | C# 12 (preview) | Compile-time call replacement | AOT, source gen optimization |

---

### Quick Reference: Patterns

```csharp
// รวม patterns ที่ใช้บ่อยที่สุด

object value = GetSomething();

string result = value switch
{
    // Type pattern
    int n                                   => $"int: {n}",
    
    // Const pattern
    "specific"                              => "exact string",
    
    // Property pattern
    Order { Status: OrderStatus.Active }    => "active order",
    
    // Property + nested
    Customer { Address: { Country: "TH" } } => "Thai customer",
    
    // Relational pattern
    int n when n is > 0 and < 100          => "small positive",
    
    // Positional pattern
    Point(0, 0)                             => "origin",
    
    // List pattern
    int[] [_, .., var last]                 => $"ends with {last}",
    
    // Null pattern
    null                                    => "null value",
    
    // Var pattern (always matches)
    var x                                   => $"other: {x}"
};
```

---

### Quick Reference: Records

```csharp
// Positional record
public record Point(double X, double Y);

// Nominal record  
public record Person
{
    public required string Name { get; init; }
    public int Age { get; init; }
}

// Record with computed property
public record Circle(double Radius)
{
    public double Area        => Math.PI * Radius * Radius;
    public double Circumference => 2 * Math.PI * Radius;
}

// Record inheritance
public abstract record Shape(string Color);
public record Square(string Color, double Side) : Shape(Color);

// with expression
var original = new Point(1, 2);
var modified = original with { X = 10 };  // Point(10, 2)

// Deconstruct
var (x, y) = new Point(3, 4);

// Value equality
new Point(1, 2) == new Point(1, 2)  // true
```

---

### Quick Reference: Nullable

```csharp
// ใน .csproj
// <Nullable>enable</Nullable>

string  required = "value";    // Non-nullable
string? optional = null;       // Nullable

// Safe access
int len = optional?.Length ?? 0;

// Null-forgiving (ใช้เมื่อมั่นใจ)
string certain = optional!;

// Pattern matching
if (optional is string s) { /* s is non-null */ }

// ThrowIfNull
ArgumentNullException.ThrowIfNull(optional);
// หลังจากนี้ optional ไม่ใช่ null

// Attributes
[return: MaybeNull]  // อาจคืน null
[NotNull]            // ไม่คืน null / parameter ไม่ null หลัง call
[NotNullWhen(true)]  // ไม่ null เมื่อ method return true
```

---

### Quick Reference: Generic Math

```csharp
using System.Numerics;

// Generic algorithm
static T Sum<T>(IEnumerable<T> items) where T : INumber<T>
{
    T result = T.Zero;
    foreach (var item in items) result += item;
    return result;
}

// ใช้งาน
Sum(new[] { 1, 2, 3 });         // int: 6
Sum(new[] { 1.5, 2.5 });       // double: 4.0
Sum(new[] { 1m, 2m, 3m });     // decimal: 6

// Constraints
where T : INumber<T>            // Full arithmetic
where T : IComparable<T>        // Comparison only
where T : IParsable<T>          // Can parse from string
where T : IAdditionOperators<T, T, T>  // Just addition
```

---

### แนวทาง Best Practices

1. **Records สำหรับ Immutable Data**
   - ใช้ record สำหรับ domain models, DTOs, value objects
   - ใช้ `with` แทนการแก้ไข properties โดยตรง
   - ใช้ record struct สำหรับ small value types (< 16 bytes)

2. **Pattern Matching แทน if-else chain**
   - switch expression อ่านง่ายกว่า nested if
   - Property patterns แสดงเจตนาชัดเจนกว่า `obj.Prop == value`
   - List patterns สำหรับ validate sequences

3. **Nullable Reference Types**
   - เปิดใช้ทุก project ใหม่
   - ใช้ `?` อย่างจงใจ ไม่ใช่เพื่อ suppress warnings
   - ใช้ null-forgiving `!` น้อยที่สุด

4. **Source Generators แทน Reflection**
   - Reflection ช้าและ error-prone
   - Source Generators ทำงาน compile-time ทำให้ type-safe และเร็ว
   - ใช้สำหรับ mapping, serialization, INotifyPropertyChanged

5. **C# 12 Features**
   - Primary constructors: ลด boilerplate ใน service classes
   - Collection expressions: ใช้ทุกที่แทน `new List<>()`, `new []`
   - `ref readonly`: ใช้เมื่อ pass large structs บ่อย

---

## 📚 การนำทาง

| | |
|--|--|
| ◀ Part ก่อนหน้า | [part55-async-advanced.md](part55-async-advanced.md) |
| ▶ Part ถัดไป    | [part57-blazor-intro.md](part57-blazor-intro.md) |

---

*Part 56 ครอบคลุม Advanced C# Language Features: Records & Immutability, Pattern Matching ขั้นสูง, Nullable Reference Types, Generic Math, Source Generators และ C# 12/13 Features*
