# Part 44: LINQ Advanced
## ขั้นตอนที่ 431-440: LINQ ขั้นสูงและ Functional Programming

---

## 🎯 เป้าหมายของ Part นี้
- LINQ method syntax vs query syntax
- Deferred execution
- IQueryable vs IEnumerable
- Custom LINQ extensions
- Functional patterns: Map, Filter, Reduce
- Parallel LINQ (PLINQ)
- LINQ to Objects: complex scenarios
- โปรแกรม Data Analysis Pipeline

---

## ขั้นตอนที่ 431: Deferred Execution

```csharp
// LINQ queries are lazy - executed when enumerated
var numbers = new List<int> { 1, 2, 3, 4, 5 };

// Query defined but NOT executed yet
var query = numbers.Where(n => n % 2 == 0).Select(n => n * 2);
Console.WriteLine("Query defined - no execution yet");

// Execution happens here (ToList, foreach, Count, etc.)
var result = query.ToList(); // [4, 8]

// Changing the source after query definition
numbers.Add(6);
var result2 = query.ToList(); // [4, 8, 12] - includes 6!

// Force immediate execution
var snapshot = numbers.Where(n => n > 3).ToArray(); // captures current state
```

---

## ขั้นตอนที่ 432: IQueryable vs IEnumerable

```csharp
// IEnumerable: executes in memory (LINQ to Objects)
// IQueryable: translates to SQL (LINQ to EF Core)

// BAD: loads all posts into memory, then filters
IEnumerable<Post> allPosts = context.Posts.ToList(); // SELECT * FROM Posts
var filtered = allPosts.Where(p => p.IsPublished); // filters in C#

// GOOD: filter in database
IQueryable<Post> query = context.Posts.Where(p => p.IsPublished); // builds SQL
var result = await query.ToListAsync(); // SELECT * WHERE IsPublished=1

// LINQ chain with IQueryable - each .Where() builds onto the SQL
var postsQuery = context.Posts
    .Where(p => p.IsPublished)          // WHERE IsPublished = 1
    .Where(p => p.CategoryId == 2)      // AND CategoryId = 2
    .OrderByDescending(p => p.CreatedAt) // ORDER BY CreatedAt DESC
    .Take(10);                           // LIMIT 10

// Dynamic query building
IQueryable<Post> dynamicQuery = context.Posts;

if (!string.IsNullOrEmpty(searchTerm))
    dynamicQuery = dynamicQuery.Where(p => p.Title.Contains(searchTerm));

if (categoryId.HasValue)
    dynamicQuery = dynamicQuery.Where(p => p.CategoryId == categoryId);

if (publishedOnly)
    dynamicQuery = dynamicQuery.Where(p => p.IsPublished);

var posts = await dynamicQuery.ToListAsync(); // single SQL query
```

---

## ขั้นตอนที่ 433: Complex LINQ Operations

```csharp
// GroupBy with aggregation
var salesByMonth = orders.GroupBy(o => new { o.OrderDate.Year, o.OrderDate.Month })
    .Select(g => new
    {
        Year = g.Key.Year,
        Month = g.Key.Month,
        Count = g.Count(),
        Revenue = g.Sum(o => o.TotalAmount),
        AvgOrder = g.Average(o => (double)o.TotalAmount)
    })
    .OrderBy(x => x.Year).ThenBy(x => x.Month)
    .ToList();

// Join (SQL-style)
var orderDetails = from o in orders
                   join c in customers on o.CustomerId equals c.Id
                   join p in products on o.ProductId equals p.Id
                   where o.Status == "Completed"
                   select new 
                   {
                       CustomerName = c.Name,
                       ProductName = p.Name,
                       Amount = o.TotalAmount,
                       Date = o.OrderDate
                   };

// GroupJoin (LEFT JOIN)
var customersWithOrders = customers.GroupJoin(
    orders,
    c => c.Id,
    o => o.CustomerId,
    (customer, orderGroup) => new
    {
        Customer = customer,
        OrderCount = orderGroup.Count(),
        TotalSpent = orderGroup.Sum(o => o.TotalAmount)
    });

// SelectMany (flatten nested collections)
var allItems = orders.SelectMany(o => o.Items); // IEnumerable<OrderItem>
var productIds = orders.SelectMany(o => o.Items.Select(i => i.ProductId)).Distinct();

// Zip (combine two sequences)
var names = new[] { "Alice", "Bob", "Charlie" };
var scores = new[] { 95, 87, 92 };
var combined = names.Zip(scores, (name, score) => new { Name = name, Score = score });

// Aggregate (fold/reduce)
var product = new[] { 1, 2, 3, 4, 5 }.Aggregate((acc, n) => acc * n); // 120
var csv = new[] { "a", "b", "c" }.Aggregate((acc, s) => $"{acc},{s}"); // "a,b,c"
var sumWithSeed = new[] { 1, 2, 3 }.Aggregate(100, (acc, n) => acc + n); // 106

// Chunk (EF 8 / .NET 8)
var batches = items.Chunk(100); // splits into batches of 100
foreach (var batch in batches)
    await ProcessBatchAsync(batch);
```

---

## ขั้นตอนที่ 434: Custom LINQ Extensions

```csharp
// Extension methods สำหรับ IEnumerable
public static class LinqExtensions
{
    // Batch processing
    public static IEnumerable<IEnumerable<T>> Batch<T>(this IEnumerable<T> source, int size)
    {
        using var e = source.GetEnumerator();
        while (e.MoveNext())
        {
            var batch = new List<T> { e.Current };
            for (int i = 1; i < size && e.MoveNext(); i++)
                batch.Add(e.Current);
            yield return batch;
        }
    }
    
    // Distinct by property
    public static IEnumerable<T> DistinctBy<T, TKey>(
        this IEnumerable<T> source, Func<T, TKey> keySelector)
    {
        var seen = new HashSet<TKey>();
        foreach (var item in source)
            if (seen.Add(keySelector(item)))
                yield return item;
    }
    
    // ForEach (like List<T>.ForEach but for any IEnumerable)
    public static void ForEach<T>(this IEnumerable<T> source, Action<T> action)
    {
        foreach (var item in source) action(item);
    }
    
    // Paginate
    public static (IEnumerable<T> Items, int TotalPages) Paginate<T>(
        this IEnumerable<T> source, int page, int pageSize)
    {
        var list = source.ToList();
        var items = list.Skip((page - 1) * pageSize).Take(pageSize);
        var totalPages = (int)Math.Ceiling(list.Count / (double)pageSize);
        return (items, totalPages);
    }
    
    // Try-catch in LINQ
    public static IEnumerable<T> WhereNotNull<T>(this IEnumerable<T?> source) where T : class
        => source.Where(x => x != null).Select(x => x!);
    
    // Moving average
    public static IEnumerable<double> MovingAverage(this IEnumerable<double> source, int window)
    {
        var queue = new Queue<double>(window);
        foreach (var value in source)
        {
            queue.Enqueue(value);
            if (queue.Count > window) queue.Dequeue();
            if (queue.Count == window) yield return queue.Average();
        }
    }
    
    // Min/Max with selector that returns the item (not just the value)
    public static T MinBy<T, TKey>(this IEnumerable<T> source, Func<T, TKey> key) where TKey : IComparable<TKey>
    {
        return source.Aggregate((min, x) => key(x).CompareTo(key(min)) < 0 ? x : min);
    }
    
    // Flatten nested hierarchy
    public static IEnumerable<T> Flatten<T>(this IEnumerable<T> source, Func<T, IEnumerable<T>> children)
    {
        foreach (var item in source)
        {
            yield return item;
            foreach (var child in children(item).Flatten(children))
                yield return child;
        }
    }
}
```

---

## ขั้นตอนที่ 435: Parallel LINQ (PLINQ)

```csharp
// PLINQ: parallel data processing
var numbers = Enumerable.Range(1, 10_000_000).ToArray();

// Sequential
var sq = numbers.Where(n => n % 2 == 0).Select(n => n * n).Sum(); // slow

// Parallel
var pq = numbers.AsParallel()
    .Where(n => n % 2 == 0)
    .Select(n => n * n)
    .Sum(); // faster on multi-core

// PLINQ options
var result = numbers.AsParallel()
    .WithDegreeOfParallelism(4)      // max 4 threads
    .WithExecutionMode(ParallelExecutionMode.ForceParallelism)
    .Where(n => n % 3 == 0)
    .ToList();

// Order preserving (WithMergeOptions.NotBuffered = streaming)
var ordered = numbers.AsParallel()
    .AsOrdered()  // preserves original order (slower)
    .Where(n => n > 500_000)
    .Take(100)
    .ToList();

// When to use PLINQ:
// ✓ CPU-intensive operations (image processing, encryption)
// ✓ Large collections (10k+)
// ✗ I/O bound (use async/await instead)
// ✗ Small collections (overhead > benefit)
// ✗ When order matters and not specified

// Parallel processing with side effects (use ConcurrentBag)
var bag = new ConcurrentBag<ProcessedItem>();
Parallel.ForEach(items, item =>
{
    var processed = HeavyProcess(item);
    bag.Add(processed);
});
var results = bag.ToList();
```

---

## ขั้นตอนที่ 436-440: Data Analysis Pipeline

```csharp
// Data Analysis Pipeline - สถิติยอดขาย
public class SalesRecord
{
    public DateTime Date { get; set; }
    public string Product { get; set; } = "";
    public string Category { get; set; } = "";
    public string Region { get; set; } = "";
    public decimal Amount { get; set; }
    public int Quantity { get; set; }
}

public class SalesAnalyzer
{
    private readonly IEnumerable<SalesRecord> _data;
    
    public SalesAnalyzer(IEnumerable<SalesRecord> data) => _data = data;
    
    // Top N products
    public IEnumerable<(string Product, decimal Revenue)> GetTopProducts(int n = 10)
        => _data
            .GroupBy(s => s.Product)
            .Select(g => (Product: g.Key, Revenue: g.Sum(s => s.Amount)))
            .OrderByDescending(x => x.Revenue)
            .Take(n);
    
    // Monthly trend
    public IEnumerable<(int Year, int Month, decimal Revenue, int Orders)> GetMonthlyTrend()
        => _data
            .GroupBy(s => new { s.Date.Year, s.Date.Month })
            .Select(g => (g.Key.Year, g.Key.Month, 
                         Revenue: g.Sum(s => s.Amount),
                         Orders: g.Count()))
            .OrderBy(x => x.Year).ThenBy(x => x.Month);
    
    // Category breakdown
    public Dictionary<string, decimal> GetCategoryBreakdown()
        => _data
            .GroupBy(s => s.Category)
            .ToDictionary(g => g.Key, g => g.Sum(s => s.Amount));
    
    // YoY growth
    public decimal GetYearOverYearGrowth(int year)
    {
        var thisYear = _data.Where(s => s.Date.Year == year).Sum(s => s.Amount);
        var lastYear = _data.Where(s => s.Date.Year == year - 1).Sum(s => s.Amount);
        return lastYear > 0 ? (thisYear - lastYear) / lastYear * 100 : 0;
    }
    
    // Cohort analysis: first purchase month
    public IEnumerable<object> GetCohortRetention(IEnumerable<(string CustomerId, DateTime FirstPurchase)> cohorts)
    {
        return from cohort in cohorts
               group cohort by new { cohort.FirstPurchase.Year, cohort.FirstPurchase.Month } into g
               select new
               {
                   CohortYear = g.Key.Year,
                   CohortMonth = g.Key.Month,
                   Size = g.Count()
               };
    }
    
    // Correlation: price vs quantity (Pearson)
    public double GetPriceQuantityCorrelation()
    {
        var data = _data.Select(s => (Price: (double)s.Amount / s.Quantity, Qty: (double)s.Quantity)).ToList();
        
        double n = data.Count;
        double sumX = data.Sum(d => d.Price);
        double sumY = data.Sum(d => d.Qty);
        double sumXY = data.Sum(d => d.Price * d.Qty);
        double sumX2 = data.Sum(d => d.Price * d.Price);
        double sumY2 = data.Sum(d => d.Qty * d.Qty);
        
        double num = n * sumXY - sumX * sumY;
        double den = Math.Sqrt((n * sumX2 - sumX * sumX) * (n * sumY2 - sumY * sumY));
        return den == 0 ? 0 : num / den;
    }
}

// Main analysis program
var records = GenerateSampleData();
var analyzer = new SalesAnalyzer(records);

Console.WriteLine("=== รายงานการวิเคราะห์ยอดขาย ===\n");

Console.WriteLine("🏆 สินค้าขายดี Top 5:");
foreach (var (product, revenue) in analyzer.GetTopProducts(5))
    Console.WriteLine($"  {product}: ฿{revenue:N0}");

Console.WriteLine("\n📊 แนวโน้มรายเดือน (ปี 2024):");
foreach (var (year, month, revenue, orders) in analyzer.GetMonthlyTrend().Where(m => m.Year == 2024))
    Console.WriteLine($"  {year}/{month:D2}: ฿{revenue:N0} ({orders} คำสั่ง)");

Console.WriteLine("\n📈 การเติบโต YoY 2024 vs 2023:");
Console.WriteLine($"  {analyzer.GetYearOverYearGrowth(2024):+0.##;-0.##}%");

Console.WriteLine("\n🗂️ สัดส่วนตามหมวดหมู่:");
foreach (var (cat, rev) in analyzer.GetCategoryBreakdown().OrderByDescending(x => x.Value))
    Console.WriteLine($"  {cat}: ฿{rev:N0}");

// Using custom LINQ extensions
var monthlyRevenues = analyzer.GetMonthlyTrend().Select(m => (double)m.Revenue);
var movingAvg = monthlyRevenues.MovingAverage(3).ToList();
Console.WriteLine($"\n📉 Moving Average (3 เดือน): {string.Join(", ", movingAvg.Select(v => $"฿{v:N0}"))}");

static List<SalesRecord> GenerateSampleData()
{
    var rng = new Random(42);
    var products = new[] { "iPhone", "MacBook", "iPad", "AirPods", "Watch" };
    var categories = new[] { "Phone", "Laptop", "Tablet", "Accessories", "Wearable" };
    var regions = new[] { "กรุงเทพ", "เชียงใหม่", "ภูเก็ต", "ขอนแก่น" };
    
    return Enumerable.Range(0, 2000).Select(i =>
    {
        int idx = rng.Next(5);
        return new SalesRecord
        {
            Date = DateTime.Now.AddDays(-rng.Next(730)),
            Product = products[idx],
            Category = categories[idx],
            Region = regions[rng.Next(4)],
            Quantity = rng.Next(1, 5),
            Amount = (decimal)(rng.Next(5000, 80000) + rng.NextDouble() * 100)
        };
    }).ToList();
}
```

---

## 📝 สรุป Part 44

| Concept | สิ่งสำคัญ |
|---------|---------|
| Deferred Execution | Query ทำงานตอน enumerate |
| IQueryable | Translates to SQL |
| IEnumerable | Runs in memory |
| GroupBy | Group and aggregate |
| SelectMany | Flatten nested collections |
| Aggregate | Fold/reduce operations |
| PLINQ | Parallel processing |
| Custom Extensions | Reusable LINQ helpers |

---

**ก่อนหน้า → [Part 43: Unit of Work](part43-ef-unit-of-work.md)**  
**ต่อไป → [Part 45: EF Core + WPF Integration](part45-ef-wpf-integration.md)**
