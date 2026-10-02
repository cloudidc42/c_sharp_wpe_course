# Part 05: Methods & Functions
## ขั้นตอนที่ 41-50: การสร้างและใช้งาน Methods

---

## 🎯 เป้าหมายของ Part นี้
- เข้าใจ Methods และการประกาศ Method
- รู้จัก Parameters ทุกประเภท (in, out, ref, params)
- ใช้ Optional Parameters และ Named Arguments
- เข้าใจ Method Overloading
- ใช้ Expression-bodied Methods และ Local Functions
- เข้าใจ Recursion

---

## ขั้นตอนที่ 41: Methods พื้นฐาน

### การประกาศ Method
```csharp
// รูปแบบ: [access modifier] [return type] [method name]([parameters])
// {
//     // method body
//     return value;  // ถ้า return type ไม่ใช่ void
// }

// Method ไม่ return ค่า (void)
void SayHello()
{
    Console.WriteLine("สวัสดี!");
}

// Method return ค่า
int Add(int a, int b)
{
    return a + b;
}

// Method ที่รับ parameters หลายตัว
double CalculateArea(double width, double height)
{
    return width * height;
}

// การเรียกใช้ Method
SayHello();                              // สวัสดี!
int result = Add(5, 3);                  // 8
double area = CalculateArea(10.5, 5.0);  // 52.5
Console.WriteLine($"พื้นที่: {area}");
```

### Access Modifiers
```csharp
class Example
{
    // public - เรียกได้จากทุกที่
    public void PublicMethod()
    {
        Console.WriteLine("Public");
    }
    
    // private - เรียกได้แค่ใน class นี้ (default สำหรับ members)
    private void PrivateMethod()
    {
        Console.WriteLine("Private");
    }
    
    // protected - เรียกได้ใน class นี้และ derived classes
    protected void ProtectedMethod()
    {
        Console.WriteLine("Protected");
    }
    
    // internal - เรียกได้ใน assembly เดียวกัน
    internal void InternalMethod()
    {
        Console.WriteLine("Internal");
    }
    
    // static - เรียกได้โดยไม่ต้องสร้าง instance
    public static void StaticMethod()
    {
        Console.WriteLine("Static");
    }
}
```

### Return Types
```csharp
// void - ไม่ return
void PrintName(string name)
{
    Console.WriteLine(name);
}

// int - return จำนวนเต็ม
int Square(int n) => n * n;  // Expression-bodied

// string - return string
string GetFullName(string first, string last)
{
    return $"{first} {last}";
}

// bool - return boolean
bool IsEven(int n) => n % 2 == 0;

// tuple - return หลายค่า (C# 7+)
(int min, int max, double average) GetStats(int[] numbers)
{
    int min = numbers[0], max = numbers[0];
    double sum = 0;
    
    foreach (int n in numbers)
    {
        if (n < min) min = n;
        if (n > max) max = n;
        sum += n;
    }
    
    return (min, max, sum / numbers.Length);
}

// การใช้งาน tuple return
int[] scores = { 85, 72, 90, 68, 95 };
var (min, max, avg) = GetStats(scores);
Console.WriteLine($"Min: {min}, Max: {max}, Avg: {avg:F2}");
```

---

## ขั้นตอนที่ 42: Parameters ประเภทต่างๆ

### Value Parameters (default)
```csharp
// Value parameters - ส่งสำเนา ของค่า (ไม่กระทบต้นฉบับ)
void ModifyValue(int x)
{
    x = x * 2;  // แก้ไขแค่ local copy
    Console.WriteLine($"Inside: {x}");
}

int num = 10;
ModifyValue(num);
Console.WriteLine($"Outside: {num}");  // ยังคงเป็น 10!

// Output:
// Inside: 20
// Outside: 10
```

### ref Parameters (ส่ง reference)
```csharp
// ref parameters - ส่ง reference จริงๆ (กระทบต้นฉบับ)
void DoubleValue(ref int x)
{
    x = x * 2;  // แก้ไขค่าต้นฉบับ
}

int num = 10;
DoubleValue(ref num);  // ต้องใส่ ref ตอนเรียกด้วย
Console.WriteLine(num);  // 20

// ใช้ ref เพื่อ swap
void Swap<T>(ref T a, ref T b)
{
    T temp = a;
    a = b;
    b = temp;
}

int x = 5, y = 10;
Swap(ref x, ref y);
Console.WriteLine($"x={x}, y={y}");  // x=10, y=5

string s1 = "Hello", s2 = "World";
Swap(ref s1, ref s2);
Console.WriteLine($"s1={s1}, s2={s2}");  // s1=World, s2=Hello
```

### out Parameters (return หลายค่า)
```csharp
// out parameters - Method ต้องกำหนดค่าก่อน return
bool TryDivide(int dividend, int divisor, out double result)
{
    if (divisor == 0)
    {
        result = 0;  // ต้องกำหนดค่า out parameter เสมอ
        return false;
    }
    
    result = (double)dividend / divisor;
    return true;
}

if (TryDivide(10, 3, out double quotient))
{
    Console.WriteLine($"ผล: {quotient:F4}");  // 3.3333
}
else
{
    Console.WriteLine("ไม่สามารถหารด้วย 0 ได้");
}

// out var (C# 7+) - ประกาศตัวแปรพร้อมกัน
if (int.TryParse("42", out var number))
{
    Console.WriteLine($"แปลงสำเร็จ: {number}");
}

// Discard _ สำหรับ out ที่ไม่ใช้
if (int.TryParse("not a number", out _))
{
    // ไม่สนใจค่า out
}
```

### in Parameters (read-only reference)
```csharp
// in parameters - ส่ง reference แต่ไม่อนุญาตให้แก้ไข (ประหยัด copy)
// ใช้กับ large structs เพื่อ performance
struct LargeStruct
{
    public int[] Data;
    public string Name;
}

double CalculateAverage(in LargeStruct data)
{
    // data.Name = "changed";  ← Error! ไม่สามารถแก้ไข
    
    double sum = 0;
    foreach (int n in data.Data)
        sum += n;
    return sum / data.Data.Length;
}

LargeStruct myData = new LargeStruct { Data = new[] { 1, 2, 3, 4, 5 }, Name = "test" };
double avg = CalculateAverage(in myData);
```

### params Parameters (จำนวนไม่แน่นอน)
```csharp
// params - รับจำนวน arguments ไม่แน่นอน
int Sum(params int[] numbers)
{
    int total = 0;
    foreach (int n in numbers)
        total += n;
    return total;
}

// สามารถเรียกได้หลายวิธี
Console.WriteLine(Sum());              // 0
Console.WriteLine(Sum(1, 2, 3));       // 6
Console.WriteLine(Sum(1, 2, 3, 4, 5)); // 15

int[] arr = { 10, 20, 30 };
Console.WriteLine(Sum(arr));           // 60 (ส่ง array ได้เลย)

// params ต้องเป็น parameter สุดท้าย
string Format(string separator, params string[] values)
{
    return string.Join(separator, values);
}

Console.WriteLine(Format(", ", "apple", "banana", "cherry"));
// apple, banana, cherry
```

---

## ขั้นตอนที่ 43: Optional Parameters และ Named Arguments

### Optional Parameters (Default Values)
```csharp
// Optional parameters - มีค่า default ถ้าไม่ส่งมา
void CreateUser(string name, int age = 18, string role = "User", bool isActive = true)
{
    Console.WriteLine($"สร้างผู้ใช้: {name}, อายุ: {age}, บทบาท: {role}, ใช้งาน: {isActive}");
}

// เรียกได้หลายวิธี
CreateUser("สมชาย");                          // ใช้ default ทั้งหมด
CreateUser("สมชาย", 25);                      // กำหนด age
CreateUser("สมชาย", 25, "Admin");             // กำหนด age และ role
CreateUser("สมชาย", 25, "Admin", false);      // กำหนดทั้งหมด

// Output:
// สร้างผู้ใช้: สมชาย, อายุ: 18, บทบาท: User, ใช้งาน: True
// สร้างผู้ใช้: สมชาย, อายุ: 25, บทบาท: User, ใช้งาน: True
// สร้างผู้ใช้: สมชาย, อายุ: 25, บทบาท: Admin, ใช้งาน: True
// สร้างผู้ใช้: สมชาย, อายุ: 25, บทบาท: Admin, ใช้งาน: False
```

### Named Arguments
```csharp
// Named arguments - ระบุชื่อ parameter ตอนเรียก
void SendEmail(string to, string subject, string body, bool isHtml = false, int priority = 0)
{
    Console.WriteLine($"ส่งถึง: {to}");
    Console.WriteLine($"หัวข้อ: {subject}");
    Console.WriteLine($"HTML: {isHtml}, Priority: {priority}");
}

// ปกติ (ตามลำดับ)
SendEmail("user@email.com", "สวัสดี", "เนื้อหา", false, 1);

// Named arguments - ลำดับไม่สำคัญ
SendEmail(
    to: "user@email.com",
    subject: "สวัสดี",
    body: "เนื้อหา",
    priority: 2,       // ข้ามบางตัวได้ถ้าใช้ Named
    isHtml: true
);

// ผสมกัน - positional ก่อน named ทีหลัง
SendEmail("user@email.com", "สวัสดี", "เนื้อหา", isHtml: true);
```

---

## ขั้นตอนที่ 44: Method Overloading

### Overloading - Method ชื่อเดิม Signature ต่างกัน
```csharp
class Calculator
{
    // Overloaded methods - ชื่อเหมือนกัน แต่ parameters ต่างกัน
    
    public int Add(int a, int b)
    {
        return a + b;
    }
    
    public double Add(double a, double b)
    {
        return a + b;
    }
    
    public int Add(int a, int b, int c)
    {
        return a + b + c;
    }
    
    public string Add(string a, string b)
    {
        return a + b;  // string concatenation
    }
    
    // ❌ Error - ไม่สามารถ overload ด้วย return type เท่านั้น
    // public double Add(int a, int b) { return a + b; }
}

var calc = new Calculator();
Console.WriteLine(calc.Add(1, 2));          // 3
Console.WriteLine(calc.Add(1.5, 2.5));      // 4
Console.WriteLine(calc.Add(1, 2, 3));       // 6
Console.WriteLine(calc.Add("Hello", "!")); // Hello!
```

### Overloading กับ Optional vs Explicit
```csharp
// ✅ ชัดเจนกว่าถ้าใช้ overloading
void Log(string message)
{
    Log(message, LogLevel.Info);
}

void Log(string message, LogLevel level)
{
    Console.WriteLine($"[{level}] {message}");
}

enum LogLevel { Debug, Info, Warning, Error }

// ⚠️ optional parameters อาจทำให้เกิด ambiguity
void Process(string data, bool validate = true) { }
void Process(string data) { }  // ← Ambiguous กับ บนเมื่อเรียก Process("test")
```

---

## ขั้นตอนที่ 45: Expression-bodied Members

### Expression Body (=> operator)
```csharp
// แทน:
int Add(int a, int b)
{
    return a + b;
}

// ใช้:
int Add(int a, int b) => a + b;

// void method
void PrintLine(string text) => Console.WriteLine(text);

// Properties
class Circle
{
    public double Radius { get; set; }
    
    // Expression-bodied property
    public double Area => Math.PI * Radius * Radius;
    public double Circumference => 2 * Math.PI * Radius;
    
    // Expression-bodied method
    public string Describe() => $"วงกลมรัศมี {Radius:F2} cm";
    
    // Expression-bodied constructor
    public Circle(double radius) => Radius = radius;
}

var circle = new Circle(5.0);
Console.WriteLine($"Area: {circle.Area:F2}");          // 78.54
Console.WriteLine($"Circumference: {circle.Circumference:F2}");  // 31.42
Console.WriteLine(circle.Describe());

// Chain expressions
bool IsValidAge(int age) => age is >= 0 and <= 120;
string GetAgeCategory(int age) => age switch
{
    < 13 => "เด็ก",
    < 18 => "วัยรุ่น",
    < 60 => "ผู้ใหญ่",
    _ => "ผู้สูงอายุ"
};
```

---

## ขั้นตอนที่ 46: Local Functions

### Local Functions - Functions ใน Functions
```csharp
// Local function - ประกาศภายใน method
void ProcessData(int[] data)
{
    // Local function - ใช้ได้แค่ใน ProcessData
    bool IsValid(int value) => value >= 0 && value <= 100;
    double Normalize(int value) => value / 100.0;
    
    foreach (int item in data)
    {
        if (!IsValid(item))
        {
            Console.WriteLine($"ข้อมูลไม่ถูกต้อง: {item}");
            continue;
        }
        
        double normalized = Normalize(item);
        Console.WriteLine($"{item} → {normalized:P1}");
    }
}

ProcessData(new[] { 75, 90, -5, 85, 110, 60 });

// Local function สามารถเข้าถึง outer variables
void CreateCounter(int start, int step)
{
    int current = start;
    
    // เข้าถึง start, step, current ได้
    void PrintAndIncrement()
    {
        Console.Write($"{current} ");
        current += step;
    }
    
    for (int i = 0; i < 5; i++)
        PrintAndIncrement();
    
    Console.WriteLine();
}

CreateCounter(1, 2);   // 1 3 5 7 9
CreateCounter(10, -2); // 10 8 6 4 2

// Static local function (C# 8+) - ไม่สามารถเข้าถึง outer variables
void ProcessWithStatic()
{
    int x = 10;
    
    static int Double(int value) => value * 2;  // ไม่เข้าถึง x ได้
    
    Console.WriteLine(Double(x));
}
```

---

## ขั้นตอนที่ 47: Recursion (การเรียกตัวเอง)

### Recursion พื้นฐาน
```csharp
// Factorial: n! = n × (n-1)!
long Factorial(int n)
{
    // Base case - หยุด recursion
    if (n <= 1) return 1;
    
    // Recursive case
    return n * Factorial(n - 1);
}

Console.WriteLine(Factorial(5));   // 120 (5×4×3×2×1)
Console.WriteLine(Factorial(10));  // 3628800

// Fibonacci
long Fibonacci(int n)
{
    if (n <= 1) return n;
    return Fibonacci(n - 1) + Fibonacci(n - 2);
}

for (int i = 0; i < 10; i++)
    Console.Write($"{Fibonacci(i)} ");  // 0 1 1 2 3 5 8 13 21 34

// ⚠️ Fibonacci แบบ recursive ช้ามากสำหรับ n ใหญ่!
// ✅ ใช้ Memoization แทน
Dictionary<int, long> memo = new();
long FibMemo(int n)
{
    if (n <= 1) return n;
    if (memo.ContainsKey(n)) return memo[n];
    
    memo[n] = FibMemo(n - 1) + FibMemo(n - 2);
    return memo[n];
}

Console.WriteLine(FibMemo(50));  // เร็วมาก!
```

### Recursion กับ Directory/Tree
```csharp
// นับไฟล์ใน Directory แบบ recursive
int CountFiles(string path)
{
    if (!Directory.Exists(path)) return 0;
    
    int count = Directory.GetFiles(path).Length;
    
    foreach (string subDir in Directory.GetDirectories(path))
    {
        count += CountFiles(subDir);  // Recursive call
    }
    
    return count;
}

// Binary Search แบบ recursive
int BinarySearch(int[] arr, int target, int left, int right)
{
    if (left > right) return -1;  // Not found
    
    int mid = (left + right) / 2;
    
    if (arr[mid] == target) return mid;
    if (arr[mid] < target) return BinarySearch(arr, target, mid + 1, right);
    return BinarySearch(arr, target, left, mid - 1);
}

int[] sorted = { 1, 3, 5, 7, 9, 11, 13, 15, 17, 19 };
int index = BinarySearch(sorted, 11, 0, sorted.Length - 1);
Console.WriteLine($"พบ 11 ที่ index {index}");  // 5

// Merge Sort
int[] MergeSort(int[] arr)
{
    if (arr.Length <= 1) return arr;
    
    int mid = arr.Length / 2;
    int[] left = MergeSort(arr[..mid]);   // Range operator (C# 8+)
    int[] right = MergeSort(arr[mid..]);
    
    return Merge(left, right);
}

int[] Merge(int[] left, int[] right)
{
    var result = new int[left.Length + right.Length];
    int i = 0, j = 0, k = 0;
    
    while (i < left.Length && j < right.Length)
    {
        if (left[i] <= right[j])
            result[k++] = left[i++];
        else
            result[k++] = right[j++];
    }
    
    while (i < left.Length) result[k++] = left[i++];
    while (j < right.Length) result[k++] = right[j++];
    
    return result;
}

int[] unsorted = { 64, 34, 25, 12, 22, 11, 90 };
int[] sorted2 = MergeSort(unsorted);
Console.WriteLine(string.Join(", ", sorted2));  // 11, 12, 22, 25, 34, 64, 90
```

---

## ขั้นตอนที่ 48: Method Chaining และ Fluent Interface

### Method Chaining
```csharp
// Builder Pattern ด้วย Method Chaining
class EmailBuilder
{
    private string _to = "";
    private string _subject = "";
    private string _body = "";
    private List<string> _cc = new();
    private bool _isHtml = false;
    
    public EmailBuilder To(string email)
    {
        _to = email;
        return this;  // return this เพื่อ chain
    }
    
    public EmailBuilder Subject(string subject)
    {
        _subject = subject;
        return this;
    }
    
    public EmailBuilder Body(string body)
    {
        _body = body;
        return this;
    }
    
    public EmailBuilder CC(params string[] emails)
    {
        _cc.AddRange(emails);
        return this;
    }
    
    public EmailBuilder AsHtml()
    {
        _isHtml = true;
        return this;
    }
    
    public string Build()
    {
        string result = $"To: {_to}\nSubject: {_subject}\n";
        if (_cc.Count > 0) result += $"CC: {string.Join(", ", _cc)}\n";
        result += $"HTML: {_isHtml}\n\n{_body}";
        return result;
    }
}

// การใช้งานแบบ Method Chaining
string email = new EmailBuilder()
    .To("user@example.com")
    .Subject("ทดสอบ")
    .CC("manager@example.com", "hr@example.com")
    .Body("<h1>สวัสดี</h1>")
    .AsHtml()
    .Build();

Console.WriteLine(email);
```

---

## ขั้นตอนที่ 49: Async Methods (Preview)

### async/await พื้นฐาน (จะเรียนละเอียดใน Part 17)
```csharp
using System.Threading.Tasks;

// Async method ต้องมี Task หรือ Task<T> เป็น return type
async Task FetchDataAsync()
{
    Console.WriteLine("เริ่มดึงข้อมูล...");
    await Task.Delay(1000);  // จำลองการรอ (1 วินาที)
    Console.WriteLine("ดึงข้อมูลเสร็จแล้ว!");
}

async Task<string> GetUserNameAsync(int userId)
{
    await Task.Delay(500);  // จำลองการ query database
    return $"User_{userId}";
}

// เรียกใช้
await FetchDataAsync();
string name = await GetUserNameAsync(42);
Console.WriteLine(name);  // User_42
```

---

## ขั้นตอนที่ 50: โปรแกรมตัวอย่าง - Library Management System

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace LibraryManagement
{
    record Book(
        int Id,
        string Title,
        string Author,
        string Genre,
        int Year,
        bool IsAvailable = true
    );
    
    class Library
    {
        private List<Book> _books = new();
        private int _nextId = 1;
        
        // Add book
        public Book AddBook(string title, string author, string genre, int year)
        {
            var book = new Book(_nextId++, title, author, genre, year);
            _books.Add(book);
            return book;
        }
        
        // Search by title (overloaded)
        public IEnumerable<Book> Search(string title)
        {
            return _books.Where(b => 
                b.Title.Contains(title, StringComparison.OrdinalIgnoreCase));
        }
        
        // Search by author
        public IEnumerable<Book> SearchByAuthor(string author)
        {
            return _books.Where(b => 
                b.Author.Contains(author, StringComparison.OrdinalIgnoreCase));
        }
        
        // Search by multiple criteria
        public IEnumerable<Book> Search(
            string? title = null,
            string? author = null,
            string? genre = null,
            int? year = null,
            bool? available = null)
        {
            var query = _books.AsEnumerable();
            
            if (title != null)
                query = query.Where(b => b.Title.Contains(title, 
                    StringComparison.OrdinalIgnoreCase));
            if (author != null)
                query = query.Where(b => b.Author.Contains(author, 
                    StringComparison.OrdinalIgnoreCase));
            if (genre != null)
                query = query.Where(b => b.Genre.Equals(genre, 
                    StringComparison.OrdinalIgnoreCase));
            if (year != null)
                query = query.Where(b => b.Year == year);
            if (available != null)
                query = query.Where(b => b.IsAvailable == available);
            
            return query;
        }
        
        // Borrow book
        public (bool success, string message) BorrowBook(int bookId)
        {
            var book = _books.FirstOrDefault(b => b.Id == bookId);
            
            if (book == null)
                return (false, "ไม่พบหนังสือ");
            
            if (!book.IsAvailable)
                return (false, "หนังสือถูกยืมไปแล้ว");
            
            int index = _books.IndexOf(book);
            _books[index] = book with { IsAvailable = false };
            
            return (true, $"ยืมหนังสือ '{book.Title}' สำเร็จ");
        }
        
        // Return book
        public (bool success, string message) ReturnBook(int bookId)
        {
            var book = _books.FirstOrDefault(b => b.Id == bookId);
            
            if (book == null)
                return (false, "ไม่พบหนังสือ");
            
            if (book.IsAvailable)
                return (false, "หนังสือยังไม่ได้ถูกยืม");
            
            int index = _books.IndexOf(book);
            _books[index] = book with { IsAvailable = true };
            
            return (true, $"คืนหนังสือ '{book.Title}' สำเร็จ");
        }
        
        // Statistics
        public (int total, int available, int borrowed) GetStats()
        {
            int available = _books.Count(b => b.IsAvailable);
            return (_books.Count, available, _books.Count - available);
        }
        
        // Display all books
        public void DisplayBooks(IEnumerable<Book> books, string header = "รายการหนังสือ")
        {
            Console.WriteLine($"\n=== {header} ===");
            
            var bookList = books.ToList();
            if (!bookList.Any())
            {
                Console.WriteLine("ไม่พบหนังสือ");
                return;
            }
            
            Console.WriteLine($"{"ID",-4} {"ชื่อหนังสือ",-30} {"ผู้แต่ง",-20} {"ปี",-6} {"สถานะ",-10}");
            Console.WriteLine(new string('-', 75));
            
            foreach (var book in bookList)
            {
                string title = book.Title.Length > 28 ? book.Title[..28] + ".." : book.Title;
                string author = book.Author.Length > 18 ? book.Author[..18] + ".." : book.Author;
                
                Console.ForegroundColor = book.IsAvailable ? ConsoleColor.White : ConsoleColor.DarkGray;
                Console.WriteLine($"{book.Id,-4} {title,-30} {author,-20} {book.Year,-6} {(book.IsAvailable ? "✅ ว่าง" : "❌ ถูกยืม"),-10}");
                Console.ResetColor();
            }
            
            Console.WriteLine(new string('-', 75));
            Console.WriteLine($"รวม: {bookList.Count} เล่ม");
        }
    }
    
    class Program
    {
        static Library library = new Library();
        
        static void Main(string[] args)
        {
            Console.OutputEncoding = System.Text.Encoding.UTF8;
            
            // เพิ่มหนังสือตัวอย่าง
            InitializeBooks();
            
            bool running = true;
            while (running)
            {
                ShowMenu();
                string? choice = Console.ReadLine();
                
                switch (choice)
                {
                    case "1": ShowAllBooks(); break;
                    case "2": SearchBooks(); break;
                    case "3": BorrowBook(); break;
                    case "4": ReturnBook(); break;
                    case "5": AddBook(); break;
                    case "6": ShowStats(); break;
                    case "0": running = false; break;
                    default: Console.WriteLine("เลือกไม่ถูกต้อง"); break;
                }
                
                if (running)
                {
                    Console.WriteLine("\nกด Enter เพื่อดำเนินการต่อ...");
                    Console.ReadLine();
                }
            }
        }
        
        static void InitializeBooks()
        {
            library.AddBook("Clean Code", "Robert C. Martin", "Programming", 2008);
            library.AddBook("The Pragmatic Programmer", "Andy Hunt", "Programming", 1999);
            library.AddBook("Design Patterns", "Gang of Four", "Programming", 1994);
            library.AddBook("Refactoring", "Martin Fowler", "Programming", 1999);
            library.AddBook("Introduction to Algorithms", "Cormen et al.", "Algorithms", 2001);
            library.AddBook("C# in Depth", "Jon Skeet", "Programming", 2019);
            library.AddBook("Pro C# 10", "Andrew Troelsen", "Programming", 2022);
            library.AddBook("Domain-Driven Design", "Eric Evans", "Architecture", 2003);
        }
        
        static void ShowMenu()
        {
            Console.Clear();
            Console.ForegroundColor = ConsoleColor.Cyan;
            Console.WriteLine("╔═══════════════════════════════╗");
            Console.WriteLine("║    📚 ระบบจัดการห้องสมุด      ║");
            Console.WriteLine("╠═══════════════════════════════╣");
            Console.ForegroundColor = ConsoleColor.White;
            Console.WriteLine("║  1. ดูหนังสือทั้งหมด          ║");
            Console.WriteLine("║  2. ค้นหาหนังสือ               ║");
            Console.WriteLine("║  3. ยืมหนังสือ                 ║");
            Console.WriteLine("║  4. คืนหนังสือ                 ║");
            Console.WriteLine("║  5. เพิ่มหนังสือ               ║");
            Console.WriteLine("║  6. สถิติ                      ║");
            Console.WriteLine("║  0. ออก                        ║");
            Console.ForegroundColor = ConsoleColor.Cyan;
            Console.WriteLine("╚═══════════════════════════════╝");
            Console.ResetColor();
            Console.Write("เลือก: ");
        }
        
        static void ShowAllBooks()
        {
            // ใช้ method overloading ที่สร้างไว้
            var allBooks = library.Search();  // ไม่ระบุ criteria = ดูทั้งหมด
            library.DisplayBooks(allBooks);
        }
        
        static void SearchBooks()
        {
            Console.Write("ค้นหาด้วย (1=ชื่อ, 2=ผู้แต่ง, 3=ขั้นสูง): ");
            string? searchType = Console.ReadLine();
            
            IEnumerable<Book> results;
            
            switch (searchType)
            {
                case "1":
                    Console.Write("ชื่อหนังสือ: ");
                    string? title = Console.ReadLine();
                    results = library.Search(title ?? "");
                    library.DisplayBooks(results, $"ผลการค้นหา: '{title}'");
                    break;
                    
                case "2":
                    Console.Write("ชื่อผู้แต่ง: ");
                    string? author = Console.ReadLine();
                    results = library.SearchByAuthor(author ?? "");
                    library.DisplayBooks(results, $"ผู้แต่ง: '{author}'");
                    break;
                    
                case "3":
                    Console.Write("ประเภท (เช่น Programming): ");
                    string? genre = Console.ReadLine();
                    Console.Write("ปีพิมพ์ (เว้นว่างถ้าไม่ระบุ): ");
                    int.TryParse(Console.ReadLine(), out int year);
                    
                    results = library.Search(genre: genre, year: year == 0 ? null : year);
                    library.DisplayBooks(results, "ผลการค้นหาขั้นสูง");
                    break;
            }
        }
        
        static void BorrowBook()
        {
            Console.Write("ระบุ ID หนังสือที่ต้องการยืม: ");
            if (!int.TryParse(Console.ReadLine(), out int id))
            {
                Console.WriteLine("ID ไม่ถูกต้อง");
                return;
            }
            
            var (success, message) = library.BorrowBook(id);
            
            Console.ForegroundColor = success ? ConsoleColor.Green : ConsoleColor.Red;
            Console.WriteLine(success ? $"✅ {message}" : $"❌ {message}");
            Console.ResetColor();
        }
        
        static void ReturnBook()
        {
            Console.Write("ระบุ ID หนังสือที่ต้องการคืน: ");
            if (!int.TryParse(Console.ReadLine(), out int id))
            {
                Console.WriteLine("ID ไม่ถูกต้อง");
                return;
            }
            
            var (success, message) = library.ReturnBook(id);
            
            Console.ForegroundColor = success ? ConsoleColor.Green : ConsoleColor.Red;
            Console.WriteLine(success ? $"✅ {message}" : $"❌ {message}");
            Console.ResetColor();
        }
        
        static void AddBook()
        {
            Console.WriteLine("=== เพิ่มหนังสือใหม่ ===");
            
            Console.Write("ชื่อหนังสือ: ");
            string? title = Console.ReadLine();
            if (string.IsNullOrWhiteSpace(title)) { Console.WriteLine("ชื่อหนังสือต้องไม่ว่าง"); return; }
            
            Console.Write("ผู้แต่ง: ");
            string? author = Console.ReadLine();
            if (string.IsNullOrWhiteSpace(author)) { Console.WriteLine("ผู้แต่งต้องไม่ว่าง"); return; }
            
            Console.Write("ประเภท: ");
            string genre = Console.ReadLine() ?? "ทั่วไป";
            
            Console.Write("ปีพิมพ์: ");
            int.TryParse(Console.ReadLine(), out int year);
            if (year == 0) year = DateTime.Now.Year;
            
            var book = library.AddBook(title, author, genre, year);
            
            Console.ForegroundColor = ConsoleColor.Green;
            Console.WriteLine($"✅ เพิ่มหนังสือ ID {book.Id}: '{book.Title}' สำเร็จ");
            Console.ResetColor();
        }
        
        static void ShowStats()
        {
            var (total, available, borrowed) = library.GetStats();
            
            Console.WriteLine("\n=== สถิติห้องสมุด ===");
            Console.WriteLine($"หนังสือทั้งหมด: {total} เล่ม");
            Console.ForegroundColor = ConsoleColor.Green;
            Console.WriteLine($"ว่าง: {available} เล่ม ({(double)available / total:P0})");
            Console.ForegroundColor = ConsoleColor.Yellow;
            Console.WriteLine($"ถูกยืม: {borrowed} เล่ม ({(double)borrowed / total:P0})");
            Console.ResetColor();
        }
    }
}
```

---

## 📝 สรุป Part 05

| หัวข้อ | รายละเอียดสำคัญ |
|--------|----------------|
| Method Declaration | access modifier, return type, parameters |
| Value Parameters | ส่ง copy - ไม่กระทบต้นฉบับ |
| ref Parameters | ส่ง reference - กระทบต้นฉบับ |
| out Parameters | Method ต้องกำหนดค่า - ใช้ return หลายค่า |
| in Parameters | Read-only reference - performance |
| params | รับ arguments จำนวนไม่แน่นอน |
| Optional Parameters | Default values สำหรับ parameters |
| Named Arguments | ระบุชื่อ parameter ตอนเรียก |
| Method Overloading | ชื่อเดิม signature ต่างกัน |
| Expression Body | => สำหรับ methods สั้นๆ |
| Local Functions | Functions ภายใน Methods |
| Recursion | Method เรียกตัวเอง + base case |

---

## 🏋️ แบบฝึกหัด Part 05

### แบบฝึกหัดที่ 1: Math Utilities
สร้าง class `MathUtils` ที่มี static methods:
- `int GCD(int a, int b)` - หา Greatest Common Divisor
- `int LCM(int a, int b)` - หา Least Common Multiple
- `bool IsPrime(int n)` - ตรวจสอบจำนวนเฉพาะ
- `int[] GetPrimes(int max)` - หาจำนวนเฉพาะทั้งหมด ≤ max

### แบบฝึกหัดที่ 2: String Utilities
สร้าง static methods:
- `string Reverse(string text)` - กลับ string
- `bool IsPalindrome(string text)` - ตรวจสอบ Palindrome
- `int CountWords(string text)` - นับคำ
- `string TitleCase(string text)` - แปลงเป็น Title Case

### แบบฝึกหัดที่ 3: Recursive Tower of Hanoi
สร้างโปรแกรม Tower of Hanoi:
- รับจำนวน disk
- แสดงขั้นตอนการย้าย disk ทีละขั้น
- นับจำนวนการเคลื่อนย้ายทั้งหมด

---

**ก่อนหน้า → [Part 04: Loops](part04-loops.md)**  
**ต่อไป → [Part 06: Arrays & Collections](../part01-10/part06-arrays-collections.md)**
