# Part 15: Delegates & Events
## ขั้นตอนที่ 141-150: Delegates, Events, และ Functional Programming

---

## 🎯 เป้าหมายของ Part นี้
- เข้าใจ Delegates และการใช้งาน
- Action, Func, Predicate built-in delegates
- Lambda expressions
- Events และ Event handling
- Multicast delegates
- Functional programming patterns

---

## ขั้นตอนที่ 141: Delegates พื้นฐาน

### Delegates คืออะไร?
```csharp
// Delegate = type-safe function pointer
// ประกาศ delegate type
delegate int MathOperation(int x, int y);
delegate string Formatter(string input);
delegate bool Validator(string input);

// สร้าง methods ที่ match delegate signature
int Add(int x, int y) => x + y;
int Multiply(int x, int y) => x * y;
int Subtract(int x, int y) => x - y;

// ใช้ delegate
MathOperation operation = Add;
Console.WriteLine(operation(5, 3));  // 8

operation = Multiply;
Console.WriteLine(operation(5, 3));  // 15

// Delegate as parameter
void ApplyOperation(int a, int b, MathOperation op)
{
    Console.WriteLine($"{a} op {b} = {op(a, b)}");
}

ApplyOperation(10, 4, Add);         // 10 op 4 = 14
ApplyOperation(10, 4, Multiply);    // 10 op 4 = 40
ApplyOperation(10, 4, Subtract);    // 10 op 4 = 6

// Return delegate
MathOperation GetOperation(string name) => name switch
{
    "add" => Add,
    "multiply" => Multiply,
    "subtract" => Subtract,
    _ => throw new ArgumentException($"Unknown: {name}")
};

var myOp = GetOperation("multiply");
Console.WriteLine(myOp(6, 7));  // 42
```

### Built-in Delegates
```csharp
// Action<T> - delegate ที่ไม่ return ค่า
Action<string> print = Console.WriteLine;
Action<int, int> printSum = (a, b) => Console.WriteLine(a + b);

print("Hello!");
printSum(5, 3);

// Func<T, TResult> - delegate ที่ return ค่า
Func<int, int, int> add = (a, b) => a + b;
Func<string, bool> isLong = s => s.Length > 10;
Func<int, string> numberToStr = n => n.ToString();

Console.WriteLine(add(5, 3));         // 8
Console.WriteLine(isLong("Hello"));  // False
Console.WriteLine(numberToStr(42));  // "42"

// Predicate<T> - delegate ที่ return bool
Predicate<int> isEven = n => n % 2 == 0;
Predicate<string> isEmail = s => s.Contains('@');

Console.WriteLine(isEven(4));          // True
Console.WriteLine(isEmail("hi@test")); // True

// ใช้กับ collections
var numbers = new List<int> { 1, 2, 3, 4, 5, 6, 7, 8 };
var evens = numbers.FindAll(isEven);  // ใช้ Predicate<T>
numbers.ForEach(print => Console.Write($"{print} "));
```

---

## ขั้นตอนที่ 142: Multicast Delegates

### Combining Delegates
```csharp
// Multicast - delegate สามารถ invoke หลาย methods
Action<string> log = msg => Console.WriteLine($"[Log] {msg}");
Action<string> save = msg => Console.WriteLine($"[Save] {msg}");
Action<string> notify = msg => Console.WriteLine($"[Notify] {msg}");

// Combine
Action<string> allHandlers = log + save + notify;
allHandlers("System started");
// [Log] System started
// [Save] System started
// [Notify] System started

// Remove
allHandlers -= save;
allHandlers("After removing save");
// [Log] After removing save
// [Notify] After removing save

// Get invocation list
foreach (var d in allHandlers.GetInvocationList())
{
    Console.WriteLine($"Handler: {d.Method.Name}");
}

// Delegate with return value (multicast returns last result)
Func<int, int> double1 = n => n * 2;
Func<int, int> addTen = n => n + 10;

Func<int, int> combined = double1 + addTen;
int result = combined(5);  // returns 15 (from addTen, last invoked)
Console.WriteLine(result);
```

---

## ขั้นตอนที่ 143: Lambda Expressions

### Lambda syntax
```csharp
// Expression lambda: (params) => expression
Func<int, int> square = x => x * x;
Func<int, int, int> add = (x, y) => x + y;
Action<string> greet = name => Console.WriteLine($"Hello {name}!");

// Statement lambda: (params) => { statements }
Func<int, string> classify = n =>
{
    if (n < 0) return "negative";
    if (n == 0) return "zero";
    if (n < 10) return "small";
    return "large";
};

// No parameters
Func<DateTime> now = () => DateTime.Now;
Action sayHi = () => Console.WriteLine("Hi!");

// Capturing variables (closures)
int multiplier = 3;
Func<int, int> triple = n => n * multiplier;  // captures multiplier

Console.WriteLine(triple(5));  // 15
multiplier = 10;
Console.WriteLine(triple(5));  // 50 (captures by reference!)

// Static lambda (C# 9+) - cannot capture outer variables
Func<int, int> pureDouble = static n => n * 2;

// Discard parameters
Action<int, string> ignored = (_, _) => Console.WriteLine("Called!");
```

### LINQ กับ Lambda
```csharp
var products = new[]
{
    new { Name = "Laptop", Price = 35000m, Category = "Electronics" },
    new { Name = "Phone", Price = 15000m, Category = "Electronics" },
    new { Name = "Desk", Price = 12000m, Category = "Furniture" },
    new { Name = "Chair", Price = 8000m, Category = "Furniture" },
    new { Name = "Mouse", Price = 500m, Category = "Electronics" }
};

// Chained lambdas
var report = products
    .Where(p => p.Price > 5000m)
    .GroupBy(p => p.Category)
    .Select(g => new
    {
        Category = g.Key,
        Count = g.Count(),
        TotalValue = g.Sum(p => p.Price),
        MaxPrice = g.Max(p => p.Price)
    })
    .OrderByDescending(x => x.TotalValue);

foreach (var item in report)
    Console.WriteLine($"{item.Category}: {item.Count} items, Total: {item.TotalValue:N0}");
```

---

## ขั้นตอนที่ 144: Events

### Events พื้นฐาน
```csharp
// Event = delegate ที่ class อื่น subscribe/unsubscribe ได้
// แต่ invoke ได้เฉพาะ class ที่ประกาศ

class Button
{
    // Event declaration
    public event EventHandler? Clicked;
    public event EventHandler<ButtonEventArgs>? RightClicked;
    
    // Raise event
    protected virtual void OnClicked(EventArgs e)
    {
        Clicked?.Invoke(this, e);  // Null-safe invoke
    }
    
    protected virtual void OnRightClicked(ButtonEventArgs e)
    {
        RightClicked?.Invoke(this, e);
    }
    
    public void SimulateClick()
    {
        Console.WriteLine("Button clicked!");
        OnClicked(EventArgs.Empty);
    }
    
    public void SimulateRightClick(string menuItem)
    {
        OnRightClicked(new ButtonEventArgs { MenuItem = menuItem });
    }
}

class ButtonEventArgs : EventArgs
{
    public string MenuItem { get; set; } = "";
}

// Subscribe to events
var button = new Button();

button.Clicked += (sender, e) => Console.WriteLine("Handler 1: Button was clicked!");
button.Clicked += (sender, e) => Console.WriteLine("Handler 2: Logging click...");

EventHandler rightClickHandler = (sender, e) => 
{
    var args = (ButtonEventArgs)e;
    Console.WriteLine($"Right click: {args.MenuItem}");
};

button.RightClicked += rightClickHandler;

// Trigger events
button.SimulateClick();
button.SimulateRightClick("Copy");

// Unsubscribe
button.RightClicked -= rightClickHandler;
```

### Custom Event Pattern
```csharp
// EventArgs with data
class StockPriceChangedEventArgs : EventArgs
{
    public string Symbol { get; init; } = "";
    public decimal OldPrice { get; init; }
    public decimal NewPrice { get; init; }
    public decimal Change => NewPrice - OldPrice;
    public double ChangePercent => OldPrice == 0 ? 0 : (double)(Change / OldPrice * 100);
}

class Stock
{
    private decimal _price;
    
    public string Symbol { get; }
    public decimal Price
    {
        get => _price;
        set
        {
            if (_price != value)
            {
                var args = new StockPriceChangedEventArgs
                {
                    Symbol = Symbol,
                    OldPrice = _price,
                    NewPrice = value
                };
                _price = value;
                OnPriceChanged(args);
            }
        }
    }
    
    public event EventHandler<StockPriceChangedEventArgs>? PriceChanged;
    
    public Stock(string symbol, decimal initialPrice)
    {
        Symbol = symbol;
        _price = initialPrice;
    }
    
    protected virtual void OnPriceChanged(StockPriceChangedEventArgs e)
    {
        PriceChanged?.Invoke(this, e);
    }
}

// Portfolio Monitor
class PortfolioMonitor
{
    private List<Stock> _stocks = new();
    
    public void AddStock(Stock stock)
    {
        _stocks.Add(stock);
        stock.PriceChanged += OnStockPriceChanged;
    }
    
    private void OnStockPriceChanged(object? sender, StockPriceChangedEventArgs e)
    {
        string direction = e.Change >= 0 ? "▲" : "▼";
        ConsoleColor color = e.Change >= 0 ? ConsoleColor.Green : ConsoleColor.Red;
        
        Console.ForegroundColor = color;
        Console.WriteLine($"{e.Symbol}: {e.NewPrice:N2} {direction} {Math.Abs(e.ChangePercent):F2}%");
        Console.ResetColor();
        
        if (Math.Abs(e.ChangePercent) > 5)
        {
            Console.ForegroundColor = ConsoleColor.Yellow;
            Console.WriteLine($"  ⚠️ ALERT: {e.Symbol} moved more than 5%!");
            Console.ResetColor();
        }
    }
}

// Simulation
var monitor = new PortfolioMonitor();
var aapl = new Stock("AAPL", 150.00m);
var msft = new Stock("MSFT", 380.00m);

monitor.AddStock(aapl);
monitor.AddStock(msft);

aapl.Price = 152.50m;  // +1.67%
msft.Price = 395.00m;  // +3.95%
aapl.Price = 140.00m;  // -8.20% → Alert!
```

---

## ขั้นตอนที่ 145-150: โปรแกรมตัวอย่าง - Workflow Engine

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

namespace WorkflowEngine
{
    // Workflow Events
    class WorkflowEventArgs : EventArgs
    {
        public string WorkflowId { get; init; } = "";
        public string StepName { get; init; } = "";
        public DateTime Timestamp { get; init; } = DateTime.UtcNow;
        public object? Data { get; init; }
    }
    
    // Generic Step
    class WorkflowStep<TInput, TOutput>
    {
        public string Name { get; }
        private readonly Func<TInput, Task<TOutput>> _execute;
        private readonly Func<TInput, bool>? _condition;
        
        public event EventHandler<WorkflowEventArgs>? Started;
        public event EventHandler<WorkflowEventArgs>? Completed;
        public event EventHandler<WorkflowEventArgs>? Failed;
        
        public WorkflowStep(string name, Func<TInput, Task<TOutput>> execute, 
            Func<TInput, bool>? condition = null)
        {
            Name = name;
            _execute = execute;
            _condition = condition;
        }
        
        public WorkflowStep(string name, Func<TInput, TOutput> execute,
            Func<TInput, bool>? condition = null)
            : this(name, input => Task.FromResult(execute(input)), condition) { }
        
        public async Task<(bool skipped, TOutput? result)> RunAsync(TInput input, string workflowId)
        {
            if (_condition != null && !_condition(input))
            {
                Console.WriteLine($"  ⏭️ Skipping: {Name}");
                return (true, default);
            }
            
            Started?.Invoke(this, new WorkflowEventArgs { WorkflowId = workflowId, StepName = Name });
            
            try
            {
                var result = await _execute(input);
                Completed?.Invoke(this, new WorkflowEventArgs 
                { 
                    WorkflowId = workflowId, 
                    StepName = Name, 
                    Data = result 
                });
                return (false, result);
            }
            catch (Exception ex)
            {
                Failed?.Invoke(this, new WorkflowEventArgs 
                { 
                    WorkflowId = workflowId, 
                    StepName = Name, 
                    Data = ex.Message 
                });
                throw;
            }
        }
    }
    
    // Order Processing Workflow
    record Order(int Id, string CustomerEmail, decimal Amount, List<string> Items);
    record ValidatedOrder(Order Original, bool IsValid, string? Error = null);
    record ProcessedPayment(ValidatedOrder Order, bool PaymentSuccess, string TransactionId);
    record FulfilledOrder(ProcessedPayment Payment, DateTime? EstimatedDelivery);
    
    class OrderWorkflow
    {
        private int _processedCount = 0;
        private int _failedCount = 0;
        
        // Steps
        private WorkflowStep<Order, ValidatedOrder> _validateStep;
        private WorkflowStep<ValidatedOrder, ProcessedPayment> _paymentStep;
        private WorkflowStep<ProcessedPayment, FulfilledOrder> _fulfillStep;
        
        public event EventHandler<WorkflowEventArgs>? OrderProcessed;
        public event EventHandler<WorkflowEventArgs>? OrderFailed;
        
        public OrderWorkflow()
        {
            // Define steps
            _validateStep = new WorkflowStep<Order, ValidatedOrder>(
                "Validate Order",
                async order =>
                {
                    await Task.Delay(50);
                    
                    if (order.Amount <= 0)
                        return new ValidatedOrder(order, false, "Amount must be positive");
                    
                    if (!order.CustomerEmail.Contains('@'))
                        return new ValidatedOrder(order, false, "Invalid email");
                    
                    if (!order.Items.Any())
                        return new ValidatedOrder(order, false, "No items in order");
                    
                    return new ValidatedOrder(order, true);
                }
            );
            
            _paymentStep = new WorkflowStep<ValidatedOrder, ProcessedPayment>(
                "Process Payment",
                async validated =>
                {
                    if (!validated.IsValid)
                        return new ProcessedPayment(validated, false, "SKIPPED");
                    
                    await Task.Delay(100);  // จำลอง payment gateway
                    
                    bool success = validated.Original.Amount < 500000m;  // Limit
                    string txId = success ? $"TXN-{Guid.NewGuid():N}"[..16] : "FAILED";
                    
                    return new ProcessedPayment(validated, success, txId);
                },
                validated => validated.IsValid  // Only run if valid
            );
            
            _fulfillStep = new WorkflowStep<ProcessedPayment, FulfilledOrder>(
                "Fulfill Order",
                async payment =>
                {
                    if (!payment.PaymentSuccess)
                        return new FulfilledOrder(payment, null);
                    
                    await Task.Delay(50);
                    
                    // Calculate delivery
                    var delivery = DateTime.Now.AddDays(
                        payment.Order.Original.Amount > 100000m ? 2 : 5
                    );
                    
                    return new FulfilledOrder(payment, delivery);
                },
                payment => payment.PaymentSuccess
            );
            
            // Subscribe to step events for logging
            _validateStep.Completed += (s, e) => Console.WriteLine($"  ✅ {e.StepName} done");
            _paymentStep.Completed += (s, e) => Console.WriteLine($"  ✅ {e.StepName} done");
            _fulfillStep.Completed += (s, e) => Console.WriteLine($"  ✅ {e.StepName} done");
        }
        
        public async Task<FulfilledOrder> ProcessOrderAsync(Order order)
        {
            string workflowId = $"WF-{Guid.NewGuid():N}"[..8];
            Console.WriteLine($"\n[{workflowId}] Processing order #{order.Id}");
            
            try
            {
                var (_, validated) = await _validateStep.RunAsync(order, workflowId);
                var (_, payment) = await _paymentStep.RunAsync(validated!, workflowId);
                var (_, fulfilled) = await _fulfillStep.RunAsync(payment!, workflowId);
                
                _processedCount++;
                
                OrderProcessed?.Invoke(this, new WorkflowEventArgs 
                { 
                    WorkflowId = workflowId, 
                    Data = fulfilled 
                });
                
                return fulfilled!;
            }
            catch (Exception ex)
            {
                _failedCount++;
                OrderFailed?.Invoke(this, new WorkflowEventArgs 
                { 
                    WorkflowId = workflowId, 
                    Data = ex.Message 
                });
                throw;
            }
        }
        
        public void PrintStats()
        {
            Console.WriteLine($"\n=== Workflow Statistics ===");
            Console.WriteLine($"Processed: {_processedCount}");
            Console.WriteLine($"Failed: {_failedCount}");
        }
    }
    
    class Program
    {
        static async Task Main()
        {
            Console.OutputEncoding = System.Text.Encoding.UTF8;
            Console.Title = "Workflow Engine";
            
            var workflow = new OrderWorkflow();
            
            // Subscribe to workflow events
            workflow.OrderProcessed += (s, e) =>
            {
                var order = e.Data as FulfilledOrder;
                Console.ForegroundColor = ConsoleColor.Green;
                if (order?.EstimatedDelivery != null)
                    Console.WriteLine($"  🚚 Delivery: {order.EstimatedDelivery:dd/MM/yyyy}");
                Console.ResetColor();
            };
            
            workflow.OrderFailed += (s, e) =>
            {
                Console.ForegroundColor = ConsoleColor.Red;
                Console.WriteLine($"  ❌ Failed: {e.Data}");
                Console.ResetColor();
            };
            
            // Test orders
            var orders = new[]
            {
                new Order(1, "alice@test.com", 15000m, new List<string> { "Laptop", "Mouse" }),
                new Order(2, "invalid-email", 5000m, new List<string> { "Phone" }),
                new Order(3, "bob@test.com", -100m, new List<string> { "Tablet" }),
                new Order(4, "charlie@test.com", 600000m, new List<string> { "Server" }),
                new Order(5, "diana@test.com", 8500m, new List<string> { "Monitor", "Keyboard" })
            };
            
            // Process all orders
            foreach (var order in orders)
            {
                try
                {
                    var result = await workflow.ProcessOrderAsync(order);
                    
                    bool success = result.Payment.PaymentSuccess;
                    Console.WriteLine($"  Result: {(success ? "✅ SUCCESS" : "❌ PAYMENT FAILED")}");
                    if (success)
                        Console.WriteLine($"  Transaction: {result.Payment.TransactionId}");
                }
                catch (Exception ex)
                {
                    Console.ForegroundColor = ConsoleColor.Red;
                    Console.WriteLine($"  Error: {ex.Message}");
                    Console.ResetColor();
                }
            }
            
            workflow.PrintStats();
            
            Console.WriteLine("\nกด Enter เพื่อออก...");
            Console.ReadLine();
        }
    }
}
```

---

## 📝 สรุป Part 15

| หัวข้อ | Key Points |
|--------|-----------|
| delegate | Type สำหรับ function pointers |
| Action<T> | Delegate ที่ไม่ return ค่า |
| Func<T, TResult> | Delegate ที่ return ค่า |
| Predicate<T> | Delegate ที่ return bool |
| Lambda `=>` | Concise delegate/anonymous method |
| Multicast | Delegate ที่มีหลาย handlers |
| event | delegate ที่จัดการ subscription |
| EventHandler<T> | Standard event pattern |
| Closure | Lambda capture outer variables |

---

**ก่อนหน้า → [Part 14: Generics](part14-generics.md)**  
**ต่อไป → [Part 16: Async/Await](part16-async-await.md)**
