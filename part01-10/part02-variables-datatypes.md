# Part 02: Variables, Data Types & Operators
## ขั้นตอนที่ 11-20: ตัวแปร ชนิดข้อมูล และตัวดำเนินการ

---

## 🎯 เป้าหมายของ Part นี้
- เข้าใจตัวแปรและการประกาศตัวแปรใน C#
- รู้จัก Value Types และ Reference Types
- ใช้ตัวดำเนินการทางคณิตศาสตร์และตรรกะ
- เข้าใจการแปลงชนิดข้อมูล (Type Conversion)
- รู้จัก Constants และ var keyword

---

## ขั้นตอนที่ 11: ตัวแปร (Variables)

### ตัวแปรคืออะไร?
ตัวแปรคือพื้นที่ในหน่วยความจำที่ใช้เก็บข้อมูล มีชื่อที่ใช้อ้างอิง และมีชนิดข้อมูล (type) กำกับ

### การประกาศตัวแปร (Variable Declaration)
```csharp
// รูปแบบ: [ชนิดข้อมูล] [ชื่อตัวแปร];
int age;           // ประกาศตัวแปรชนิด int ชื่อ age
string name;       // ประกาศตัวแปรชนิด string ชื่อ name
double salary;     // ประกาศตัวแปรชนิด double ชื่อ salary
bool isActive;     // ประกาศตัวแปรชนิด bool ชื่อ isActive

// การประกาศพร้อมกำหนดค่า (Declaration + Initialization)
int age = 25;
string name = "สมชาย";
double salary = 35000.50;
bool isActive = true;

// การประกาศหลายตัวแปรในบรรทัดเดียว (ไม่แนะนำ)
int x = 1, y = 2, z = 3;
```

### กฎการตั้งชื่อตัวแปร (Naming Rules)
```csharp
// ✅ ถูกต้อง
int age = 25;
string firstName = "สมชาย";      // camelCase (แนะนำ)
double totalAmount = 1000.0;
bool isValid = true;
int _privateField = 0;           // ขึ้นต้นด้วย _ ก็ได้

// ✅ ใช้ได้ แต่ไม่แนะนำ
int Age = 25;                    // PascalCase (ใช้กับ Properties/Methods)
string CONSTANT_NAME = "value";  // UPPER_CASE (ใช้กับ constants บางครั้ง)

// ❌ ผิด - ไม่สามารถใช้ได้
// int 1age = 25;     // ขึ้นต้นด้วยตัวเลขไม่ได้
// int my-age = 25;   // มี - ไม่ได้
// int int = 25;      // ใช้ keyword ไม่ได้
// int my age = 25;   // มี space ไม่ได้
```

### Naming Conventions ใน C#
```csharp
// camelCase - ตัวแปร local และ parameters
int myAge = 25;
string firstName = "John";
double totalPrice = 999.99;

// PascalCase - Classes, Methods, Properties, Events
public class CustomerOrder { }
public void CalculateTotal() { }
public string FirstName { get; set; }

// _camelCase - Private fields (underscore prefix)
private int _count = 0;
private string _name = "";

// ALL_CAPS - Constants (บางทีม)
const double PI = 3.14159;
const int MAX_SIZE = 100;

// IPascalCase - Interfaces
public interface IRepository { }
public interface IDisposable { }
```

---

## ขั้นตอนที่ 12: Value Types - ชนิดข้อมูลแบบค่า

### Integral Types (จำนวนเต็ม)
```csharp
// sbyte: -128 ถึง 127 (1 byte)
sbyte smallValue = -100;

// byte: 0 ถึง 255 (1 byte)
byte byteValue = 200;

// short: -32,768 ถึง 32,767 (2 bytes)
short shortValue = -1000;

// ushort: 0 ถึง 65,535 (2 bytes)
ushort ushortValue = 60000;

// int: -2,147,483,648 ถึง 2,147,483,647 (4 bytes) ← ใช้บ่อยที่สุด
int number = 1000000;
int negativeNumber = -500000;

// uint: 0 ถึง 4,294,967,295 (4 bytes)
uint positiveOnly = 4000000000;

// long: -9.2 × 10^18 ถึง 9.2 × 10^18 (8 bytes)
long bigNumber = 9000000000L;  // ใส่ L suffix

// ulong: 0 ถึง 18.4 × 10^18 (8 bytes)
ulong veryBigNumber = 18000000000000000000UL;

// แสดงขนาดและช่วงของ int
Console.WriteLine($"int.MinValue: {int.MinValue}");    // -2147483648
Console.WriteLine($"int.MaxValue: {int.MaxValue}");    // 2147483647
Console.WriteLine($"Size: {sizeof(int)} bytes");       // 4

// Numeric Literals (C# 7+)
int million = 1_000_000;          // underscore separator ช่วยอ่านง่าย
int binary = 0b1010_1010;         // Binary literal (0b prefix)
int hex = 0xFF_FF;                // Hexadecimal (0x prefix)
```

### Floating-Point Types (ทศนิยม)
```csharp
// float: ±1.5 × 10^-45 ถึง ±3.4 × 10^38 (4 bytes, ~7 digits precision)
float temperature = 36.5f;  // ต้องใส่ f suffix
float pi = 3.14159f;

// double: ±5.0 × 10^-324 ถึง ±1.7 × 10^308 (8 bytes, ~15-17 digits) ← ใช้บ่อย
double price = 1999.99;
double bigDecimal = 1.23456789012345;

// decimal: ±1.0 × 10^-28 ถึง ±7.9 × 10^28 (16 bytes, ~28-29 digits)
// ← ใช้กับเงิน/การเงิน เพราะแม่นยำที่สุด
decimal moneyAmount = 1234567890.99m;  // ต้องใส่ m suffix
decimal exactValue = 0.1m + 0.2m;     // = 0.3 (ถูกต้อง!)

// ⚠️ ความแตกต่าง float/double กับ decimal
double d = 0.1 + 0.2;
Console.WriteLine(d);          // 0.30000000000000004 (ไม่แม่นยำ!)

decimal dec = 0.1m + 0.2m;
Console.WriteLine(dec);        // 0.3 (แม่นยำ!)

// Special Values
double infinity = double.PositiveInfinity;
double negInfinity = double.NegativeInfinity;
double notANumber = double.NaN;
Console.WriteLine(double.IsNaN(notANumber));  // True
```

### Boolean Type
```csharp
// bool: true หรือ false เท่านั้น (1 byte)
bool isLoggedIn = true;
bool hasPermission = false;
bool isAdult = age >= 18;

// Boolean operations
bool result1 = true && false;   // AND: false
bool result2 = true || false;   // OR: true
bool result3 = !true;           // NOT: false
bool result4 = true ^ false;    // XOR: true (exclusive or)
```

### Character Type
```csharp
// char: ตัวอักขระ Unicode เดี่ยว (2 bytes)
char letter = 'A';
char digit = '5';
char thai = 'ก';
char emoji = '😀';
char newline = '\n';    // escape sequence
char tab = '\t';
char quote = '\'';
char backslash = '\\';
char unicode = 'A';  // Unicode: 'A'

// char operations
Console.WriteLine((int)'A');      // 65 (ASCII code)
Console.WriteLine((char)65);      // A
Console.WriteLine(char.IsLetter('A'));    // True
Console.WriteLine(char.IsDigit('5'));     // True
Console.WriteLine(char.ToUpper('a'));     // A
Console.WriteLine(char.ToLower('Z'));     // z
```

### Struct Types
```csharp
// DateTime - วันและเวลา
DateTime now = DateTime.Now;
DateTime today = DateTime.Today;
DateTime birthday = new DateTime(1995, 1, 15);
DateTime specificTime = new DateTime(2024, 12, 25, 10, 30, 0);

Console.WriteLine(now.Year);           // 2026
Console.WriteLine(now.Month);          // 10
Console.WriteLine(now.Day);            // 2
Console.WriteLine(now.Hour);           // ชั่วโมง
Console.WriteLine(now.ToString("dd/MM/yyyy HH:mm:ss"));

// TimeSpan - ช่วงเวลา
TimeSpan duration = new TimeSpan(2, 30, 0);  // 2 ชั่วโมง 30 นาที
TimeSpan diff = DateTime.Now - birthday;
Console.WriteLine($"อายุ: {diff.Days / 365} ปี");

// Guid - Globally Unique Identifier
Guid id = Guid.NewGuid();
Console.WriteLine(id);  // เช่น: a3b4c5d6-e7f8-90ab-cdef-123456789012
```

---

## ขั้นตอนที่ 13: Reference Types - ชนิดข้อมูลแบบอ้างอิง

### String Type
```csharp
// string: sequence ของ char (reference type แต่ทำงานคล้าย value type)
string greeting = "Hello, World!";
string name = "สมชาย วิชาการ";
string empty = "";
string nullString = null;       // string สามารถเป็น null ได้
string? nullableString = null;  // Nullable string (C# 8+)

// String properties
Console.WriteLine(greeting.Length);          // 13
Console.WriteLine(greeting.ToUpper());       // HELLO, WORLD!
Console.WriteLine(greeting.ToLower());       // hello, world!
Console.WriteLine(greeting.Trim());          // ลบ whitespace หัวท้าย

// String methods ที่ใช้บ่อย
string text = "  Hello, C# World!  ";

// Trim - ลบ whitespace
Console.WriteLine(text.Trim());          // "Hello, C# World!"
Console.WriteLine(text.TrimStart());     // "Hello, C# World!  "
Console.WriteLine(text.TrimEnd());       // "  Hello, C# World!"

// Contains, StartsWith, EndsWith
Console.WriteLine(text.Contains("C#"));         // True
Console.WriteLine(text.StartsWith("  Hello"));  // True
Console.WriteLine(text.EndsWith("!  "));        // True

// IndexOf, Substring
string sentence = "The quick brown fox";
int index = sentence.IndexOf("quick");      // 4
string sub = sentence.Substring(4, 5);      // "quick"
string sub2 = sentence.Substring(10);       // "brown fox"

// Replace
string replaced = sentence.Replace("fox", "cat");
Console.WriteLine(replaced);  // "The quick brown cat"

// Split
string csv = "apple,banana,cherry,date";
string[] fruits = csv.Split(',');
foreach (string fruit in fruits)
{
    Console.WriteLine(fruit);  // apple, banana, cherry, date
}

// Join
string joined = string.Join(" | ", fruits);
Console.WriteLine(joined);  // "apple | banana | cherry | date"

// String.IsNullOrEmpty, IsNullOrWhiteSpace
Console.WriteLine(string.IsNullOrEmpty(""));       // True
Console.WriteLine(string.IsNullOrEmpty(null));     // True
Console.WriteLine(string.IsNullOrEmpty("hello")); // False
Console.WriteLine(string.IsNullOrWhiteSpace("  ")); // True (รวม whitespace ด้วย)
```

### String Interpolation และ Formatting
```csharp
string name = "สมชาย";
int age = 25;
double salary = 35000.50;
DateTime birthdate = new DateTime(1999, 5, 15);

// String Interpolation
Console.WriteLine($"ชื่อ: {name}, อายุ: {age} ปี");
Console.WriteLine($"เงินเดือน: {salary:C2}");        // Currency format
Console.WriteLine($"เงินเดือน: {salary:N2}");        // Number with 2 decimals
Console.WriteLine($"เปอร์เซ็นต์: {0.856:P1}");      // 85.6%
Console.WriteLine($"วันเกิด: {birthdate:dd MMMM yyyy}");
Console.WriteLine($"เลขฐาน 16: {255:X}");           // FF

// Multiline string interpolation
string report = $"""
    รายงานพนักงาน
    ชื่อ: {name}
    อายุ: {age} ปี
    เงินเดือน: {salary:N2} บาท
    """;
Console.WriteLine(report);
```

### Object Type
```csharp
// object: base type ของทุก type ใน C#
object anyValue;
anyValue = 42;          // int
anyValue = "hello";     // string
anyValue = 3.14;        // double
anyValue = true;        // bool

// Boxing - แปลง value type เป็น object (reference type)
int num = 42;
object boxed = num;     // Boxing

// Unboxing - แปลง object กลับเป็น value type
int unboxed = (int)boxed;  // Unboxing (ต้อง cast)

Console.WriteLine(anyValue.GetType().Name);  // ชนิดข้อมูลจริงๆ
```

---

## ขั้นตอนที่ 14: var และ Type Inference

### var keyword
```csharp
// var - compiler จะ infer type จากค่าที่กำหนด
var number = 42;            // int
var name = "สมชาย";        // string
var pi = 3.14159;          // double
var isValid = true;         // bool
var today = DateTime.Now;  // DateTime

// ⚠️ var ต้องกำหนดค่าตอนประกาศ
// var x;  ← Error! ต้องกำหนดค่า

// ตรวจสอบ type
Console.WriteLine(number.GetType().Name);  // Int32
Console.WriteLine(name.GetType().Name);    // String

// เมื่อควรใช้ var?
// ✅ เมื่อ type ชัดเจนจากการอ่าน
var customers = new List<Customer>();
var result = GetQueryResult();

// ❌ เมื่อ type ไม่ชัดเจน
// var x = GetValue();  ← ไม่รู้ว่า GetValue() return อะไร
```

### dynamic keyword
```csharp
// dynamic - type จะถูกตรวจสอบตอน runtime (ช้ากว่า var)
dynamic dyn = 42;
Console.WriteLine(dyn + 8);  // 50

dyn = "Hello";
Console.WriteLine(dyn.Length);  // 5

dyn = new { Name = "John", Age = 25 };
Console.WriteLine(dyn.Name);  // John

// ⚠️ Error จะเกิดตอน runtime ไม่ใช่ compile time
// dyn.NonExistentMethod();  ← RuntimeBinderException
```

---

## ขั้นตอนที่ 15: Constants และ Readonly

### Constants
```csharp
// const - ค่าคงที่ที่กำหนดตอน compile time
const double PI = 3.14159265358979;
const int MAX_SIZE = 100;
const string APP_NAME = "My Application";
const decimal TAX_RATE = 0.07m;

// ✅ ใช้ const กับ: numbers, strings, booleans
// ❌ ไม่สามารถใช้ const กับ objects หรือ DateTime

// Readonly - อ่านได้อย่างเดียว (กำหนดค่าได้ใน constructor)
class Configuration
{
    public readonly string ConnectionString;
    public readonly DateTime StartTime;
    
    public Configuration(string connectionString)
    {
        ConnectionString = connectionString;
        StartTime = DateTime.Now;
        // หลังจาก constructor เสร็จ ไม่สามารถเปลี่ยน readonly ได้
    }
}
```

### Static vs Instance
```csharp
class Counter
{
    // Instance field - แต่ละ object มีของตัวเอง
    int instanceCount = 0;
    
    // Static field - ใช้ร่วมกันทุก object ของ class นี้
    static int totalCount = 0;
    
    // Static readonly - กำหนดได้ครั้งเดียว
    static readonly string ClassName = "Counter";
    
    public void Increment()
    {
        instanceCount++;
        totalCount++;
    }
    
    public static int GetTotalCount() => totalCount;
}

// การใช้งาน
Counter c1 = new Counter();
Counter c2 = new Counter();
c1.Increment();
c1.Increment();
c2.Increment();

// Counter.totalCount = 3 (ทุก object ใช้ร่วมกัน)
```

---

## ขั้นตอนที่ 16: Operators - ตัวดำเนินการ

### Arithmetic Operators (ตัวดำเนินการทางคณิตศาสตร์)
```csharp
int a = 10, b = 3;

// พื้นฐาน
Console.WriteLine(a + b);   // 13 - บวก
Console.WriteLine(a - b);   // 7  - ลบ
Console.WriteLine(a * b);   // 30 - คูณ
Console.WriteLine(a / b);   // 3  - หาร (ผลลัพธ์เป็น int - ตัดทศนิยม)
Console.WriteLine(a % b);   // 1  - หารเอาเศษ (modulo)

// หารได้ผลทศนิยม
double c = 10.0, d = 3.0;
Console.WriteLine(c / d);   // 3.3333...

// หรือ cast ก่อนหาร
Console.WriteLine((double)a / b);  // 3.3333...

// Increment/Decrement
int x = 5;
x++;        // x = 6 (post-increment)
++x;        // x = 7 (pre-increment)
x--;        // x = 6 (post-decrement)
--x;        // x = 5 (pre-decrement)

// ความแตกต่าง post และ pre
int y = 5;
int postResult = y++;   // postResult = 5, y = 6 (ใช้ค่าเดิมก่อน แล้วค่อยเพิ่ม)
int preResult = ++y;    // preResult = 7, y = 7 (เพิ่มก่อน แล้วค่อยใช้)

// Math operations
Console.WriteLine(Math.Pow(2, 10));     // 1024 (2^10)
Console.WriteLine(Math.Sqrt(144));      // 12 (square root)
Console.WriteLine(Math.Abs(-42));       // 42 (absolute value)
Console.WriteLine(Math.Round(3.567, 2)); // 3.57
Console.WriteLine(Math.Floor(3.9));     // 3 (ปัดลง)
Console.WriteLine(Math.Ceiling(3.1));   // 4 (ปัดขึ้น)
Console.WriteLine(Math.Min(10, 20));    // 10
Console.WriteLine(Math.Max(10, 20));    // 20
```

### Comparison Operators (ตัวดำเนินการเปรียบเทียบ)
```csharp
int a = 10, b = 20;

Console.WriteLine(a == b);   // False - เท่ากัน
Console.WriteLine(a != b);   // True  - ไม่เท่ากัน
Console.WriteLine(a > b);    // False - มากกว่า
Console.WriteLine(a < b);    // True  - น้อยกว่า
Console.WriteLine(a >= b);   // False - มากกว่าหรือเท่ากัน
Console.WriteLine(a <= b);   // True  - น้อยกว่าหรือเท่ากัน

// String comparison
string s1 = "Hello";
string s2 = "hello";
Console.WriteLine(s1 == s2);                     // False (case-sensitive)
Console.WriteLine(s1.Equals(s2, StringComparison.OrdinalIgnoreCase));  // True
```

### Logical Operators (ตัวดำเนินการตรรกะ)
```csharp
bool a = true, b = false;

// && (AND) - true ถ้าทั้งสองเป็น true
Console.WriteLine(a && b);   // False
Console.WriteLine(a && a);   // True

// || (OR) - true ถ้าอย่างน้อยหนึ่งเป็น true
Console.WriteLine(a || b);   // True
Console.WriteLine(b || b);   // False

// ! (NOT) - กลับค่า
Console.WriteLine(!a);       // False
Console.WriteLine(!b);       // True

// ^ (XOR) - true ถ้าต่างกัน
Console.WriteLine(a ^ b);    // True
Console.WriteLine(a ^ a);    // False

// Short-circuit evaluation
// && และ || จะหยุดประเมินเมื่อรู้คำตอบแล้ว
bool result = IsValid() && CheckPermission();
// ถ้า IsValid() = false จะไม่เรียก CheckPermission()

bool canAccess = HasAccount() || CreateAccount();
// ถ้า HasAccount() = true จะไม่เรียก CreateAccount()
```

### Assignment Operators
```csharp
int x = 10;

x += 5;    // x = x + 5 = 15
x -= 3;    // x = x - 3 = 12
x *= 2;    // x = x * 2 = 24
x /= 4;    // x = x / 4 = 6
x %= 4;    // x = x % 4 = 2

// C# 8+ Null-coalescing assignment
string? name = null;
name ??= "ค่าเริ่มต้น";    // name = "ค่าเริ่มต้น" (ถ้า name เป็น null)
Console.WriteLine(name);

string? existing = "มีค่าแล้ว";
existing ??= "ค่าใหม่";    // ไม่เปลี่ยน เพราะ existing ไม่ใช่ null
Console.WriteLine(existing);  // "มีค่าแล้ว"
```

---

## ขั้นตอนที่ 17: Ternary และ Null Operators

### Ternary Operator (? :)
```csharp
int age = 20;

// รูปแบบ: condition ? valueIfTrue : valueIfFalse
string status = age >= 18 ? "บรรลุนิติภาวะ" : "ยังไม่บรรลุนิติภาวะ";
Console.WriteLine(status);  // บรรลุนิติภาวะ

// ซ้อนกัน (ไม่แนะนำ - อ่านยาก)
string grade = age < 13 ? "เด็ก" : age < 18 ? "วัยรุ่น" : "ผู้ใหญ่";

// ดีกว่าใช้ if-else แทน
string grade2;
if (age < 13) grade2 = "เด็ก";
else if (age < 18) grade2 = "วัยรุ่น";
else grade2 = "ผู้ใหญ่";
```

### Null-Coalescing Operator (??)
```csharp
string? name = null;

// ?? - ถ้า left เป็น null ใช้ right
string displayName = name ?? "ไม่ระบุชื่อ";
Console.WriteLine(displayName);  // ไม่ระบุชื่อ

string? firstName = null;
string? lastName = "Smith";
string fullName = firstName ?? lastName ?? "Unknown";
Console.WriteLine(fullName);  // Smith

// Null-conditional operator (?.)
string? text = null;
int? length = text?.Length;         // null (ไม่ throw NullReferenceException)
string? upper = text?.ToUpper();    // null

string? text2 = "Hello";
int? length2 = text2?.Length;       // 5
Console.WriteLine(length2 ?? 0);    // 5

// ?.[] - Null-conditional indexer
int[]? numbers = null;
int? first = numbers?[0];  // null (ไม่ throw IndexOutOfRangeException)

int[]? numbers2 = { 1, 2, 3 };
int? first2 = numbers2?[0];  // 1
```

---

## ขั้นตอนที่ 18: Type Conversion (การแปลงชนิดข้อมูล)

### Implicit Conversion (แปลงอัตโนมัติ - ไม่สูญเสียข้อมูล)
```csharp
// เล็ก → ใหญ่ (implicit)
byte b = 100;
short s = b;       // byte → short (OK)
int i = s;         // short → int (OK)
long l = i;        // int → long (OK)
float f = l;       // long → float (OK)
double d = f;      // float → double (OK)

int num = 1000;
long bigNum = num;  // implicit: int → long
double pi = 3.14;
float fPi = (float)pi;  // explicit: double → float (อาจสูญเสีย precision)
```

### Explicit Conversion / Casting (แปลงแบบบังคับ)
```csharp
// ใหญ่ → เล็ก ต้อง cast (อาจสูญเสียข้อมูล)
double d = 3.99;
int i = (int)d;        // i = 3 (ตัดทศนิยม ไม่ปัดเศษ!)
Console.WriteLine(i);  // 3

long bigNum = 3000000000L;
int smallNum = (int)bigNum;  // อาจ overflow!
Console.WriteLine(smallNum); // ค่าผิด! (-1294967296)

// ตรวจสอบก่อน cast
if (bigNum <= int.MaxValue && bigNum >= int.MinValue)
{
    int safe = (int)bigNum;
}

// checked keyword - throw exception เมื่อ overflow
try
{
    checked
    {
        int overflow = (int)bigNum;  // OverflowException!
    }
}
catch (OverflowException ex)
{
    Console.WriteLine($"Overflow: {ex.Message}");
}
```

### Convert Class
```csharp
// System.Convert - แปลงระหว่าง types ต่างๆ
string strNum = "42";
int parsed = Convert.ToInt32(strNum);    // "42" → 42
double parsedDouble = Convert.ToDouble("3.14");
bool parsedBool = Convert.ToBoolean("true");
string numToStr = Convert.ToString(123);

// ⚠️ Convert.ToInt32(null) = 0 (ไม่ throw exception)
// ⚠️ Convert.ToInt32("abc") = FormatException!
```

### Parse และ TryParse
```csharp
// Parse - แปลง string เป็น type (throw exception ถ้าล้มเหลว)
int num1 = int.Parse("42");
double num2 = double.Parse("3.14");
bool flag = bool.Parse("true");
DateTime date = DateTime.Parse("2024-01-15");

// TryParse - ปลอดภัยกว่า (ไม่ throw exception)
string input = "12345";
if (int.TryParse(input, out int result))
{
    Console.WriteLine($"แปลงสำเร็จ: {result}");
}
else
{
    Console.WriteLine("แปลงไม่ได้");
}

// รับ input จาก user แล้ว parse
Console.Write("ใส่ตัวเลข: ");
string userInput = Console.ReadLine();
if (double.TryParse(userInput, out double number))
{
    Console.WriteLine($"ตัวเลขที่ใส่: {number}");
    Console.WriteLine($"กำลังสอง: {number * number}");
}
else
{
    Console.WriteLine("กรุณาใส่ตัวเลขที่ถูกต้อง");
}
```

### as และ is Operators
```csharp
object obj = "Hello, World!";

// is - ตรวจสอบ type
if (obj is string)
{
    Console.WriteLine("obj เป็น string");
}

// is with pattern matching (C# 7+)
if (obj is string str)
{
    Console.WriteLine($"Length: {str.Length}");
}

// as - cast แบบปลอดภัย (return null ถ้า fail)
string? text = obj as string;      // "Hello, World!"
int[]? nums = obj as int[];        // null (ไม่ใช่ int[])

if (text != null)
{
    Console.WriteLine(text.ToUpper());
}
```

---

## ขั้นตอนที่ 19: Nullable Types

### Nullable Value Types (T?)
```csharp
// Value types ปกติไม่สามารถเป็น null ได้
int age = 25;
// int noValue = null;  ← Error!

// Nullable<T> หรือ T? - ทำให้ value type เป็น null ได้
int? nullableAge = null;
int? age2 = 25;

// ตรวจสอบ null
if (nullableAge.HasValue)
{
    Console.WriteLine(nullableAge.Value);
}
else
{
    Console.WriteLine("ไม่มีค่า");
}

// ?? operator
int displayAge = nullableAge ?? 0;  // 0 ถ้าเป็น null
int displayAge2 = age2 ?? 0;        // 25

// GetValueOrDefault
int val = nullableAge.GetValueOrDefault();     // 0 (default of int)
int val2 = nullableAge.GetValueOrDefault(18);  // 18 (custom default)

// Nullable comparisons
int? x = 5;
int? y = null;
Console.WriteLine(x > y);    // False (any comparison with null = false)
Console.WriteLine(x == null); // False
Console.WriteLine(y == null); // True
```

### Nullable Reference Types (C# 8+)
```csharp
// เปิดใช้ใน .csproj: <Nullable>enable</Nullable>

// ไม่มี ? - ไม่ควรเป็น null (compiler จะเตือน)
string nonNullable = "Hello";
// nonNullable = null;  ← Warning!

// มี ? - สามารถเป็น null ได้
string? nullable = null;
nullable = "World";
nullable = null;  // OK

// Null-forgiving operator (!)
string? text = GetText();
int length = text!.Length;  // บอก compiler ว่าแน่ใจว่าไม่ null
// ⚠️ อันตราย - ถ้า text จริงๆ เป็น null จะ throw NullReferenceException

// Null checks ที่ดี
string? name = GetName();
if (name is not null)
{
    Console.WriteLine(name.Length);  // ปลอดภัย
}

// หรือใช้ ?.
Console.WriteLine(name?.Length ?? 0);
```

---

## ขั้นตอนที่ 20: โปรแกรมตัวอย่างรวมทุกอย่าง

### ตัวอย่าง: ระบบคำนวณคะแนนนักศึกษา

```csharp
using System;

namespace StudentGradeCalculator
{
    class Program
    {
        static void Main(string[] args)
        {
            Console.OutputEncoding = System.Text.Encoding.UTF8;
            Console.Title = "Student Grade Calculator";
            
            // ==========================================
            // รับข้อมูลนักศึกษา
            // ==========================================
            Console.WriteLine("=== ระบบคำนวณเกรดนักศึกษา ===\n");
            
            Console.Write("ชื่อ-นามสกุล: ");
            string? name = Console.ReadLine();
            if (string.IsNullOrWhiteSpace(name)) name = "นักศึกษา";
            
            Console.Write("รหัสนักศึกษา: ");
            string? studentId = Console.ReadLine() ?? "000000";
            
            // ==========================================
            // รับคะแนน
            // ==========================================
            double[] scores = new double[5];
            string[] subjects = { "คณิตศาสตร์", "วิทยาศาสตร์", "ภาษาไทย", "ภาษาอังกฤษ", "สังคม" };
            
            Console.WriteLine("\nกรอกคะแนน (0-100) สำหรับแต่ละวิชา:");
            
            for (int i = 0; i < subjects.Length; i++)
            {
                bool isValid = false;
                while (!isValid)
                {
                    Console.Write($"{subjects[i]}: ");
                    string? input = Console.ReadLine();
                    
                    if (double.TryParse(input, out double score) && 
                        score >= 0 && score <= 100)
                    {
                        scores[i] = score;
                        isValid = true;
                    }
                    else
                    {
                        Console.ForegroundColor = ConsoleColor.Red;
                        Console.WriteLine("กรุณาใส่คะแนนระหว่าง 0-100");
                        Console.ResetColor();
                    }
                }
            }
            
            // ==========================================
            // คำนวณผลลัพธ์
            // ==========================================
            double total = 0;
            double highest = scores[0];
            double lowest = scores[0];
            
            for (int i = 0; i < scores.Length; i++)
            {
                total += scores[i];
                if (scores[i] > highest) highest = scores[i];
                if (scores[i] < lowest) lowest = scores[i];
            }
            
            double average = total / scores.Length;
            
            // คำนวณเกรด
            string grade;
            string gradePoint;
            
            if (average >= 80)
            {
                grade = "A";
                gradePoint = "4.0";
            }
            else if (average >= 75)
            {
                grade = "B+";
                gradePoint = "3.5";
            }
            else if (average >= 70)
            {
                grade = "B";
                gradePoint = "3.0";
            }
            else if (average >= 65)
            {
                grade = "C+";
                gradePoint = "2.5";
            }
            else if (average >= 60)
            {
                grade = "C";
                gradePoint = "2.0";
            }
            else if (average >= 55)
            {
                grade = "D+";
                gradePoint = "1.5";
            }
            else if (average >= 50)
            {
                grade = "D";
                gradePoint = "1.0";
            }
            else
            {
                grade = "F";
                gradePoint = "0.0";
            }
            
            bool isPassed = average >= 50;
            
            // ==========================================
            // แสดงผลลัพธ์
            // ==========================================
            Console.Clear();
            Console.WriteLine("╔══════════════════════════════════════════╗");
            Console.WriteLine("║          ใบรายงานผลการเรียน              ║");
            Console.WriteLine("╚══════════════════════════════════════════╝");
            Console.WriteLine();
            
            Console.WriteLine($"ชื่อ: {name}");
            Console.WriteLine($"รหัส: {studentId}");
            Console.WriteLine($"วันที่: {DateTime.Now:dd/MM/yyyy}");
            Console.WriteLine();
            
            Console.WriteLine("┌─────────────────┬──────────┬─────────┐");
            Console.WriteLine("│ วิชา            │ คะแนน    │ สถานะ   │");
            Console.WriteLine("├─────────────────┼──────────┼─────────┤");
            
            for (int i = 0; i < subjects.Length; i++)
            {
                string status = scores[i] >= 50 ? "ผ่าน" : "ไม่ผ่าน";
                Console.ForegroundColor = scores[i] >= 50 ? 
                    ConsoleColor.Green : ConsoleColor.Red;
                Console.WriteLine($"│ {subjects[i],-15} │ {scores[i],8:F1} │ {status,-7} │");
                Console.ResetColor();
            }
            
            Console.WriteLine("├─────────────────┼──────────┼─────────┤");
            Console.WriteLine($"│ {"รวม",-15} │ {total,8:F1} │         │");
            Console.WriteLine($"│ {"เฉลี่ย",-15} │ {average,8:F2} │         │");
            Console.WriteLine("└─────────────────┴──────────┴─────────┘");
            Console.WriteLine();
            
            // แสดงสถิติ
            Console.WriteLine($"คะแนนสูงสุด: {highest:F1}");
            Console.WriteLine($"คะแนนต่ำสุด: {lowest:F1}");
            Console.WriteLine();
            
            // แสดงเกรด
            Console.Write("เกรด: ");
            Console.ForegroundColor = isPassed ? ConsoleColor.Green : ConsoleColor.Red;
            Console.WriteLine($"{grade} (GPA: {gradePoint})");
            Console.ResetColor();
            
            Console.Write("ผลการเรียน: ");
            Console.ForegroundColor = isPassed ? ConsoleColor.Green : ConsoleColor.Red;
            Console.WriteLine(isPassed ? "✅ ผ่านการประเมิน" : "❌ ไม่ผ่านการประเมิน");
            Console.ResetColor();
            
            Console.WriteLine();
            Console.WriteLine("กด Enter เพื่อออก...");
            Console.ReadLine();
        }
    }
}
```

---

## 📝 สรุป Part 02

| หัวข้อ | รายละเอียดสำคัญ |
|--------|----------------|
| Value Types | int, double, bool, char, struct, enum - เก็บค่าโดยตรง |
| Reference Types | string, object, class, array - เก็บ reference ไปยัง heap |
| var keyword | Compiler infer type อัตโนมัติ ต้องกำหนดค่าตอนประกาศ |
| Constants | const กำหนดตอน compile, readonly กำหนดได้ใน constructor |
| Arithmetic | +, -, *, /, % และ Math class |
| Comparison | ==, !=, >, <, >=, <= |
| Logical | &&, \|\|, !, ^ |
| Null Handling | ??, ?., ??= operators |
| Type Conversion | Implicit, Explicit, Parse, TryParse, Convert |
| Nullable Types | int?, string? สำหรับค่าที่อาจเป็น null |

---

## 🏋️ แบบฝึกหัด Part 02

### แบบฝึกหัดที่ 1: Currency Converter
สร้างโปรแกรมแปลงค่าเงิน:
- รับจำนวนเงินบาท
- แปลงเป็น USD, EUR, JPY, GBP
- แสดงผลในรูปแบบตาราง

### แบบฝึกหัดที่ 2: BMI Calculator
สร้างเครื่องคำนวณ BMI:
- รับน้ำหนัก (kg) และส่วนสูง (cm)
- คำนวณ BMI = weight / (height * height)
- แสดงผลและบอก category (Underweight/Normal/Overweight/Obese)

### แบบฝึกหัดที่ 3: Time Calculator
สร้างโปรแกรมคำนวณเวลา:
- รับวันเกิดของผู้ใช้
- คำนวณอายุ (ปี, เดือน, วัน)
- แสดงวันเกิดวันต่อไป
- แสดงจำนวนวันจนถึงวันเกิดหน้า

---

**ก่อนหน้า → [Part 01: Introduction](part01-introduction-csharp.md)**  
**ต่อไป → [Part 03: Control Flow](part03-control-flow.md)**
