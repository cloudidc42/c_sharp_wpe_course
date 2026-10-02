# Part 18: Tuples & Records
## ขั้นตอนที่ 171-180: Modern Data Types

---

## 🎯 เป้าหมายของ Part นี้
- Tuples: ValueTuple และ named tuples
- Records: immutable data types
- Records กับ with expressions
- Record structs (C# 10)
- Primary constructors (C# 12)
- ออกแบบ Data Models อย่างมืออาชีพ

---

## ขั้นตอนที่ 171: Tuples

### ValueTuple - ส่งหลายค่ากลับ
```csharp
// Tuple พื้นฐาน
var point = (3, 4);
Console.WriteLine($"X: {point.Item1}, Y: {point.Item2}");

// Named tuple
var namedPoint = (X: 3, Y: 4);
Console.WriteLine($"X: {namedPoint.X}, Y: {namedPoint.Y}");

// Deconstruction
var (x, y) = namedPoint;
Console.WriteLine($"x={x}, y={y}");

// Tuple return values
static (double distance, double angle) PolarCoordinates(double x, double y)
{
    double distance = Math.Sqrt(x * x + y * y);
    double angle = Math.Atan2(y, x) * 180 / Math.PI;
    return (distance, angle);
}

var (dist, angle) = PolarCoordinates(3, 4);
Console.WriteLine($"Distance: {dist:F2}, Angle: {angle:F2}°");

// Tuple as dictionary key
var occupancy = new Dictionary<(int Row, int Col), string>();
occupancy[(1, 1)] = "Alice";
occupancy[(1, 2)] = "Bob";
occupancy[(2, 1)] = "Charlie";

foreach (var ((row, col), name) in occupancy)
    Console.WriteLine($"[{row},{col}]: {name}");

// Tuple swap
int a = 5, b = 10;
(a, b) = (b, a);  // Swap!
Console.WriteLine($"a={a}, b={b}");  // a=10, b=5

// Tuple equality
var t1 = (1, "hello", true);
var t2 = (1, "hello", true);
Console.WriteLine(t1 == t2);  // True (structural equality)
```

---

## ขั้นตอนที่ 172: Records

### Record types
```csharp
// Record = immutable reference type ที่ value equality
record Person(string Name, int Age);

var alice = new Person("Alice", 30);
var alice2 = new Person("Alice", 30);
var bob = new Person("Bob", 25);

// Value equality (ไม่ใช่ reference equality)
Console.WriteLine(alice == alice2);  // True
Console.WriteLine(alice == bob);     // False
Console.WriteLine(alice.Equals(alice2));  // True

// ToString (auto-generated)
Console.WriteLine(alice);  // Person { Name = Alice, Age = 30 }

// Deconstruction (auto-generated)
var (name, age) = alice;
Console.WriteLine($"{name} is {age} years old");

// With expression - non-destructive mutation
var alice31 = alice with { Age = 31 };
Console.WriteLine(alice);    // Person { Name = Alice, Age = 30 } (unchanged)
Console.WriteLine(alice31);  // Person { Name = Alice, Age = 31 }

// Records are immutable
// alice.Name = "Bob";  // Error! init-only

// Records with additional members
record Product(string Name, decimal Price, string Category)
{
    // Additional computed property
    public decimal PriceWithVat => Price * 1.07m;
    
    // Custom ToString
    public override string ToString() => $"{Name} ({Category}): {Price:N2} บาท";
    
    // Validate in constructor
    public Product : this(Name, Price, Category)
    {
        if (Price < 0) throw new ArgumentException("Price cannot be negative");
        if (string.IsNullOrEmpty(Name)) throw new ArgumentException("Name required");
    }
}

var laptop = new Product("Laptop Pro", 45000m, "Electronics");
Console.WriteLine(laptop);
Console.WriteLine($"With VAT: {laptop.PriceWithVat:N2}");
```

---

## ขั้นตอนที่ 173: Record Inheritance

### Inheriting Records
```csharp
// Record inheritance
record Vehicle(string Make, string Model, int Year);
record Car(string Make, string Model, int Year, int Doors) : Vehicle(Make, Model, Year);
record ElectricCar(string Make, string Model, int Year, int Doors, int RangeKm) 
    : Car(Make, Model, Year, Doors);

var tesla = new ElectricCar("Tesla", "Model 3", 2024, 4, 600);
Console.WriteLine(tesla);
// ElectricCar { Make = Tesla, Model = Model 3, Year = 2024, Doors = 4, RangeKm = 600 }

// with expression works with inheritance
var longRange = tesla with { RangeKm = 800 };
Console.WriteLine(longRange);

// Type checking
Vehicle v = tesla;
if (v is ElectricCar ev)
    Console.WriteLine($"Electric range: {ev.RangeKm}km");

// Abstract record
abstract record Shape
{
    public abstract double Area { get; }
    public abstract double Perimeter { get; }
}

record Circle(double Radius) : Shape
{
    public override double Area => Math.PI * Radius * Radius;
    public override double Perimeter => 2 * Math.PI * Radius;
}

record Rectangle(double Width, double Height) : Shape
{
    public override double Area => Width * Height;
    public override double Perimeter => 2 * (Width + Height);
}

Shape[] shapes = { new Circle(5), new Rectangle(4, 6) };
foreach (var shape in shapes)
    Console.WriteLine($"{shape}: Area={shape.Area:F2}, Perimeter={shape.Perimeter:F2}");
```

---

## ขั้นตอนที่ 174: Record Structs

### Record struct (C# 10)
```csharp
// Record struct = value type + value equality
record struct Point2D(double X, double Y)
{
    public double DistanceTo(Point2D other)
    {
        double dx = X - other.X;
        double dy = Y - other.Y;
        return Math.Sqrt(dx * dx + dy * dy);
    }
    
    public static Point2D Origin => new(0, 0);
}

var p1 = new Point2D(3, 4);
var p2 = new Point2D(0, 0);

Console.WriteLine(p1);                        // Point2D { X = 3, Y = 4 }
Console.WriteLine(p1.DistanceTo(p2));         // 5
Console.WriteLine(p1 == new Point2D(3, 4));  // True

// Record struct is mutable by default (unlike class records)
var mutablePoint = new Point2D(1, 2);
mutablePoint.X = 10;  // OK!

// readonly record struct - fully immutable
readonly record struct ImmutablePoint(double X, double Y);

var ip = new ImmutablePoint(3, 4);
// ip.X = 10;  // Error! Read-only

// Comparison
var p3 = new Point2D(3, 4);
Console.WriteLine(p1 == p3);   // True (value equality)
Console.WriteLine(p1.Equals(p3));  // True
```

---

## ขั้นตอนที่ 175: Primary Constructors (C# 12)

### Primary Constructors
```csharp
// C# 12 Primary Constructors ใน regular classes
class Database(string connectionString, int timeout = 30)
{
    // Parameters are in scope throughout the class
    public string ConnectionString { get; } = connectionString;
    
    public void Query(string sql)
    {
        Console.WriteLine($"Query [{timeout}s timeout]: {sql}");
    }
    
    // Primary constructor parameter used in initialization
    private readonly string _logPrefix = $"[DB:{connectionString[..Math.Min(20, connectionString.Length)]}]";
    
    public void Log(string message)
        => Console.WriteLine($"{_logPrefix} {message}");
}

class UserRepository(Database database)
{
    public void GetUser(int id)
    {
        database.Query($"SELECT * FROM users WHERE id = {id}");
    }
}

// ใช้งาน
var db = new Database("Server=localhost;DB=myapp", timeout: 60);
db.Query("SELECT * FROM users");
db.Log("Connection established");

var userRepo = new UserRepository(db);
userRepo.GetUser(1);

// Struct with primary constructor
struct Color(byte r, byte g, byte b)
{
    public byte R { get; } = r;
    public byte G { get; } = g;
    public byte B { get; } = b;
    
    public string ToHex() => $"#{R:X2}{G:X2}{B:X2}";
    public override string ToString() => $"rgb({R}, {G}, {B})";
}

var red = new Color(255, 0, 0);
Console.WriteLine(red.ToHex());  // #FF0000
Console.WriteLine(red);          // rgb(255, 0, 0)
```

---

## ขั้นตอนที่ 176-180: โปรแกรมตัวอย่าง - Immutable Domain Models

```csharp
using System;
using System.Collections.Generic;
using System.Collections.Immutable;
using System.Linq;

namespace ImmutableDomain
{
    // Immutable value objects
    record Money(decimal Amount, string Currency = "THB")
    {
        public Money Add(Money other)
        {
            if (Currency != other.Currency)
                throw new InvalidOperationException($"Cannot add {Currency} and {other.Currency}");
            return this with { Amount = Amount + other.Amount };
        }
        
        public Money Subtract(Money other)
        {
            if (Currency != other.Currency)
                throw new InvalidOperationException("Currency mismatch");
            return this with { Amount = Amount - other.Amount };
        }
        
        public Money Multiply(decimal factor) => this with { Amount = Amount * factor };
        
        public static Money operator +(Money a, Money b) => a.Add(b);
        public static Money operator -(Money a, Money b) => a.Subtract(b);
        public static Money operator *(Money m, decimal f) => m.Multiply(f);
        
        public bool IsNegative => Amount < 0;
        public bool IsZero => Amount == 0;
        
        public override string ToString() => $"{Amount:N2} {Currency}";
    }
    
    record Address(
        string Street, 
        string City, 
        string Province, 
        string PostalCode,
        string Country = "Thailand"
    )
    {
        public string FullAddress => $"{Street}, {City}, {Province} {PostalCode}, {Country}";
    }
    
    // Entity with immutable updates
    record Customer
    {
        public Guid Id { get; init; } = Guid.NewGuid();
        public string Name { get; init; } = "";
        public string Email { get; init; } = "";
        public Address? ShippingAddress { get; init; }
        public ImmutableList<Order> Orders { get; init; } = ImmutableList<Order>.Empty;
        public DateTime CreatedAt { get; init; } = DateTime.UtcNow;
        public CustomerTier Tier => CalculateTier();
        
        private CustomerTier CalculateTier()
        {
            decimal total = Orders.Sum(o => o.Total.Amount);
            return total switch
            {
                >= 500000m => CustomerTier.Platinum,
                >= 100000m => CustomerTier.Gold,
                >= 30000m => CustomerTier.Silver,
                _ => CustomerTier.Bronze
            };
        }
        
        public Customer AddOrder(Order order)
            => this with { Orders = Orders.Add(order) };
        
        public Customer UpdateAddress(Address address)
            => this with { ShippingAddress = address };
        
        public Customer UpdateEmail(string email)
        {
            if (!email.Contains('@'))
                throw new ArgumentException("Invalid email");
            return this with { Email = email };
        }
        
        public Money TotalSpent => Orders.Aggregate(
            new Money(0), (acc, o) => acc + o.Total);
    }
    
    enum CustomerTier { Bronze, Silver, Gold, Platinum }
    
    record OrderItem(
        string ProductId,
        string ProductName,
        int Quantity,
        Money UnitPrice
    )
    {
        public Money Subtotal => UnitPrice * Quantity;
        public override string ToString() => $"{ProductName} x{Quantity} @ {UnitPrice} = {Subtotal}";
    }
    
    record Order
    {
        public Guid Id { get; init; } = Guid.NewGuid();
        public string CustomerId { get; init; } = "";
        public ImmutableList<OrderItem> Items { get; init; } = ImmutableList<OrderItem>.Empty;
        public DateTime OrderDate { get; init; } = DateTime.UtcNow;
        public OrderStatus Status { get; init; } = OrderStatus.Pending;
        public Money? Discount { get; init; }
        
        public Money Subtotal => Items.Aggregate(new Money(0), (acc, item) => acc + item.Subtotal);
        public Money Total => Discount is null ? Subtotal : Subtotal - Discount;
        
        public Order AddItem(OrderItem item) => this with { Items = Items.Add(item) };
        public Order RemoveItem(string productId) 
            => this with { Items = Items.RemoveAll(i => i.ProductId == productId) };
        
        public Order ApplyDiscount(Money discount) => this with { Discount = discount };
        public Order UpdateStatus(OrderStatus status) => this with { Status = status };
        
        public void PrintReceipt()
        {
            Console.WriteLine($"\n{'=',40}");
            Console.WriteLine($"{"ORDER RECEIPT":^40}");
            Console.WriteLine($"{'=',40}");
            Console.WriteLine($"Order ID: {Id:N}"[..18]);
            Console.WriteLine($"Date: {OrderDate:dd/MM/yyyy HH:mm}");
            Console.WriteLine($"Status: {Status}");
            Console.WriteLine(new string('-', 40));
            
            foreach (var item in Items)
                Console.WriteLine($"  {item.ProductName,-20} {item.Subtotal,10}");
            
            Console.WriteLine(new string('-', 40));
            Console.WriteLine($"  {"Subtotal",-20} {Subtotal,10}");
            
            if (Discount is not null)
                Console.WriteLine($"  {"Discount",-20} -{Discount,9}");
            
            Console.WriteLine($"  {"TOTAL",-20} {Total,10}");
            Console.WriteLine(new string('=', 40));
        }
    }
    
    enum OrderStatus { Pending, Confirmed, Processing, Shipped, Delivered, Cancelled }
    
    // Repository (immutable store)
    class CustomerStore
    {
        private ImmutableDictionary<Guid, Customer> _customers 
            = ImmutableDictionary<Guid, Customer>.Empty;
        
        public Customer Create(string name, string email)
        {
            var customer = new Customer { Name = name, Email = email };
            _customers = _customers.Add(customer.Id, customer);
            return customer;
        }
        
        public Customer? Find(Guid id) => _customers.GetValueOrDefault(id);
        
        public Customer Update(Customer customer)
        {
            _customers = _customers.SetItem(customer.Id, customer);
            return customer;
        }
        
        public IEnumerable<Customer> GetAll() => _customers.Values;
        
        public IEnumerable<Customer> GetByTier(CustomerTier tier)
            => _customers.Values.Where(c => c.Tier == tier);
    }
    
    class Program
    {
        static void Main()
        {
            Console.OutputEncoding = System.Text.Encoding.UTF8;
            Console.Title = "Immutable Domain Models";
            
            var store = new CustomerStore();
            
            // Create customers
            var alice = store.Create("Alice Smith", "alice@example.com");
            var bob = store.Create("Bob Jones", "bob@example.com");
            
            // Alice places an order
            var order1 = new Order { CustomerId = alice.Id.ToString() }
                .AddItem(new OrderItem("P001", "Laptop Pro", 1, new Money(45000m)))
                .AddItem(new OrderItem("P002", "Wireless Mouse", 2, new Money(1500m)))
                .AddItem(new OrderItem("P003", "USB Hub", 1, new Money(800m)))
                .ApplyDiscount(new Money(2000m))
                .UpdateStatus(OrderStatus.Confirmed);
            
            order1.PrintReceipt();
            
            // Update customer with order
            alice = alice
                .AddOrder(order1)
                .UpdateAddress(new Address("123 Main St", "Bangkok", "Bangkok", "10100"));
            
            store.Update(alice);
            
            // Bob places multiple orders
            var order2 = new Order { CustomerId = bob.Id.ToString() }
                .AddItem(new OrderItem("P004", "4K Monitor", 2, new Money(18000m)))
                .UpdateStatus(OrderStatus.Shipped);
            
            var order3 = new Order { CustomerId = bob.Id.ToString() }
                .AddItem(new OrderItem("P005", "Mechanical Keyboard", 1, new Money(4500m)))
                .UpdateStatus(OrderStatus.Delivered);
            
            bob = bob.AddOrder(order2).AddOrder(order3);
            store.Update(bob);
            
            // Display customer summary
            Console.WriteLine("\n=== Customer Summary ===");
            foreach (var customer in store.GetAll())
            {
                Console.WriteLine($"\n{customer.Name} ({customer.Email})");
                Console.WriteLine($"  Tier: {customer.Tier} 🏆");
                Console.WriteLine($"  Total Spent: {customer.TotalSpent}");
                Console.WriteLine($"  Orders: {customer.Orders.Count}");
                if (customer.ShippingAddress is not null)
                    Console.WriteLine($"  Address: {customer.ShippingAddress.FullAddress}");
            }
            
            // Immutability demonstration
            Console.WriteLine("\n=== Immutability Demo ===");
            var originalAlice = store.Find(alice.Id)!;
            var modifiedAlice = originalAlice.UpdateEmail("newalice@example.com");
            
            Console.WriteLine($"Original email: {originalAlice.Email}");
            Console.WriteLine($"Modified email: {modifiedAlice.Email}");
            Console.WriteLine($"Same object? {ReferenceEquals(originalAlice, modifiedAlice)}");  // False!
            
            // Money operations
            Console.WriteLine("\n=== Money Operations ===");
            var price1 = new Money(100m);
            var price2 = new Money(50m);
            var total = price1 + price2;
            var discounted = total * 0.9m;
            
            Console.WriteLine($"{price1} + {price2} = {total}");
            Console.WriteLine($"{total} * 0.9 = {discounted}");
            
            Console.WriteLine("\nกด Enter เพื่อออก...");
            Console.ReadLine();
        }
    }
}
```

---

## 📝 สรุป Part 18

| หัวข้อ | Key Points |
|--------|-----------|
| ValueTuple | `(int x, int y)` - value types |
| Named tuple | `(X: 3, Y: 4)` |
| Deconstruction | `var (x, y) = tuple;` |
| record | Immutable reference type + value equality |
| with expression | `obj with { Prop = newValue }` |
| record struct | Value type record |
| readonly record struct | Fully immutable value type |
| Primary constructors | `class Foo(int x, int y)` |

---

**ก่อนหน้า → [Part 17: Pattern Matching](part17-pattern-matching.md)**  
**ต่อไป → [Part 19: Extension Methods](part19-extension-methods.md)**
