# Part 19: Extension Methods & Nullable
## ขั้นตอนที่ 181-190: Extension Methods และ Nullable Reference Types

---

## 🎯 เป้าหมายของ Part นี้
- Extension Methods: เพิ่ม methods ให้ existing types
- Fluent API ด้วย Extensions
- Nullable Reference Types (C# 8+)
- Null-coalescing operators
- Null-conditional operators
- สร้าง Utility Library จริง

---

## ขั้นตอนที่ 181: Extension Methods

### สร้าง Extension Methods
```csharp
// Extension methods อยู่ใน static class
static class StringExtensions
{
    // this = type ที่ extend
    public static bool IsEmail(this string value)
        => !string.IsNullOrWhiteSpace(value) && value.Contains('@') && value.Contains('.');
    
    public static bool IsThaiPhone(this string value)
        => System.Text.RegularExpressions.Regex.IsMatch(value ?? "", @"^0[689]\d{8}$");
    
    public static string TruncateAt(this string value, int maxLength, string suffix = "...")
    {
        if (value.Length <= maxLength) return value;
        return value[..(maxLength - suffix.Length)] + suffix;
    }
    
    public static string ToSlug(this string value)
        => value.ToLower()
            .Replace(" ", "-")
            .Replace("_", "-")
            .Where(c => char.IsLetterOrDigit(c) || c == '-')
            .ToArray()
            .Let(chars => new string(chars));
    
    public static string Repeat(this string value, int times)
        => string.Concat(Enumerable.Repeat(value, times));
    
    public static string Capitalize(this string value)
        => string.IsNullOrEmpty(value) ? value : char.ToUpper(value[0]) + value[1..];
    
    public static T? TryConvert<T>(this string value) where T : struct
    {
        try { return (T)Convert.ChangeType(value, typeof(T)); }
        catch { return null; }
    }
}

// ใช้งาน
string email = "user@example.com";
Console.WriteLine(email.IsEmail());          // True

string title = "Hello World This Is Long";
Console.WriteLine(title.TruncateAt(15));     // Hello World...

string slug = "My Blog Post Title";
Console.WriteLine(slug.ToSlug());            // my-blog-post-title

string divider = "-".Repeat(40);
Console.WriteLine(divider);                  // ----------------------------------------

// ใช้กับ null
string? nullStr = null;
// nullStr.IsEmail();  // NullReferenceException! (ระวัง)
Console.WriteLine(nullStr?.IsEmail() ?? false);  // False

// TryConvert
string numStr = "42";
int? num = numStr.TryConvert<int>();
Console.WriteLine(num);  // 42
```

---

## ขั้นตอนที่ 182: Collection Extensions

### Extensions สำหรับ Collections
```csharp
static class CollectionExtensions
{
    // Batch/Chunk
    public static IEnumerable<IEnumerable<T>> Batch<T>(this IEnumerable<T> source, int batchSize)
    {
        var batch = new List<T>(batchSize);
        foreach (var item in source)
        {
            batch.Add(item);
            if (batch.Count == batchSize)
            {
                yield return batch;
                batch = new List<T>(batchSize);
            }
        }
        if (batch.Any()) yield return batch;
    }
    
    // Shuffle (Fisher-Yates)
    public static List<T> Shuffle<T>(this IEnumerable<T> source, Random? random = null)
    {
        var list = source.ToList();
        var rng = random ?? Random.Shared;
        
        for (int i = list.Count - 1; i > 0; i--)
        {
            int j = rng.Next(i + 1);
            (list[i], list[j]) = (list[j], list[i]);
        }
        
        return list;
    }
    
    // SafeGet with default
    public static TValue GetValueOrDefault<TKey, TValue>(
        this IDictionary<TKey, TValue> dict, TKey key, TValue defaultValue)
        => dict.TryGetValue(key, out var value) ? value : defaultValue;
    
    // AddRange to HashSet
    public static void AddRange<T>(this HashSet<T> set, IEnumerable<T> items)
    {
        foreach (var item in items) set.Add(item);
    }
    
    // ToHashSet with key selector
    public static HashSet<TKey> ToHashSet<T, TKey>(
        this IEnumerable<T> source, Func<T, TKey> keySelector)
        => new HashSet<TKey>(source.Select(keySelector));
    
    // Flatten nested collections
    public static IEnumerable<T> Flatten<T>(this IEnumerable<IEnumerable<T>> source)
        => source.SelectMany(x => x);
    
    // Index of with predicate
    public static int IndexOf<T>(this IList<T> list, Func<T, bool> predicate)
    {
        for (int i = 0; i < list.Count; i++)
            if (predicate(list[i])) return i;
        return -1;
    }
    
    // Safe First/Last
    public static T? SafeFirst<T>(this IEnumerable<T> source) where T : class
        => source.FirstOrDefault();
    
    public static T? SafeLast<T>(this IEnumerable<T> source) where T : class
        => source.LastOrDefault();
    
    // Min/Max By
    public static T? MinBy<T, TKey>(this IEnumerable<T> source, Func<T, TKey> keySelector)
        where TKey : IComparable<TKey>
        => source.OrderBy(keySelector).FirstOrDefault();
    
    // Zip with index
    public static IEnumerable<(int Index, T Item)> WithIndex<T>(this IEnumerable<T> source)
        => source.Select((item, index) => (index, item));
    
    // Partition into two lists
    public static (List<T> True, List<T> False) Partition<T>(
        this IEnumerable<T> source, Func<T, bool> predicate)
    {
        var trueList = new List<T>();
        var falseList = new List<T>();
        
        foreach (var item in source)
        {
            if (predicate(item)) trueList.Add(item);
            else falseList.Add(item);
        }
        
        return (trueList, falseList);
    }
}

// Helper extension
static class ObjectExtensions
{
    public static T Let<T>(this T value, Func<T, T> transform) => transform(value);
    public static TResult Map<T, TResult>(this T value, Func<T, TResult> transform) => transform(value);
    public static T Also<T>(this T value, Action<T> action) { action(value); return value; }
    public static bool IsNull<T>(this T? value) where T : class => value is null;
    public static bool IsNotNull<T>(this T? value) where T : class => value is not null;
}

// การใช้งาน
var numbers = Enumerable.Range(1, 20).ToList();

// Batch
foreach (var batch in numbers.Batch(5))
    Console.WriteLine(string.Join(", ", batch));

// Shuffle
var shuffled = numbers.Shuffle();
Console.WriteLine(string.Join(", ", shuffled));

// Partition
var (evens, odds) = numbers.Partition(n => n % 2 == 0);
Console.WriteLine($"Evens: {evens.Count}, Odds: {odds.Count}");

// WithIndex
foreach (var (index, item) in new[] { "a", "b", "c" }.WithIndex())
    Console.WriteLine($"[{index}] {item}");
```

---

## ขั้นตอนที่ 183: Fluent API

### สร้าง Fluent API ด้วย Extension Methods
```csharp
// Query Builder ด้วย Fluent API
class QueryBuilder
{
    private string _table = "";
    private readonly List<string> _conditions = new();
    private readonly List<string> _columns = new();
    private string? _orderBy;
    private bool _orderDesc = false;
    private int? _limit;
    private int? _offset;
    
    public string Table => _table;
    public IReadOnlyList<string> Conditions => _conditions.AsReadOnly();
    
    public QueryBuilder From(string table) { _table = table; return this; }
    public QueryBuilder Select(params string[] columns) { _columns.AddRange(columns); return this; }
    public QueryBuilder Where(string condition) { _conditions.Add(condition); return this; }
    public QueryBuilder OrderBy(string column, bool desc = false) 
    { 
        _orderBy = column; 
        _orderDesc = desc; 
        return this; 
    }
    public QueryBuilder Limit(int limit) { _limit = limit; return this; }
    public QueryBuilder Offset(int offset) { _offset = offset; return this; }
    
    public string Build()
    {
        var cols = _columns.Any() ? string.Join(", ", _columns) : "*";
        var sql = $"SELECT {cols} FROM {_table}";
        
        if (_conditions.Any())
            sql += " WHERE " + string.Join(" AND ", _conditions);
        
        if (_orderBy is not null)
            sql += $" ORDER BY {_orderBy}{(_orderDesc ? " DESC" : "")}";
        
        if (_limit.HasValue)
            sql += $" LIMIT {_limit}";
        
        if (_offset.HasValue)
            sql += $" OFFSET {_offset}";
        
        return sql;
    }
}

// Fluent usage
var query = new QueryBuilder()
    .From("users")
    .Select("id", "name", "email", "created_at")
    .Where("is_active = 1")
    .Where("age >= 18")
    .OrderBy("created_at", desc: true)
    .Limit(20)
    .Offset(0)
    .Build();

Console.WriteLine(query);
// SELECT id, name, email, created_at FROM users WHERE is_active = 1 AND age >= 18 ORDER BY created_at DESC LIMIT 20 OFFSET 0
```

---

## ขั้นตอนที่ 184: Nullable Reference Types

### C# 8+ Nullable Reference Types
```csharp
// เปิดใช้ Nullable Reference Types ใน .csproj:
// <Nullable>enable</Nullable>

// ? = nullable, non-? = non-nullable
string nonNullable = "Hello";     // Cannot be null
string? nullable = null;          // Can be null

// nonNullable = null;  // Warning! CS8600

// Null checks
void ProcessName(string? name)
{
    // name.Length  // Warning! Possible null dereference
    
    if (name is not null)
        Console.WriteLine(name.Length);  // OK
    
    // Null-forgiving operator (!)
    string definitelyNotNull = name!;  // Suppresses warning (use carefully!)
    
    // ArgumentNullException.ThrowIfNull (C# 10+)
    ArgumentNullException.ThrowIfNull(name);
    Console.WriteLine(name.Length);  // Safe now
}

// Null-coalescing ??
string? input = null;
string result = input ?? "default";
Console.WriteLine(result);  // default

// Null-coalescing assignment ??=
string? value = null;
value ??= "initialized";  // Only assigns if null
Console.WriteLine(value);  // initialized

// Null-conditional ?.
string? str = null;
int? length = str?.Length;      // null (no exception)
string? upper = str?.ToUpper(); // null

// Chain
var user = GetUser(1);
string? city = user?.Address?.City?.ToUpper();

// Null-conditional with indexer
string[]? arr = null;
string? first = arr?[0];  // null (no exception)

// Elvis operator for methods
int? count = user?.GetOrderCount();

User? GetUser(int id) => id == 1 ? new User("Alice") : null;

class User(string name)
{
    public string Name { get; } = name;
    public Address? Address { get; set; }
    public int GetOrderCount() => 5;
}

class Address
{
    public string? City { get; set; }
}
```

---

## ขั้นตอนที่ 185: Null Safety Patterns

### Null Object Pattern
```csharp
// Null Object Pattern
interface ILogger
{
    void Log(string message);
    void LogError(string message);
}

class ConsoleLogger : ILogger
{
    public void Log(string message) => Console.WriteLine($"[LOG] {message}");
    public void LogError(string message) 
    {
        Console.ForegroundColor = ConsoleColor.Red;
        Console.WriteLine($"[ERR] {message}");
        Console.ResetColor();
    }
}

// Null Object - does nothing
class NullLogger : ILogger
{
    public static NullLogger Instance { get; } = new();
    private NullLogger() { }
    public void Log(string message) { }
    public void LogError(string message) { }
}

// ใช้งาน
class DataProcessor
{
    private readonly ILogger _logger;
    
    public DataProcessor(ILogger? logger = null)
    {
        _logger = logger ?? NullLogger.Instance;  // Use null object if not provided
    }
    
    public void Process(string data)
    {
        _logger.Log($"Processing: {data}");
        // ... process
        _logger.Log("Done");
    }
}

// ไม่ต้อง check null ทุกที่
var processor1 = new DataProcessor(new ConsoleLogger());
var processor2 = new DataProcessor();  // Uses NullLogger - no errors!

processor1.Process("data1");
processor2.Process("data2");  // Works silently
```

---

## ขั้นตอนที่ 186-190: โปรแกรมตัวอย่าง - Fluent Validation Library

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text.RegularExpressions;

namespace FluentValidation
{
    // Validation Result
    record ValidationError(string Field, string Message, string Code = "INVALID");
    
    class ValidationResult
    {
        private readonly List<ValidationError> _errors = new();
        
        public bool IsValid => !_errors.Any();
        public IReadOnlyList<ValidationError> Errors => _errors.AsReadOnly();
        
        public void AddError(string field, string message, string code = "INVALID")
            => _errors.Add(new ValidationError(field, message, code));
        
        public void Merge(ValidationResult other)
            => _errors.AddRange(other.Errors);
        
        public override string ToString()
            => IsValid ? "Valid" : string.Join("\n", _errors.Select(e => $"  {e.Field}: {e.Message}"));
    }
    
    // Fluent Validator
    class Validator<T>
    {
        private readonly T _target;
        private readonly ValidationResult _result = new();
        private string _currentField = "";
        private object? _currentValue;
        
        public Validator(T target) => _target = target;
        
        public Validator<T> RuleFor(string fieldName, Func<T, object?> selector)
        {
            _currentField = fieldName;
            _currentValue = selector(_target);
            return this;
        }
        
        // String rules
        public Validator<T> NotEmpty()
        {
            if (_currentValue is string s && string.IsNullOrWhiteSpace(s))
                _result.AddError(_currentField, "Field cannot be empty", "REQUIRED");
            return this;
        }
        
        public Validator<T> MinLength(int min)
        {
            if (_currentValue is string s && s.Length < min)
                _result.AddError(_currentField, $"Minimum length is {min}", "MIN_LENGTH");
            return this;
        }
        
        public Validator<T> MaxLength(int max)
        {
            if (_currentValue is string s && s.Length > max)
                _result.AddError(_currentField, $"Maximum length is {max}", "MAX_LENGTH");
            return this;
        }
        
        public Validator<T> Matches(string pattern, string errorMessage)
        {
            if (_currentValue is string s && !Regex.IsMatch(s, pattern))
                _result.AddError(_currentField, errorMessage, "PATTERN_MISMATCH");
            return this;
        }
        
        public Validator<T> IsEmail()
            => Matches(@"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$", "Invalid email format");
        
        public Validator<T> IsPhoneNumber()
            => Matches(@"^0[689]\d{8}$", "Invalid Thai phone number");
        
        // Number rules
        public Validator<T> GreaterThan<TNum>(TNum min) where TNum : IComparable<TNum>
        {
            if (_currentValue is TNum value && value.CompareTo(min) <= 0)
                _result.AddError(_currentField, $"Must be greater than {min}", "MIN_VALUE");
            return this;
        }
        
        public Validator<T> LessThan<TNum>(TNum max) where TNum : IComparable<TNum>
        {
            if (_currentValue is TNum value && value.CompareTo(max) >= 0)
                _result.AddError(_currentField, $"Must be less than {max}", "MAX_VALUE");
            return this;
        }
        
        public Validator<T> Between<TNum>(TNum min, TNum max) where TNum : IComparable<TNum>
        {
            if (_currentValue is TNum value && (value.CompareTo(min) < 0 || value.CompareTo(max) > 0))
                _result.AddError(_currentField, $"Must be between {min} and {max}", "OUT_OF_RANGE");
            return this;
        }
        
        // Custom rules
        public Validator<T> Must(Func<T, bool> predicate, string errorMessage)
        {
            if (!predicate(_target))
                _result.AddError(_currentField, errorMessage, "CUSTOM");
            return this;
        }
        
        public Validator<T> NotNull()
        {
            if (_currentValue is null)
                _result.AddError(_currentField, "Field cannot be null", "NULL");
            return this;
        }
        
        public ValidationResult Validate() => _result;
    }
    
    // Extension method to create validator
    static class ValidatorExtensions
    {
        public static Validator<T> Validate<T>(this T target) => new Validator<T>(target);
    }
    
    // Models
    record RegisterRequest(
        string Username,
        string Email,
        string Password,
        string ConfirmPassword,
        int Age,
        string Phone
    );
    
    record ProductRequest(
        string Name,
        string? Description,
        decimal Price,
        int Stock,
        string Category
    );
    
    class Program
    {
        static void Main()
        {
            Console.OutputEncoding = System.Text.Encoding.UTF8;
            Console.Title = "Fluent Validation Library";
            
            // Test Register Validation
            Console.WriteLine("=== Register Validation ===\n");
            
            var validRequest = new RegisterRequest(
                "alice_smith",
                "alice@example.com",
                "Secure@Pass123",
                "Secure@Pass123",
                25,
                "0812345678"
            );
            
            var invalidRequest = new RegisterRequest(
                "ab",
                "not-an-email",
                "short",
                "different",
                15,
                "1234567"
            );
            
            void ValidateRegister(RegisterRequest req)
            {
                var result = req.Validate()
                    .RuleFor("Username", r => r.Username)
                        .NotEmpty()
                        .MinLength(3)
                        .MaxLength(20)
                        .Matches(@"^[a-zA-Z0-9_]+$", "Only letters, numbers, underscores")
                    .RuleFor("Email", r => r.Email)
                        .NotEmpty()
                        .IsEmail()
                    .RuleFor("Password", r => r.Password)
                        .NotEmpty()
                        .MinLength(8)
                        .Matches(@"[A-Z]", "Must contain uppercase")
                        .Matches(@"[0-9]", "Must contain digit")
                    .RuleFor("ConfirmPassword", r => r.ConfirmPassword)
                        .Must(r => r.Password == r.ConfirmPassword, "Passwords must match")
                    .RuleFor("Age", r => r.Age)
                        .Between(18, 120)
                    .RuleFor("Phone", r => r.Phone)
                        .IsPhoneNumber()
                    .Validate();
                
                if (result.IsValid)
                {
                    Console.ForegroundColor = ConsoleColor.Green;
                    Console.WriteLine("✅ Validation passed!");
                    Console.ResetColor();
                }
                else
                {
                    Console.ForegroundColor = ConsoleColor.Red;
                    Console.WriteLine("❌ Validation failed:");
                    Console.ResetColor();
                    Console.WriteLine(result.ToString());
                }
            }
            
            Console.WriteLine("Test valid request:");
            ValidateRegister(validRequest);
            
            Console.WriteLine("\nTest invalid request:");
            ValidateRegister(invalidRequest);
            
            // Test Product Validation
            Console.WriteLine("\n=== Product Validation ===\n");
            
            var validProduct = new ProductRequest("Laptop", "High-end laptop", 45000m, 10, "Electronics");
            var invalidProduct = new ProductRequest("", null, -100m, -5, "");
            
            void ValidateProduct(ProductRequest prod)
            {
                var result = prod.Validate()
                    .RuleFor("Name", p => p.Name)
                        .NotEmpty()
                        .MinLength(2)
                        .MaxLength(100)
                    .RuleFor("Price", p => p.Price)
                        .GreaterThan(0m)
                        .LessThan(10_000_000m)
                    .RuleFor("Stock", p => p.Stock)
                        .GreaterThan(-1)
                    .RuleFor("Category", p => p.Category)
                        .NotEmpty()
                    .Validate();
                
                Console.WriteLine(result.IsValid ? "✅ Valid" : $"❌ Invalid:\n{result}");
            }
            
            Console.Write("Valid product: ");
            ValidateProduct(validProduct);
            
            Console.Write("\nInvalid product: ");
            ValidateProduct(invalidProduct);
            
            Console.WriteLine("\nกด Enter เพื่อออก...");
            Console.ReadLine();
        }
    }
}
```

---

## 📝 สรุป Part 19

| หัวข้อ | Key Points |
|--------|-----------|
| Extension Methods | `this Type name` ใน static class |
| Fluent API | Method chaining pattern |
| Nullable `?` | เปิด nullable context ป้องกัน NullRef |
| `?.` | Null-conditional operator |
| `??` | Null-coalescing (default value) |
| `??=` | Null-coalescing assignment |
| `!` | Null-forgiving operator |
| Null Object | Pattern แทน null reference |

---

**ก่อนหน้า → [Part 18: Tuples & Records](part18-tuples-records.md)**  
**ต่อไป → [Part 20: Attributes & Reflection](part20-attributes-reflection.md)**
