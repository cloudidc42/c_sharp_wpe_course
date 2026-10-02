# Part 11: String Manipulation
## ขั้นตอนที่ 101-110: การจัดการ Strings อย่างมืออาชีพ

---

## 🎯 เป้าหมายของ Part นี้
- เข้าใจ string immutability และ StringBuilder
- ใช้ String methods ทั้งหมดได้อย่างคล่องแคล่ว
- String Interpolation, Format, และ Verbatim strings
- Regular Expressions เบื้องต้น
- Span<char> สำหรับ performance
- สร้าง Text Processing utilities

---

## ขั้นตอนที่ 101: String Basics และ Immutability

### String เป็น Immutable
```csharp
// String เป็น reference type แต่ immutable
string s1 = "Hello";
string s2 = s1;
s1 = s1 + " World";  // สร้าง string ใหม่, s2 ยังคงเป็น "Hello"

Console.WriteLine(s1);  // Hello World
Console.WriteLine(s2);  // Hello

// String interning
string a = "Hello";
string b = "Hello";
string c = new string(new char[] { 'H', 'e', 'l', 'l', 'o' });

Console.WriteLine(ReferenceEquals(a, b));  // True (interned)
Console.WriteLine(ReferenceEquals(a, c));  // False (new instance)
Console.WriteLine(a == c);                 // True (value comparison)

// String เทียบกับ null
string? nullable = null;
Console.WriteLine(nullable == null);           // True
Console.WriteLine(nullable?.Length ?? 0);      // 0
Console.WriteLine(string.IsNullOrEmpty(nullable));    // True
Console.WriteLine(string.IsNullOrWhiteSpace("  ")); // True
```

### String Methods ที่ใช้บ่อย
```csharp
string text = "  Hello, World! Hello, C#!  ";

// Trim
Console.WriteLine(text.Trim());           // "Hello, World! Hello, C#!"
Console.WriteLine(text.TrimStart());      // "Hello, World! Hello, C#!  "
Console.WriteLine(text.TrimEnd());        // "  Hello, World! Hello, C#!"

string clean = text.Trim();

// Case
Console.WriteLine(clean.ToUpper());       // HELLO, WORLD! HELLO, C#!
Console.WriteLine(clean.ToLower());       // hello, world! hello, c#!
Console.WriteLine(clean.ToUpperInvariant());  // สำหรับ culture-independent

// Search
Console.WriteLine(clean.Contains("World"));         // True
Console.WriteLine(clean.StartsWith("Hello"));        // True
Console.WriteLine(clean.EndsWith("!"));              // True
Console.WriteLine(clean.IndexOf("Hello"));           // 0
Console.WriteLine(clean.LastIndexOf("Hello"));       // 14
Console.WriteLine(clean.IndexOf("hello", StringComparison.OrdinalIgnoreCase)); // 0

// Extract
Console.WriteLine(clean.Substring(7, 5));    // World
Console.WriteLine(clean[7..12]);             // World (Range syntax C# 8+)
Console.WriteLine(clean[^4..]);              // C#!

// Replace
Console.WriteLine(clean.Replace("Hello", "Hi"));
// Hi, World! Hi, C#!

// Split
string[] parts = clean.Split(',');
foreach (var part in parts)
    Console.WriteLine($"  '{part.Trim()}'");

// Split ด้วยหลาย separators
string csv = "one,two;three|four";
string[] items = csv.Split(new char[] { ',', ';', '|' });

// Join
string joined = string.Join(" - ", items);
Console.WriteLine(joined);  // one - two - three - four
```

---

## ขั้นตอนที่ 102: String Formatting

### String Interpolation
```csharp
string name = "Alice";
int age = 30;
decimal salary = 85000.50m;
DateTime birthDate = new DateTime(1994, 3, 15);

// String interpolation
string info = $"ชื่อ: {name}, อายุ: {age} ปี";

// Format specifiers
string formatted = $"""
    ชื่อ: {name}
    อายุ: {age} ปี
    เงินเดือน: {salary:N2} บาท
    วันเกิด: {birthDate:dd/MM/yyyy}
    """;
Console.WriteLine(formatted);

// Alignment
Console.WriteLine($"{'Product',20} {'Price',10} {'Qty',5}");
Console.WriteLine($"{'Laptop',20} {35000m,10:N2} {5,5}");
Console.WriteLine($"{'Mouse',20} {500m,10:N2} {20,5}");

// Conditional expressions
bool isPremium = true;
Console.WriteLine($"Status: {(isPremium ? "Premium" : "Standard")}");

// Nested interpolation
var data = new[] { 1, 2, 3, 4, 5 };
Console.WriteLine($"Sum: {data.Sum()}, Avg: {data.Average():F2}");

// Raw string literals (C# 11+)
string json = """
    {
        "name": "Alice",
        "age": 30,
        "email": "alice@example.com"
    }
    """;

string regex = """"
    Pattern: \d+\.\d+
    """";
```

### String.Format และ PadLeft/PadRight
```csharp
// String.Format
string result = string.Format("{0,-20} {1,10:N2} {2,5}", "Laptop", 35000m, 5);

// Number formats
double pi = Math.PI;
Console.WriteLine($"Default: {pi}");
Console.WriteLine($"F2: {pi:F2}");        // 3.14
Console.WriteLine($"E3: {pi:E3}");        // 3.142E+000
Console.WriteLine($"N0: {pi:N0}");        // 3
Console.WriteLine($"P2: {0.1234:P2}");    // 12.34%

// เงิน
decimal price = 1234567.89m;
Console.WriteLine($"C: {price:C}");       // ฿1,234,567.89 (depends on culture)
Console.WriteLine($"N2: {price:N2}");     // 1,234,567.89

// วันที่
var now = DateTime.Now;
Console.WriteLine($"d: {now:d}");    // 10/2/2026
Console.WriteLine($"D: {now:D}");    // Friday, October 2, 2026
Console.WriteLine($"t: {now:t}");    // 3:45 PM
Console.WriteLine($"T: {now:T}");    // 3:45:30 PM
Console.WriteLine($"f: {now:f}");    // Friday, October 2, 2026 3:45 PM
Console.WriteLine($"s: {now:s}");    // 2026-10-02T15:45:30
Console.WriteLine($"Custom: {now:dd-MM-yyyy HH:mm}");  // 02-10-2026 15:45

// PadLeft/PadRight
Console.WriteLine("ID".PadRight(10) + "Name".PadRight(20) + "Score".PadLeft(8));
Console.WriteLine("001".PadRight(10) + "Alice".PadRight(20) + "95".PadLeft(8));
```

---

## ขั้นตอนที่ 103: StringBuilder

### StringBuilder สำหรับ Performance
```csharp
using System.Text;

// ❌ Bad - สร้าง string ใหม่ทุกครั้ง O(n²)
string bad = "";
for (int i = 0; i < 10000; i++)
    bad += i.ToString();  // slow!

// ✅ Good - StringBuilder O(n)
var sb = new StringBuilder();
for (int i = 0; i < 10000; i++)
    sb.Append(i);
string good = sb.ToString();

// StringBuilder methods
var builder = new StringBuilder();
builder.Append("Hello");
builder.Append(", ");
builder.AppendLine("World!");            // + newline
builder.Insert(5, " Beautiful");        // insert at index
builder.Replace("Beautiful", "Lovely"); // replace
builder.Remove(5, 7);                   // remove " Lovely"

Console.WriteLine(builder.ToString());
Console.WriteLine($"Length: {builder.Length}");

// Method chaining
var html = new StringBuilder()
    .AppendLine("<html>")
    .AppendLine("<body>")
    .AppendLine("  <h1>Hello World</h1>")
    .AppendLine("</body>")
    .AppendLine("</html>")
    .ToString();

// AppendFormat
var report = new StringBuilder();
report.AppendLine("=== Sales Report ===");
report.AppendFormat("{0,-20} {1,10} {2,10}\n", "Product", "Qty", "Total");
report.AppendFormat("{0,-20} {1,10} {2,10:N2}\n", "Laptop", 5, 175000m);
report.AppendFormat("{0,-20} {1,10} {2,10:N2}\n", "Phone", 10, 150000m);

Console.WriteLine(report.ToString());
```

---

## ขั้นตอนที่ 104: Regular Expressions

### Regex พื้นฐาน
```csharp
using System.Text.RegularExpressions;

// IsMatch - ตรวจสอบว่า match หรือไม่
bool IsEmail(string email)
    => Regex.IsMatch(email, @"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$");

bool IsThaiPhone(string phone)
    => Regex.IsMatch(phone, @"^0[689]\d{8}$");

bool IsThaiId(string id)
    => Regex.IsMatch(id, @"^\d{13}$");

// Match - หา match แรก
string text = "Contact us at info@example.com or support@test.org";
var match = Regex.Match(text, @"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}");

if (match.Success)
    Console.WriteLine($"First email: {match.Value}");

// Matches - หา match ทั้งหมด
var matches = Regex.Matches(text, @"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}");
Console.WriteLine($"Found {matches.Count} emails:");
foreach (Match m in matches)
    Console.WriteLine($"  {m.Value} at index {m.Index}");

// Replace
string cleaned = Regex.Replace(
    "Phone: 081-234-5678 or 02-999-8888",
    @"\d{3}-\d{3}-\d{4}",
    "***-***-****"
);
Console.WriteLine(cleaned);

// Groups
string dateText = "Meeting on 2026-10-15 and 2026-11-20";
var datePattern = new Regex(@"(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})");

foreach (Match m in datePattern.Matches(dateText))
{
    Console.WriteLine($"Date: {m.Groups["year"].Value}/{m.Groups["month"].Value}/{m.Groups["day"].Value}");
}

// Compiled Regex (for performance when used many times)
private static readonly Regex _emailRegex = new Regex(
    @"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$",
    RegexOptions.Compiled | RegexOptions.IgnoreCase
);

// Source Generator Regex (C# 11+)
[GeneratedRegex(@"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$", 
    RegexOptions.IgnoreCase)]
private static partial Regex EmailRegex();
```

---

## ขั้นตอนที่ 105: Span<char> และ Performance

### Span<char> สำหรับ zero-allocation string processing
```csharp
// Span<char> - work with string slices without allocation
void ProcessWithSpan(string data)
{
    ReadOnlySpan<char> span = data.AsSpan();
    
    // Slice โดยไม่ allocate string ใหม่
    ReadOnlySpan<char> firstWord = span[..5];
    ReadOnlySpan<char> rest = span[6..];
    
    Console.WriteLine(firstWord.ToString());
    Console.WriteLine(rest.ToString());
}

// Parse numbers from span (faster than string.Parse)
string numbers = "123,456,789";
ReadOnlySpan<char> numSpan = numbers.AsSpan();

int total = 0;
foreach (var numStr in numSpan.ToString().Split(','))
{
    if (int.TryParse(numStr, out int n))
        total += n;
}
Console.WriteLine($"Total: {total}");

// เปรียบเทียบ string operations
string TestStringPerformance(int iterations)
{
    var sw = System.Diagnostics.Stopwatch.StartNew();
    
    string result = "";
    for (int i = 0; i < iterations; i++)
        result += i;
    
    sw.Stop();
    return $"String concat: {sw.ElapsedMilliseconds}ms";
}

string TestStringBuilderPerformance(int iterations)
{
    var sw = System.Diagnostics.Stopwatch.StartNew();
    
    var sb = new StringBuilder();
    for (int i = 0; i < iterations; i++)
        sb.Append(i);
    _ = sb.ToString();
    
    sw.Stop();
    return $"StringBuilder: {sw.ElapsedMilliseconds}ms";
}

Console.WriteLine(TestStringPerformance(10_000));
Console.WriteLine(TestStringBuilderPerformance(10_000));
```

---

## ขั้นตอนที่ 106-110: โปรแกรมตัวอย่าง - Text Processing Engine

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Text.RegularExpressions;

namespace TextProcessingEngine
{
    // Text Statistics
    record TextStats(
        int WordCount,
        int SentenceCount,
        int ParagraphCount,
        int CharCount,
        double AverageWordLength,
        IReadOnlyDictionary<string, int> WordFrequency,
        string MostCommonWord
    );
    
    // Text Transformer
    class TextTransformer
    {
        // camelCase → snake_case
        public string ToSnakeCase(string input)
        {
            var sb = new StringBuilder();
            for (int i = 0; i < input.Length; i++)
            {
                char c = input[i];
                if (char.IsUpper(c) && i > 0)
                    sb.Append('_');
                sb.Append(char.ToLower(c));
            }
            return sb.ToString();
        }
        
        // snake_case → PascalCase
        public string ToPascalCase(string input)
        {
            return string.Join("", input.Split('_')
                .Where(s => !string.IsNullOrEmpty(s))
                .Select(s => char.ToUpper(s[0]) + s[1..].ToLower()));
        }
        
        // Title Case
        public string ToTitleCase(string input)
        {
            var smallWords = new HashSet<string> { "a", "an", "the", "and", "or", "but", "in", "on", "at", "to", "for" };
            
            var words = input.ToLower().Split(' ');
            var result = words.Select((word, index) => 
                index == 0 || !smallWords.Contains(word)
                    ? char.ToUpper(word[0]) + word[1..]
                    : word);
            
            return string.Join(" ", result);
        }
        
        // Word Wrap
        public string WordWrap(string text, int maxWidth = 80)
        {
            var sb = new StringBuilder();
            int currentWidth = 0;
            
            foreach (string word in text.Split(' '))
            {
                if (currentWidth + word.Length + 1 > maxWidth && currentWidth > 0)
                {
                    sb.AppendLine();
                    currentWidth = 0;
                }
                
                if (currentWidth > 0)
                {
                    sb.Append(' ');
                    currentWidth++;
                }
                
                sb.Append(word);
                currentWidth += word.Length;
            }
            
            return sb.ToString();
        }
        
        // Remove extra whitespace
        public string Normalize(string input)
        {
            return Regex.Replace(input.Trim(), @"\s+", " ");
        }
        
        // Truncate with ellipsis
        public string Truncate(string input, int maxLength, string suffix = "...")
        {
            if (input.Length <= maxLength) return input;
            return input[..(maxLength - suffix.Length)] + suffix;
        }
        
        // Mask sensitive data
        public string MaskEmail(string email)
        {
            int atIndex = email.IndexOf('@');
            if (atIndex <= 0) return email;
            
            string local = email[..atIndex];
            string domain = email[atIndex..];
            
            string maskedLocal = local.Length <= 2 
                ? new string('*', local.Length)
                : local[0] + new string('*', local.Length - 2) + local[^1];
            
            return maskedLocal + domain;
        }
        
        public string MaskCreditCard(string cardNumber)
        {
            string digits = Regex.Replace(cardNumber, @"\D", "");
            if (digits.Length < 4) return "****";
            return new string('*', digits.Length - 4) + digits[^4..];
        }
    }
    
    // Text Analyzer
    class TextAnalyzer
    {
        private static readonly Regex WordPattern = new Regex(@"\b[a-zA-Zก-๙]+\b", RegexOptions.Compiled);
        private static readonly Regex SentencePattern = new Regex(@"[.!?]+", RegexOptions.Compiled);
        
        public TextStats Analyze(string text)
        {
            if (string.IsNullOrWhiteSpace(text))
                return new TextStats(0, 0, 0, 0, 0, new Dictionary<string, int>(), "");
            
            // Words
            var words = WordPattern.Matches(text.ToLower())
                .Select(m => m.Value)
                .ToList();
            
            // Word frequency
            var frequency = words
                .GroupBy(w => w)
                .ToDictionary(g => g.Key, g => g.Count());
            
            // Most common
            string mostCommon = frequency.Any() 
                ? frequency.MaxBy(kvp => kvp.Value).Key
                : "";
            
            // Sentences
            int sentences = Math.Max(1, SentencePattern.Matches(text).Count);
            
            // Paragraphs
            int paragraphs = text.Split(new[] { "\n\n", "\r\n\r\n" }, 
                StringSplitOptions.RemoveEmptyEntries).Length;
            
            // Average word length
            double avgLength = words.Any() ? words.Average(w => w.Length) : 0;
            
            return new TextStats(
                words.Count,
                sentences,
                paragraphs,
                text.Length,
                avgLength,
                frequency,
                mostCommon
            );
        }
        
        public string GenerateWordCloud(string text, int topN = 10)
        {
            var stats = Analyze(text);
            var sb = new StringBuilder();
            
            sb.AppendLine("=== Word Cloud ===");
            
            var topWords = stats.WordFrequency
                .OrderByDescending(kvp => kvp.Value)
                .Take(topN);
            
            int maxCount = topWords.FirstOrDefault().Value;
            
            foreach (var (word, count) in topWords)
            {
                int barLength = (int)(count * 20.0 / maxCount);
                string bar = new string('█', barLength);
                sb.AppendLine($"{word,15} {bar} ({count})");
            }
            
            return sb.ToString();
        }
    }
    
    // Template Engine
    class SimpleTemplateEngine
    {
        private readonly Dictionary<string, string> _variables = new();
        
        public void SetVariable(string name, string value)
            => _variables[name] = value;
        
        public void SetVariables(Dictionary<string, string> variables)
        {
            foreach (var (key, value) in variables)
                _variables[key] = value;
        }
        
        public string Render(string template)
        {
            string result = template;
            
            foreach (var (key, value) in _variables)
            {
                result = result.Replace($"{{{{{key}}}}}", value);
            }
            
            // แจ้ง variables ที่ยังไม่ได้กำหนด
            var unresolved = Regex.Matches(result, @"\{\{(\w+)\}\}")
                .Select(m => m.Groups[1].Value)
                .Distinct()
                .ToList();
            
            if (unresolved.Any())
                Console.WriteLine($"⚠️ Unresolved variables: {string.Join(", ", unresolved)}");
            
            return result;
        }
    }
    
    class Program
    {
        static void Main()
        {
            Console.OutputEncoding = Encoding.UTF8;
            
            var transformer = new TextTransformer();
            var analyzer = new TextAnalyzer();
            var template = new SimpleTemplateEngine();
            
            // Test Transformer
            Console.WriteLine("=== Text Transformer ===");
            string camel = "myVariableName";
            string snake = transformer.ToSnakeCase(camel);
            string pascal = transformer.ToPascalCase(snake);
            Console.WriteLine($"camelCase: {camel}");
            Console.WriteLine($"snake_case: {snake}");
            Console.WriteLine($"PascalCase: {pascal}");
            
            string title = "the quick brown fox jumps over the lazy dog";
            Console.WriteLine($"\nOriginal: {title}");
            Console.WriteLine($"Title Case: {transformer.ToTitleCase(title)}");
            
            // Masking
            Console.WriteLine("\n=== Data Masking ===");
            Console.WriteLine(transformer.MaskEmail("alice@example.com"));
            Console.WriteLine(transformer.MaskEmail("bob@test.org"));
            Console.WriteLine(transformer.MaskCreditCard("4111 1111 1111 1111"));
            
            // Truncate
            string longText = "This is a very long text that needs to be truncated for display purposes.";
            Console.WriteLine($"\nOriginal: {longText}");
            Console.WriteLine($"Truncated: {transformer.Truncate(longText, 40)}");
            
            // Analyzer
            Console.WriteLine("\n=== Text Analyzer ===");
            string sample = @"The quick brown fox jumps over the lazy dog.
The fox was very quick and the dog was very lazy.
Both the fox and the dog were animals.

This is a second paragraph about animals.
Animals are interesting creatures.";
            
            var stats = analyzer.Analyze(sample);
            Console.WriteLine($"Words: {stats.WordCount}");
            Console.WriteLine($"Sentences: {stats.SentenceCount}");
            Console.WriteLine($"Paragraphs: {stats.ParagraphCount}");
            Console.WriteLine($"Characters: {stats.CharCount}");
            Console.WriteLine($"Avg word length: {stats.AverageWordLength:F2}");
            Console.WriteLine($"Most common: '{stats.MostCommonWord}'");
            
            Console.WriteLine(analyzer.GenerateWordCloud(sample, 8));
            
            // Template Engine
            Console.WriteLine("=== Template Engine ===");
            string emailTemplate = @"เรียน {{name}},

ขอบคุณสำหรับการสั่งซื้อ Order #{{orderId}}
สินค้าของคุณ: {{product}}
จำนวน: {{quantity}} ชิ้น
ราคารวม: {{total}} บาท

กรุณาชำระภายใน {{dueDate}}

ขอบคุณ,
ทีมงาน {{company}}";
            
            template.SetVariables(new Dictionary<string, string>
            {
                { "name", "คุณสมชาย ใจดี" },
                { "orderId", "ORD-2026-001234" },
                { "product", "Laptop ASUS ROG" },
                { "quantity", "1" },
                { "total", "45,000.00" },
                { "dueDate", "10 ตุลาคม 2026" },
                { "company", "TechShop Thailand" }
            });
            
            Console.WriteLine(template.Render(emailTemplate));
        }
    }
}
```

---

## 📝 สรุป Part 11

| หัวข้อ | Key Points |
|--------|-----------|
| String Immutability | สร้าง object ใหม่ทุกครั้งที่ modify |
| StringBuilder | ใช้เมื่อ concatenate หลายครั้ง |
| Interpolation | `$"..."` ใช้แทน string.Format |
| Raw strings | `"""..."""` สำหรับ multi-line |
| Regex | Pattern matching, extract, replace |
| Span<char> | Zero-allocation string slicing |
| Format codes | N2, C, d, D, s สำหรับ number/date |

---

**ก่อนหน้า → [Part 10: Exception Handling](../part01-10/part10-exception-handling.md)**  
**ต่อไป → [Part 12: File I/O](part12-file-io.md)**
