# Part 07: Object-Oriented Programming - Classes
## ขั้นตอนที่ 61-70: คลาสและการเขียนโปรแกรมเชิงวัตถุ

---

## 🎯 เป้าหมายของ Part นี้
- เข้าใจ Class, Object, Instance
- ใช้ Fields, Properties, Methods, Constructors
- รู้จัก Access Modifiers
- ใช้ static vs instance members
- เข้าใจ Records (C# 9+)
- ใช้ Object Initializers และ Anonymous Types

---

## ขั้นตอนที่ 61: Class และ Object

### Class คืออะไร?
```csharp
// Class = พิมพ์เขียว (blueprint)
// Object = instance ที่สร้างจาก class
// Method = พฤติกรรม (behavior)
// Field/Property = สถานะ (state)

// ตัวอย่าง Class
class Person
{
    // Fields - เก็บสถานะ (state)
    private string _name;
    private int _age;
    
    // Constructor - สร้าง instance
    public Person(string name, int age)
    {
        _name = name;
        _age = age;
    }
    
    // Properties - เข้าถึง fields
    public string Name
    {
        get { return _name; }
        set { _name = value; }
    }
    
    public int Age
    {
        get { return _age; }
        set
        {
            if (value < 0) throw new ArgumentException("Age cannot be negative");
            _age = value;
        }
    }
    
    // Methods - พฤติกรรม
    public void Greet()
    {
        Console.WriteLine($"สวัสดี ฉันชื่อ {_name} อายุ {_age} ปี");
    }
    
    public string GetInfo()
    {
        return $"{_name} ({_age})";
    }
    
    // Override ToString
    public override string ToString()
    {
        return $"Person: {_name}, {_age}";
    }
}

// สร้าง Object จาก Class
Person person1 = new Person("สมชาย", 25);
Person person2 = new Person("สมหญิง", 30);

// เรียกใช้ Methods
person1.Greet();                          // สวัสดี ฉันชื่อ สมชาย อายุ 25 ปี
Console.WriteLine(person1.GetInfo());     // สมชาย (25)
Console.WriteLine(person1);              // Person: สมชาย, 25

// Access Properties
Console.WriteLine(person2.Name);         // สมหญิง
person2.Age = 31;
Console.WriteLine(person2.Age);          // 31

// new keyword (C# 9+)
Person person3 = new("นางสาว", 22);  // Target-typed new
```

---

## ขั้นตอนที่ 62: Properties

### Properties แบบต่างๆ
```csharp
class Product
{
    // Full property with backing field
    private string _name = "";
    public string Name
    {
        get { return _name; }
        set
        {
            if (string.IsNullOrWhiteSpace(value))
                throw new ArgumentException("Name cannot be empty");
            _name = value.Trim();
        }
    }
    
    // Auto-implemented property (C# 3+)
    public int Quantity { get; set; }
    public string Category { get; set; } = "General";
    
    // Read-only auto property (C# 6+)
    public string Id { get; } = Guid.NewGuid().ToString()[..8];
    
    // Computed property (Expression body)
    private decimal _price;
    public decimal Price
    {
        get => _price;
        set => _price = value < 0 ? throw new ArgumentException("Price cannot be negative") : value;
    }
    
    // Computed (read-only) property
    public decimal TotalValue => Price * Quantity;
    public bool IsAvailable => Quantity > 0;
    public string DisplayName => $"{Category}: {Name}";
    
    // Init-only property (C# 9+)
    public DateTime CreatedAt { get; init; } = DateTime.Now;
    
    // Required property (C# 11+)
    public required string SKU { get; set; }
    
    // Property with different access levels
    public string Description { get; private set; } = "";
    
    public void UpdateDescription(string desc)
    {
        Description = desc;  // สามารถ set ได้ภายใน class
    }
}

// การใช้งาน
var product = new Product 
{ 
    SKU = "P-001",
    Name = "iPhone 15",
    Price = 35000m,
    Quantity = 10,
    Category = "Electronics"
};

Console.WriteLine(product.TotalValue);   // 350000
Console.WriteLine(product.IsAvailable);  // True
Console.WriteLine(product.DisplayName);  // Electronics: iPhone 15
Console.WriteLine(product.Id);           // Random 8 chars
Console.WriteLine(product.CreatedAt);    // Creation time

// product.CreatedAt = DateTime.Now;  ← Error! init-only
```

---

## ขั้นตอนที่ 63: Constructors

### Constructor ประเภทต่างๆ
```csharp
class BankAccount
{
    private decimal _balance;
    
    // Default constructor (no parameters)
    public BankAccount()
    {
        _balance = 0;
        AccountNumber = GenerateAccountNumber();
        CreatedAt = DateTime.Now;
    }
    
    // Parameterized constructor
    public BankAccount(string ownerName, decimal initialBalance = 0)
        : this()  // เรียก default constructor ก่อน
    {
        OwnerName = ownerName;
        
        if (initialBalance < 0)
            throw new ArgumentException("Initial balance cannot be negative");
        _balance = initialBalance;
    }
    
    // Copy constructor
    public BankAccount(BankAccount other)
    {
        AccountNumber = GenerateAccountNumber();  // ใหม่ทุกครั้ง
        OwnerName = other.OwnerName;
        _balance = other._balance;
        CreatedAt = DateTime.Now;
    }
    
    public string AccountNumber { get; }
    public string OwnerName { get; set; } = "";
    public decimal Balance => _balance;
    public DateTime CreatedAt { get; }
    
    public void Deposit(decimal amount)
    {
        if (amount <= 0) throw new ArgumentException("Amount must be positive");
        _balance += amount;
    }
    
    public bool Withdraw(decimal amount)
    {
        if (amount <= 0) throw new ArgumentException("Amount must be positive");
        if (amount > _balance) return false;
        _balance -= amount;
        return true;
    }
    
    private string GenerateAccountNumber()
    {
        return $"ACC-{DateTime.Now:yyyyMMdd}-{Random.Shared.Next(1000, 9999)}";
    }
    
    public override string ToString() =>
        $"[{AccountNumber}] {OwnerName}: {_balance:C2}";
}

// Static constructor (เรียกครั้งเดียว ก่อน instance แรก)
class Database
{
    private static string _connectionString;
    
    static Database()
    {
        // Initialize static members
        _connectionString = "Server=localhost;Database=mydb;";
        Console.WriteLine("Database class initialized");
    }
    
    public Database()
    {
        // Instance initialization
    }
}
```

### Constructor Chaining
```csharp
class Person
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public int Age { get; set; }
    public string Email { get; set; }
    public DateTime BirthDate { get; set; }
    
    // Primary constructor (C# 12)
    // class Person(string firstName, string lastName) { ... }
    
    // Constructor chaining
    public Person(string firstName, string lastName) 
        : this(firstName, lastName, 0) { }
    
    public Person(string firstName, string lastName, int age)
        : this(firstName, lastName, age, "") { }
    
    public Person(string firstName, string lastName, int age, string email)
    {
        FirstName = firstName;
        LastName = lastName;
        Age = age;
        Email = email;
        BirthDate = DateTime.Now.AddYears(-age);
    }
    
    public string FullName => $"{FirstName} {LastName}";
}
```

---

## ขั้นตอนที่ 64: Static Members

### Static Fields, Properties, Methods
```csharp
class Counter
{
    // Static field - ใช้ร่วมกันทุก instance
    private static int _totalCount = 0;
    private static readonly object _lock = new object();
    
    // Instance field
    private int _instanceId;
    
    public Counter()
    {
        lock (_lock)
        {
            _totalCount++;
            _instanceId = _totalCount;
        }
    }
    
    // Static property
    public static int TotalCount => _totalCount;
    
    // Instance property
    public int Id => _instanceId;
    
    // Static method
    public static void ResetAll()
    {
        _totalCount = 0;
        Console.WriteLine("Reset all counters");
    }
    
    // Destructor (Finalizer)
    ~Counter()
    {
        // เรียกตอน GC เก็บ object (ไม่แนะนำ เพราะ timing ไม่แน่นอน)
        _totalCount--;
    }
}

// Static Class - ทุก member ต้องเป็น static
static class MathHelper
{
    public static double DegToRad(double degrees) => degrees * Math.PI / 180;
    public static double RadToDeg(double radians) => radians * 180 / Math.PI;
    
    public static bool IsPrime(int n)
    {
        if (n < 2) return false;
        for (int i = 2; i <= Math.Sqrt(n); i++)
            if (n % i == 0) return false;
        return true;
    }
    
    public static IEnumerable<int> Primes(int max)
    {
        for (int n = 2; n <= max; n++)
            if (IsPrime(n)) yield return n;
    }
}

// การใช้งาน
Counter c1 = new Counter();
Counter c2 = new Counter();
Counter c3 = new Counter();

Console.WriteLine($"ID: {c1.Id}, {c2.Id}, {c3.Id}");  // 1, 2, 3
Console.WriteLine($"Total: {Counter.TotalCount}");     // 3

Console.WriteLine(MathHelper.IsPrime(17));  // True
Console.WriteLine(string.Join(", ", MathHelper.Primes(20)));  // 2, 3, 5, 7, 11, 13, 17, 19
```

---

## ขั้นตอนที่ 65: Records (C# 9+)

### Record Types
```csharp
// Record - immutable reference type
record Point(double X, double Y);

var p1 = new Point(1.0, 2.0);
var p2 = new Point(1.0, 2.0);
var p3 = new Point(3.0, 4.0);

Console.WriteLine(p1 == p2);  // True (value equality!)
Console.WriteLine(p1 == p3);  // False
Console.WriteLine(p1);        // Point { X = 1, Y = 2 }

// With expression - สร้าง copy พร้อมเปลี่ยนค่า
var p4 = p1 with { X = 10.0 };
Console.WriteLine(p4);  // Point { X = 10, Y = 2 }

// Record ที่ซับซ้อน
record Customer
{
    public required string Id { get; init; }
    public required string Name { get; init; }
    public string Email { get; init; } = "";
    public decimal Balance { get; init; } = 0;
    public DateTime CreatedAt { get; init; } = DateTime.Now;
    
    // Custom method ใน record
    public string GetDisplayName() => $"{Name} ({Id})";
    
    // Deconstruct
    public void Deconstruct(out string id, out string name, out decimal balance)
    {
        id = Id;
        name = Name;
        balance = Balance;
    }
}

var customer = new Customer { Id = "C001", Name = "สมชาย", Balance = 5000m };
Console.WriteLine(customer);  // Customer { Id = C001, Name = สมชาย, ... }

// Deconstruct
var (id, name, balance) = customer;
Console.WriteLine($"{id}: {name} = {balance:C2}");

// Record struct (C# 10)
record struct Coordinate(double Lat, double Lng);

// Positional record with body
record PersonRecord(string FirstName, string LastName) 
{
    public string FullName => $"{FirstName} {LastName}";
    
    // Can have additional properties
    public string? Email { get; init; }
}
```

---

## ขั้นตอนที่ 66: Object Initializers

### Object Initializer Syntax
```csharp
class Address
{
    public string Street { get; set; } = "";
    public string City { get; set; } = "";
    public string Province { get; set; } = "";
    public string ZipCode { get; set; } = "";
    public string Country { get; set; } = "Thailand";
}

class Employee
{
    public string Id { get; set; } = "";
    public string Name { get; set; } = "";
    public int Age { get; set; }
    public decimal Salary { get; set; }
    public Address? HomeAddress { get; set; }
    public List<string> Skills { get; set; } = new();
    public Dictionary<string, string> Contacts { get; set; } = new();
}

// Object Initializer
var emp = new Employee
{
    Id = "E001",
    Name = "สมชาย วิชาการ",
    Age = 28,
    Salary = 45000m,
    HomeAddress = new Address  // Nested object initializer
    {
        Street = "123 ถนนสุขุมวิท",
        City = "กรุงเทพ",
        Province = "กรุงเทพมหานคร",
        ZipCode = "10110"
    },
    Skills = new List<string> { "C#", "SQL", "WPF" },  // Collection initializer
    Contacts = new Dictionary<string, string>
    {
        { "email", "somchai@example.com" },
        { "phone", "081-234-5678" }
    }
};

Console.WriteLine($"{emp.Name} ({emp.Id})");
Console.WriteLine($"เมือง: {emp.HomeAddress?.City}");
Console.WriteLine($"ทักษะ: {string.Join(", ", emp.Skills)}");
```

---

## ขั้นตอนที่ 67: Anonymous Types

### Anonymous Types
```csharp
// Anonymous type - ไม่มีชื่อ class, อ่านอย่างเดียว
var person = new { Name = "สมชาย", Age = 25, Score = 95.5 };
Console.WriteLine(person.Name);   // สมชาย
Console.WriteLine(person.Age);    // 25
Console.WriteLine(person);        // { Name = สมชาย, Age = 25, Score = 95.5 }

// ใช้บ่อยกับ LINQ Select
var employees = new List<Employee>
{
    new() { Id = "E001", Name = "Alice", Salary = 50000m },
    new() { Id = "E002", Name = "Bob", Salary = 45000m },
    new() { Id = "E003", Name = "Charlie", Salary = 55000m }
};

var summary = employees.Select(e => new 
{
    e.Id,
    e.Name,
    FormattedSalary = e.Salary.ToString("N2"),
    IsHighEarner = e.Salary > 50000
});

foreach (var s in summary)
    Console.WriteLine($"{s.Name}: {s.FormattedSalary} ({(s.IsHighEarner ? "High" : "Normal")})");

// Tuple vs Anonymous Type
// Tuple - ไม่มีชื่อ members (หรือมีแบบ named)
var tuple = (Name: "สมชาย", Age: 25);
Console.WriteLine(tuple.Name);

// Anonymous type - มีชื่อ members แต่ immutable
var anon = new { Name = "สมชาย", Age = 25 };
Console.WriteLine(anon.Name);
// anon.Name = "อื่น";  ← Error! read-only
```

---

## ขั้นตอนที่ 68: Nested Classes และ Partial Classes

### Nested Classes
```csharp
class Order
{
    private List<OrderItem> _items = new();
    
    // Nested class
    public class OrderItem
    {
        public string ProductId { get; set; } = "";
        public string ProductName { get; set; } = "";
        public int Quantity { get; set; }
        public decimal UnitPrice { get; set; }
        public decimal Total => UnitPrice * Quantity;
    }
    
    // Private nested class (ใช้ภายใน Order เท่านั้น)
    private class DiscountCalculator
    {
        public decimal Calculate(decimal amount, string memberType) => memberType switch
        {
            "Gold" => amount * 0.10m,
            "Platinum" => amount * 0.20m,
            _ => 0m
        };
    }
    
    public void AddItem(string id, string name, int qty, decimal price)
    {
        _items.Add(new OrderItem 
        { 
            ProductId = id, 
            ProductName = name,
            Quantity = qty, 
            UnitPrice = price 
        });
    }
    
    public decimal GetTotal(string memberType = "")
    {
        decimal subtotal = _items.Sum(i => i.Total);
        var calculator = new DiscountCalculator();
        decimal discount = calculator.Calculate(subtotal, memberType);
        return subtotal - discount;
    }
    
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();
}

// การใช้งาน Nested Class
var order = new Order();
order.AddItem("P001", "Apple", 2, 50m);
order.AddItem("P002", "Banana", 3, 20m);

// Access nested class จากภายนอก
var item = new Order.OrderItem { ProductName = "Cherry", Quantity = 1, UnitPrice = 100m };

foreach (var i in order.Items)
    Console.WriteLine($"{i.ProductName}: {i.Total:N2}");
```

### Partial Classes
```csharp
// ไฟล์ที่ 1: Person.cs
public partial class PersonModel
{
    public string Id { get; set; } = "";
    public string Name { get; set; } = "";
    public int Age { get; set; }
    
    public bool IsValid()
    {
        return !string.IsNullOrEmpty(Name) && Age >= 0;
    }
}

// ไฟล์ที่ 2: Person.Generated.cs (auto-generated)
public partial class PersonModel
{
    // Generated by tool
    public string GetDisplayName() => Name;
    public override string ToString() => $"{Name} ({Age})";
}

// Partial Methods
public partial class DataProcessor
{
    public void Process(string data)
    {
        OnBeforeProcess(data);  // partial method call
        
        // main logic
        Console.WriteLine($"Processing: {data}");
        
        OnAfterProcess(data);  // partial method call
    }
    
    partial void OnBeforeProcess(string data);  // declaration
    partial void OnAfterProcess(string data);   // declaration
}

// Implementation (อาจอยู่ไฟล์อื่น)
public partial class DataProcessor
{
    partial void OnBeforeProcess(string data)
    {
        Console.WriteLine($"Before: {data}");
    }
    
    partial void OnAfterProcess(string data)
    {
        Console.WriteLine($"After: {data}");
    }
}
```

---

## ขั้นตอนที่ 69: Struct vs Class

### Struct - Value Type
```csharp
// Struct = Value Type (เก็บใน Stack, copy เมื่อส่งผ่าน)
struct Point2D
{
    public double X;
    public double Y;
    
    public Point2D(double x, double y)
    {
        X = x;
        Y = y;
    }
    
    public double DistanceTo(Point2D other)
    {
        double dx = X - other.X;
        double dy = Y - other.Y;
        return Math.Sqrt(dx * dx + dy * dy);
    }
    
    public Point2D Translate(double dx, double dy)
    {
        return new Point2D(X + dx, Y + dy);  // Return new struct
    }
    
    public override string ToString() => $"({X:F2}, {Y:F2})";
}

// Struct ถูก copy เมื่อส่งผ่าน
Point2D p1 = new Point2D(1, 2);
Point2D p2 = p1;  // Copy!
p2.X = 10;

Console.WriteLine(p1);  // (1.00, 2.00) - ไม่เปลี่ยน!
Console.WriteLine(p2);  // (10.00, 2.00)

// Class ถูกส่ง reference
class PointClass
{
    public double X, Y;
    public PointClass(double x, double y) { X = x; Y = y; }
}

var c1 = new PointClass(1, 2);
var c2 = c1;  // Reference copy!
c2.X = 10;

Console.WriteLine($"{c1.X}, {c1.Y}");  // 10, 2 - เปลี่ยนด้วย!

/*
Struct ดีกว่าเมื่อ:
- ขนาดเล็ก (< 16 bytes แนะนำ)
- Immutable
- Short-lived objects
- Value semantics ที่สมเหตุสมผล

Class ดีกว่าเมื่อ:
- ขนาดใหญ่
- Mutable
- Long-lived objects
- Reference semantics
- ต้องการ inheritance
*/
```

---

## ขั้นตอนที่ 70: โปรแกรมตัวอย่าง - School Management System

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace SchoolManagement
{
    // Enums
    enum Grade { A, B_Plus, B, C_Plus, C, D_Plus, D, F }
    enum Department { Science, Arts, Business, Engineering, Medicine }
    
    // Records
    record Address(string Street, string City, string Province, string ZipCode);
    
    // Base data class
    class Person
    {
        private static int _nextId = 1;
        
        public int Id { get; } = _nextId++;
        public string FirstName { get; set; }
        public string LastName { get; set; }
        public DateTime DateOfBirth { get; set; }
        public Address? HomeAddress { get; set; }
        
        public Person(string firstName, string lastName, DateTime dob)
        {
            FirstName = firstName;
            LastName = lastName;
            DateOfBirth = dob;
        }
        
        public string FullName => $"{FirstName} {LastName}";
        public int Age => (int)((DateTime.Now - DateOfBirth).Days / 365.25);
        
        public override string ToString() => $"{Id}: {FullName}";
    }
    
    // Student class
    class Student : Person
    {
        public string StudentId { get; }
        public Department Department { get; set; }
        public int Year { get; set; }
        public List<CourseEnrollment> Enrollments { get; } = new();
        
        public Student(string firstName, string lastName, DateTime dob, Department dept, int year)
            : base(firstName, lastName, dob)
        {
            StudentId = $"STU{Id:0000}";
            Department = dept;
            Year = year;
        }
        
        public double GPA
        {
            get
            {
                if (!Enrollments.Any()) return 0;
                return Enrollments.Where(e => e.IsCompleted)
                    .Average(e => GradeToPoint(e.Grade));
            }
        }
        
        private double GradeToPoint(Grade grade) => grade switch
        {
            Grade.A => 4.0,
            Grade.B_Plus => 3.5,
            Grade.B => 3.0,
            Grade.C_Plus => 2.5,
            Grade.C => 2.0,
            Grade.D_Plus => 1.5,
            Grade.D => 1.0,
            Grade.F => 0.0,
            _ => 0.0
        };
        
        public void EnrollCourse(Course course)
        {
            if (Enrollments.Any(e => e.CourseId == course.Id && !e.IsCompleted))
                throw new InvalidOperationException($"Already enrolled in {course.Name}");
            
            Enrollments.Add(new CourseEnrollment(course.Id, course.Name));
        }
        
        public bool CompleteCourse(string courseId, Grade grade)
        {
            var enrollment = Enrollments.FirstOrDefault(
                e => e.CourseId == courseId && !e.IsCompleted);
            
            if (enrollment == null) return false;
            
            enrollment.Complete(grade);
            return true;
        }
        
        public override string ToString() => 
            $"[{StudentId}] {FullName} - {Department}, Year {Year}, GPA: {GPA:F2}";
    }
    
    // Course Enrollment
    class CourseEnrollment
    {
        public string CourseId { get; }
        public string CourseName { get; }
        public DateTime EnrolledDate { get; } = DateTime.Now;
        public DateTime? CompletedDate { get; private set; }
        public Grade Grade { get; private set; }
        public bool IsCompleted { get; private set; }
        
        public CourseEnrollment(string courseId, string courseName)
        {
            CourseId = courseId;
            CourseName = courseName;
        }
        
        public void Complete(Grade grade)
        {
            Grade = grade;
            CompletedDate = DateTime.Now;
            IsCompleted = true;
        }
    }
    
    // Course
    class Course
    {
        private static int _nextId = 1;
        
        public string Id { get; }
        public string Name { get; set; }
        public Department Department { get; set; }
        public int Credits { get; set; }
        public string Description { get; set; } = "";
        public int MaxStudents { get; set; } = 40;
        
        public Course(string name, Department dept, int credits)
        {
            Id = $"CRS{_nextId++:000}";
            Name = name;
            Department = dept;
            Credits = credits;
        }
        
        public override string ToString() => $"[{Id}] {Name} ({Credits} credits)";
    }
    
    // School
    class School
    {
        public string Name { get; }
        private List<Student> _students = new();
        private List<Course> _courses = new();
        
        public School(string name)
        {
            Name = name;
        }
        
        public Student AddStudent(string firstName, string lastName, 
            DateTime dob, Department dept, int year)
        {
            var student = new Student(firstName, lastName, dob, dept, year);
            _students.Add(student);
            return student;
        }
        
        public Course AddCourse(string name, Department dept, int credits)
        {
            var course = new Course(name, dept, credits);
            _courses.Add(course);
            return course;
        }
        
        public Student? FindStudent(string studentId)
            => _students.FirstOrDefault(s => s.StudentId == studentId);
        
        public Course? FindCourse(string courseId)
            => _courses.FirstOrDefault(c => c.Id == courseId);
        
        public IEnumerable<Student> GetTopStudents(int count = 10)
            => _students.OrderByDescending(s => s.GPA).Take(count);
        
        public Dictionary<Department, double> GetAverageGPAByDepartment()
            => _students
                .GroupBy(s => s.Department)
                .ToDictionary(
                    g => g.Key,
                    g => g.Average(s => s.GPA)
                );
        
        public void PrintReport()
        {
            Console.ForegroundColor = ConsoleColor.Cyan;
            Console.WriteLine($"\n╔══════════════════════════════════╗");
            Console.WriteLine($"║  รายงาน: {Name,-24}║");
            Console.WriteLine($"╚══════════════════════════════════╝");
            Console.ResetColor();
            
            Console.WriteLine($"\nนักศึกษาทั้งหมด: {_students.Count} คน");
            Console.WriteLine($"รายวิชาทั้งหมด: {_courses.Count} วิชา");
            
            Console.WriteLine("\n--- นักศึกษา GPA สูงสุด 5 คน ---");
            int rank = 1;
            foreach (var student in GetTopStudents(5))
            {
                Console.ForegroundColor = rank == 1 ? ConsoleColor.Yellow : ConsoleColor.White;
                Console.WriteLine($"{rank}. {student.FullName} - GPA: {student.GPA:F2}");
                Console.ResetColor();
                rank++;
            }
            
            Console.WriteLine("\n--- GPA เฉลี่ยตามคณะ ---");
            foreach (var (dept, avgGpa) in GetAverageGPAByDepartment().OrderByDescending(x => x.Value))
                Console.WriteLine($"{dept}: {avgGpa:F2}");
        }
    }
    
    class Program
    {
        static void Main(string[] args)
        {
            Console.OutputEncoding = System.Text.Encoding.UTF8;
            
            var school = new School("มหาวิทยาลัยวิทยาศาสตร์และเทคโนโลยี");
            
            // เพิ่มรายวิชา
            var cs101 = school.AddCourse("Introduction to CS", Department.Engineering, 3);
            var math201 = school.AddCourse("Calculus II", Department.Science, 4);
            var prog301 = school.AddCourse("Advanced Programming", Department.Engineering, 3);
            var db401 = school.AddCourse("Database Systems", Department.Engineering, 3);
            var ai501 = school.AddCourse("Artificial Intelligence", Department.Engineering, 4);
            
            // เพิ่มนักศึกษา
            var s1 = school.AddStudent("สมชาย", "วิชาการ", new DateTime(2002, 5, 15), 
                Department.Engineering, 3);
            var s2 = school.AddStudent("สมหญิง", "ใจดี", new DateTime(2003, 8, 22), 
                Department.Engineering, 2);
            var s3 = school.AddStudent("นายเก่ง", "เรียนดี", new DateTime(2001, 3, 10), 
                Department.Science, 4);
            var s4 = school.AddStudent("นางสาวสวย", "เก่งกาจ", new DateTime(2002, 11, 5), 
                Department.Business, 3);
            var s5 = school.AddStudent("วิชาญ", "โปรแกรมมิ่ง", new DateTime(2000, 7, 18), 
                Department.Engineering, 4);
            
            // ลงทะเบียนและให้เกรด
            s1.EnrollCourse(cs101);
            s1.EnrollCourse(math201);
            s1.EnrollCourse(prog301);
            s1.CompleteCourse(cs101.Id, Grade.A);
            s1.CompleteCourse(math201.Id, Grade.B_Plus);
            s1.CompleteCourse(prog301.Id, Grade.A);
            
            s2.EnrollCourse(cs101);
            s2.EnrollCourse(math201);
            s2.CompleteCourse(cs101.Id, Grade.B);
            s2.CompleteCourse(math201.Id, Grade.C_Plus);
            
            s3.EnrollCourse(math201);
            s3.EnrollCourse(db401);
            s3.EnrollCourse(ai501);
            s3.CompleteCourse(math201.Id, Grade.A);
            s3.CompleteCourse(db401.Id, Grade.A);
            s3.CompleteCourse(ai501.Id, Grade.B_Plus);
            
            s5.EnrollCourse(cs101);
            s5.EnrollCourse(prog301);
            s5.EnrollCourse(db401);
            s5.EnrollCourse(ai501);
            s5.CompleteCourse(cs101.Id, Grade.A);
            s5.CompleteCourse(prog301.Id, Grade.A);
            s5.CompleteCourse(db401.Id, Grade.A);
            s5.CompleteCourse(ai501.Id, Grade.A);
            
            // แสดงรายงาน
            school.PrintReport();
            
            // แสดงรายละเอียดนักศึกษา
            Console.WriteLine("\n--- รายละเอียดนักศึกษา ---");
            Console.WriteLine(s1);
            Console.WriteLine($"รายวิชาที่ผ่าน: {s1.Enrollments.Count(e => e.IsCompleted)}");
            
            Console.WriteLine("\nกด Enter เพื่อออก...");
            Console.ReadLine();
        }
    }
}
```

---

## 📝 สรุป Part 07

| หัวข้อ | รายละเอียดสำคัญ |
|--------|----------------|
| Class | Blueprint สำหรับสร้าง objects |
| Object | Instance ของ class |
| Fields | เก็บสถานะ (state) |
| Properties | เข้าถึง fields พร้อม validation |
| Constructors | สร้าง instance, initializing state |
| Static Members | ใช้ร่วมกันทุก instance |
| Records | Immutable data objects, value equality |
| Structs | Value type, ขนาดเล็ก |
| Object Initializers | สร้าง object พร้อมกำหนดค่า |
| Nested Classes | Classes ภายใน classes |

---

## 🏋️ แบบฝึกหัด Part 07

### แบบฝึกหัดที่ 1: Bank System
สร้าง class `BankAccount` ที่มี:
- Properties: AccountNumber, Owner, Balance, Type
- Methods: Deposit, Withdraw, Transfer, GetHistory
- Validation ที่เหมาะสม

### แบบฝึกหัดที่ 2: Shopping Cart
สร้าง class `ShoppingCart` ที่มี:
- AddItem, RemoveItem, UpdateQuantity
- CalculateTotal (พร้อมส่วนลด)
- Checkout

### แบบฝึกหัดที่ 3: Record & Immutability
สร้าง record `Transaction` แบบ immutable สำหรับระบบบัญชี

---

**ก่อนหน้า → [Part 06: Arrays & Collections](part06-arrays-collections.md)**  
**ต่อไป → [Part 08: OOP - Inheritance & Polymorphism](part08-oop-inheritance.md)**
