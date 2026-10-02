# Part 09: Interfaces & Abstract Classes
## ขั้นตอนที่ 81-90: Interfaces และการออกแบบ Contracts

---

## 🎯 เป้าหมายของ Part นี้
- เข้าใจ Interfaces และความแตกต่างจาก Abstract Classes
- Implement หลาย Interfaces พร้อมกัน
- รู้จัก Default Interface Methods (C# 8+)
- ใช้ Interfaces เพื่อ Dependency Injection
- ออกแบบโปรแกรมโดยใช้ Interface Segregation
- รู้จัก Common .NET Interfaces

---

## ขั้นตอนที่ 81: Interface พื้นฐาน

### Interface คืออะไร?
```csharp
// Interface = Contract (สัญญา) ที่ class ต้องปฏิบัติตาม
// ไม่มี state (fields)
// ทุก member เป็น public โดยอัตโนมัติ

interface IDrawable
{
    void Draw();                          // Method
    string Color { get; set; }           // Property
    event EventHandler? OnDrawn;          // Event
}

interface IResizable
{
    void Resize(double factor);
    double Width { get; }
    double Height { get; }
}

// Class implement interface
class Rectangle : IDrawable, IResizable
{
    private double _width;
    private double _height;
    
    public string Color { get; set; } = "Black";
    public event EventHandler? OnDrawn;
    
    public double Width => _width;
    public double Height => _height;
    
    public Rectangle(double width, double height)
    {
        _width = width;
        _height = height;
    }
    
    public void Draw()
    {
        Console.WriteLine($"Drawing {Color} rectangle ({_width}x{_height})");
        OnDrawn?.Invoke(this, EventArgs.Empty);
    }
    
    public void Resize(double factor)
    {
        _width *= factor;
        _height *= factor;
    }
}

// ใช้งานผ่าน interface type (polymorphism)
IDrawable drawable = new Rectangle(10, 5);
drawable.Draw();
drawable.Color = "Red";
drawable.Draw();

// ตรวจสอบ interface
if (drawable is IResizable resizable)
{
    resizable.Resize(2.0);
    Console.WriteLine($"After resize: {resizable.Width}x{resizable.Height}");
}
```

### Interface vs Abstract Class
```csharp
/*
Interface:
✅ Multiple implementation (class สามารถ implement หลาย interface)
✅ No state (no fields)
✅ Contract only (ก่อน C# 8)
✅ Default implementation (C# 8+)
❌ No constructor
❌ No access modifiers (all public)

Abstract Class:
✅ Single inheritance only
✅ Can have state (fields)
✅ Can have constructor
✅ Mix of abstract and concrete methods
✅ Access modifiers allowed
❌ Cannot implement multiple abstract classes
*/

// เมื่อไหร่ใช้อะไร:
// Interface: เมื่อต้องการ contract ที่ไม่เกี่ยวข้องกับ hierarchy
//   - IComparable, IDisposable, IEnumerable
//   - IPaymentProcessor, ILogger, IRepository

// Abstract Class: เมื่อมี "is-a" relationship และ shared behavior
//   - Animal (Dog is Animal)
//   - Shape (Circle is Shape)
//   - Employee (Manager is Employee)
```

---

## ขั้นตอนที่ 82: Common .NET Interfaces

### IComparable<T> และ IComparer<T>
```csharp
class Product : IComparable<Product>
{
    public string Name { get; set; }
    public decimal Price { get; set; }
    public int Rating { get; set; }
    
    public Product(string name, decimal price, int rating)
    {
        Name = name;
        Price = price;
        Rating = rating;
    }
    
    // IComparable<T> - default comparison
    public int CompareTo(Product? other)
    {
        if (other == null) return 1;
        return Price.CompareTo(other.Price);  // Sort by price by default
    }
    
    public override string ToString() => $"{Name} ({Price:N2} บาท, ⭐{Rating})";
}

// Custom Comparer
class ProductByRatingComparer : IComparer<Product>
{
    public int Compare(Product? x, Product? y)
    {
        if (x == null && y == null) return 0;
        if (x == null) return -1;
        if (y == null) return 1;
        
        // Sort by rating desc, then by name asc
        int ratingComp = y.Rating.CompareTo(x.Rating);
        return ratingComp != 0 ? ratingComp : 
            string.Compare(x.Name, y.Name, StringComparison.Ordinal);
    }
}

// การใช้งาน
var products = new List<Product>
{
    new Product("Laptop", 35000m, 4),
    new Product("Phone", 15000m, 5),
    new Product("Tablet", 20000m, 4),
    new Product("Watch", 8000m, 3)
};

products.Sort();  // ใช้ IComparable (by price)
Console.WriteLine("เรียงตามราคา:");
products.ForEach(p => Console.WriteLine($"  {p}"));

products.Sort(new ProductByRatingComparer());  // ใช้ IComparer
Console.WriteLine("\nเรียงตาม Rating:");
products.ForEach(p => Console.WriteLine($"  {p}"));
```

### IEquatable<T>
```csharp
class Point : IEquatable<Point>
{
    public double X { get; }
    public double Y { get; }
    
    public Point(double x, double y) => (X, Y) = (x, y);
    
    public bool Equals(Point? other)
    {
        if (other is null) return false;
        return Math.Abs(X - other.X) < 1e-10 && Math.Abs(Y - other.Y) < 1e-10;
    }
    
    public override bool Equals(object? obj) => Equals(obj as Point);
    
    public override int GetHashCode() => HashCode.Combine(X, Y);
    
    public static bool operator ==(Point? a, Point? b)
    {
        if (a is null && b is null) return true;
        if (a is null || b is null) return false;
        return a.Equals(b);
    }
    
    public static bool operator !=(Point? a, Point? b) => !(a == b);
    
    public double DistanceTo(Point other)
    {
        double dx = X - other.X;
        double dy = Y - other.Y;
        return Math.Sqrt(dx * dx + dy * dy);
    }
    
    public override string ToString() => $"({X:F2}, {Y:F2})";
}
```

### IDisposable - Resource Management
```csharp
// IDisposable - สำหรับ cleanup resources
class DatabaseConnection : IDisposable
{
    private bool _disposed = false;
    private bool _isConnected = false;
    
    public string ConnectionString { get; }
    
    public DatabaseConnection(string connectionString)
    {
        ConnectionString = connectionString;
        Connect();
    }
    
    private void Connect()
    {
        // จำลองการเชื่อมต่อ
        _isConnected = true;
        Console.WriteLine($"เชื่อมต่อ: {ConnectionString}");
    }
    
    public void ExecuteQuery(string sql)
    {
        if (_disposed) throw new ObjectDisposedException(nameof(DatabaseConnection));
        if (!_isConnected) throw new InvalidOperationException("Not connected");
        
        Console.WriteLine($"Execute: {sql}");
    }
    
    // IDisposable implementation
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);  // บอก GC ว่าทำ cleanup แล้ว
    }
    
    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
            {
                // Cleanup managed resources
                if (_isConnected)
                {
                    _isConnected = false;
                    Console.WriteLine("ปิดการเชื่อมต่อ");
                }
            }
            
            // Cleanup unmanaged resources (ถ้ามี)
            
            _disposed = true;
        }
    }
    
    // Finalizer - ถ้าลืม Dispose()
    ~DatabaseConnection()
    {
        Dispose(false);
    }
}

// การใช้งานที่ถูกต้อง - ใช้ using statement
using (var conn = new DatabaseConnection("Server=localhost;DB=test"))
{
    conn.ExecuteQuery("SELECT * FROM Users");
    conn.ExecuteQuery("SELECT * FROM Products");
}
// Dispose() ถูกเรียกอัตโนมัติ ไม่ว่าจะมี exception หรือไม่

// ใช้ using declaration (C# 8+)
using var conn2 = new DatabaseConnection("Server=localhost;DB=test");
conn2.ExecuteQuery("SELECT * FROM Orders");
// Dispose() ถูกเรียกตอนออกจาก scope
```

---

## ขั้นตอนที่ 83: IEnumerable<T> และ Custom Iterators

### IEnumerable<T>
```csharp
// IEnumerable<T> = สามารถ iterate ได้ด้วย foreach
// IEnumerator<T> = state machine สำหรับ iteration

class NumberSequence : IEnumerable<int>
{
    private readonly int _start;
    private readonly int _end;
    private readonly Func<int, bool>? _filter;
    
    public NumberSequence(int start, int end, Func<int, bool>? filter = null)
    {
        _start = start;
        _end = end;
        _filter = filter;
    }
    
    // IEnumerable<int>
    public IEnumerator<int> GetEnumerator()
    {
        for (int i = _start; i <= _end; i++)
        {
            if (_filter == null || _filter(i))
                yield return i;
        }
    }
    
    // Non-generic IEnumerable
    System.Collections.IEnumerator System.Collections.IEnumerable.GetEnumerator()
        => GetEnumerator();
}

// การใช้งาน
var evens = new NumberSequence(1, 20, n => n % 2 == 0);
Console.Write("คู่: ");
foreach (int n in evens)
    Console.Write($"{n} ");
// คู่: 2 4 6 8 10 12 14 16 18 20

var primes = new NumberSequence(2, 50, IsPrime);
Console.Write("\nจำนวนเฉพาะ: ");
foreach (int p in primes)
    Console.Write($"{p} ");

bool IsPrime(int n)
{
    for (int i = 2; i <= Math.Sqrt(n); i++)
        if (n % i == 0) return false;
    return true;
}

// LINQ ใช้ได้กับ IEnumerable<T>
int sumOfPrimes = primes.Sum();
Console.WriteLine($"\nผลรวมจำนวนเฉพาะ: {sumOfPrimes}");
```

---

## ขั้นตอนที่ 84: Default Interface Methods (C# 8+)

### Default Implementation ใน Interface
```csharp
interface ILogger
{
    void Log(string message);
    void LogError(string message) => Log($"[ERROR] {message}");   // Default implementation
    void LogWarning(string message) => Log($"[WARNING] {message}");
    void LogInfo(string message) => Log($"[INFO] {message}");
    
    // Static members in interface (C# 8+)
    static string FormatMessage(string level, string message)
        => $"[{DateTime.Now:HH:mm:ss}] [{level}] {message}";
}

class ConsoleLogger : ILogger
{
    private ConsoleColor _defaultColor = Console.ForegroundColor;
    
    // Must implement only Log()
    public void Log(string message)
    {
        Console.WriteLine(message);
    }
    
    // Can override default methods
    public void LogError(string message)
    {
        Console.ForegroundColor = ConsoleColor.Red;
        Log($"[ERROR] {message}");
        Console.ForegroundColor = _defaultColor;
    }
}

class FileLogger : ILogger
{
    private readonly string _filePath;
    
    public FileLogger(string filePath)
    {
        _filePath = filePath;
    }
    
    public void Log(string message)
    {
        // จำลองเขียนไฟล์
        Console.WriteLine($"[FILE:{_filePath}] {message}");
    }
    // ใช้ default implementations สำหรับ LogError, LogWarning, LogInfo
}

// การใช้งาน
ILogger logger = new ConsoleLogger();
logger.Log("Normal message");
logger.LogError("Something went wrong!");
logger.LogWarning("Be careful");
logger.LogInfo("Application started");
```

---

## ขั้นตอนที่ 85: Interface Segregation

### Interface Segregation Principle (ISP)
```csharp
// ❌ Bad: Fat Interface - บังคับ implement ทุกอย่าง
interface IEmployee
{
    void Work();
    void ManageTeam();       // ไม่ใช่ทุกคนที่ manage team
    void ApproveLeave();     // เฉพาะ manager
    void WriteCode();        // เฉพาะ developer
    void DesignSystem();     // เฉพาะ architect
    void HandleCustomer();   // เฉพาะ support
}

// ✅ Good: Segregated Interfaces
interface IWorker
{
    void Work();
    string GetWorkReport();
}

interface IManager
{
    void ManageTeam();
    void ApproveLeave(Employee employee);
    IReadOnlyList<Employee> GetTeam();
}

interface IDeveloper
{
    void WriteCode(string feature);
    void ReviewCode(string code);
    string RunTests();
}

interface ISystemArchitect
{
    void DesignSystem(string requirements);
    string CreateArchitectureDocument();
}

// Classes implement only what they need
class Developer : IWorker, IDeveloper
{
    public string Name { get; set; }
    public List<string> Skills { get; set; } = new();
    
    public Developer(string name) => Name = name;
    
    public void Work() => Console.WriteLine($"{Name} is developing...");
    
    public string GetWorkReport() => $"{Name} completed 5 tasks";
    
    public void WriteCode(string feature) 
        => Console.WriteLine($"{Name} writing code for: {feature}");
    
    public void ReviewCode(string code)
        => Console.WriteLine($"{Name} reviewing: {code[..Math.Min(50, code.Length)]}...");
    
    public string RunTests() => "All 42 tests passed";
}

class TechLead : Developer, IManager, ISystemArchitect
{
    private List<Employee> _team = new();
    
    public TechLead(string name) : base(name) { }
    
    public void ManageTeam() => Console.WriteLine($"{Name} is managing team");
    
    public void ApproveLeave(Employee employee)
        => Console.WriteLine($"{Name} approved leave for {employee.Name}");
    
    public IReadOnlyList<Employee> GetTeam() => _team.AsReadOnly();
    
    public void DesignSystem(string requirements)
        => Console.WriteLine($"{Name} designing: {requirements}");
    
    public string CreateArchitectureDocument() => "Architecture v1.0";
    
    public void AddTeamMember(Employee emp) => _team.Add(emp);
}

class Employee { public string Name { get; set; } = ""; }
```

---

## ขั้นตอนที่ 86: Explicit Interface Implementation

### Explicit Implementation
```csharp
// เมื่อ 2 interfaces มี method ชื่อเดียวกัน
interface IAnimal
{
    void MakeSound();
    string GetName();
}

interface IVehicle
{
    void MakeSound();  // engine sound
    string GetName();
}

// Class implement ทั้งสอง - ต้องใช้ explicit implementation
class RobotPet : IAnimal, IVehicle
{
    public string Name { get; }
    
    public RobotPet(string name) => Name = name;
    
    // Explicit - เฉพาะเมื่อเรียกผ่าน IAnimal
    void IAnimal.MakeSound() => Console.WriteLine($"{Name} goes: Woof! (animal)");
    string IAnimal.GetName() => $"Animal: {Name}";
    
    // Explicit - เฉพาะเมื่อเรียกผ่าน IVehicle
    void IVehicle.MakeSound() => Console.WriteLine($"{Name} goes: VROOM! (vehicle)");
    string IVehicle.GetName() => $"Vehicle: {Name}";
    
    // Public method (accessible directly)
    public void Introduce() => Console.WriteLine($"I am {Name}, a robot pet!");
}

// การใช้งาน
var robo = new RobotPet("R2D2");

robo.Introduce();
// robo.MakeSound();    ← Error! Ambiguous

IAnimal animal = robo;
animal.MakeSound();    // Woof! (animal)
Console.WriteLine(animal.GetName());

IVehicle vehicle = robo;
vehicle.MakeSound();   // VROOM! (vehicle)
Console.WriteLine(vehicle.GetName());
```

---

## ขั้นตอนที่ 87: Generic Interfaces

### Generic Interfaces
```csharp
// Generic interface
interface IRepository<T, TKey> where T : class where TKey : notnull
{
    T? GetById(TKey id);
    IEnumerable<T> GetAll();
    T Add(T entity);
    T Update(T entity);
    bool Delete(TKey id);
    int Count();
}

// Generic interface สำหรับ filtering
interface IFilterable<T>
{
    IEnumerable<T> Filter(Func<T, bool> predicate);
    IEnumerable<T> OrderBy<TKey>(Func<T, TKey> keySelector);
}

// Implementation
class InMemoryRepository<T> : IRepository<T, int>, IFilterable<T>
    where T : class
{
    private Dictionary<int, T> _storage = new();
    private int _nextId = 1;
    private readonly Func<T, int> _getIdFunc;
    private readonly Action<T, int> _setIdFunc;
    
    public InMemoryRepository(Func<T, int> getId, Action<T, int> setId)
    {
        _getIdFunc = getId;
        _setIdFunc = setId;
    }
    
    public T? GetById(int id) => _storage.GetValueOrDefault(id);
    
    public IEnumerable<T> GetAll() => _storage.Values.ToList();
    
    public T Add(T entity)
    {
        int id = _nextId++;
        _setIdFunc(entity, id);
        _storage[id] = entity;
        return entity;
    }
    
    public T Update(T entity)
    {
        int id = _getIdFunc(entity);
        if (!_storage.ContainsKey(id))
            throw new KeyNotFoundException($"Entity with id {id} not found");
        _storage[id] = entity;
        return entity;
    }
    
    public bool Delete(int id)
    {
        return _storage.Remove(id);
    }
    
    public int Count() => _storage.Count;
    
    public IEnumerable<T> Filter(Func<T, bool> predicate)
        => _storage.Values.Where(predicate).ToList();
    
    public IEnumerable<T> OrderBy<TKey>(Func<T, TKey> keySelector)
        => _storage.Values.OrderBy(keySelector).ToList();
}
```

---

## ขั้นตอนที่ 88: Interfaces กับ Dependency Injection

### Dependency Injection ด้วย Interfaces
```csharp
// Interfaces for dependencies
interface IEmailService
{
    Task SendAsync(string to, string subject, string body);
    bool IsAvailable { get; }
}

interface ISmsService
{
    Task SendAsync(string phone, string message);
}

interface INotificationService
{
    Task NotifyAsync(string userId, string message, NotificationType type);
}

enum NotificationType { Email, SMS, Both }

// Implementations
class SmtpEmailService : IEmailService
{
    private readonly string _smtpServer;
    
    public SmtpEmailService(string smtpServer)
    {
        _smtpServer = smtpServer;
    }
    
    public bool IsAvailable => true;
    
    public async Task SendAsync(string to, string subject, string body)
    {
        await Task.Delay(100);  // จำลองการส่ง
        Console.WriteLine($"[SMTP:{_smtpServer}] Email to {to}: {subject}");
    }
}

class MockEmailService : IEmailService  // สำหรับ testing
{
    public List<(string to, string subject, string body)> SentEmails { get; } = new();
    public bool IsAvailable => true;
    
    public async Task SendAsync(string to, string subject, string body)
    {
        SentEmails.Add((to, subject, body));
        Console.WriteLine($"[MOCK EMAIL] to: {to}, subject: {subject}");
    }
}

class TwilioSmsService : ISmsService
{
    public async Task SendAsync(string phone, string message)
    {
        await Task.Delay(50);
        Console.WriteLine($"[Twilio SMS] to {phone}: {message}");
    }
}

// Composite notification service - depends on interfaces, not concrete classes
class NotificationService : INotificationService
{
    private readonly IEmailService _emailService;
    private readonly ISmsService _smsService;
    private readonly Dictionary<string, (string email, string phone)> _users;
    
    // Constructor Injection
    public NotificationService(IEmailService emailService, ISmsService smsService)
    {
        _emailService = emailService;
        _smsService = smsService;
        
        // จำลอง user data
        _users = new()
        {
            { "U001", ("alice@example.com", "081-111-1111") },
            { "U002", ("bob@example.com", "082-222-2222") }
        };
    }
    
    public async Task NotifyAsync(string userId, string message, NotificationType type)
    {
        if (!_users.TryGetValue(userId, out var userInfo))
        {
            Console.WriteLine($"User {userId} not found");
            return;
        }
        
        var tasks = new List<Task>();
        
        if (type is NotificationType.Email or NotificationType.Both)
        {
            if (_emailService.IsAvailable)
                tasks.Add(_emailService.SendAsync(userInfo.email, "Notification", message));
        }
        
        if (type is NotificationType.SMS or NotificationType.Both)
        {
            tasks.Add(_smsService.SendAsync(userInfo.phone, message));
        }
        
        await Task.WhenAll(tasks);
    }
}

// การใช้งาน
async Task RunNotificationExample()
{
    // Production
    var emailService = new SmtpEmailService("smtp.example.com");
    var smsService = new TwilioSmsService();
    var notifier = new NotificationService(emailService, smsService);
    
    await notifier.NotifyAsync("U001", "Your order has been shipped!", NotificationType.Both);
    
    // Testing - inject mock
    var mockEmail = new MockEmailService();
    var testNotifier = new NotificationService(mockEmail, smsService);
    
    await testNotifier.NotifyAsync("U002", "Test notification", NotificationType.Email);
    Console.WriteLine($"Emails sent in test: {mockEmail.SentEmails.Count}");
}

await RunNotificationExample();
```

---

## ขั้นตอนที่ 89: Interface Hierarchies

### Interface Hierarchies
```csharp
// Base interfaces
interface IReadable
{
    string Read();
}

interface IWritable
{
    void Write(string data);
}

// Combined interface
interface IReadWritable : IReadable, IWritable
{
    void Clear();
}

// More specific
interface IStreamable : IReadWritable
{
    long Position { get; set; }
    long Length { get; }
    void Seek(long position);
}

// Implementation
class MemoryBuffer : IStreamable
{
    private List<string> _data = new();
    private long _position = 0;
    
    public long Position
    {
        get => _position;
        set => _position = Math.Clamp(value, 0, Length);
    }
    
    public long Length => _data.Count;
    
    public string Read()
    {
        if (_position >= _data.Count) return "";
        return _data[(int)_position++];
    }
    
    public void Write(string data)
    {
        _data.Add(data);
    }
    
    public void Clear()
    {
        _data.Clear();
        _position = 0;
    }
    
    public void Seek(long position)
    {
        Position = position;
    }
}

// ใช้งานผ่าน interface levels
MemoryBuffer buffer = new();
buffer.Write("Line 1");
buffer.Write("Line 2");
buffer.Write("Line 3");

IReadable readable = buffer;
buffer.Seek(0);
Console.WriteLine(readable.Read());  // Line 1

IStreamable stream = buffer;
stream.Seek(2);
Console.WriteLine(stream.Read());    // Line 3
```

---

## ขั้นตอนที่ 90: โปรแกรมตัวอย่าง - Plugin System

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

namespace PluginSystem
{
    // Plugin interfaces
    interface IPlugin
    {
        string Name { get; }
        string Version { get; }
        string Description { get; }
        
        Task InitializeAsync();
        Task ShutdownAsync();
    }
    
    interface IDataTransformPlugin : IPlugin
    {
        string Transform(string input);
        bool CanTransform(string input);
    }
    
    interface IValidationPlugin : IPlugin
    {
        (bool isValid, string error) Validate(string data);
    }
    
    interface IStoragePlugin : IPlugin
    {
        Task<bool> SaveAsync(string key, string data);
        Task<string?> LoadAsync(string key);
        Task<bool> DeleteAsync(string key);
        Task<IEnumerable<string>> GetKeysAsync();
    }
    
    // Abstract base plugin
    abstract class BasePlugin : IPlugin
    {
        public abstract string Name { get; }
        public abstract string Version { get; }
        public virtual string Description => $"{Name} v{Version}";
        
        public virtual async Task InitializeAsync()
        {
            Console.WriteLine($"Initializing plugin: {Name}");
            await OnInitializeAsync();
        }
        
        public virtual async Task ShutdownAsync()
        {
            Console.WriteLine($"Shutting down plugin: {Name}");
            await OnShutdownAsync();
        }
        
        protected virtual Task OnInitializeAsync() => Task.CompletedTask;
        protected virtual Task OnShutdownAsync() => Task.CompletedTask;
    }
    
    // Concrete plugins
    class UpperCasePlugin : BasePlugin, IDataTransformPlugin
    {
        public override string Name => "UpperCase Transform";
        public override string Version => "1.0.0";
        
        public bool CanTransform(string input) => !string.IsNullOrEmpty(input);
        
        public string Transform(string input) => input.ToUpper();
    }
    
    class TrimPlugin : BasePlugin, IDataTransformPlugin
    {
        public override string Name => "Trim Whitespace";
        public override string Version => "1.0.0";
        
        public bool CanTransform(string input) => input?.Contains(' ') == true;
        
        public string Transform(string input) => input.Trim();
    }
    
    class EmailValidationPlugin : BasePlugin, IValidationPlugin
    {
        public override string Name => "Email Validator";
        public override string Version => "2.0.0";
        
        public (bool isValid, string error) Validate(string data)
        {
            if (string.IsNullOrWhiteSpace(data))
                return (false, "Email cannot be empty");
            
            if (!data.Contains('@'))
                return (false, "Email must contain @");
            
            string[] parts = data.Split('@');
            if (parts.Length != 2 || string.IsNullOrEmpty(parts[0]) || !parts[1].Contains('.'))
                return (false, "Invalid email format");
            
            return (true, "");
        }
    }
    
    class LengthValidationPlugin : BasePlugin, IValidationPlugin
    {
        private readonly int _minLength;
        private readonly int _maxLength;
        
        public LengthValidationPlugin(int min = 3, int max = 100)
        {
            _minLength = min;
            _maxLength = max;
        }
        
        public override string Name => "Length Validator";
        public override string Version => "1.0.0";
        public override string Description => $"Validates length between {_minLength}-{_maxLength}";
        
        public (bool isValid, string error) Validate(string data)
        {
            if (data.Length < _minLength)
                return (false, $"Too short (min {_minLength})");
            if (data.Length > _maxLength)
                return (false, $"Too long (max {_maxLength})");
            return (true, "");
        }
    }
    
    class InMemoryStoragePlugin : BasePlugin, IStoragePlugin
    {
        private Dictionary<string, string> _storage = new();
        
        public override string Name => "In-Memory Storage";
        public override string Version => "1.0.0";
        
        public async Task<bool> SaveAsync(string key, string data)
        {
            await Task.Delay(1);  // จำลอง async
            _storage[key] = data;
            return true;
        }
        
        public async Task<string?> LoadAsync(string key)
        {
            await Task.Delay(1);
            return _storage.GetValueOrDefault(key);
        }
        
        public async Task<bool> DeleteAsync(string key)
        {
            await Task.Delay(1);
            return _storage.Remove(key);
        }
        
        public async Task<IEnumerable<string>> GetKeysAsync()
        {
            await Task.Delay(1);
            return _storage.Keys.ToList();
        }
    }
    
    // Plugin Manager
    class PluginManager
    {
        private List<IPlugin> _plugins = new();
        
        public void Register(IPlugin plugin)
        {
            _plugins.Add(plugin);
            Console.WriteLine($"✅ Registered: {plugin.Name}");
        }
        
        public async Task InitializeAllAsync()
        {
            Console.WriteLine("\n=== Initializing Plugins ===");
            var tasks = _plugins.Select(p => p.InitializeAsync());
            await Task.WhenAll(tasks);
        }
        
        public async Task ShutdownAllAsync()
        {
            Console.WriteLine("\n=== Shutting Down Plugins ===");
            var tasks = _plugins.Select(p => p.ShutdownAsync());
            await Task.WhenAll(tasks);
        }
        
        public IEnumerable<IDataTransformPlugin> GetTransformPlugins()
            => _plugins.OfType<IDataTransformPlugin>();
        
        public IEnumerable<IValidationPlugin> GetValidationPlugins()
            => _plugins.OfType<IValidationPlugin>();
        
        public IStoragePlugin? GetStoragePlugin()
            => _plugins.OfType<IStoragePlugin>().FirstOrDefault();
        
        // Pipeline: Transform then Validate then Store
        public async Task<(bool success, string result, IEnumerable<string> errors)> 
            ProcessAsync(string data, string? storageKey = null)
        {
            var errors = new List<string>();
            string processed = data;
            
            // Transform
            foreach (var transformer in GetTransformPlugins())
            {
                if (transformer.CanTransform(processed))
                {
                    processed = transformer.Transform(processed);
                    Console.WriteLine($"[Transform] {transformer.Name}: '{data}' → '{processed}'");
                }
            }
            
            // Validate
            foreach (var validator in GetValidationPlugins())
            {
                var (isValid, error) = validator.Validate(processed);
                if (!isValid)
                {
                    errors.Add($"{validator.Name}: {error}");
                }
            }
            
            if (errors.Any()) return (false, processed, errors);
            
            // Store (if storage key provided)
            if (storageKey != null)
            {
                var storage = GetStoragePlugin();
                if (storage != null)
                {
                    await storage.SaveAsync(storageKey, processed);
                    Console.WriteLine($"[Storage] Saved to key: {storageKey}");
                }
            }
            
            return (true, processed, errors);
        }
        
        public void ShowPluginInfo()
        {
            Console.WriteLine("\n=== Registered Plugins ===");
            foreach (var plugin in _plugins)
            {
                string type = plugin switch
                {
                    IDataTransformPlugin => "Transform",
                    IValidationPlugin => "Validation",
                    IStoragePlugin => "Storage",
                    _ => "General"
                };
                Console.WriteLine($"  [{type}] {plugin.Name} v{plugin.Version}");
                Console.WriteLine($"    {plugin.Description}");
            }
        }
    }
    
    class Program
    {
        static async Task Main(string[] args)
        {
            Console.OutputEncoding = System.Text.Encoding.UTF8;
            Console.Title = "Plugin System Demo";
            
            var manager = new PluginManager();
            
            // Register plugins
            manager.Register(new TrimPlugin());
            manager.Register(new UpperCasePlugin());
            manager.Register(new LengthValidationPlugin(3, 50));
            manager.Register(new EmailValidationPlugin());
            manager.Register(new InMemoryStoragePlugin());
            
            // Initialize
            await manager.InitializeAllAsync();
            
            // Show info
            manager.ShowPluginInfo();
            
            // Test pipeline
            Console.WriteLine("\n=== Testing Pipeline ===");
            
            var testData = new[]
            {
                "  hello world  ",
                "  valid@email.com  ",
                "ab",
                "  no-at-sign  ",
                "  test@example.org  "
            };
            
            foreach (var data in testData)
            {
                Console.WriteLine($"\nInput: '{data}'");
                
                var (success, result, errors) = await manager.ProcessAsync(
                    data, 
                    success: true ? $"key_{data.Trim()[..Math.Min(10, data.Trim().Length)]}" : null
                );
                
                if (success)
                {
                    Console.ForegroundColor = ConsoleColor.Green;
                    Console.WriteLine($"✅ Success! Result: '{result}'");
                }
                else
                {
                    Console.ForegroundColor = ConsoleColor.Red;
                    Console.WriteLine($"❌ Failed!");
                    foreach (var error in errors)
                        Console.WriteLine($"  - {error}");
                }
                Console.ResetColor();
            }
            
            // Show stored data
            var storage = manager.GetStoragePlugin();
            if (storage != null)
            {
                Console.WriteLine("\n=== Stored Data ===");
                var keys = await storage.GetKeysAsync();
                foreach (var key in keys)
                {
                    var value = await storage.LoadAsync(key);
                    Console.WriteLine($"  {key}: {value}");
                }
            }
            
            // Shutdown
            await manager.ShutdownAllAsync();
            
            Console.WriteLine("\nกด Enter เพื่อออก...");
            Console.ReadLine();
        }
    }
}
```

---

## 📝 สรุป Part 09

| หัวข้อ | รายละเอียดสำคัญ |
|--------|----------------|
| Interface | Contract ที่ class ต้อง implement |
| Multiple Interface | Class สามารถ implement หลาย interfaces |
| Default Methods | C# 8+ - default implementation ใน interface |
| IComparable<T> | Custom sorting |
| IDisposable | Resource cleanup ด้วย using |
| IEnumerable<T> | Custom iteration |
| Explicit Implementation | แก้ปัญหา name conflict |
| Generic Interfaces | Interface ที่ parameterized ด้วย type |
| Dependency Injection | Inject interface แทน concrete class |
| Interface Segregation | แยก interface ตาม responsibility |

---

## 🏋️ แบบฝึกหัด Part 09

### แบบฝึกหัดที่ 1: Shape Factory
สร้าง `IShape` interface และ factory ที่สร้าง shapes ตาม string parameter

### แบบฝึกหัดที่ 2: Cache System
สร้าง `ICache<TKey, TValue>` interface พร้อม implementations:
- MemoryCache
- (ถ้าต้องการ) FileCache

### แบบฝึกหัดที่ 3: Observer Pattern
ใช้ interfaces สร้าง event system:
- `IObservable<T>` - Observable
- `IObserver<T>` - Observer
- `IEventBus` - message bus

---

**ก่อนหน้า → [Part 08: Inheritance](part08-oop-inheritance.md)**  
**ต่อไป → [Part 10: Exception Handling](part10-exception-handling.md)**
