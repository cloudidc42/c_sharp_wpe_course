# Part 04: Loops - for, while, foreach, do-while
## ขั้นตอนที่ 31-40: การวนซ้ำในโปรแกรม

---

## 🎯 เป้าหมายของ Part นี้
- ใช้ for loop อย่างมีประสิทธิภาพ
- ใช้ while และ do-while loop ให้ถูกกรณี
- ใช้ foreach กับ collections ต่างๆ
- ใช้ break, continue, return ควบคุม loop
- เขียน nested loops และเข้าใจ performance implications

---

## ขั้นตอนที่ 31: for Loop

### รูปแบบ for Loop
```csharp
// รูปแบบ: for (initializer; condition; iterator)
for (int i = 0; i < 5; i++)
{
    Console.WriteLine($"รอบที่ {i}");
}
// Output: รอบที่ 0, 1, 2, 3, 4

// นับขึ้น
for (int i = 1; i <= 10; i++)
{
    Console.Write($"{i} ");
}
// Output: 1 2 3 4 5 6 7 8 9 10

// นับลง
for (int i = 10; i >= 1; i--)
{
    Console.Write($"{i} ");
}
// Output: 10 9 8 7 6 5 4 3 2 1

// นับทีละ 2
for (int i = 0; i <= 20; i += 2)
{
    Console.Write($"{i} ");
}
// Output: 0 2 4 6 8 10 12 14 16 18 20

// ประกาศหลายตัวแปร
for (int i = 0, j = 10; i < j; i++, j--)
{
    Console.WriteLine($"i={i}, j={j}");
}
```

### for Loop กับ Arrays
```csharp
string[] names = { "สมชาย", "สมหญิง", "สมศักดิ์", "สมปอง" };

// วนผ่าน array ด้วย index
for (int i = 0; i < names.Length; i++)
{
    Console.WriteLine($"{i + 1}. {names[i]}");
}

// แก้ไขค่าใน array
int[] scores = { 85, 72, 90, 68, 95 };
for (int i = 0; i < scores.Length; i++)
{
    scores[i] += 5;  // เพิ่มคะแนน 5 ทุกคน (แต่ไม่เกิน 100)
    if (scores[i] > 100) scores[i] = 100;
}

// วน array แบบย้อนกลับ
for (int i = names.Length - 1; i >= 0; i--)
{
    Console.Write($"{names[i]} ");
}
```

### for Loop แบบ Infinite (ใช้ break ออก)
```csharp
// Loop ไม่มีที่สิ้นสุด - ต้องใช้ break
for (;;)  // เทียบเท่า while(true)
{
    Console.Write("ใส่ตัวเลข (0 เพื่อออก): ");
    if (int.TryParse(Console.ReadLine(), out int num))
    {
        if (num == 0) break;
        Console.WriteLine($"กำลังสอง: {num * num}");
    }
}
```

---

## ขั้นตอนที่ 32: while Loop

### รูปแบบ while Loop
```csharp
// while - ตรวจสอบ condition ก่อน (อาจไม่ทำงานเลย)
int count = 0;
while (count < 5)
{
    Console.WriteLine($"count = {count}");
    count++;
}

// ใช้ while กับ input validation
int age = -1;
while (age < 0 || age > 120)
{
    Console.Write("ใส่อายุ (0-120): ");
    if (!int.TryParse(Console.ReadLine(), out age))
    {
        age = -1;  // reset ถ้าไม่ใช่ตัวเลข
        Console.WriteLine("กรุณาใส่ตัวเลข");
    }
}
Console.WriteLine($"อายุ: {age}");

// while loop กับ file reading
// using (StreamReader reader = new StreamReader("file.txt"))
// {
//     string? line;
//     while ((line = reader.ReadLine()) != null)
//     {
//         Console.WriteLine(line);
//     }
// }
```

### do-while Loop
```csharp
// do-while - ทำงานอย่างน้อยหนึ่งครั้งก่อน
int number;
do
{
    Console.Write("ใส่ตัวเลขระหว่าง 1-10: ");
    int.TryParse(Console.ReadLine(), out number);
    
    if (number < 1 || number > 10)
    {
        Console.WriteLine("ตัวเลขไม่อยู่ในช่วงที่กำหนด");
    }
} while (number < 1 || number > 10);

Console.WriteLine($"คุณใส่: {number}");

// เมนูที่แสดงอย่างน้อยหนึ่งครั้ง
string? choice;
do
{
    Console.WriteLine("\n=== เมนู ===");
    Console.WriteLine("1. เลือกที่ 1");
    Console.WriteLine("2. เลือกที่ 2");
    Console.WriteLine("Q. ออก");
    Console.Write("เลือก: ");
    choice = Console.ReadLine();
    
    switch (choice?.ToUpper())
    {
        case "1": Console.WriteLine("เลือกที่ 1"); break;
        case "2": Console.WriteLine("เลือกที่ 2"); break;
        case "Q": Console.WriteLine("ออกจากระบบ"); break;
        default: Console.WriteLine("เลือกไม่ถูกต้อง"); break;
    }
} while (choice?.ToUpper() != "Q");
```

---

## ขั้นตอนที่ 33: foreach Loop

### foreach กับ Collections ต่างๆ
```csharp
// foreach กับ Array
string[] fruits = { "แอปเปิ้ล", "กล้วย", "มะม่วง", "ส้ม", "องุ่น" };
foreach (string fruit in fruits)
{
    Console.WriteLine($"• {fruit}");
}

// foreach กับ List<T>
var numbers = new List<int> { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
int sum = 0;
foreach (int num in numbers)
{
    sum += num;
}
Console.WriteLine($"ผลรวม: {sum}");  // 55

// foreach กับ Dictionary
var population = new Dictionary<string, int>
{
    { "กรุงเทพ", 10000000 },
    { "เชียงใหม่", 1200000 },
    { "ขอนแก่น", 1800000 },
    { "ภูเก็ต", 900000 }
};

foreach (KeyValuePair<string, int> city in population)
{
    Console.WriteLine($"{city.Key}: {city.Value:N0} คน");
}

// ใช้ var (สั้นกว่า)
foreach (var (city, pop) in population)  // Deconstruct tuple
{
    Console.WriteLine($"{city}: {pop:N0} คน");
}

// foreach กับ string (วนผ่านแต่ละตัวอักษร)
string word = "Hello";
foreach (char c in word)
{
    Console.Write($"{c} ");  // H e l l o
}
```

### ⚠️ ข้อจำกัดของ foreach
```csharp
int[] numbers = { 1, 2, 3, 4, 5 };

// ❌ ไม่สามารถแก้ไข collection ขณะ foreach
foreach (int num in numbers)
{
    // numbers[index] = num * 2;  ← Error หรือไม่ได้ผล
}

// ✅ ใช้ for แทนถ้าต้องการแก้ไข
for (int i = 0; i < numbers.Length; i++)
{
    numbers[i] *= 2;
}

// ❌ ไม่สามารถ Add/Remove ใน List ขณะ foreach
var list = new List<int> { 1, 2, 3, 4, 5 };
// foreach (int num in list)
// {
//     if (num == 3) list.Remove(num);  ← InvalidOperationException!
// }

// ✅ วิธีแก้: เก็บไว้ก่อนแล้วค่อย Remove
var toRemove = new List<int>();
foreach (int num in list)
{
    if (num == 3) toRemove.Add(num);
}
foreach (int num in toRemove)
{
    list.Remove(num);
}

// ✅ หรือใช้ RemoveAll
list.RemoveAll(num => num == 3);
```

---

## ขั้นตอนที่ 34: break และ continue

### break - หยุด loop ทันที
```csharp
// หา element แรกที่ตรงเงื่อนไข
int[] numbers = { 3, 7, 2, 9, 5, 8, 1, 6 };
int target = 9;
int foundIndex = -1;

for (int i = 0; i < numbers.Length; i++)
{
    if (numbers[i] == target)
    {
        foundIndex = i;
        break;  // พอเจอแล้วหยุด ไม่ต้องวนต่อ
    }
}

if (foundIndex >= 0)
    Console.WriteLine($"พบ {target} ที่ index {foundIndex}");
else
    Console.WriteLine($"ไม่พบ {target}");

// break ใน while
int guess = 0;
int secret = 42;
int attempts = 0;

while (true)
{
    attempts++;
    Console.Write($"เดาตัวเลข (ครั้งที่ {attempts}): ");
    
    if (!int.TryParse(Console.ReadLine(), out guess))
        continue;
    
    if (guess == secret)
    {
        Console.WriteLine($"ถูกต้อง! ใช้ {attempts} ครั้ง");
        break;
    }
    else if (guess < secret)
        Console.WriteLine("น้อยเกินไป");
    else
        Console.WriteLine("มากเกินไป");
    
    if (attempts >= 10)
    {
        Console.WriteLine($"หมดโอกาส! ตัวเลขคือ {secret}");
        break;
    }
}
```

### continue - ข้ามรอบนี้
```csharp
// ข้ามตัวเลขคู่ (แสดงแค่คี่)
for (int i = 1; i <= 20; i++)
{
    if (i % 2 == 0) continue;  // ข้ามถ้าคู่
    Console.Write($"{i} ");
}
// Output: 1 3 5 7 9 11 13 15 17 19

// กรองข้อมูล
string[] students = { "สมชาย", "", "สมหญิง", null, "สมศักดิ์" };
foreach (string? student in students)
{
    if (string.IsNullOrWhiteSpace(student)) continue;  // ข้ามที่ว่าง
    Console.WriteLine($"นักเรียน: {student}");
}

// Process valid data only
int[] data = { 5, -3, 8, -1, 12, -7, 0, 4 };
int positiveSum = 0;
foreach (int num in data)
{
    if (num <= 0) continue;  // ข้ามค่า 0 หรือติดลบ
    positiveSum += num;
}
Console.WriteLine($"ผลรวมบวก: {positiveSum}");  // 29
```

---

## ขั้นตอนที่ 35: Nested Loops

### Nested for Loops
```csharp
// ตาราง Multiplication
Console.WriteLine("ตาราง Multiplication (1-5)");
Console.Write("    ");
for (int i = 1; i <= 5; i++) Console.Write($"{i,4}");
Console.WriteLine();
Console.WriteLine("    " + new string('-', 20));

for (int i = 1; i <= 5; i++)
{
    Console.Write($"{i,3}|");
    for (int j = 1; j <= 5; j++)
    {
        Console.Write($"{i * j,4}");
    }
    Console.WriteLine();
}

// สร้าง Pattern
// *
// **
// ***
// ****
// *****
for (int row = 1; row <= 5; row++)
{
    for (int col = 1; col <= row; col++)
    {
        Console.Write("*");
    }
    Console.WriteLine();
}

// Diamond Pattern
int n = 5;
// ครึ่งบน
for (int i = 1; i <= n; i++)
{
    // Spaces
    for (int j = 1; j <= n - i; j++) Console.Write(" ");
    // Stars
    for (int j = 1; j <= 2 * i - 1; j++) Console.Write("*");
    Console.WriteLine();
}
// ครึ่งล่าง
for (int i = n - 1; i >= 1; i--)
{
    for (int j = 1; j <= n - i; j++) Console.Write(" ");
    for (int j = 1; j <= 2 * i - 1; j++) Console.Write("*");
    Console.WriteLine();
}
```

### 2D Array กับ Nested Loops
```csharp
// สร้าง 2D Array
int[,] matrix = new int[3, 4];

// กำหนดค่า
for (int row = 0; row < 3; row++)
{
    for (int col = 0; col < 4; col++)
    {
        matrix[row, col] = row * 4 + col + 1;
    }
}

// แสดงผล
Console.WriteLine("Matrix:");
for (int row = 0; row < 3; row++)
{
    for (int col = 0; col < 4; col++)
    {
        Console.Write($"{matrix[row, col],4}");
    }
    Console.WriteLine();
}
// Output:
//    1   2   3   4
//    5   6   7   8
//    9  10  11  12

// หา max ใน matrix
int max = matrix[0, 0];
for (int row = 0; row < matrix.GetLength(0); row++)
{
    for (int col = 0; col < matrix.GetLength(1); col++)
    {
        if (matrix[row, col] > max)
            max = matrix[row, col];
    }
}
Console.WriteLine($"Max: {max}");  // 12
```

### Break ออกจาก Nested Loops
```csharp
// วิธีที่ 1: ใช้ flag
bool found = false;
int[,] grid = { { 1, 2, 3 }, { 4, 5, 6 }, { 7, 8, 9 } };
int searchValue = 5;
int foundRow = -1, foundCol = -1;

for (int row = 0; row < 3 && !found; row++)
{
    for (int col = 0; col < 3 && !found; col++)
    {
        if (grid[row, col] == searchValue)
        {
            foundRow = row;
            foundCol = col;
            found = true;
        }
    }
}

if (found)
    Console.WriteLine($"พบที่ [{foundRow}, {foundCol}]");

// วิธีที่ 2: ใช้ goto (บางครั้งใช้ได้)
for (int row = 0; row < 3; row++)
{
    for (int col = 0; col < 3; col++)
    {
        if (grid[row, col] == searchValue)
        {
            foundRow = row;
            foundCol = col;
            goto FoundIt;
        }
    }
}
FoundIt:
Console.WriteLine($"พบที่ [{foundRow}, {foundCol}]");

// วิธีที่ 3: แยก Method (ดีที่สุด!)
(int row, int col) FindInGrid(int[,] g, int value)
{
    for (int r = 0; r < g.GetLength(0); r++)
        for (int c = 0; c < g.GetLength(1); c++)
            if (g[r, c] == value)
                return (r, c);
    return (-1, -1);
}

var (r, c) = FindInGrid(grid, searchValue);
Console.WriteLine($"พบที่ [{r}, {c}]");
```

---

## ขั้นตอนที่ 36: LINQ กับ Loops (Preview)

### LINQ แทน Loops ที่ซับซ้อน
```csharp
int[] numbers = { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

// แบบ for loop
int sumOfEven = 0;
for (int i = 0; i < numbers.Length; i++)
{
    if (numbers[i] % 2 == 0)
        sumOfEven += numbers[i];
}

// แบบ LINQ (กระชับกว่า)
int sumOfEvenLinq = numbers.Where(n => n % 2 == 0).Sum();

// ทั้งสองให้ผลเหมือนกัน: 30
Console.WriteLine(sumOfEven);      // 30
Console.WriteLine(sumOfEvenLinq);  // 30

// ตัวอย่างเพิ่มเติม
string[] cities = { "กรุงเทพ", "เชียงใหม่", "ขอนแก่น", "ภูเก็ต", "หาดใหญ่" };

// แบบ loop
var shortCities = new List<string>();
foreach (var city in cities)
{
    if (city.Length <= 5)
        shortCities.Add(city);
}

// แบบ LINQ
var shortCitiesLinq = cities.Where(c => c.Length <= 5).ToList();
```

---

## ขั้นตอนที่ 37: Loop Performance

### เทคนิค Performance สำหรับ Loops
```csharp
// ✅ Cache Length ออกมา (บางกรณี)
string[] bigArray = new string[100000];

// แบบปกติ
for (int i = 0; i < bigArray.Length; i++)  // .Length ถูก evaluate ทุก iteration

// แบบ cache (อาจเร็วกว่าเล็กน้อยสำหรับ array ปกติ)
int len = bigArray.Length;
for (int i = 0; i < len; i++)

// ✅ หลีกเลี่ยงการสร้าง object ใน loop
// ❌ Bad
for (int i = 0; i < 10000; i++)
{
    StringBuilder sb = new StringBuilder();  // สร้างใหม่ทุกรอบ!
    sb.Append($"Item {i}");
}

// ✅ Good
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++)
{
    sb.Clear();
    sb.Append($"Item {i}");
}

// ✅ ใช้ Span<T> สำหรับ performance-critical code (C# 7.2+)
int[] data = { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
Span<int> span = data;
foreach (ref int item in span)
{
    item *= 2;  // แก้ไขได้โดยตรง ไม่ต้องสร้าง copy
}

// ✅ Parallel.For สำหรับงานที่สามารถขนานกันได้
// (จะเรียนใน Part ขั้นสูง)
Parallel.For(0, 100, i =>
{
    // แต่ละ iteration รันใน thread ต่างๆ
    DoHeavyWork(i);
});

void DoHeavyWork(int i) { /* heavy computation */ }
```

---

## ขั้นตอนที่ 38: Iterator และ yield

### yield return - สร้าง sequence แบบ lazy
```csharp
// สร้าง sequence ของตัวเลขฟีโบนักชี
IEnumerable<long> Fibonacci()
{
    long a = 0, b = 1;
    while (true)  // sequence ไม่มีที่สิ้นสุด!
    {
        yield return a;
        (a, b) = (b, a + b);  // tuple swap
    }
}

// ใช้ Take() เพื่อจำกัดจำนวน
foreach (long num in Fibonacci().Take(15))
{
    Console.Write($"{num} ");
}
// Output: 0 1 1 2 3 5 8 13 21 34 55 89 144 233 377

// สร้าง number range
IEnumerable<int> Range(int start, int end, int step = 1)
{
    for (int i = start; i <= end; i += step)
    {
        yield return i;
    }
}

foreach (int n in Range(1, 20, 3))
{
    Console.Write($"{n} ");  // 1 4 7 10 13 16 19
}

// yield break
IEnumerable<int> TakeWhilePositive(int[] numbers)
{
    foreach (int n in numbers)
    {
        if (n <= 0) yield break;  // หยุด iteration
        yield return n;
    }
}

int[] mixed = { 3, 7, 2, -1, 5, 8 };  // หยุดที่ -1
foreach (int n in TakeWhilePositive(mixed))
{
    Console.Write($"{n} ");  // 3 7 2
}
```

---

## ขั้นตอนที่ 39: Loop Patterns ที่พบบ่อย

### Common Loop Patterns
```csharp
// Pattern 1: Accumulator (สะสมค่า)
int[] scores = { 85, 72, 90, 68, 95, 88, 76 };
double sum = 0;
foreach (int score in scores)
{
    sum += score;
}
double average = sum / scores.Length;
Console.WriteLine($"เฉลี่ย: {average:F2}");

// Pattern 2: Find Max/Min
int max = int.MinValue;
int min = int.MaxValue;
foreach (int score in scores)
{
    if (score > max) max = score;
    if (score < min) min = score;
}
Console.WriteLine($"สูงสุด: {max}, ต่ำสุด: {min}");

// Pattern 3: Count/Filter
int countPassed = 0;
var passedScores = new List<int>();
foreach (int score in scores)
{
    if (score >= 70)
    {
        countPassed++;
        passedScores.Add(score);
    }
}
Console.WriteLine($"ผ่าน: {countPassed}/{scores.Length}");

// Pattern 4: Transform (แปลงข้อมูล)
double[] normalizedScores = new double[scores.Length];
for (int i = 0; i < scores.Length; i++)
{
    normalizedScores[i] = (double)scores[i] / 100.0;
}

// Pattern 5: Build String
var parts = new List<string> { "apple", "banana", "cherry" };
string result = "";
for (int i = 0; i < parts.Count; i++)
{
    result += parts[i];
    if (i < parts.Count - 1) result += ", ";
}
Console.WriteLine(result);  // apple, banana, cherry

// ✅ ดีกว่า: ใช้ string.Join
result = string.Join(", ", parts);

// Pattern 6: Pagination
int[] items = Enumerable.Range(1, 100).ToArray();
int pageSize = 10;
int pageNumber = 3;  // หน้าที่ 3

int startIndex = (pageNumber - 1) * pageSize;
int endIndex = Math.Min(startIndex + pageSize, items.Length);

Console.WriteLine($"หน้า {pageNumber} (แสดง {pageSize} รายการ):");
for (int i = startIndex; i < endIndex; i++)
{
    Console.Write($"{items[i]} ");
}
// Output: หน้า 3 (แสดง 10 รายการ): 21 22 23 24 25 26 27 28 29 30
```

---

## ขั้นตอนที่ 40: โปรแกรมตัวอย่าง - เกมเดาตัวเลข (Advanced)

```csharp
using System;
using System.Collections.Generic;

namespace NumberGuessingGame
{
    class Program
    {
        record GameResult(int Attempts, bool Won, int SecretNumber, DateTime PlayedAt);
        
        static List<GameResult> gameHistory = new();
        static Random random = new Random();
        
        static void Main(string[] args)
        {
            Console.OutputEncoding = System.Text.Encoding.UTF8;
            Console.Title = "Number Guessing Game";
            
            bool playing = true;
            while (playing)
            {
                ShowMainMenu();
                
                Console.Write("\nเลือก: ");
                string? choice = Console.ReadLine();
                
                switch (choice)
                {
                    case "1":
                        PlayGame(difficulty: "easy");
                        break;
                    case "2":
                        PlayGame(difficulty: "medium");
                        break;
                    case "3":
                        PlayGame(difficulty: "hard");
                        break;
                    case "4":
                        ShowHistory();
                        break;
                    case "5":
                        ShowStatistics();
                        break;
                    case "0":
                        playing = false;
                        break;
                    default:
                        Console.WriteLine("เลือกไม่ถูกต้อง");
                        break;
                }
                
                if (playing && choice != "0")
                {
                    Console.WriteLine("\nกด Enter เพื่อดำเนินการต่อ...");
                    Console.ReadLine();
                }
            }
            
            Console.ForegroundColor = ConsoleColor.Yellow;
            Console.WriteLine("\nขอบคุณที่เล่น! แล้วพบกันใหม่!");
            Console.ResetColor();
            Console.ReadLine();
        }
        
        static void ShowMainMenu()
        {
            Console.Clear();
            Console.ForegroundColor = ConsoleColor.Cyan;
            Console.WriteLine("╔════════════════════════════════════╗");
            Console.WriteLine("║      🎮 เกมเดาตัวเลข              ║");
            Console.WriteLine("╠════════════════════════════════════╣");
            Console.ForegroundColor = ConsoleColor.White;
            Console.WriteLine("║  1. เริ่มเกม (ง่าย: 1-50, 10ครั้ง) ║");
            Console.WriteLine("║  2. เริ่มเกม (กลาง: 1-100, 7ครั้ง) ║");
            Console.WriteLine("║  3. เริ่มเกม (ยาก: 1-200, 5ครั้ง)  ║");
            Console.WriteLine("║  4. ประวัติการเล่น                  ║");
            Console.WriteLine("║  5. สถิติ                           ║");
            Console.WriteLine("║  0. ออก                             ║");
            Console.ForegroundColor = ConsoleColor.Cyan;
            Console.WriteLine("╚════════════════════════════════════╝");
            Console.ResetColor();
            
            if (gameHistory.Count > 0)
            {
                int wins = 0;
                foreach (var result in gameHistory)
                    if (result.Won) wins++;
                Console.WriteLine($"\nเล่นแล้ว: {gameHistory.Count} ครั้ง | ชนะ: {wins} ครั้ง");
            }
        }
        
        static void PlayGame(string difficulty)
        {
            int maxNumber, maxAttempts;
            
            switch (difficulty)
            {
                case "easy":
                    maxNumber = 50;
                    maxAttempts = 10;
                    break;
                case "medium":
                    maxNumber = 100;
                    maxAttempts = 7;
                    break;
                case "hard":
                    maxNumber = 200;
                    maxAttempts = 5;
                    break;
                default:
                    maxNumber = 100;
                    maxAttempts = 7;
                    break;
            }
            
            int secretNumber = random.Next(1, maxNumber + 1);
            int attempts = 0;
            bool won = false;
            
            Console.Clear();
            Console.ForegroundColor = ConsoleColor.Yellow;
            Console.WriteLine($"🎯 เดาตัวเลขระหว่าง 1 ถึง {maxNumber}");
            Console.WriteLine($"⏳ คุณมี {maxAttempts} โอกาส");
            Console.ResetColor();
            Console.WriteLine(new string('─', 40));
            
            // แสดง hint bar
            var guesses = new List<int>();
            
            while (attempts < maxAttempts && !won)
            {
                int remainingAttempts = maxAttempts - attempts;
                
                // แสดง progress
                Console.Write($"\nโอกาสที่เหลือ: ");
                Console.ForegroundColor = remainingAttempts switch
                {
                    > 5 => ConsoleColor.Green,
                    > 2 => ConsoleColor.Yellow,
                    _ => ConsoleColor.Red
                };
                Console.Write(new string('❤', remainingAttempts));
                Console.ResetColor();
                Console.WriteLine();
                
                // แสดงช่วงที่เหลือ
                int lowerBound = 1, upperBound = maxNumber;
                foreach (int g in guesses)
                {
                    if (g < secretNumber && g > lowerBound) lowerBound = g;
                    if (g > secretNumber && g < upperBound) upperBound = g;
                }
                if (guesses.Count > 0)
                    Console.WriteLine($"ช่วงที่เป็นไปได้: {lowerBound} - {upperBound}");
                
                Console.Write("เดา: ");
                string? input = Console.ReadLine();
                
                if (!int.TryParse(input, out int guess))
                {
                    Console.ForegroundColor = ConsoleColor.Red;
                    Console.WriteLine("กรุณาใส่ตัวเลข");
                    Console.ResetColor();
                    continue;
                }
                
                if (guess < 1 || guess > maxNumber)
                {
                    Console.ForegroundColor = ConsoleColor.Red;
                    Console.WriteLine($"กรุณาเดาระหว่าง 1-{maxNumber}");
                    Console.ResetColor();
                    continue;
                }
                
                if (guesses.Contains(guess))
                {
                    Console.ForegroundColor = ConsoleColor.Yellow;
                    Console.WriteLine("คุณเดาตัวเลขนี้ไปแล้ว!");
                    Console.ResetColor();
                    continue;
                }
                
                attempts++;
                guesses.Add(guess);
                
                if (guess == secretNumber)
                {
                    won = true;
                    Console.ForegroundColor = ConsoleColor.Green;
                    Console.WriteLine($"\n🎉 ยินดีด้วย! ถูกต้อง! ตัวเลขคือ {secretNumber}");
                    Console.WriteLine($"คุณเดาถูกใน {attempts} ครั้ง!");
                    Console.ResetColor();
                    
                    // คะแนน
                    int score = (maxAttempts - attempts + 1) * 100 / maxAttempts;
                    Console.WriteLine($"คะแนน: {score}%");
                }
                else if (guess < secretNumber)
                {
                    Console.ForegroundColor = ConsoleColor.Yellow;
                    int diff = secretNumber - guess;
                    string hint = diff switch
                    {
                        < 5 => "🔥 ร้อนมาก! ใกล้มากๆ",
                        < 15 => "☀️ อุ่นๆ ใกล้แล้ว",
                        < 30 => "❄️ เย็นๆ ยังห่างอยู่",
                        _ => "🧊 เย็นมาก! ห่างมาก"
                    };
                    Console.WriteLine($"น้อยเกินไป {hint}");
                    Console.ResetColor();
                }
                else
                {
                    Console.ForegroundColor = ConsoleColor.Magenta;
                    int diff = guess - secretNumber;
                    string hint = diff switch
                    {
                        < 5 => "🔥 ร้อนมาก! ใกล้มากๆ",
                        < 15 => "☀️ อุ่นๆ ใกล้แล้ว",
                        < 30 => "❄️ เย็นๆ ยังห่างอยู่",
                        _ => "🧊 เย็นมาก! ห่างมาก"
                    };
                    Console.WriteLine($"มากเกินไป {hint}");
                    Console.ResetColor();
                }
            }
            
            if (!won)
            {
                Console.ForegroundColor = ConsoleColor.Red;
                Console.WriteLine($"\n😢 หมดโอกาสแล้ว! ตัวเลขคือ {secretNumber}");
                Console.ResetColor();
            }
            
            // บันทึกผล
            gameHistory.Add(new GameResult(attempts, won, secretNumber, DateTime.Now));
        }
        
        static void ShowHistory()
        {
            Console.Clear();
            Console.WriteLine("=== ประวัติการเล่น ===\n");
            
            if (gameHistory.Count == 0)
            {
                Console.WriteLine("ยังไม่มีประวัติการเล่น");
                return;
            }
            
            Console.WriteLine($"{"#",-4} {"ผล",-8} {"ตัวเลข",-10} {"ครั้ง",-8} {"เวลา",-20}");
            Console.WriteLine(new string('-', 55));
            
            for (int i = 0; i < gameHistory.Count; i++)
            {
                var result = gameHistory[i];
                Console.ForegroundColor = result.Won ? ConsoleColor.Green : ConsoleColor.Red;
                Console.WriteLine($"{i + 1,-4} {(result.Won ? "✅ ชนะ" : "❌ แพ้"),-8} {result.SecretNumber,-10} {result.Attempts,-8} {result.PlayedAt:dd/MM HH:mm:ss}");
                Console.ResetColor();
            }
        }
        
        static void ShowStatistics()
        {
            Console.Clear();
            Console.WriteLine("=== สถิติ ===\n");
            
            if (gameHistory.Count == 0)
            {
                Console.WriteLine("ยังไม่มีข้อมูล");
                return;
            }
            
            int totalGames = gameHistory.Count;
            int totalWins = 0;
            int totalAttempts = 0;
            int minAttempts = int.MaxValue;
            int maxAttempts = int.MinValue;
            
            foreach (var result in gameHistory)
            {
                if (result.Won)
                {
                    totalWins++;
                    if (result.Attempts < minAttempts) minAttempts = result.Attempts;
                    if (result.Attempts > maxAttempts) maxAttempts = result.Attempts;
                }
                totalAttempts += result.Attempts;
            }
            
            double winRate = (double)totalWins / totalGames * 100;
            double avgAttempts = (double)totalAttempts / totalGames;
            
            Console.WriteLine($"จำนวนเกมทั้งหมด: {totalGames}");
            Console.WriteLine($"ชนะ: {totalWins} ({winRate:F1}%)");
            Console.WriteLine($"แพ้: {totalGames - totalWins}");
            Console.WriteLine($"เฉลี่ยจำนวนครั้งต่อเกม: {avgAttempts:F1}");
            
            if (totalWins > 0)
            {
                Console.WriteLine($"ครั้งน้อยสุด (ที่ชนะ): {minAttempts}");
                Console.WriteLine($"ครั้งมากสุด (ที่ชนะ): {maxAttempts}");
            }
            
            // Win rate bar
            Console.Write("\nอัตราชนะ: [");
            int bars = (int)(winRate / 5);
            Console.ForegroundColor = ConsoleColor.Green;
            Console.Write(new string('█', bars));
            Console.ResetColor();
            Console.Write(new string('░', 20 - bars));
            Console.WriteLine($"] {winRate:F1}%");
        }
    }
}
```

---

## 📝 สรุป Part 04

| Loop Type | เมื่อใช้ |
|-----------|---------|
| for | รู้จำนวนรอบล่วงหน้า, ต้องการ index |
| while | ไม่รู้จำนวนรอบ, ตรวจเงื่อนไขก่อน |
| do-while | ต้องทำอย่างน้อย 1 ครั้ง |
| foreach | วนผ่าน collection, อ่านง่ายกว่า for |

| Keyword | ใช้สำหรับ |
|---------|-----------|
| break | หยุด loop ทันที |
| continue | ข้ามรอบปัจจุบัน |
| return | ออกจาก method (หยุด loop ด้วย) |
| yield return | สร้าง lazy sequence |

---

## 🏋️ แบบฝึกหัด Part 04

### แบบฝึกหัดที่ 1: Prime Numbers
สร้างโปรแกรมหาจำนวนเฉพาะ (Prime Numbers) ระหว่าง 1-100

### แบบฝึกหัดที่ 2: Sorting Algorithm
เขียน Bubble Sort ด้วย nested loops สำหรับเรียงลำดับ array

### แบบฝึกหัดที่ 3: Pascal's Triangle
สร้าง Pascal's Triangle โดยใช้ nested loops แสดง 10 แถว

---

**ก่อนหน้า → [Part 03: Control Flow](part03-control-flow.md)**  
**ต่อไป → [Part 05: Methods & Functions](part05-methods.md)**
