# Part 14: Generics
## ขั้นตอนที่ 131-140: Generic Programming

---

## 🎯 เป้าหมายของ Part นี้
- เข้าใจ Generics และประโยชน์
- Generic Classes, Methods, Interfaces
- Type Constraints
- Covariance และ Contravariance
- สร้าง Generic Data Structures
- Generic patterns ระดับมืออาชีพ

---

## ขั้นตอนที่ 131: Generics พื้นฐาน

### ทำไมต้องใช้ Generics?
```csharp
// ❌ Without Generics - ต้อง cast, ไม่ type-safe
class ObjectBox
{
    private object _value;
    public void Set(object value) => _value = value;
    public object Get() => _value;
}

var box = new ObjectBox();
box.Set(42);
int value = (int)box.Get();  // Must cast
box.Set("Hello");            // Can set wrong type - runtime error!

// ✅ With Generics - type-safe, no boxing
class Box<T>
{
    private T _value = default!;
    public void Set(T value) => _value = value;
    public T Get() => _value;
    public bool HasValue => _value is not null;
    public override string ToString() => _value?.ToString() ?? "Empty";
}

var intBox = new Box<int>();
intBox.Set(42);
int num = intBox.Get();  // No cast needed

var strBox = new Box<string>();
strBox.Set("Hello");
// strBox.Set(42);  // ← Compile error! Type safety

// Benefits of Generics:
// 1. Type safety at compile time
// 2. No boxing/unboxing for value types
// 3. Code reuse without duplication
// 4. Better performance
```

### Generic Methods
```csharp
// Generic method
static T Max<T>(T a, T b) where T : IComparable<T>
    => a.CompareTo(b) >= 0 ? a : b;

int maxInt = Max(5, 3);           // T inferred as int
double maxDouble = Max(3.14, 2.71);  // T inferred as double
string maxStr = Max("banana", "apple");  // T inferred as string

// Multiple type parameters
static Dictionary<TKey, TValue> ZipToDictionary<TKey, TValue>(
    IEnumerable<TKey> keys, 
    IEnumerable<TValue> values) where TKey : notnull
{
    return keys.Zip(values, (k, v) => (k, v))
        .ToDictionary(x => x.k, x => x.v);
}

var keys = new[] { "a", "b", "c" };
var vals = new[] { 1, 2, 3 };
var dict = ZipToDictionary(keys, vals);
// { "a": 1, "b": 2, "c": 3 }

// Generic extension method
static class Extensions
{
    public static T? FirstOrNull<T>(this IEnumerable<T> source, Func<T, bool> predicate)
        where T : struct
    {
        foreach (var item in source)
        {
            if (predicate(item)) return item;
        }
        return null;
    }
    
    public static void ForEach<T>(this IEnumerable<T> source, Action<T> action)
    {
        foreach (var item in source) action(item);
    }
    
    public static bool None<T>(this IEnumerable<T> source, Func<T, bool> predicate)
        => !source.Any(predicate);
}
```

---

## ขั้นตอนที่ 132: Type Constraints

### where constraints
```csharp
// class constraint - T must be a reference type
class Repository<T> where T : class
{
    private List<T> _items = new();
    public void Add(T item) => _items.Add(item);
    public IReadOnlyList<T> GetAll() => _items.AsReadOnly();
}

// struct constraint - T must be a value type
struct Optional<T> where T : struct
{
    private T _value;
    public bool HasValue { get; private set; }
    
    public static Optional<T> Of(T value) => new() { _value = value, HasValue = true };
    public static Optional<T> Empty() => new() { HasValue = false };
    
    public T Value => HasValue ? _value : throw new InvalidOperationException("No value");
    public T GetValueOrDefault(T defaultValue = default) => HasValue ? _value : defaultValue;
}

// new() constraint - T must have parameterless constructor
class Factory<T> where T : new()
{
    public T Create() => new T();
    public List<T> CreateMany(int count) => Enumerable.Range(0, count).Select(_ => new T()).ToList();
}

// Interface constraint
class Sorter<T> where T : IComparable<T>
{
    public T[] BubbleSort(T[] array)
    {
        var result = (T[])array.Clone();
        for (int i = 0; i < result.Length - 1; i++)
            for (int j = 0; j < result.Length - i - 1; j++)
                if (result[j].CompareTo(result[j + 1]) > 0)
                    (result[j], result[j + 1]) = (result[j + 1], result[j]);
        return result;
    }
}

// Base class constraint
class Animal { public virtual string Speak() => "..."; }
class Dog : Animal { public override string Speak() => "Woof!"; }
class Cat : Animal { public override string Speak() => "Meow!"; }

class AnimalShelter<T> where T : Animal
{
    private List<T> _animals = new();
    public void Admit(T animal) => _animals.Add(animal);
    public void AllSpeak() => _animals.ForEach(a => Console.WriteLine(a.Speak()));
}

// Multiple constraints
class Service<T> where T : class, IComparable<T>, new()
{
    public T Create() => new T();
    public T Max(T a, T b) => a.CompareTo(b) >= 0 ? a : b;
}

// Unmanaged constraint (C# 7.3+) - for unsafe code
class Buffer<T> where T : unmanaged
{
    private T[] _data;
    public Buffer(int size) => _data = new T[size];
    public T this[int i] { get => _data[i]; set => _data[i] = value; }
}
```

---

## ขั้นตอนที่ 133: Generic Collections

### สร้าง Generic Stack
```csharp
class Stack<T>
{
    private T[] _items;
    private int _top = -1;
    
    public Stack(int capacity = 16)
    {
        _items = new T[capacity];
    }
    
    public int Count => _top + 1;
    public bool IsEmpty => _top < 0;
    public bool IsFull => _top == _items.Length - 1;
    
    public void Push(T item)
    {
        if (IsFull) Resize();
        _items[++_top] = item;
    }
    
    public T Pop()
    {
        if (IsEmpty) throw new InvalidOperationException("Stack is empty");
        T item = _items[_top];
        _items[_top--] = default!;  // clear reference
        return item;
    }
    
    public T Peek()
    {
        if (IsEmpty) throw new InvalidOperationException("Stack is empty");
        return _items[_top];
    }
    
    public bool TryPop(out T item)
    {
        if (IsEmpty) { item = default!; return false; }
        item = Pop();
        return true;
    }
    
    private void Resize()
    {
        Array.Resize(ref _items, _items.Length * 2);
    }
    
    public IEnumerable<T> ToEnumerable()
    {
        for (int i = _top; i >= 0; i--)
            yield return _items[i];
    }
}

// Generic Priority Queue
class PriorityQueue<T> where T : IComparable<T>
{
    private List<T> _heap = new();
    
    public int Count => _heap.Count;
    
    public void Enqueue(T item)
    {
        _heap.Add(item);
        HeapifyUp(_heap.Count - 1);
    }
    
    public T Dequeue()
    {
        if (_heap.Count == 0) throw new InvalidOperationException("Queue is empty");
        
        T top = _heap[0];
        int last = _heap.Count - 1;
        _heap[0] = _heap[last];
        _heap.RemoveAt(last);
        
        if (_heap.Count > 0) HeapifyDown(0);
        
        return top;
    }
    
    public T Peek() => _heap.Count > 0 ? _heap[0] : throw new InvalidOperationException();
    
    private void HeapifyUp(int index)
    {
        while (index > 0)
        {
            int parent = (index - 1) / 2;
            if (_heap[index].CompareTo(_heap[parent]) < 0)
            {
                (_heap[index], _heap[parent]) = (_heap[parent], _heap[index]);
                index = parent;
            }
            else break;
        }
    }
    
    private void HeapifyDown(int index)
    {
        while (true)
        {
            int left = 2 * index + 1;
            int right = 2 * index + 2;
            int smallest = index;
            
            if (left < _heap.Count && _heap[left].CompareTo(_heap[smallest]) < 0) smallest = left;
            if (right < _heap.Count && _heap[right].CompareTo(_heap[smallest]) < 0) smallest = right;
            
            if (smallest == index) break;
            
            (_heap[index], _heap[smallest]) = (_heap[smallest], _heap[index]);
            index = smallest;
        }
    }
}
```

---

## ขั้นตอนที่ 134: Covariance และ Contravariance

### out (Covariant) และ in (Contravariant)
```csharp
// Covariance (out) - IEnumerable<Dog> can be IEnumerable<Animal>
interface IProducer<out T>
{
    T Produce();
}

class DogProducer : IProducer<Dog>
{
    public Dog Produce() => new Dog("Rex");
}

// Covariant assignment
IProducer<Dog> dogProducer = new DogProducer();
IProducer<Animal> animalProducer = dogProducer;  // OK! (covariant)
Animal animal = animalProducer.Produce();

// Contravariance (in) - IComparer<Animal> can be IComparer<Dog>
interface IConsumer<in T>
{
    void Consume(T item);
}

class AnimalPrinter : IConsumer<Animal>
{
    public void Consume(Animal item) => Console.WriteLine(item.Name);
}

// Contravariant assignment
IConsumer<Animal> animalConsumer = new AnimalPrinter();
IConsumer<Dog> dogConsumer = animalConsumer;  // OK! (contravariant)
dogConsumer.Consume(new Dog("Buddy"));

class Animal { public string Name { get; set; } = ""; }
class Dog : Animal { }

// Real world: IEnumerable<T> is covariant
List<Dog> dogs = new() { new Dog { Name = "Rex" }, new Dog { Name = "Buddy" } };
IEnumerable<Animal> animals = dogs;  // Works because IEnumerable<out T>

foreach (Animal a in animals)
    Console.WriteLine(a.Name);
```

---

## ขั้นตอนที่ 135-140: โปรแกรมตัวอย่าง - Generic Data Pipeline

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

namespace GenericPipeline
{
    // Generic Pipeline Step
    interface IPipelineStep<TIn, TOut>
    {
        Task<TOut> ExecuteAsync(TIn input);
    }
    
    // Generic Pipeline
    class Pipeline<TInput, TOutput>
    {
        private readonly Func<TInput, Task<TOutput>> _execute;
        
        private Pipeline(Func<TInput, Task<TOutput>> execute)
        {
            _execute = execute;
        }
        
        public static Pipeline<T, T> Create<T>()
        {
            return new Pipeline<T, T>(input => Task.FromResult(input));
        }
        
        public Pipeline<TInput, TNext> Then<TNext>(Func<TOutput, Task<TNext>> next)
        {
            return new Pipeline<TInput, TNext>(async input =>
            {
                var intermediate = await _execute(input);
                return await next(intermediate);
            });
        }
        
        public Pipeline<TInput, TNext> Then<TNext>(Func<TOutput, TNext> next)
        {
            return Then<TNext>(input => Task.FromResult(next(input)));
        }
        
        public async Task<TOutput> RunAsync(TInput input) => await _execute(input);
    }
    
    // Generic Cache
    class Cache<TKey, TValue> where TKey : notnull
    {
        private readonly Dictionary<TKey, (TValue Value, DateTime Expiry)> _store = new();
        private readonly TimeSpan _defaultTtl;
        
        public Cache(TimeSpan? defaultTtl = null)
        {
            _defaultTtl = defaultTtl ?? TimeSpan.FromMinutes(5);
        }
        
        public void Set(TKey key, TValue value, TimeSpan? ttl = null)
        {
            var expiry = DateTime.UtcNow.Add(ttl ?? _defaultTtl);
            _store[key] = (value, expiry);
        }
        
        public bool TryGet(TKey key, out TValue? value)
        {
            if (_store.TryGetValue(key, out var entry) && entry.Expiry > DateTime.UtcNow)
            {
                value = entry.Value;
                return true;
            }
            
            _store.Remove(key);
            value = default;
            return false;
        }
        
        public async Task<TValue> GetOrSetAsync(TKey key, Func<Task<TValue>> factory, TimeSpan? ttl = null)
        {
            if (TryGet(key, out var cached)) return cached!;
            var value = await factory();
            Set(key, value, ttl);
            return value;
        }
        
        public void Evict(TKey key) => _store.Remove(key);
        
        public void Clear() => _store.Clear();
        
        public int Count => _store.Count(x => x.Value.Expiry > DateTime.UtcNow);
    }
    
    // Generic Event Bus
    class EventBus
    {
        private readonly Dictionary<Type, List<Delegate>> _handlers = new();
        
        public void Subscribe<TEvent>(Action<TEvent> handler)
        {
            var type = typeof(TEvent);
            if (!_handlers.ContainsKey(type))
                _handlers[type] = new List<Delegate>();
            _handlers[type].Add(handler);
        }
        
        public void Publish<TEvent>(TEvent @event)
        {
            var type = typeof(TEvent);
            if (!_handlers.TryGetValue(type, out var handlers)) return;
            
            foreach (var handler in handlers)
                ((Action<TEvent>)handler)(@event);
        }
        
        public void Unsubscribe<TEvent>(Action<TEvent> handler)
        {
            var type = typeof(TEvent);
            if (_handlers.TryGetValue(type, out var handlers))
                handlers.Remove(handler);
        }
    }
    
    // Generic Result type
    class Result<T>
    {
        public bool IsSuccess { get; }
        public T? Value { get; }
        public string? Error { get; }
        public Exception? Exception { get; }
        
        private Result(T value) { IsSuccess = true; Value = value; }
        private Result(string error, Exception? ex = null) { Error = error; Exception = ex; }
        
        public static Result<T> Ok(T value) => new(value);
        public static Result<T> Fail(string error, Exception? ex = null) => new(error, ex);
        
        public Result<TNext> Map<TNext>(Func<T, TNext> transform)
            => IsSuccess ? Result<TNext>.Ok(transform(Value!)) : Result<TNext>.Fail(Error!, Exception);
        
        public async Task<Result<TNext>> MapAsync<TNext>(Func<T, Task<TNext>> transform)
        {
            if (!IsSuccess) return Result<TNext>.Fail(Error!, Exception);
            try { return Result<TNext>.Ok(await transform(Value!)); }
            catch (Exception ex) { return Result<TNext>.Fail(ex.Message, ex); }
        }
        
        public override string ToString() 
            => IsSuccess ? $"Ok({Value})" : $"Fail({Error})";
    }
    
    // Demo Events
    record OrderPlaced(int OrderId, decimal Amount);
    record PaymentProcessed(int OrderId, bool Success);
    record OrderShipped(int OrderId, string TrackingNumber);
    
    class Program
    {
        static async Task Main()
        {
            Console.OutputEncoding = System.Text.Encoding.UTF8;
            
            // Test Pipeline
            Console.WriteLine("=== Generic Pipeline ===");
            
            var pipeline = Pipeline<string, string>.Create<string>()
                .Then(s => s.Trim())
                .Then(s => s.ToUpper())
                .Then(s => s.Replace(" ", "_"))
                .Then(async s => { await Task.Delay(10); return $"processed_{s}"; });
            
            string result = await pipeline.RunAsync("  Hello World  ");
            Console.WriteLine($"Pipeline result: {result}");
            
            // Test Cache
            Console.WriteLine("\n=== Generic Cache ===");
            var cache = new Cache<string, string>(TimeSpan.FromSeconds(30));
            
            int fetchCount = 0;
            
            async Task<string> FetchData(string key)
            {
                fetchCount++;
                await Task.Delay(100);  // Simulate async fetch
                return $"Data for {key} (fetch #{fetchCount})";
            }
            
            // First call - fetches
            var data1 = await cache.GetOrSetAsync("user:1", () => FetchData("user:1"));
            Console.WriteLine($"First call: {data1}");
            
            // Second call - from cache
            var data2 = await cache.GetOrSetAsync("user:1", () => FetchData("user:1"));
            Console.WriteLine($"Second call: {data2}");
            
            Console.WriteLine($"Total fetches: {fetchCount} (expected 1)");
            
            // Test Event Bus
            Console.WriteLine("\n=== Event Bus ===");
            var bus = new EventBus();
            
            bus.Subscribe<OrderPlaced>(e => 
                Console.WriteLine($"[Handler1] Order placed: #{e.OrderId} for {e.Amount:N0} บาท"));
            
            bus.Subscribe<OrderPlaced>(e => 
                Console.WriteLine($"[Handler2] Sending confirmation for order #{e.OrderId}"));
            
            bus.Subscribe<PaymentProcessed>(e => 
                Console.WriteLine($"[PaymentHandler] Order #{e.OrderId}: {(e.Success ? "Paid" : "Failed")}"));
            
            bus.Subscribe<OrderShipped>(e => 
                Console.WriteLine($"[ShipHandler] Order #{e.OrderId} shipped! Tracking: {e.TrackingNumber}"));
            
            // Simulate order flow
            bus.Publish(new OrderPlaced(1001, 35000m));
            bus.Publish(new PaymentProcessed(1001, true));
            bus.Publish(new OrderShipped(1001, "TH1234567890"));
            
            // Test Result
            Console.WriteLine("\n=== Generic Result ===");
            
            Result<int> ParseNumber(string s)
            {
                return int.TryParse(s, out int n) 
                    ? Result<int>.Ok(n) 
                    : Result<int>.Fail($"Cannot parse '{s}' as number");
            }
            
            var r1 = ParseNumber("42")
                .Map(n => n * 2)
                .Map(n => $"Result: {n}");
            Console.WriteLine(r1);
            
            var r2 = ParseNumber("abc")
                .Map(n => n * 2);
            Console.WriteLine(r2);
            
            Console.WriteLine("\nกด Enter เพื่อออก...");
            Console.ReadLine();
        }
    }
}
```

---

## 📝 สรุป Part 14

| หัวข้อ | Key Points |
|--------|-----------|
| Generic Classes | `class Box<T>` |
| Generic Methods | `T Max<T>(T a, T b)` |
| Type Constraints | `where T : class, IComparable<T>, new()` |
| Covariance | `out T` - assign derived to base |
| Contravariance | `in T` - assign base to derived |
| Generic Collections | Stack<T>, Queue<T>, List<T> |
| Generic Patterns | Pipeline, Cache, EventBus, Result |

---

**ก่อนหน้า → [Part 13: LINQ](part13-linq.md)**  
**ต่อไป → [Part 15: Delegates & Events](part15-delegates-events.md)**
