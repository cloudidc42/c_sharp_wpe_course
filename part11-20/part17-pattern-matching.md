# Part 17: Pattern Matching
## ขั้นตอนที่ 161-170: Pattern Matching C# 7-12

---

## 🎯 เป้าหมายของ Part นี้
- Pattern matching ตั้งแต่ C# 7 ถึง C# 12
- Type patterns, Property patterns, Positional patterns
- List patterns (C# 11)
- Switch expressions
- Guard clauses
- สร้างโปรแกรมที่ใช้ patterns อย่างมืออาชีพ

---

## ขั้นตอนที่ 161: Type Patterns

### is และ switch type patterns
```csharp
// C# 7: Type pattern
object obj = "Hello World";

// is with type pattern
if (obj is string str)
{
    Console.WriteLine($"String of length: {str.Length}");
}

// Switch with type pattern
static string Describe(object obj) => obj switch
{
    int n => $"Integer: {n}",
    double d => $"Double: {d:F2}",
    string s => $"String: '{s}'",
    bool b => $"Boolean: {b}",
    null => "Null",
    _ => $"Unknown: {obj.GetType().Name}"
};

Console.WriteLine(Describe(42));
Console.WriteLine(Describe(3.14));
Console.WriteLine(Describe("Hello"));
Console.WriteLine(Describe(true));
Console.WriteLine(Describe(null!));

// Null check pattern
string? nullable = null;
if (nullable is not null)
    Console.WriteLine(nullable.Length);

// var pattern (always matches)
if (obj is var anything)
    Console.WriteLine($"Captured: {anything}");
```

### Constant Patterns
```csharp
// Constant patterns
int score = 85;

string grade = score switch
{
    100 => "A+",
    >= 90 => "A",
    >= 80 => "B",
    >= 70 => "C",
    >= 60 => "D",
    _ => "F"
};

Console.WriteLine($"Grade: {grade}");

// With negative ranges
static string ClassifyTemperature(double celsius) => celsius switch
{
    < 0 => "น้ำแข็ง",
    0 => "จุดเยือกแข็ง",
    > 0 and < 15 => "หนาวมาก",
    >= 15 and < 20 => "หนาว",
    >= 20 and < 25 => "เย็นสบาย",
    >= 25 and < 30 => "อบอุ่น",
    >= 30 and < 35 => "ร้อน",
    >= 35 => "ร้อนมาก",
};

for (double temp = -5; temp <= 40; temp += 5)
    Console.WriteLine($"{temp,5}°C: {ClassifyTemperature(temp)}");
```

---

## ขั้นตอนที่ 162: Property Patterns

### Property Pattern
```csharp
record Address(string City, string Country, string PostalCode);
record Person(string Name, int Age, Address? Address);

// Property pattern
static string GetShippingZone(Person person) => person switch
{
    { Address: null } => "Unknown",
    { Address.Country: "Thailand" } => "Domestic",
    { Address.Country: "US" or "Canada" } => "North America",
    { Address.Country: "UK" or "France" or "Germany" } => "Europe",
    _ => "International"
};

var people = new[]
{
    new Person("Alice", 30, new Address("Bangkok", "Thailand", "10100")),
    new Person("Bob", 45, new Address("New York", "US", "10001")),
    new Person("Charlie", 28, null),
    new Person("Diana", 35, new Address("London", "UK", "SW1A"))
};

foreach (var p in people)
    Console.WriteLine($"{p.Name}: {GetShippingZone(p)}");

// Complex property patterns
static decimal CalculateDiscount(Person customer, decimal orderAmount) 
    => (customer, orderAmount) switch
    {
        // VIP เมื่ออายุ > 65
        ({ Age: > 65 }, > 1000m) => 0.20m,
        ({ Age: > 65 }, _) => 0.10m,
        
        // Bangkok ได้ discount พิเศษ
        ({ Address.City: "Bangkok" }, > 5000m) => 0.15m,
        ({ Address.City: "Bangkok" }, _) => 0.05m,
        
        // ยอดซื้อสูง
        (_, > 10000m) => 0.12m,
        (_, > 5000m) => 0.08m,
        
        // Default
        _ => 0.0m
    };
```

---

## ขั้นตอนที่ 163: Positional Patterns

### Deconstruct กับ Patterns
```csharp
// Positional pattern ใช้กับ record หรือ class ที่มี Deconstruct
record Point(double X, double Y);
record Rectangle(Point TopLeft, Point BottomRight);

static string DescribePoint(Point p) => p switch
{
    (0, 0) => "Origin",
    (0, _) => "On Y-axis",
    (_, 0) => "On X-axis",
    (> 0, > 0) => "First quadrant",
    (< 0, > 0) => "Second quadrant",
    (< 0, < 0) => "Third quadrant",
    (> 0, < 0) => "Fourth quadrant",
    _ => "Unknown"
};

var points = new[]
{
    new Point(0, 0),
    new Point(3, 4),
    new Point(-2, 5),
    new Point(1, 0)
};

foreach (var p in points)
    Console.WriteLine($"({p.X}, {p.Y}): {DescribePoint(p)}");

// Tuple patterns
static string ClassifyTriangle(double a, double b, double c) => (a, b, c) switch
{
    _ when a + b <= c || a + c <= b || b + c <= a => "Not a triangle",
    var (x, y, z) when x == y && y == z => "Equilateral",
    var (x, y, z) when x == y || y == z || x == z => "Isosceles",
    _ => "Scalene"
};

Console.WriteLine(ClassifyTriangle(3, 3, 3));   // Equilateral
Console.WriteLine(ClassifyTriangle(3, 3, 4));   // Isosceles
Console.WriteLine(ClassifyTriangle(3, 4, 5));   // Scalene
```

---

## ขั้นตอนที่ 164: List Patterns (C# 11)

### Pattern matching กับ Arrays และ Lists
```csharp
// List patterns (C# 11)
int[] numbers = { 1, 2, 3, 4, 5 };

// Match specific elements
bool matchesPattern = numbers is [1, 2, 3, 4, 5];
Console.WriteLine(matchesPattern);  // True

// Match with wildcards
bool startsWithOne = numbers is [1, ..];
bool endsWithFive = numbers is [.., 5];
bool hasOneAndFive = numbers is [1, .., 5];

Console.WriteLine(startsWithOne);  // True
Console.WriteLine(endsWithFive);   // True
Console.WriteLine(hasOneAndFive);  // True

// Extract elements
if (numbers is [int first, int second, ..])
    Console.WriteLine($"First two: {first}, {second}");

if (numbers is [.., int last])
    Console.WriteLine($"Last: {last}");

// Function with list pattern
static string DescribeList(int[] list) => list switch
{
    [] => "Empty list",
    [var single] => $"Single element: {single}",
    [var head, var tail] => $"Two elements: {head} and {tail}",
    [1, 2, 3] => "Exactly [1, 2, 3]",
    [var h, .. var middle, var t] => $"Starts with {h}, {middle.Length} in middle, ends with {t}",
};

Console.WriteLine(DescribeList(Array.Empty<int>()));
Console.WriteLine(DescribeList(new[] { 42 }));
Console.WriteLine(DescribeList(new[] { 1, 2 }));
Console.WriteLine(DescribeList(new[] { 1, 2, 3 }));
Console.WriteLine(DescribeList(new[] { 1, 2, 3, 4, 5 }));
```

---

## ขั้นตอนที่ 165: Advanced Switch Expressions

### Switch with Guards
```csharp
// Guard clauses (when)
record HttpRequest(string Method, string Path, Dictionary<string, string>? Headers = null);
record HttpResponse(int StatusCode, string Body);

static HttpResponse HandleRequest(HttpRequest req) => req switch
{
    { Method: "GET", Path: "/" } => new HttpResponse(200, "<html>Home</html>"),
    
    { Method: "GET", Path: var p } when p.StartsWith("/api/") 
        => HandleApiRequest(p),
    
    { Method: "POST", Path: "/api/users" } when req.Headers?.ContainsKey("Authorization") == true 
        => new HttpResponse(201, "User created"),
    
    { Method: "POST", Path: "/api/users" } 
        => new HttpResponse(401, "Unauthorized"),
    
    { Method: "DELETE" } 
        => new HttpResponse(405, "Method not allowed"),
    
    _ => new HttpResponse(404, "Not found")
};

static HttpResponse HandleApiRequest(string path) 
    => new HttpResponse(200, $"API response for {path}");

var requests = new[]
{
    new HttpRequest("GET", "/"),
    new HttpRequest("GET", "/api/products"),
    new HttpRequest("POST", "/api/users", new Dictionary<string, string> { { "Authorization", "Bearer token" } }),
    new HttpRequest("POST", "/api/users"),
    new HttpRequest("DELETE", "/api/items/1")
};

foreach (var req in requests)
{
    var response = HandleRequest(req);
    Console.WriteLine($"{req.Method} {req.Path} → {response.StatusCode}: {response.Body}");
}
```

### Recursive Patterns
```csharp
// Pattern matching กับ tree structures
abstract class Expression { }
record NumberExpr(double Value) : Expression;
record BinaryExpr(Expression Left, string Operator, Expression Right) : Expression;
record UnaryExpr(string Operator, Expression Operand) : Expression;

static double Evaluate(Expression expr) => expr switch
{
    NumberExpr(var n) => n,
    
    UnaryExpr("-", var operand) => -Evaluate(operand),
    UnaryExpr("+", var operand) => Evaluate(operand),
    
    BinaryExpr(var left, "+", var right) => Evaluate(left) + Evaluate(right),
    BinaryExpr(var left, "-", var right) => Evaluate(left) - Evaluate(right),
    BinaryExpr(var left, "*", var right) => Evaluate(left) * Evaluate(right),
    BinaryExpr(var left, "/", var right) => Evaluate(left) / Evaluate(right),
    
    _ => throw new ArgumentException($"Unknown expression: {expr}")
};

// (3 + 4) * 2 - 1
var expr1 = new BinaryExpr(
    new BinaryExpr(
        new BinaryExpr(new NumberExpr(3), "+", new NumberExpr(4)),
        "*",
        new NumberExpr(2)
    ),
    "-",
    new NumberExpr(1)
);

Console.WriteLine(Evaluate(expr1));  // 13
```

---

## ขั้นตอนที่ 166-170: โปรแกรมตัวอย่าง - Rule Engine

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace RuleEngine
{
    // Domain models
    record Customer(
        string Id,
        string Name,
        string Tier,          // "Bronze", "Silver", "Gold", "Platinum"
        int PurchaseCount,
        decimal TotalSpent,
        DateTime JoinDate,
        string Country
    );
    
    record Order(
        string Id,
        string CustomerId,
        decimal Amount,
        string[] Items,
        DateTime Date,
        string? DiscountCode = null
    );
    
    record DiscountResult(
        decimal DiscountPercent,
        string Reason,
        bool FreeShipping
    );
    
    // Rule Engine
    class DiscountEngine
    {
        // Main entry point
        public DiscountResult CalculateDiscount(Customer customer, Order order)
        {
            // Apply rules in priority order
            var baseDiscount = GetTierDiscount(customer.Tier);
            var loyaltyBonus = GetLoyaltyBonus(customer);
            var orderBonus = GetOrderSizeBonus(order.Amount);
            var specialBonus = GetSpecialDiscount(customer, order);
            
            decimal totalDiscount = Math.Min(baseDiscount + loyaltyBonus + orderBonus + specialBonus, 0.40m);
            bool freeShipping = ShouldGetFreeShipping(customer, order, totalDiscount);
            
            string reason = BuildReason(customer, order, baseDiscount, loyaltyBonus, orderBonus, specialBonus);
            
            return new DiscountResult(totalDiscount, reason, freeShipping);
        }
        
        private decimal GetTierDiscount(string tier) => tier switch
        {
            "Platinum" => 0.15m,
            "Gold" => 0.10m,
            "Silver" => 0.05m,
            "Bronze" or _ => 0.02m
        };
        
        private decimal GetLoyaltyBonus(Customer customer) => customer switch
        {
            { PurchaseCount: >= 100 } => 0.08m,
            { PurchaseCount: >= 50 } => 0.05m,
            { PurchaseCount: >= 20 } => 0.03m,
            { PurchaseCount: >= 10 } => 0.01m,
            _ => 0m
        };
        
        private decimal GetOrderSizeBonus(decimal amount) => amount switch
        {
            >= 100000m => 0.10m,
            >= 50000m => 0.07m,
            >= 20000m => 0.05m,
            >= 10000m => 0.03m,
            _ => 0m
        };
        
        private decimal GetSpecialDiscount(Customer customer, Order order)
        {
            decimal discount = 0m;
            
            // Birthday month bonus
            if (customer.JoinDate.Month == order.Date.Month)
                discount += 0.03m;
            
            // Discount code
            if (order.DiscountCode is not null)
            {
                discount += order.DiscountCode switch
                {
                    "SAVE10" => 0.10m,
                    "VIPONLY" when customer.Tier is "Gold" or "Platinum" => 0.15m,
                    "NEWUSER" when customer.PurchaseCount <= 3 => 0.20m,
                    _ => 0m
                };
            }
            
            return discount;
        }
        
        private bool ShouldGetFreeShipping(Customer customer, Order order, decimal discount) =>
            (customer, order, discount) switch
            {
                // Platinum always free shipping
                ({ Tier: "Platinum" }, _, _) => true,
                
                // High discount implies free shipping
                (_, _, >= 0.20m) => true,
                
                // Large order
                (_, { Amount: >= 50000m }, _) => true,
                
                // Gold + big order
                ({ Tier: "Gold" }, { Amount: >= 20000m }, _) => true,
                
                _ => false
            };
        
        private string BuildReason(Customer c, Order o, decimal tier, decimal loyalty, decimal order, decimal special)
        {
            var reasons = new List<string>();
            
            if (tier > 0) reasons.Add($"{c.Tier} tier: {tier:P0}");
            if (loyalty > 0) reasons.Add($"Loyalty ({c.PurchaseCount} purchases): {loyalty:P0}");
            if (order > 0) reasons.Add($"Order size: {order:P0}");
            if (special > 0) reasons.Add($"Special: {special:P0}");
            
            return reasons.Any() ? string.Join(" + ", reasons) : "Standard pricing";
        }
    }
    
    // Pricing Engine
    class PricingEngine
    {
        private readonly DiscountEngine _discountEngine = new();
        
        public record PriceQuote(
            decimal OriginalAmount,
            decimal DiscountAmount,
            decimal FinalAmount,
            decimal ShippingCost,
            decimal TotalAmount,
            string DiscountDetails,
            bool FreeShipping
        );
        
        public PriceQuote Quote(Customer customer, Order order)
        {
            var discount = _discountEngine.CalculateDiscount(customer, order);
            
            decimal discountAmount = order.Amount * discount.DiscountPercent;
            decimal finalAmount = order.Amount - discountAmount;
            
            decimal shippingCost = CalculateShipping(customer, order, discount.FreeShipping);
            
            return new PriceQuote(
                order.Amount,
                discountAmount,
                finalAmount,
                shippingCost,
                finalAmount + shippingCost,
                discount.Reason,
                discount.FreeShipping
            );
        }
        
        private decimal CalculateShipping(Customer customer, Order order, bool freeShipping)
        {
            if (freeShipping) return 0m;
            
            return (customer.Country, order.Amount) switch
            {
                ("Thailand", >= 1000m) => 0m,
                ("Thailand", _) => 50m,
                ("US" or "Canada", _) => 500m,
                (_, _) => 800m
            };
        }
    }
    
    class Program
    {
        static void Main()
        {
            Console.OutputEncoding = System.Text.Encoding.UTF8;
            
            var engine = new PricingEngine();
            
            var customers = new[]
            {
                new Customer("C001", "Alice (Platinum)", "Platinum", 150, 500000m, 
                    new DateTime(2020, DateTime.Now.Month, 1), "Thailand"),
                new Customer("C002", "Bob (Gold)", "Gold", 75, 200000m, 
                    new DateTime(2021, 6, 15), "Thailand"),
                new Customer("C003", "Charlie (Silver)", "Silver", 25, 80000m, 
                    new DateTime(2022, 3, 20), "US"),
                new Customer("C004", "Diana (Bronze)", "Bronze", 5, 15000m, 
                    new DateTime(2025, 1, 10), "Thailand"),
            };
            
            var orders = new[]
            {
                new Order("O001", "C001", 35000m, new[] { "Laptop", "Mouse" }, DateTime.Now),
                new Order("O002", "C002", 55000m, new[] { "TV", "Speaker" }, DateTime.Now, "VIPONLY"),
                new Order("O003", "C003", 8000m, new[] { "Keyboard" }, DateTime.Now, "SAVE10"),
                new Order("O004", "C004", 2500m, new[] { "Cable" }, DateTime.Now, "NEWUSER"),
            };
            
            Console.WriteLine("=".PadRight(70, '='));
            Console.WriteLine($"{"PRICE QUOTES":^70}");
            Console.WriteLine("=".PadRight(70, '='));
            
            foreach (var (customer, order) in customers.Zip(orders))
            {
                var quote = engine.Quote(customer, order);
                
                Console.WriteLine($"\n👤 {customer.Name}");
                Console.WriteLine($"  Order #{order.Id}: {order.Items.Length} items");
                Console.WriteLine($"  Original:  {quote.OriginalAmount,12:N2} บาท");
                Console.WriteLine($"  Discount:  -{quote.DiscountAmount,11:N2} บาท  ({quote.DiscountAmount / quote.OriginalAmount:P1})");
                Console.WriteLine($"  After:     {quote.FinalAmount,12:N2} บาท");
                Console.WriteLine($"  Shipping:  {(quote.FreeShipping ? "FREE" : $"{quote.ShippingCost:N2} บาท"),12}");
                Console.WriteLine($"  ─────────────────────────");
                Console.WriteLine($"  TOTAL:     {quote.TotalAmount,12:N2} บาท");
                Console.WriteLine($"  Reason: {quote.DiscountDetails}");
            }
            
            Console.WriteLine("\n" + "=".PadRight(70, '='));
            Console.WriteLine("\nกด Enter เพื่อออก...");
            Console.ReadLine();
        }
    }
}
```

---

## 📝 สรุป Part 17

| Pattern | Example | C# Version |
|---------|---------|-----------|
| Type pattern | `obj is string s` | C# 7 |
| Constant pattern | `x is 42` | C# 7 |
| Property pattern | `{ Name: "Alice" }` | C# 8 |
| Positional pattern | `(0, 0)` | C# 8 |
| Switch expression | `x switch { ... }` | C# 8 |
| Relational pattern | `>= 90` | C# 9 |
| Logical patterns | `and`, `or`, `not` | C# 9 |
| List pattern | `[1, 2, ..]` | C# 11 |
| Slice pattern | `[.., var last]` | C# 11 |

---

**ก่อนหน้า → [Part 16: Async/Await](part16-async-await.md)**  
**ต่อไป → [Part 18: Tuples & Records](part18-tuples-records.md)**
