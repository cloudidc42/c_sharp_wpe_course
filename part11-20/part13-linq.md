# Part 13: LINQ
## ขั้นตอนที่ 121-130: Language Integrated Query

---

## 🎯 เป้าหมายของ Part นี้
- เข้าใจ LINQ และ deferred execution
- ใช้ Query syntax และ Method syntax
- รู้จัก LINQ operators ทั้งหมด
- LINQ กับ Collections, Files, XML
- Performance และ best practices
- สร้าง Query Builder ระดับมืออาชีพ

---

## ขั้นตอนที่ 121: LINQ พื้นฐาน

### Query Syntax vs Method Syntax
```csharp
var numbers = new[] { 5, 3, 8, 1, 9, 2, 7, 4, 6 };

// Query Syntax (SQL-like)
var queryResult = from n in numbers
                  where n > 4
                  orderby n
                  select n;

// Method Syntax (Lambda)
var methodResult = numbers
    .Where(n => n > 4)
    .OrderBy(n => n);

// ผลลัพธ์เหมือนกัน
Console.WriteLine(string.Join(", ", queryResult));
// 5, 6, 7, 8, 9

// Deferred Execution
var query = numbers.Where(n => n > 4);
// ยังไม่ execute!

// Execute เมื่อ iterate
foreach (var n in query) Console.Write($"{n} ");

// หรือ force execute
var list = query.ToList();
var array = query.ToArray();
```

### Data Models
```csharp
record Customer(int Id, string Name, string City, int Age, decimal TotalPurchases);
record Order(int Id, int CustomerId, DateTime Date, decimal Amount, string Product);
record Product(int Id, string Name, string Category, decimal Price, int Stock);

// Sample data
var customers = new List<Customer>
{
    new(1, "Alice Smith", "Bangkok", 30, 150000m),
    new(2, "Bob Jones", "Chiang Mai", 45, 85000m),
    new(3, "Charlie Brown", "Bangkok", 28, 220000m),
    new(4, "Diana Prince", "Phuket", 35, 95000m),
    new(5, "Eve Wilson", "Bangkok", 52, 310000m),
    new(6, "Frank Miller", "Chiang Mai", 38, 45000m)
};

var orders = new List<Order>
{
    new(1, 1, new DateTime(2026, 1, 15), 15000m, "Laptop"),
    new(2, 1, new DateTime(2026, 2, 3), 5000m, "Mouse"),
    new(3, 2, new DateTime(2026, 1, 20), 35000m, "TV"),
    new(4, 3, new DateTime(2026, 2, 10), 80000m, "MacBook"),
    new(5, 3, new DateTime(2026, 3, 5), 25000m, "iPad"),
    new(6, 4, new DateTime(2026, 1, 8), 12000m, "Phone"),
    new(7, 5, new DateTime(2026, 2, 18), 45000m, "Camera"),
    new(8, 5, new DateTime(2026, 3, 25), 8000m, "Lens"),
    new(9, 6, new DateTime(2026, 1, 30), 3000m, "Keyboard")
};
```

---

## ขั้นตอนที่ 122: Filtering และ Projection

### Where, Select, SelectMany
```csharp
// Where - กรอง
var bangkokCustomers = customers.Where(c => c.City == "Bangkok");

// Where with multiple conditions
var premiumBangkok = customers
    .Where(c => c.City == "Bangkok" && c.TotalPurchases > 100000m);

// Select - projection (transform)
var names = customers.Select(c => c.Name);
var nameAge = customers.Select(c => new { c.Name, c.Age });
var nameCityStr = customers.Select(c => $"{c.Name} ({c.City})");

// Select with index
var numbered = customers.Select((c, index) => $"{index + 1}. {c.Name}");
foreach (var item in numbered)
    Console.WriteLine(item);

// SelectMany - flatten collections
var customerOrders = customers
    .SelectMany(c => 
        orders.Where(o => o.CustomerId == c.Id),
        (customer, order) => new { customer.Name, order.Product, order.Amount }
    );

// Example with nested lists
var data = new List<List<int>> { new() { 1, 2, 3 }, new() { 4, 5 }, new() { 6, 7, 8, 9 } };
var flat = data.SelectMany(list => list);
Console.WriteLine(string.Join(", ", flat));  // 1, 2, 3, 4, 5, 6, 7, 8, 9
```

---

## ขั้นตอนที่ 123: Sorting และ Grouping

### OrderBy, GroupBy
```csharp
// OrderBy / OrderByDescending
var byAge = customers.OrderBy(c => c.Age);
var byPurchaseDesc = customers.OrderByDescending(c => c.TotalPurchases);

// ThenBy - secondary sort
var sorted = customers
    .OrderBy(c => c.City)
    .ThenByDescending(c => c.TotalPurchases);

Console.WriteLine("Sorted by City, then Purchase (desc):");
foreach (var c in sorted)
    Console.WriteLine($"  {c.City,-15} {c.Name,-20} {c.TotalPurchases:N0}");

// GroupBy
var byCity = customers.GroupBy(c => c.City);

Console.WriteLine("\nGrouped by City:");
foreach (var group in byCity.OrderBy(g => g.Key))
{
    Console.WriteLine($"\n{group.Key} ({group.Count()} customers):");
    foreach (var c in group.OrderByDescending(c => c.TotalPurchases))
        Console.WriteLine($"  {c.Name}: {c.TotalPurchases:N0} บาท");
}

// GroupBy with aggregate
var cityStats = customers
    .GroupBy(c => c.City)
    .Select(g => new
    {
        City = g.Key,
        Count = g.Count(),
        TotalPurchases = g.Sum(c => c.TotalPurchases),
        AvgAge = g.Average(c => c.Age),
        MaxPurchase = g.Max(c => c.TotalPurchases)
    })
    .OrderByDescending(x => x.TotalPurchases);

Console.WriteLine("\nCity Statistics:");
Console.WriteLine($"{"City",15} {"Count",6} {"Total",15} {"AvgAge",8}");
foreach (var stat in cityStats)
    Console.WriteLine($"{stat.City,15} {stat.Count,6} {stat.TotalPurchases,15:N0} {stat.AvgAge,8:F1}");
```

---

## ขั้นตอนที่ 124: Join Operations

### Join, GroupJoin, Left Join
```csharp
// Inner Join
var customerOrders2 = customers
    .Join(
        orders,
        customer => customer.Id,         // Outer key
        order => order.CustomerId,        // Inner key
        (customer, order) => new          // Result selector
        {
            CustomerName = customer.Name,
            order.Product,
            order.Amount,
            order.Date
        }
    );

Console.WriteLine("Customer Orders (Inner Join):");
foreach (var co in customerOrders2.OrderBy(x => x.CustomerName))
    Console.WriteLine($"  {co.CustomerName,-20} {co.Product,-15} {co.Amount:N0}");

// GroupJoin (Left Join equivalent)
var customersWithOrders = customers
    .GroupJoin(
        orders,
        customer => customer.Id,
        order => order.CustomerId,
        (customer, customerOrders) => new
        {
            customer.Name,
            OrderCount = customerOrders.Count(),
            TotalSpent = customerOrders.Sum(o => o.Amount),
            Orders = customerOrders.ToList()
        }
    );

Console.WriteLine("\nAll Customers with Order Summary:");
foreach (var c in customersWithOrders)
    Console.WriteLine($"  {c.Name,-20} {c.OrderCount} orders, Total: {c.TotalSpent:N0}");

// Left Join (include customers with no orders)
var leftJoin = customers
    .GroupJoin(orders, c => c.Id, o => o.CustomerId,
        (customer, customerOrders) => new { customer, customerOrders })
    .SelectMany(
        x => x.customerOrders.DefaultIfEmpty(),
        (x, order) => new
        {
            x.customer.Name,
            Product = order?.Product ?? "No orders",
            Amount = order?.Amount ?? 0m
        }
    );
```

---

## ขั้นตอนที่ 125: Aggregation และ Quantifiers

### Sum, Count, Min, Max, Average, Any, All
```csharp
// Aggregations
decimal totalRevenue = orders.Sum(o => o.Amount);
decimal avgOrderAmount = orders.Average(o => o.Amount);
decimal maxOrder = orders.Max(o => o.Amount);
decimal minOrder = orders.Min(o => o.Amount);
int orderCount = orders.Count();
int bangkokOrderCount = orders.Count(o => customers
    .Any(c => c.Id == o.CustomerId && c.City == "Bangkok"));

Console.WriteLine($"Total Revenue: {totalRevenue:N0}");
Console.WriteLine($"Avg Order: {avgOrderAmount:N0}");
Console.WriteLine($"Max Order: {maxOrder:N0}");
Console.WriteLine($"Min Order: {minOrder:N0}");

// Quantifiers
bool anyHighValue = orders.Any(o => o.Amount > 50000m);
bool allPositive = orders.All(o => o.Amount > 0);
bool noneNegative = !orders.Any(o => o.Amount < 0);

Console.WriteLine($"Has high value order: {anyHighValue}");
Console.WriteLine($"All positive: {allPositive}");

// Aggregate (custom aggregation)
decimal runningTotal = orders
    .OrderBy(o => o.Date)
    .Aggregate(0m, (acc, order) => acc + order.Amount);

string allProductNames = orders
    .Select(o => o.Product)
    .Aggregate((acc, name) => acc + ", " + name);

// Distinct
var uniqueProducts = orders.Select(o => o.Product).Distinct().OrderBy(p => p);
Console.WriteLine($"\nProducts sold: {string.Join(", ", uniqueProducts)}");

// DistinctBy (C# 6+)
var uniqueCustomerOrders = orders.DistinctBy(o => o.CustomerId);
```

---

## ขั้นตอนที่ 126: Set Operations และ Partitioning

### Union, Intersect, Except, Take, Skip
```csharp
var set1 = new[] { 1, 2, 3, 4, 5 };
var set2 = new[] { 3, 4, 5, 6, 7 };

// Set operations
var union = set1.Union(set2);              // 1,2,3,4,5,6,7
var intersect = set1.Intersect(set2);     // 3,4,5
var except = set1.Except(set2);           // 1,2

Console.WriteLine($"Union: {string.Join(",", union)}");
Console.WriteLine($"Intersect: {string.Join(",", intersect)}");
Console.WriteLine($"Except: {string.Join(",", except)}");

// Partitioning
var topOrders = orders.OrderByDescending(o => o.Amount).Take(3);
var remainingOrders = orders.OrderByDescending(o => o.Amount).Skip(3);

// Pagination
int pageSize = 2;
int pageNumber = 1;
var page = orders
    .OrderBy(o => o.Date)
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize);

// TakeWhile / SkipWhile
var numbers = new[] { 1, 3, 5, 7, 2, 4, 6, 8 };
var odds = numbers.TakeWhile(n => n % 2 != 0);    // 1, 3, 5, 7
var afterOdds = numbers.SkipWhile(n => n % 2 != 0); // 2, 4, 6, 8

// Chunk (C# 6+)
var chunks = numbers.Chunk(3);
foreach (var chunk in chunks)
    Console.WriteLine($"Chunk: [{string.Join(", ", chunk)}]");
```

---

## ขั้นตอนที่ 127-130: โปรแกรมตัวอย่าง - Sales Analytics Dashboard

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;

namespace SalesAnalytics
{
    record Customer(int Id, string Name, string City, string Segment, int JoinYear);
    record Product(int Id, string Name, string Category, decimal UnitPrice);
    record SaleRecord(int Id, int CustomerId, int ProductId, DateTime Date, int Quantity, decimal Discount);
    
    class SalesDashboard
    {
        private readonly List<Customer> _customers;
        private readonly List<Product> _products;
        private readonly List<SaleRecord> _sales;
        
        public SalesDashboard(List<Customer> customers, List<Product> products, List<SaleRecord> sales)
        {
            _customers = customers;
            _products = products;
            _sales = sales;
        }
        
        // Total revenue
        public decimal TotalRevenue(int? year = null)
        {
            var query = _sales.AsEnumerable();
            if (year.HasValue) query = query.Where(s => s.Date.Year == year);
            
            return query.Join(_products, s => s.ProductId, p => p.Id,
                (s, p) => p.UnitPrice * s.Quantity * (1 - s.Discount))
                .Sum();
        }
        
        // Monthly revenue
        public IEnumerable<(string Month, decimal Revenue)> MonthlyRevenue(int year)
        {
            return _sales
                .Where(s => s.Date.Year == year)
                .Join(_products, s => s.ProductId, p => p.Id,
                    (s, p) => new { s.Date, Revenue = p.UnitPrice * s.Quantity * (1 - s.Discount) })
                .GroupBy(x => x.Date.Month)
                .Select(g => (
                    Month: new DateTime(year, g.Key, 1).ToString("MMMM"),
                    Revenue: g.Sum(x => x.Revenue)
                ))
                .OrderBy(x => x.Month);
        }
        
        // Top customers by revenue
        public IEnumerable<(Customer Customer, decimal Revenue, int OrderCount)> TopCustomers(int topN = 10)
        {
            return _customers
                .GroupJoin(_sales, c => c.Id, s => s.CustomerId,
                    (customer, sales) => new
                    {
                        Customer = customer,
                        Sales = sales.ToList()
                    })
                .Where(x => x.Sales.Any())
                .Select(x => (
                    x.Customer,
                    Revenue: x.Sales.Join(_products, s => s.ProductId, p => p.Id,
                        (s, p) => p.UnitPrice * s.Quantity * (1 - s.Discount)).Sum(),
                    OrderCount: x.Sales.Count
                ))
                .OrderByDescending(x => x.Revenue)
                .Take(topN);
        }
        
        // Category performance
        public IEnumerable<(string Category, decimal Revenue, int UnitsSold, decimal AvgDiscount)> CategoryPerformance()
        {
            return _sales
                .Join(_products, s => s.ProductId, p => p.Id,
                    (s, p) => new { s, p })
                .GroupBy(x => x.p.Category)
                .Select(g => (
                    Category: g.Key,
                    Revenue: g.Sum(x => x.p.UnitPrice * x.s.Quantity * (1 - x.s.Discount)),
                    UnitsSold: g.Sum(x => x.s.Quantity),
                    AvgDiscount: g.Average(x => x.s.Discount)
                ))
                .OrderByDescending(x => x.Revenue);
        }
        
        // Customer segment analysis
        public IEnumerable<(string Segment, int Count, decimal TotalRevenue, decimal AvgRevenue)> SegmentAnalysis()
        {
            var customerRevenue = _customers
                .GroupJoin(_sales, c => c.Id, s => s.CustomerId,
                    (customer, sales) => new
                    {
                        customer.Segment,
                        Revenue = sales.Join(_products, s => s.ProductId, p => p.Id,
                            (s, p) => p.UnitPrice * s.Quantity * (1 - s.Discount)).Sum()
                    });
            
            return customerRevenue
                .GroupBy(x => x.Segment)
                .Select(g => (
                    Segment: g.Key,
                    Count: g.Count(),
                    TotalRevenue: g.Sum(x => x.Revenue),
                    AvgRevenue: g.Average(x => x.Revenue)
                ))
                .OrderByDescending(x => x.TotalRevenue);
        }
        
        // Product sales ranking
        public IEnumerable<(int Rank, string ProductName, string Category, int Sold, decimal Revenue)> ProductRanking()
        {
            return _sales
                .Join(_products, s => s.ProductId, p => p.Id,
                    (s, p) => new { s.Quantity, Revenue = p.UnitPrice * s.Quantity * (1 - s.Discount), p.Name, p.Category })
                .GroupBy(x => new { x.Name, x.Category })
                .Select(g => new
                {
                    ProductName = g.Key.Name,
                    Category = g.Key.Category,
                    Sold = g.Sum(x => x.Quantity),
                    Revenue = g.Sum(x => x.Revenue)
                })
                .OrderByDescending(x => x.Revenue)
                .Select((x, i) => (
                    Rank: i + 1,
                    x.ProductName,
                    x.Category,
                    x.Sold,
                    x.Revenue
                ));
        }
        
        // Display dashboard
        public void DisplayDashboard(int year)
        {
            Console.OutputEncoding = Encoding.UTF8;
            
            Console.WriteLine(new string('═', 60));
            Console.WriteLine($"           SALES ANALYTICS DASHBOARD - {year}");
            Console.WriteLine(new string('═', 60));
            
            // Summary
            decimal totalRev = TotalRevenue(year);
            int totalOrders = _sales.Count(s => s.Date.Year == year);
            decimal avgOrder = totalOrders > 0 ? _sales
                .Where(s => s.Date.Year == year)
                .Join(_products, s => s.ProductId, p => p.Id,
                    (s, p) => p.UnitPrice * s.Quantity * (1 - s.Discount))
                .Average() : 0;
            
            Console.WriteLine($"\n📊 Summary");
            Console.WriteLine($"  Total Revenue: {totalRev:N0} บาท");
            Console.WriteLine($"  Total Orders:  {totalOrders:N0}");
            Console.WriteLine($"  Average Order: {avgOrder:N0} บาท");
            
            // Category Performance
            Console.WriteLine($"\n📦 Category Performance");
            Console.WriteLine($"{"Category",20} {"Revenue",15} {"Units",8} {"Avg.Disc",10}");
            Console.WriteLine(new string('─', 55));
            foreach (var (cat, rev, units, disc) in CategoryPerformance())
                Console.WriteLine($"{cat,20} {rev,15:N0} {units,8} {disc,10:P1}");
            
            // Top Customers
            Console.WriteLine($"\n👥 Top 5 Customers");
            Console.WriteLine($"{"Customer",25} {"Revenue",15} {"Orders",8}");
            Console.WriteLine(new string('─', 50));
            foreach (var (customer, rev, count) in TopCustomers(5))
                Console.WriteLine($"{customer.Name,25} {rev,15:N0} {count,8}");
            
            // Product Ranking
            Console.WriteLine($"\n🏆 Top 5 Products");
            Console.WriteLine($"{"#",4} {"Product",20} {"Category",12} {"Sold",6} {"Revenue",12}");
            Console.WriteLine(new string('─', 58));
            foreach (var (rank, name, cat, sold, rev) in ProductRanking().Take(5))
                Console.WriteLine($"{rank,4} {name,20} {cat,12} {sold,6} {rev,12:N0}");
            
            Console.WriteLine(new string('═', 60));
        }
    }
    
    class Program
    {
        static void Main()
        {
            // Sample data
            var customers = new List<Customer>
            {
                new(1, "Alice Corp", "Bangkok", "Enterprise", 2020),
                new(2, "Bob Ltd", "Chiang Mai", "SME", 2021),
                new(3, "Charlie Inc", "Bangkok", "Enterprise", 2019),
                new(4, "Diana Co", "Phuket", "SME", 2022),
                new(5, "Eve Trading", "Bangkok", "Enterprise", 2018),
                new(6, "Frank Shop", "Chiang Mai", "Retail", 2023),
                new(7, "Grace Store", "Bangkok", "Retail", 2022),
            };
            
            var products = new List<Product>
            {
                new(1, "Laptop Pro", "Electronics", 45000m),
                new(2, "Wireless Mouse", "Accessories", 1500m),
                new(3, "4K Monitor", "Electronics", 25000m),
                new(4, "Mechanical Keyboard", "Accessories", 4500m),
                new(5, "Office Chair", "Furniture", 15000m),
                new(6, "Standing Desk", "Furniture", 30000m),
                new(7, "Webcam HD", "Electronics", 3500m),
            };
            
            var random = new Random(42);
            var sales = new List<SaleRecord>();
            int saleId = 1;
            
            for (int month = 1; month <= 12; month++)
            {
                int salesCount = random.Next(15, 30);
                for (int i = 0; i < salesCount; i++)
                {
                    sales.Add(new SaleRecord(
                        saleId++,
                        random.Next(1, 8),
                        random.Next(1, 8),
                        new DateTime(2026, month, random.Next(1, 28)),
                        random.Next(1, 6),
                        random.NextDouble() < 0.3 ? Math.Round(random.NextDouble() * 0.2, 2) : 0
                    ));
                }
            }
            
            var dashboard = new SalesDashboard(customers, products, sales);
            dashboard.DisplayDashboard(2026);
            
            Console.WriteLine("\nกด Enter เพื่อออก...");
            Console.ReadLine();
        }
    }
}
```

---

## 📝 สรุป Part 13

| Operator | ใช้สำหรับ |
|----------|----------|
| Where | กรองข้อมูล |
| Select | แปลงรูปแบบ |
| SelectMany | Flatten nested collections |
| OrderBy/ThenBy | จัดเรียง |
| GroupBy | จัดกลุ่ม |
| Join/GroupJoin | รวม 2 collections |
| Any/All/Contains | ตรวจสอบ condition |
| Sum/Count/Min/Max/Average | คำนวณ aggregate |
| Take/Skip | แบ่งหน้า |
| Union/Intersect/Except | Set operations |
| Distinct/DistinctBy | ลบ duplicate |

---

**ก่อนหน้า → [Part 12: File I/O](part12-file-io.md)**  
**ต่อไป → [Part 14: Generics](part14-generics.md)**
