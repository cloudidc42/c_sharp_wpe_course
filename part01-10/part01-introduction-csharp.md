# Part 01: Introduction to C# & .NET
## ขั้นตอนที่ 1-10: เริ่มต้นกับ C# และ .NET Framework

---

## 🎯 เป้าหมายของ Part นี้
- เข้าใจว่า C# คืออะไร และทำงานอย่างไร
- ติดตั้ง Visual Studio 2022
- เขียนโปรแกรม "Hello, World!" แรก
- เข้าใจโครงสร้างพื้นฐานของโปรแกรม C#
- รู้จัก .NET Runtime และ CLR

---

## ขั้นตอนที่ 1: C# คืออะไร?

C# (อ่านว่า "ซี-ชาร์ป") เป็นภาษาโปรแกรมมิ่งที่พัฒนาโดย **Microsoft** ในปี 2000 โดย **Anders Hejlsberg** ซึ่งเป็นผู้สร้าง Turbo Pascal และ Delphi ด้วย

### ลักษณะเด่นของ C#
- **Object-Oriented**: รองรับการเขียนโปรแกรมเชิงวัตถุอย่างสมบูรณ์
- **Type-Safe**: ตรวจสอบชนิดข้อมูลตอน compile
- **Managed Memory**: จัดการ memory อัตโนมัติผ่าน Garbage Collector
- **Cross-platform**: รันได้บน Windows, macOS, Linux ผ่าน .NET Core/.NET 5+
- **Modern Features**: รองรับ async/await, LINQ, pattern matching, records

### C# ใช้ทำอะไรได้บ้าง?
```
✅ Windows Desktop Apps (WPF, WinForms)
✅ Web Applications (ASP.NET Core)
✅ Mobile Apps (Xamarin, MAUI)
✅ Game Development (Unity)
✅ Cloud Services (Azure Functions)
✅ Microservices
✅ IoT Applications
✅ Machine Learning (ML.NET)
```

---

## ขั้นตอนที่ 2: .NET Ecosystem

### .NET Framework vs .NET Core vs .NET 5+

```
┌─────────────────────────────────────────────────────┐
│                   .NET Timeline                       │
├─────────────────┬─────────────────┬─────────────────┤
│  .NET Framework │   .NET Core      │    .NET 5+       │
│  (2002-2019)    │  (2016-2020)    │  (2020-ปัจจุบัน) │
│  Windows only   │  Cross-platform  │  Unified         │
│  1.0 - 4.8      │  1.0 - 3.1      │  5, 6, 7, 8, 9  │
└─────────────────┴─────────────────┴─────────────────┘
```

### .NET Architecture
```
┌──────────────────────────────────────┐
│           Your Application           │
├──────────────────────────────────────┤
│         Base Class Library (BCL)     │
│  System.IO, System.Collections, etc. │
├──────────────────────────────────────┤
│    Common Language Runtime (CLR)     │
│  JIT Compiler, GC, Security, etc.   │
├──────────────────────────────────────┤
│           Operating System           │
└──────────────────────────────────────┘
```

### CLR (Common Language Runtime) ทำอะไร?
1. **JIT Compilation**: แปลง IL (Intermediate Language) เป็น Machine Code
2. **Garbage Collection**: จัดการ memory อัตโนมัติ
3. **Type Safety**: ตรวจสอบความถูกต้องของชนิดข้อมูล
4. **Exception Handling**: จัดการข้อผิดพลาด
5. **Security**: ตรวจสอบความปลอดภัยของโค้ด

---

## ขั้นตอนที่ 3: ติดตั้ง Visual Studio 2022

### ขั้นตอนการติดตั้ง

**Step 3.1**: ดาวน์โหลด Visual Studio 2022 Community (ฟรี)
- ไปที่ `https://visualstudio.microsoft.com/downloads/`
- คลิก "Free download" ใต้ Community

**Step 3.2**: เลือก Workloads ที่ต้องการ
```
✅ .NET desktop development        ← สำหรับ WPF, WinForms
✅ ASP.NET and web development     ← สำหรับ Web Apps
✅ Universal Windows Platform dev  ← สำหรับ UWP (optional)
```

**Step 3.3**: ติดตั้งและรอจนเสร็จ (ประมาณ 10-20 นาที)

### ทางเลือกอื่น: Visual Studio Code + C# Extension
```bash
# ติดตั้ง .NET SDK
# ดาวน์โหลดจาก https://dotnet.microsoft.com/download

# ติดตั้ง VS Code extension
code --install-extension ms-dotnettools.csharp
```

### ตรวจสอบการติดตั้ง
```bash
# เปิด Command Prompt หรือ Terminal แล้วพิมพ์:
dotnet --version
# ควรแสดง: 8.0.xxx หรือเวอร์ชันล่าสุด

dotnet --list-sdks
# แสดงรายการ SDK ที่ติดตั้ง
```

---

## ขั้นตอนที่ 4: สร้างโปรเจ็กต์แรก

### สร้างผ่าน Visual Studio 2022

```
1. เปิด Visual Studio 2022
2. คลิก "Create a new project"
3. ค้นหา "Console App"
4. เลือก "Console App" (C#, .NET)
5. ตั้งชื่อโปรเจ็กต์: "HelloWorld"
6. เลือก Location ที่ต้องการ
7. เลือก Framework: ".NET 8.0"
8. คลิก "Create"
```

### สร้างผ่าน Command Line
```bash
# สร้าง Console Application
dotnet new console -n HelloWorld
cd HelloWorld

# สร้าง Solution
dotnet new sln -n MyCSharpCourse
dotnet sln add HelloWorld/HelloWorld.csproj

# รันโปรแกรม
dotnet run
```

### โครงสร้างโปรเจ็กต์ที่สร้างขึ้น
```
HelloWorld/
├── HelloWorld.csproj    ← Project configuration file
├── Program.cs           ← Main source file
└── obj/                 ← Build output (auto-generated)
    └── ...
```

---

## ขั้นตอนที่ 5: โปรแกรม Hello, World! แรก

### Program.cs (Top-level statements - .NET 6+)
```csharp
// This is a comment - คำอธิบายโค้ด ไม่ถูก compile

// แสดงข้อความ "Hello, World!" บน Console
Console.WriteLine("Hello, World!");
Console.WriteLine("สวัสดี โลก!");

// รอรับ input จากผู้ใช้ก่อนปิดหน้าต่าง
Console.ReadLine();
```

### Program.cs (Traditional style - .NET 5 และก่อนหน้า)
```csharp
using System;  // ใช้ namespace System ซึ่งมี Console class อยู่

namespace HelloWorld  // กำหนด namespace ของโปรเจ็กต์
{
    class Program  // คลาสหลักของโปรแกรม
    {
        static void Main(string[] args)  // Method หลัก - จุดเริ่มต้นโปรแกรม
        {
            Console.WriteLine("Hello, World!");
            Console.WriteLine("สวัสดี โลก!");
            Console.ReadLine();
        }
    }
}
```

### ผลลัพธ์ที่ได้
```
Hello, World!
สวัสดี โลก!
```

---

## ขั้นตอนที่ 6: เข้าใจโครงสร้างพื้นฐาน

### องค์ประกอบหลักของโปรแกรม C#

```csharp
// 1. Using Directives - นำเข้า namespace ที่ต้องการใช้
using System;
using System.Collections.Generic;
using System.IO;

// 2. Namespace Declaration - กลุ่มของ classes ที่เกี่ยวข้องกัน
namespace MyApplication
{
    // 3. Class Declaration - พิมพ์เขียวสำหรับสร้าง objects
    class Program
    {
        // 4. Method Declaration - ฟังก์ชันที่ทำงานได้
        // static = เรียกใช้ได้โดยไม่ต้องสร้าง object
        // void = ไม่ return ค่าใดๆ
        // Main = Method พิเศษที่เป็นจุดเริ่มต้นของโปรแกรม
        // string[] args = รับ arguments จาก command line
        static void Main(string[] args)
        {
            // 5. Statements - คำสั่งที่ทำงาน
            Console.WriteLine("Hello, World!");
            
            // 6. Semicolon - จบแต่ละ statement ด้วย ;
        }
    }
}
```

### Curly Braces `{ }` ใช้สำหรับอะไร?
```csharp
namespace Example    // เปิด namespace block
{
    class MyClass    // เปิด class block
    {
        void MyMethod()  // เปิด method block
        {
            // โค้ดอยู่ใน block นี้
            if (true)    // เปิด if block
            {
                Console.WriteLine("Inside if block");
            }            // ปิด if block
        }                // ปิด method block
    }                    // ปิด class block
}                        // ปิด namespace block
```

---

## ขั้นตอนที่ 7: Console Methods ที่ใช้บ่อย

### Console.Write vs Console.WriteLine
```csharp
// Console.Write - พิมพ์ข้อความโดยไม่ขึ้นบรรทัดใหม่
Console.Write("Hello");
Console.Write(", ");
Console.Write("World!");
// Output: Hello, World!

// Console.WriteLine - พิมพ์ข้อความแล้วขึ้นบรรทัดใหม่
Console.WriteLine("Line 1");
Console.WriteLine("Line 2");
// Output:
// Line 1
// Line 2

// Console.WriteLine() ไม่มีพารามิเตอร์ - แค่ขึ้นบรรทัดใหม่
Console.WriteLine("First");
Console.WriteLine();  // บรรทัดว่าง
Console.WriteLine("Third");
// Output:
// First
//
// Third
```

### Console.ReadLine และ Console.ReadKey
```csharp
// Console.ReadLine() - รับ input เป็น string จนกด Enter
Console.Write("กรุณาใส่ชื่อของคุณ: ");
string name = Console.ReadLine();
Console.WriteLine("สวัสดี " + name + "!");

// Console.ReadKey() - รับ input เป็น 1 ตัวอักษร
Console.WriteLine("กด Enter เพื่อออก...");
Console.ReadKey();

// Console.Read() - รับ input เป็น integer (character code)
int charCode = Console.Read();
char character = (char)charCode;
```

### Console.Clear และ Console.SetTitle
```csharp
// ล้างหน้าจอ Console
Console.Clear();

// ตั้งชื่อหน้าต่าง Console
Console.Title = "My C# Application";

// เปลี่ยนสีข้อความและพื้นหลัง
Console.ForegroundColor = ConsoleColor.Green;
Console.BackgroundColor = ConsoleColor.Black;
Console.WriteLine("ข้อความสีเขียว");
Console.ResetColor();  // รีเซ็ตสีกลับเป็นค่าเริ่มต้น
```

### String Formatting
```csharp
string name = "สมชาย";
int age = 25;
double salary = 35000.50;

// วิธีที่ 1: String Concatenation (ต่อ string)
Console.WriteLine("ชื่อ: " + name + ", อายุ: " + age);

// วิธีที่ 2: String.Format
Console.WriteLine(String.Format("ชื่อ: {0}, อายุ: {1}, เงินเดือน: {2:N2}", 
    name, age, salary));

// วิธีที่ 3: String Interpolation (C# 6+) - แนะนำ
Console.WriteLine($"ชื่อ: {name}, อายุ: {age}, เงินเดือน: {salary:N2}");

// วิธีที่ 4: Verbatim String
string path = @"C:\Users\User\Documents\file.txt";
Console.WriteLine(path);

// ผลลัพธ์ของทุกวิธี:
// ชื่อ: สมชาย, อายุ: 25, เงินเดือน: 35,000.50
```

---

## ขั้นตอนที่ 8: Comments และ Documentation

### ประเภทของ Comments
```csharp
// Single-line comment - comment บรรทัดเดียว

/* Multi-line comment
   สามารถเขียนได้หลายบรรทัด
   ใช้สำหรับอธิบายโค้ดที่ซับซ้อน */

/// <summary>
/// XML Documentation Comment - ใช้สร้าง documentation อัตโนมัติ
/// </summary>
/// <param name="name">ชื่อของบุคคล</param>
/// <returns>ข้อความทักทาย</returns>
string GetGreeting(string name)
{
    return $"สวัสดี {name}!";
}
```

### ตัวอย่างการใช้ Comments อย่างถูกต้อง
```csharp
// ✅ Good Comments - อธิบาย WHY ไม่ใช่ WHAT
// คำนวณ discount สำหรับสมาชิก VIP ตามนโยบายบริษัท v2.1
decimal discount = price * 0.15m;

// ✅ XML Documentation
/// <summary>
/// คำนวณ Body Mass Index (BMI)
/// </summary>
/// <param name="weight">น้ำหนักในหน่วย กิโลกรัม</param>
/// <param name="height">ส่วนสูงในหน่วย เมตร</param>
/// <returns>ค่า BMI</returns>
double CalculateBMI(double weight, double height)
{
    return weight / (height * height);
}

// ❌ Bad Comments - อธิบาย WHAT ที่โค้ดชัดเจนอยู่แล้ว
// เพิ่ม 1 ให้กับ counter  ← ไม่จำเป็น
counter++;
```

---

## ขั้นตอนที่ 9: ทำความเข้าใจ .csproj File

### HelloWorld.csproj
```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <!-- Target Framework - .NET version ที่ใช้ -->
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    
    <!-- Nullable Reference Types - C# 8+ feature -->
    <Nullable>enable</Nullable>
    
    <!-- ใช้ implicit usings (using System, etc. ไม่ต้องเขียนเอง) -->
    <ImplicitUsings>enable</ImplicitUsings>
    
    <!-- Root namespace ของโปรเจ็กต์ -->
    <RootNamespace>HelloWorld</RootNamespace>
  </PropertyGroup>

</Project>
```

### การเพิ่ม NuGet Package ใน .csproj
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <!-- เพิ่ม NuGet packages ที่ต้องการใช้ -->
  <ItemGroup>
    <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
    <PackageReference Include="Microsoft.EntityFrameworkCore" Version="8.0.0" />
  </ItemGroup>
</Project>
```

### dotnet CLI Commands ที่ต้องรู้
```bash
# สร้างโปรเจ็กต์ใหม่
dotnet new console -n MyApp        # Console App
dotnet new winforms -n MyWinApp    # WinForms App
dotnet new wpf -n MyWpfApp         # WPF App
dotnet new webapi -n MyWebApi      # Web API
dotnet new classlib -n MyLibrary   # Class Library

# Build และ Run
dotnet build                        # Build โปรเจ็กต์
dotnet run                          # Build และ Run
dotnet run --configuration Release  # Run แบบ Release

# Package Management
dotnet add package Newtonsoft.Json           # เพิ่ม NuGet package
dotnet remove package Newtonsoft.Json        # ลบ NuGet package
dotnet restore                               # Restore packages
dotnet list package                          # แสดงรายการ packages

# Testing
dotnet test                         # Run unit tests

# Publish
dotnet publish -c Release           # Publish แบบ Release
```

---

## ขั้นตอนที่ 10: โปรแกรมแรกที่สมบูรณ์

มาเขียนโปรแกรมแรกที่สมบูรณ์เพื่อรวบรวมสิ่งที่ได้เรียนมา:

### ตัวอย่าง: โปรแกรมทักทายแบบสมบูรณ์

```csharp
// Program.cs - Complete Hello World Application
// Part 01 - Step 10: Complete First Program

using System;

namespace HelloWorldApp
{
    class Program
    {
        static void Main(string[] args)
        {
            // ==========================================
            // ขั้นตอนที่ 1: ตั้งค่า Console
            // ==========================================
            Console.Title = "My First C# Program";
            Console.OutputEncoding = System.Text.Encoding.UTF8;
            
            // เปลี่ยนสีข้อความ
            Console.ForegroundColor = ConsoleColor.Cyan;
            
            // ==========================================
            // ขั้นตอนที่ 2: แสดง Header
            // ==========================================
            Console.WriteLine("╔════════════════════════════════════╗");
            Console.WriteLine("║      Welcome to My First C# App!   ║");
            Console.WriteLine("╚════════════════════════════════════╝");
            Console.WriteLine();
            
            // รีเซ็ตสีกลับปกติ
            Console.ResetColor();
            
            // ==========================================
            // ขั้นตอนที่ 3: รับ input จากผู้ใช้
            // ==========================================
            Console.Write("กรุณาใส่ชื่อของคุณ: ");
            string? name = Console.ReadLine();
            
            // ตรวจสอบว่าชื่อไม่ว่างเปล่า
            if (string.IsNullOrWhiteSpace(name))
            {
                name = "ผู้ใช้";  // ใช้ชื่อเริ่มต้น
            }
            
            Console.Write("กรุณาใส่อายุของคุณ: ");
            string? ageInput = Console.ReadLine();
            
            // แปลง string เป็น int
            int age = 0;
            if (!int.TryParse(ageInput, out age))
            {
                age = 0;  // ถ้าแปลงไม่ได้ ใช้ค่าเริ่มต้น
            }
            
            // ==========================================
            // ขั้นตอนที่ 4: ประมวลผลและแสดงผล
            // ==========================================
            Console.WriteLine();
            Console.ForegroundColor = ConsoleColor.Yellow;
            Console.WriteLine($"สวัสดีคุณ {name}!");
            Console.ResetColor();
            
            // แสดงข้อมูลตามอายุ
            if (age < 18)
            {
                Console.WriteLine($"คุณอายุ {age} ปี - ยังเป็นเยาวชน");
            }
            else if (age < 60)
            {
                Console.WriteLine($"คุณอายุ {age} ปี - วัยทำงาน");
            }
            else
            {
                Console.WriteLine($"คุณอายุ {age} ปี - ผู้อาวุโส");
            }
            
            // ==========================================
            // ขั้นตอนที่ 5: แสดงวันและเวลา
            // ==========================================
            Console.WriteLine();
            DateTime now = DateTime.Now;
            Console.WriteLine($"วันที่วันนี้: {now:dd/MM/yyyy}");
            Console.WriteLine($"เวลาปัจจุบัน: {now:HH:mm:ss}");
            Console.WriteLine($"วันในสัปดาห์: {now.DayOfWeek}");
            
            // ==========================================
            // ขั้นตอนที่ 6: แสดงข้อมูล System
            // ==========================================
            Console.WriteLine();
            Console.WriteLine("=== ข้อมูล System ===");
            Console.WriteLine($"Operating System: {Environment.OSVersion}");
            Console.WriteLine($".NET Version: {Environment.Version}");
            Console.WriteLine($"Machine Name: {Environment.MachineName}");
            Console.WriteLine($"User Name: {Environment.UserName}");
            
            // ==========================================
            // ขั้นตอนที่ 7: รอรับ input ก่อนออก
            // ==========================================
            Console.WriteLine();
            Console.ForegroundColor = ConsoleColor.Green;
            Console.WriteLine("โปรแกรมทำงานเสร็จแล้ว! กด Enter เพื่อออก...");
            Console.ResetColor();
            Console.ReadLine();
        }
    }
}
```

### ผลลัพธ์ที่คาดหวัง
```
╔════════════════════════════════════╗
║      Welcome to My First C# App!   ║
╚════════════════════════════════════╝

กรุณาใส่ชื่อของคุณ: สมชาย
กรุณาใส่อายุของคุณ: 25

สวัสดีคุณ สมชาย!
คุณอายุ 25 ปี - วัยทำงาน

วันที่วันนี้: 02/10/2026
เวลาปัจจุบัน: 14:30:00
วันในสัปดาห์: Friday

=== ข้อมูล System ===
Operating System: Unix 6.18.44.0
.NET Version: 8.0.x
Machine Name: MY-PC
User Name: User

โปรแกรมทำงานเสร็จแล้ว! กด Enter เพื่อออก...
```

---

## 📝 สรุป Part 01

| สิ่งที่เรียนรู้ | รายละเอียด |
|----------------|-----------|
| C# คืออะไร | ภาษาโปรแกรมมิ่ง Object-Oriented จาก Microsoft |
| .NET Ecosystem | Framework, Core, .NET 5+ และบทบาทของแต่ละส่วน |
| CLR | Common Language Runtime จัดการ memory และ security |
| Visual Studio | IDE หลักสำหรับพัฒนา C# |
| Console Methods | WriteLine, ReadLine, Write, Clear, Colors |
| Program Structure | namespace, class, Main method, statements |
| Comments | Single-line, Multi-line, XML Documentation |
| .csproj | Project configuration file |
| dotnet CLI | คำสั่งพื้นฐานสำหรับ build, run, manage packages |

---

## 🏋️ แบบฝึกหัด Part 01

### แบบฝึกหัดที่ 1: Calculator แบบง่าย
สร้างโปรแกรมที่:
1. รับตัวเลข 2 ตัวจากผู้ใช้
2. แสดงผลลัพธ์ของ +, -, *, /
3. จัดรูปแบบการแสดงผลให้สวยงาม

```csharp
// ตัวอย่างโค้ด (แนวทาง)
Console.Write("ใส่ตัวเลขที่ 1: ");
string input1 = Console.ReadLine();
double num1 = double.Parse(input1);

Console.Write("ใส่ตัวเลขที่ 2: ");
string input2 = Console.ReadLine();
double num2 = double.Parse(input2);

Console.WriteLine($"บวก: {num1} + {num2} = {num1 + num2}");
Console.WriteLine($"ลบ: {num1} - {num2} = {num1 - num2}");
Console.WriteLine($"คูณ: {num1} × {num2} = {num1 * num2}");

if (num2 != 0)
    Console.WriteLine($"หาร: {num1} ÷ {num2} = {num1 / num2:F2}");
else
    Console.WriteLine("หาร: ไม่สามารถหารด้วย 0 ได้");
```

### แบบฝึกหัดที่ 2: Personal Information Display
สร้างโปรแกรมที่:
1. รับชื่อ, นามสกุล, อายุ, เงินเดือน
2. แสดงข้อมูลในรูปแบบตาราง
3. คำนวณ tax (ภาษี 10%) และแสดงเงินเดือนสุทธิ

### แบบฝึกหัดที่ 3: ASCII Art
สร้างโปรแกรมแสดง ASCII art โดยใช้ Console.WriteLine
```
    *
   ***
  *****
 *******
*********
```

---

## 🔗 แหล่งเรียนรู้เพิ่มเติม
- [Microsoft C# Documentation](https://docs.microsoft.com/en-us/dotnet/csharp/)
- [.NET Documentation](https://docs.microsoft.com/en-us/dotnet/)
- [C# Programming Guide](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/)

---

**ต่อไป → [Part 02: Variables, Data Types & Operators](../part01-10/part02-variables-datatypes.md)**
