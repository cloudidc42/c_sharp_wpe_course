# Part 03: Control Flow - if/else, switch
## ขั้นตอนที่ 21-30: การควบคุมการทำงานของโปรแกรม

---

## 🎯 เป้าหมายของ Part นี้
- ใช้ if, else if, else สำหรับตัดสินใจ
- ใช้ switch statement และ switch expression
- เข้าใจ Pattern Matching ใน C# 9+
- ใช้ goto (และรู้ว่าไม่ควรใช้เมื่อไหร่)
- เขียน Conditional Logic ที่ดีและอ่านง่าย

---

## ขั้นตอนที่ 21: if Statement พื้นฐาน

### รูปแบบ if Statement
```csharp
// รูปแบบที่ 1: if เดี่ยว
int score = 75;
if (score >= 60)
{
    Console.WriteLine("ผ่านการสอบ");
}

// รูปแบบที่ 2: if-else
if (score >= 60)
{
    Console.WriteLine("ผ่านการสอบ");
}
else
{
    Console.WriteLine("ไม่ผ่านการสอบ");
}

// รูปแบบที่ 3: if-else if-else
if (score >= 80)
{
    Console.WriteLine("เกรด A - ยอดเยี่ยม");
}
else if (score >= 70)
{
    Console.WriteLine("เกรด B - ดี");
}
else if (score >= 60)
{
    Console.WriteLine("เกรด C - พอใช้");
}
else
{
    Console.WriteLine("เกรด F - ไม่ผ่าน");
}

// รูปแบบที่ 4: if บรรทัดเดียว (ไม่แนะนำสำหรับ logic ซับซ้อน)
if (score >= 60) Console.WriteLine("ผ่าน");
else Console.WriteLine("ไม่ผ่าน");
```

### Nested if (if ซ้อน)
```csharp
int age = 25;
bool hasLicense = true;
bool hasCar = false;

if (age >= 18)
{
    Console.WriteLine("อายุเพียงพอในการขับรถ");
    
    if (hasLicense)
    {
        Console.WriteLine("มีใบขับขี่");
        
        if (hasCar)
        {
            Console.WriteLine("ขับรถได้เลย!");
        }
        else
        {
            Console.WriteLine("ต้องหารถก่อน");
        }
    }
    else
    {
        Console.WriteLine("ต้องไปทำใบขับขี่ก่อน");
    }
}
else
{
    Console.WriteLine("อายุยังน้อยเกินไป");
}
```

### ✅ เคล็ดลับการเขียน if ที่ดี
```csharp
// ❌ Bad: ซ้อนกันเยอะเกินไป (Arrow Code / Pyramid of Doom)
if (user != null)
{
    if (user.IsActive)
    {
        if (user.HasPermission)
        {
            if (score >= 60)
            {
                DoSomething();
            }
        }
    }
}

// ✅ Good: Early Return / Guard Clauses
if (user == null) return;
if (!user.IsActive) return;
if (!user.HasPermission) return;
if (score < 60) return;

DoSomething();  // ถึงตรงนี้ได้แน่ว่าผ่านเงื่อนไขทั้งหมด
```

---

## ขั้นตอนที่ 22: Comparison และ Boolean Conditions

### Complex Conditions
```csharp
int age = 25;
string membership = "Gold";
double balance = 5000.0;
bool isVerified = true;

// AND (&&) - ทุกเงื่อนไขต้องเป็นจริง
if (age >= 18 && isVerified && balance > 0)
{
    Console.WriteLine("สามารถทำธุรกรรมได้");
}

// OR (||) - อย่างน้อยหนึ่งเงื่อนไขต้องเป็นจริง
if (membership == "Gold" || membership == "Platinum" || balance > 10000)
{
    Console.WriteLine("ได้รับสิทธิพิเศษ");
}

// การรวม AND และ OR
// ⚠️ ระวังลำดับ precedence! ใช้ () เพื่อความชัดเจน
bool canApply = (age >= 18 && isVerified) && (membership == "Gold" || balance >= 5000);

if (canApply)
{
    Console.WriteLine("สมัครได้");
}

// NOT (!)
if (!string.IsNullOrEmpty(membership) && !(membership == "Basic"))
{
    Console.WriteLine("สมาชิก Premium");
}
```

### String Comparisons
```csharp
string status = "active";

// ⚠️ case-sensitive!
if (status == "active")         // True
if (status == "Active")         // False
if (status == "ACTIVE")         // False

// Case-insensitive comparison
if (status.Equals("active", StringComparison.OrdinalIgnoreCase))
{
    Console.WriteLine("สถานะ Active");
}

// หรือแปลงก่อนเปรียบเทียบ
if (status.ToLower() == "active")
{
    Console.WriteLine("สถานะ Active");
}

// ตรวจสอบหลายค่า
if (status == "active" || status == "enabled" || status == "on")
{
    Console.WriteLine("ใช้งานอยู่");
}

// ใช้ HashSet แทน (ดีกว่าเมื่อมีหลายค่า)
var activeStatuses = new HashSet<string>(StringComparer.OrdinalIgnoreCase) 
    { "active", "enabled", "on" };
    
if (activeStatuses.Contains(status))
{
    Console.WriteLine("ใช้งานอยู่");
}
```

---

## ขั้นตอนที่ 23: switch Statement

### switch Statement พื้นฐาน
```csharp
int day = 3;
string dayName;

switch (day)
{
    case 1:
        dayName = "จันทร์";
        break;  // ต้องมี break!
    case 2:
        dayName = "อังคาร";
        break;
    case 3:
        dayName = "พุธ";
        break;
    case 4:
        dayName = "พฤหัสบดี";
        break;
    case 5:
        dayName = "ศุกร์";
        break;
    case 6:
        dayName = "เสาร์";
        break;
    case 7:
        dayName = "อาทิตย์";
        break;
    default:  // ถ้าไม่ตรงกับ case ใดเลย
        dayName = "ไม่ทราบ";
        break;
}

Console.WriteLine($"วัน: {dayName}");
```

### Multiple cases (Fall-through)
```csharp
int month = 4;
string season;

switch (month)
{
    case 12:
    case 1:
    case 2:
        season = "ฤดูหนาว";
        break;
    case 3:
    case 4:
    case 5:
        season = "ฤดูร้อน";
        break;
    case 6:
    case 7:
    case 8:
    case 9:
    case 10:
        season = "ฤดูฝน";
        break;
    case 11:
        season = "ปลายฤดูฝน";
        break;
    default:
        season = "ไม่ทราบ";
        break;
}

Console.WriteLine($"ฤดูกาล: {season}");
```

### switch กับ string
```csharp
string command = "start";

switch (command.ToLower())
{
    case "start":
    case "begin":
    case "go":
        Console.WriteLine("เริ่มต้น...");
        break;
    case "stop":
    case "end":
    case "quit":
        Console.WriteLine("หยุด...");
        break;
    case "pause":
        Console.WriteLine("หยุดชั่วคราว...");
        break;
    default:
        Console.WriteLine($"ไม่รู้จักคำสั่ง: {command}");
        break;
}
```

---

## ขั้นตอนที่ 24: switch Expression (C# 8+)

### switch Expression - กระชับกว่ามาก!
```csharp
// switch statement (แบบเดิม)
string GetDayOld(int day)
{
    switch (day)
    {
        case 1: return "จันทร์";
        case 2: return "อังคาร";
        case 3: return "พุธ";
        default: return "ไม่ทราบ";
    }
}

// switch expression (C# 8+) - กระชับกว่า!
string GetDay(int day) => day switch
{
    1 => "จันทร์",
    2 => "อังคาร",
    3 => "พุธ",
    4 => "พฤหัสบดี",
    5 => "ศุกร์",
    6 => "เสาร์",
    7 => "อาทิตย์",
    _ => "ไม่ทราบ"  // _ เทียบเท่า default
};

Console.WriteLine(GetDay(3));  // พุธ

// ใช้กับตัวแปรได้เลย
int month = 7;
string season = month switch
{
    12 or 1 or 2 => "ฤดูหนาว",      // or pattern (C# 9+)
    3 or 4 or 5 => "ฤดูร้อน",
    >= 6 and <= 10 => "ฤดูฝน",       // relational pattern
    11 => "ปลายฤดูฝน",
    _ => throw new ArgumentException($"Invalid month: {month}")
};

Console.WriteLine($"ฤดูกาล: {season}");
```

### switch Expression กับ Tuple
```csharp
// ตรวจสอบหลายค่าพร้อมกัน
(bool hasAccount, bool isVerified, decimal balance) user = (true, true, 5000m);

string status = user switch
{
    (false, _, _) => "ไม่มีบัญชี",
    (true, false, _) => "รอการยืนยัน",
    (true, true, < 0) => "ยอดเงินติดลบ",
    (true, true, 0) => "ยอดเงินเป็นศูนย์",
    (true, true, > 0) => "ใช้งานได้ปกติ",
    _ => "ไม่ทราบสถานะ"
};

Console.WriteLine(status);  // ใช้งานได้ปกติ
```

---

## ขั้นตอนที่ 25: Pattern Matching (C# 7-11)

### Type Patterns
```csharp
object obj = "Hello, World!";

// is pattern (C# 7+)
if (obj is string text)
{
    Console.WriteLine($"String ยาว {text.Length} ตัวอักษร");
}

if (obj is int num)
{
    Console.WriteLine($"Number: {num}");
}
else if (obj is string str)
{
    Console.WriteLine($"String: {str}");
}
else if (obj is bool flag)
{
    Console.WriteLine($"Bool: {flag}");
}
else
{
    Console.WriteLine($"Type: {obj?.GetType().Name ?? "null"}");
}
```

### Relational Patterns (C# 9+)
```csharp
double bmi = 22.5;

string bmiCategory = bmi switch
{
    < 18.5 => "น้ำหนักน้อย (Underweight)",
    >= 18.5 and < 25.0 => "น้ำหนักปกติ (Normal)",
    >= 25.0 and < 30.0 => "น้ำหนักเกิน (Overweight)",
    >= 30.0 and < 35.0 => "โรคอ้วนระดับ 1 (Obese I)",
    >= 35.0 and < 40.0 => "โรคอ้วนระดับ 2 (Obese II)",
    >= 40.0 => "โรคอ้วนระดับ 3 (Obese III)",
    _ => "ไม่ทราบ"
};

Console.WriteLine($"BMI: {bmi} - {bmiCategory}");
```

### Property Patterns (C# 8+)
```csharp
// สมมติมี record/class Customer
record Customer(string Name, int Age, string MemberType, decimal Balance);

Customer customer = new Customer("สมชาย", 25, "Gold", 15000m);

// Property pattern matching
string discount = customer switch
{
    { MemberType: "Platinum", Balance: >= 10000 } => "ส่วนลด 20%",
    { MemberType: "Gold", Age: >= 60 } => "ส่วนลด 15% (ผู้สูงอายุ)",
    { MemberType: "Gold" } => "ส่วนลด 10%",
    { MemberType: "Silver", Balance: > 5000 } => "ส่วนลด 7%",
    { MemberType: "Silver" } => "ส่วนลด 5%",
    { Age: < 18 } => "ส่วนลด 5% (เยาวชน)",
    _ => "ไม่มีส่วนลด"
};

Console.WriteLine($"{customer.Name}: {discount}");  // สมชาย: ส่วนลด 10%
```

### List Patterns (C# 11+)
```csharp
// List/Array patterns
int[] numbers = { 1, 2, 3, 4, 5 };

string description = numbers switch
{
    [] => "ว่างเปล่า",
    [var single] => $"มีตัวเดียว: {single}",
    [var first, var second] => $"มีสองตัว: {first}, {second}",
    [1, 2, ..] => "เริ่มด้วย 1, 2",
    [.., 4, 5] => "จบด้วย 4, 5",
    _ => $"มี {numbers.Length} ตัว"
};

Console.WriteLine(description);  // เริ่มด้วย 1, 2
```

---

## ขั้นตอนที่ 26: Conditional Expressions ขั้นสูง

### Null Checks Pattern
```csharp
// C# 9+ - not null pattern
object? obj = GetValue();

if (obj is not null)
{
    // obj ไม่เป็น null
    Console.WriteLine(obj.ToString());
}

// null pattern
if (obj is null)
{
    Console.WriteLine("เป็น null");
}

// รวมกัน
if (obj is string { Length: > 0 } text)
{
    Console.WriteLine($"String ที่ไม่ว่าง: {text}");
}
```

### Logical Patterns (C# 9+)
```csharp
int value = 42;

// and
if (value is > 0 and < 100)
    Console.WriteLine("อยู่ในช่วง 1-99");

// or
if (value is 0 or 42 or 100)
    Console.WriteLine("เป็นค่าพิเศษ");

// not
if (value is not 0)
    Console.WriteLine("ไม่ใช่ 0");

// รวมกัน
bool isValidAge = age is >= 18 and <= 65 and not 30;  // แปลก แต่ syntactically valid
```

---

## ขั้นตอนที่ 27: goto Statement (ใช้น้อยๆ)

### goto - ควรหลีกเลี่ยง แต่มีบางกรณีที่ใช้ได้
```csharp
// ❌ การใช้ goto ที่ไม่ดี - ทำให้โค้ดอ่านยาก
int i = 0;
start:
Console.WriteLine(i);
i++;
if (i < 5) goto start;

// ✅ ดีกว่าใช้ loop:
for (int j = 0; j < 5; j++)
{
    Console.WriteLine(j);
}

// ✅ กรณีที่ goto มีประโยชน์: ออกจาก nested loops
for (int x = 0; x < 5; x++)
{
    for (int y = 0; y < 5; y++)
    {
        if (x == 2 && y == 3)
        {
            goto foundTarget;  // ออกจากทั้งสอง loops
        }
        Console.Write($"({x},{y}) ");
    }
}
foundTarget:
Console.WriteLine("\nพบเป้าหมาย!");

// goto ใน switch (ให้ case ไป case อื่น)
int option = 2;
switch (option)
{
    case 1:
        Console.WriteLine("Option 1");
        goto case 3;  // ไปทำ case 3 ต่อ
    case 2:
        Console.WriteLine("Option 2");
        goto case 3;
    case 3:
        Console.WriteLine("Option 3 (รันเสมอ)");
        break;
}
```

---

## ขั้นตอนที่ 28: Debugging Conditional Logic

### เทคนิคการ Debug if/switch
```csharp
// ใช้ Console.WriteLine ตรวจสอบค่าตัวแปร
int score = 75;
Console.WriteLine($"DEBUG: score = {score}");

string grade = "";
if (score >= 80)
{
    Console.WriteLine("DEBUG: เข้า branch >= 80");
    grade = "A";
}
else if (score >= 70)
{
    Console.WriteLine("DEBUG: เข้า branch >= 70");
    grade = "B";
}
else
{
    Console.WriteLine("DEBUG: เข้า branch else");
    grade = "F";
}

// ใช้ conditional breakpoints ใน VS2022:
// คลิกขวาที่ breakpoint → Conditions → เขียน condition

// ใช้ Debug.Assert
System.Diagnostics.Debug.Assert(score >= 0 && score <= 100, 
    "Score ต้องอยู่ในช่วง 0-100");
```

### Unit Test สำหรับ Conditional Logic
```csharp
// เขียน test cases สำหรับทุก branch
void TestGradeCalculation()
{
    // Test case 1: ขอบเขตบน
    TestGrade(100, "A");
    TestGrade(80, "A");
    
    // Test case 2: ขอบเขตล่าง
    TestGrade(79, "B");
    TestGrade(70, "B");
    
    // Test case 3: ขอบเขต
    TestGrade(69, "C");
    TestGrade(60, "C");
    
    // Test case 4: Fail
    TestGrade(59, "F");
    TestGrade(0, "F");
    
    Console.WriteLine("ทุก test ผ่านหมด!");
}

void TestGrade(int score, string expectedGrade)
{
    string actualGrade = CalculateGrade(score);
    if (actualGrade != expectedGrade)
    {
        Console.WriteLine($"FAIL: score={score}, expected={expectedGrade}, actual={actualGrade}");
    }
}

string CalculateGrade(int score)
{
    return score switch
    {
        >= 80 => "A",
        >= 70 => "B",
        >= 60 => "C",
        _ => "F"
    };
}
```

---

## ขั้นตอนที่ 29: Best Practices สำหรับ Conditional Logic

### หลักการเขียน if/switch ที่ดี
```csharp
// ✅ 1. เรียงเงื่อนไขจาก specific ไป general
string GetDiscount(Customer customer)
{
    // Specific cases ก่อน
    if (customer.IsVIP && customer.Balance > 100000)
        return "30%";
    
    if (customer.IsVIP)
        return "20%";
    
    if (customer.MemberYears > 5)
        return "10%";
    
    // General case สุดท้าย
    return "0%";
}

// ✅ 2. Return early แทนการ nest
bool ValidateOrder(Order order)
{
    if (order == null) return false;
    if (!order.HasItems) return false;
    if (order.TotalAmount <= 0) return false;
    if (!order.Customer.IsActive) return false;
    
    // ถ้าผ่านทั้งหมด
    return true;
}

// ✅ 3. ใช้ switch expression แทน long if-else chain
string GetStatusMessage(int statusCode) => statusCode switch
{
    200 => "OK",
    201 => "Created",
    400 => "Bad Request",
    401 => "Unauthorized",
    403 => "Forbidden",
    404 => "Not Found",
    500 => "Internal Server Error",
    _ => $"Unknown Status: {statusCode}"
};

// ✅ 4. แยก complex condition เป็น method หรือตัวแปร
bool CanApplyForLoan(Customer customer, decimal amount)
{
    bool hasGoodCredit = customer.CreditScore >= 700;
    bool hasStableIncome = customer.MonthlyIncome >= amount / 36;
    bool notOverExtended = customer.TotalDebt < customer.AnnualIncome * 3;
    bool hasHistory = customer.AccountAgeMonths >= 12;
    
    return hasGoodCredit && hasStableIncome && notOverExtended && hasHistory;
}
```

---

## ขั้นตอนที่ 30: โปรแกรมตัวอย่างรวม - ATM Simulator

```csharp
using System;
using System.Collections.Generic;

namespace ATMSimulator
{
    class Program
    {
        // ข้อมูลบัญชีจำลอง
        static Dictionary<string, (string pin, decimal balance, string name)> accounts = new()
        {
            { "1234567890", ("1234", 50000m, "สมชาย วิชาการ") },
            { "0987654321", ("5678", 25000m, "สมหญิง ใจดี") },
            { "1111222233", ("9999", 100000m, "นายร่ำรวย มีเงิน") }
        };
        
        static void Main(string[] args)
        {
            Console.OutputEncoding = System.Text.Encoding.UTF8;
            Console.Title = "ATM Simulator";
            
            bool continueSession = true;
            
            while (continueSession)
            {
                ShowWelcome();
                
                // รับ Account Number
                string? accountNumber = GetAccountNumber();
                if (accountNumber == null) break;
                
                // ตรวจสอบบัญชี
                if (!accounts.ContainsKey(accountNumber))
                {
                    ShowError("ไม่พบหมายเลขบัญชีนี้");
                    continue;
                }
                
                // รับ PIN
                string? pin = GetPIN();
                if (pin == null) break;
                
                // ตรวจสอบ PIN
                var account = accounts[accountNumber];
                if (account.pin != pin)
                {
                    ShowError("PIN ไม่ถูกต้อง");
                    continue;
                }
                
                // เข้าสู่ Main Menu
                bool loggedIn = true;
                while (loggedIn)
                {
                    ShowMainMenu(account.name, account.balance);
                    
                    Console.Write("\nเลือกรายการ: ");
                    string? choice = Console.ReadLine();
                    
                    switch (choice)
                    {
                        case "1":
                            // ตรวจสอบยอดเงิน
                            ShowBalance(accountNumber, accounts);
                            break;
                            
                        case "2":
                            // ถอนเงิน
                            loggedIn = ProcessWithdrawal(accountNumber);
                            break;
                            
                        case "3":
                            // ฝากเงิน
                            ProcessDeposit(accountNumber);
                            break;
                            
                        case "4":
                            // โอนเงิน
                            ProcessTransfer(accountNumber);
                            break;
                            
                        case "5":
                            // ออกจากระบบ
                            Console.WriteLine("\nขอบคุณที่ใช้บริการ!");
                            loggedIn = false;
                            break;
                            
                        default:
                            Console.ForegroundColor = ConsoleColor.Red;
                            Console.WriteLine("กรุณาเลือกรายการ 1-5");
                            Console.ResetColor();
                            break;
                    }
                    
                    if (loggedIn)
                    {
                        Console.WriteLine("\nกด Enter เพื่อดำเนินการต่อ...");
                        Console.ReadLine();
                    }
                }
                
                Console.Write("\nทำรายการอื่นอีกไหม? (Y/N): ");
                string? again = Console.ReadLine();
                continueSession = again?.ToUpper() == "Y";
            }
            
            Console.WriteLine("ขอบคุณที่ใช้บริการ ATM");
            Console.ReadLine();
        }
        
        static void ShowWelcome()
        {
            Console.Clear();
            Console.ForegroundColor = ConsoleColor.Cyan;
            Console.WriteLine("╔══════════════════════════════════╗");
            Console.WriteLine("║        ยินดีต้อนรับสู่ ATM       ║");
            Console.WriteLine("║         THAI BANK ATM             ║");
            Console.WriteLine("╚══════════════════════════════════╝");
            Console.ResetColor();
            Console.WriteLine();
        }
        
        static string? GetAccountNumber()
        {
            Console.Write("กรุณาใส่หมายเลขบัญชี (10 หลัก): ");
            string? input = Console.ReadLine();
            
            if (input == null || input.Length != 10 || !long.TryParse(input, out _))
            {
                ShowError("หมายเลขบัญชีต้องเป็นตัวเลข 10 หลัก");
                return null;
            }
            
            return input;
        }
        
        static string? GetPIN()
        {
            Console.Write("กรุณาใส่ PIN (4 หลัก): ");
            
            // ซ่อน PIN ขณะพิมพ์
            string pin = "";
            ConsoleKeyInfo key;
            
            do
            {
                key = Console.ReadKey(intercept: true);
                
                if (key.Key != ConsoleKey.Enter && key.Key != ConsoleKey.Backspace)
                {
                    if (char.IsDigit(key.KeyChar) && pin.Length < 4)
                    {
                        pin += key.KeyChar;
                        Console.Write("*");
                    }
                }
                else if (key.Key == ConsoleKey.Backspace && pin.Length > 0)
                {
                    pin = pin[..^1];  // ลบตัวสุดท้าย
                    Console.Write("\b \b");
                }
            } while (key.Key != ConsoleKey.Enter);
            
            Console.WriteLine();
            
            if (pin.Length != 4)
            {
                ShowError("PIN ต้องเป็นตัวเลข 4 หลัก");
                return null;
            }
            
            return pin;
        }
        
        static void ShowMainMenu(string name, decimal balance)
        {
            Console.Clear();
            Console.WriteLine($"สวัสดีคุณ {name}");
            Console.WriteLine($"ยอดเงินในบัญชี: {balance:N2} บาท");
            Console.WriteLine();
            Console.WriteLine("╔════════════════════════╗");
            Console.WriteLine("║      เมนูหลัก           ║");
            Console.WriteLine("╠════════════════════════╣");
            Console.WriteLine("║ 1. ตรวจสอบยอดเงิน      ║");
            Console.WriteLine("║ 2. ถอนเงิน              ║");
            Console.WriteLine("║ 3. ฝากเงิน              ║");
            Console.WriteLine("║ 4. โอนเงิน              ║");
            Console.WriteLine("║ 5. ออกจากระบบ           ║");
            Console.WriteLine("╚════════════════════════╝");
        }
        
        static void ShowBalance(string accountNumber, 
            Dictionary<string, (string pin, decimal balance, string name)> accounts)
        {
            var account = accounts[accountNumber];
            Console.Clear();
            Console.ForegroundColor = ConsoleColor.Green;
            Console.WriteLine("=== ยอดเงินในบัญชี ===");
            Console.WriteLine($"ชื่อบัญชี: {account.name}");
            Console.WriteLine($"เลขที่บัญชี: {accountNumber}");
            Console.WriteLine($"ยอดเงิน: {account.balance:N2} บาท");
            Console.ResetColor();
        }
        
        static bool ProcessWithdrawal(string accountNumber)
        {
            Console.Clear();
            Console.WriteLine("=== ถอนเงิน ===");
            
            var account = accounts[accountNumber];
            Console.WriteLine($"ยอดเงินปัจจุบัน: {account.balance:N2} บาท");
            Console.WriteLine();
            
            // แสดงจำนวนเงินให้เลือก
            Console.WriteLine("เลือกจำนวนเงิน:");
            decimal[] amounts = { 500, 1000, 2000, 5000, 10000 };
            
            for (int i = 0; i < amounts.Length; i++)
            {
                Console.WriteLine($"{i + 1}. {amounts[i]:N0} บาท");
            }
            Console.WriteLine("6. ระบุจำนวนเอง");
            Console.WriteLine("0. ยกเลิก");
            
            Console.Write("\nเลือก: ");
            string? choice = Console.ReadLine();
            
            decimal withdrawAmount = 0;
            
            switch (choice)
            {
                case "1": withdrawAmount = amounts[0]; break;
                case "2": withdrawAmount = amounts[1]; break;
                case "3": withdrawAmount = amounts[2]; break;
                case "4": withdrawAmount = amounts[3]; break;
                case "5": withdrawAmount = amounts[4]; break;
                case "6":
                    Console.Write("ระบุจำนวนเงิน: ");
                    if (!decimal.TryParse(Console.ReadLine(), out withdrawAmount))
                    {
                        ShowError("จำนวนเงินไม่ถูกต้อง");
                        return true;
                    }
                    break;
                case "0":
                    return true;
                default:
                    ShowError("เลือกไม่ถูกต้อง");
                    return true;
            }
            
            // ตรวจสอบเงื่อนไข
            if (withdrawAmount <= 0)
            {
                ShowError("จำนวนเงินต้องมากกว่า 0");
                return true;
            }
            
            if (withdrawAmount > account.balance)
            {
                ShowError("ยอดเงินในบัญชีไม่เพียงพอ");
                return true;
            }
            
            if (withdrawAmount % 100 != 0)
            {
                ShowError("จำนวนเงินต้องเป็นผลคูณของ 100");
                return true;
            }
            
            // ทำรายการ
            var updatedAccount = account with { balance = account.balance - withdrawAmount };
            accounts[accountNumber] = updatedAccount;
            
            Console.ForegroundColor = ConsoleColor.Green;
            Console.WriteLine($"\n✅ ถอนเงินสำเร็จ: {withdrawAmount:N2} บาท");
            Console.WriteLine($"ยอดเงินคงเหลือ: {updatedAccount.balance:N2} บาท");
            Console.ResetColor();
            
            return true;
        }
        
        static void ProcessDeposit(string accountNumber)
        {
            Console.Clear();
            Console.WriteLine("=== ฝากเงิน ===");
            
            Console.Write("ระบุจำนวนเงินที่ต้องการฝาก: ");
            if (!decimal.TryParse(Console.ReadLine(), out decimal depositAmount) || depositAmount <= 0)
            {
                ShowError("จำนวนเงินไม่ถูกต้อง");
                return;
            }
            
            var account = accounts[accountNumber];
            var updatedAccount = account with { balance = account.balance + depositAmount };
            accounts[accountNumber] = updatedAccount;
            
            Console.ForegroundColor = ConsoleColor.Green;
            Console.WriteLine($"\n✅ ฝากเงินสำเร็จ: {depositAmount:N2} บาท");
            Console.WriteLine($"ยอดเงินคงเหลือ: {updatedAccount.balance:N2} บาท");
            Console.ResetColor();
        }
        
        static void ProcessTransfer(string accountNumber)
        {
            Console.Clear();
            Console.WriteLine("=== โอนเงิน ===");
            
            Console.Write("ระบุหมายเลขบัญชีปลายทาง: ");
            string? targetAccount = Console.ReadLine();
            
            if (targetAccount == accountNumber)
            {
                ShowError("ไม่สามารถโอนไปยังบัญชีตัวเองได้");
                return;
            }
            
            if (!accounts.ContainsKey(targetAccount ?? ""))
            {
                ShowError("ไม่พบบัญชีปลายทาง");
                return;
            }
            
            Console.Write("ระบุจำนวนเงินที่ต้องการโอน: ");
            if (!decimal.TryParse(Console.ReadLine(), out decimal transferAmount) || transferAmount <= 0)
            {
                ShowError("จำนวนเงินไม่ถูกต้อง");
                return;
            }
            
            var sourceAccount = accounts[accountNumber];
            if (transferAmount > sourceAccount.balance)
            {
                ShowError("ยอดเงินในบัญชีไม่เพียงพอ");
                return;
            }
            
            var targetAcc = accounts[targetAccount!];
            
            // ยืนยันการโอน
            Console.WriteLine($"\nยืนยันการโอนเงิน:");
            Console.WriteLine($"ไปยัง: {targetAcc.name} ({targetAccount})");
            Console.WriteLine($"จำนวน: {transferAmount:N2} บาท");
            Console.Write("ยืนยัน? (Y/N): ");
            
            string? confirm = Console.ReadLine();
            if (confirm?.ToUpper() != "Y") return;
            
            // ทำรายการ
            accounts[accountNumber] = sourceAccount with 
                { balance = sourceAccount.balance - transferAmount };
            accounts[targetAccount!] = targetAcc with 
                { balance = targetAcc.balance + transferAmount };
            
            Console.ForegroundColor = ConsoleColor.Green;
            Console.WriteLine($"\n✅ โอนเงินสำเร็จ!");
            Console.WriteLine($"โอนไปยัง: {targetAcc.name}");
            Console.WriteLine($"จำนวน: {transferAmount:N2} บาท");
            Console.WriteLine($"ยอดเงินคงเหลือ: {accounts[accountNumber].balance:N2} บาท");
            Console.ResetColor();
        }
        
        static void ShowError(string message)
        {
            Console.ForegroundColor = ConsoleColor.Red;
            Console.WriteLine($"\n❌ Error: {message}");
            Console.ResetColor();
            System.Threading.Thread.Sleep(1500);
        }
    }
}
```

---

## 📝 สรุป Part 03

| หัวข้อ | รายละเอียดสำคัญ |
|--------|----------------|
| if/else | การตัดสินใจพื้นฐาน, nested if, early return |
| switch statement | Multiple cases, fall-through, string switch |
| switch expression | C# 8+ กระชับกว่า, arm patterns |
| Pattern Matching | Type, Relational, Property, List patterns |
| Null Patterns | is null, is not null, null-coalescing |
| goto | ควรหลีกเลี่ยง ยกเว้น break nested loops |
| Best Practices | Guard clauses, early return, readable conditions |

---

## 🏋️ แบบฝึกหัด Part 03

### แบบฝึกหัดที่ 1: Traffic Light Simulator
สร้างโปรแกรมจำลองสัญญาณไฟจราจร:
- รับสี (red/yellow/green)
- แสดงคำแนะนำสำหรับผู้ขับขี่
- ใช้ switch expression

### แบบฝึกหัดที่ 2: Shipping Cost Calculator
คำนวณค่าส่งตามน้ำหนักและระยะทาง:
- ในเมือง < 5kg: 30บ, 5-10kg: 50บ, >10kg: 80บ
- ต่างจังหวัด: เพิ่ม 50%
- ไปรษณีย์EMS: เพิ่ม 30บ

### แบบฝึกหัดที่ 3: Simple Quiz Game
สร้างเกม Quiz:
- 5 คำถาม
- 4 ตัวเลือก
- คำนวณคะแนน
- แสดงผลพร้อมเกรด

---

**ก่อนหน้า → [Part 02: Variables & Data Types](part02-variables-datatypes.md)**  
**ต่อไป → [Part 04: Loops](part04-loops.md)**
