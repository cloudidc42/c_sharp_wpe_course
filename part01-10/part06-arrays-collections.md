# Part 06: Arrays & Collections
## ขั้นตอนที่ 51-60: Arrays, List, Dictionary และ Collections

---

## 🎯 เป้าหมายของ Part นี้
- ใช้ Arrays ทุกรูปแบบ (1D, 2D, Jagged)
- รู้จัก Generic Collections: List<T>, Dictionary<TKey,TValue>
- ใช้ Stack<T>, Queue<T>, HashSet<T>
- เข้าใจ IEnumerable<T> และ iteration
- ใช้ Collection Initializers
- รู้จัก Span<T> และ Memory<T>

---

## ขั้นตอนที่ 51: Arrays

### Array พื้นฐาน
```csharp
// การสร้าง Array
int[] numbers = new int[5];          // สร้าง array ขนาด 5, ค่า default = 0
string[] names = new string[3];      // ค่า default = null
bool[] flags = new bool[4];          // ค่า default = false

// Initialization
int[] scores = { 85, 72, 90, 68, 95 };
string[] fruits = new string[] { "apple", "banana", "cherry" };
double[] temps = new double[] { 36.5, 37.0, 36.8 };

// Array properties
Console.WriteLine(scores.Length);     // 5
Console.WriteLine(scores.Rank);       // 1 (มิติ)
Console.WriteLine(scores[0]);         // 85 (ตัวแรก)
Console.WriteLine(scores[^1]);        // 95 (ตัวสุดท้าย, C# 8+)
Console.WriteLine(scores[^2]);        // 68 (สองตัวสุดท้าย)

// Range operator (..)
int[] subset = scores[1..4];          // [72, 90, 68] (index 1 ถึง 3)
int[] first3 = scores[..3];           // [85, 72, 90] (3 ตัวแรก)
int[] last2 = scores[^2..];           // [68, 95] (2 ตัวสุดท้าย)
int[] copy = scores[..];              // copy ทั้งหมด

// แก้ไขค่า
scores[0] = 100;
scores[^1] = 99;
```

### Array Methods
```csharp
int[] numbers = { 5, 3, 8, 1, 9, 2, 7, 4, 6 };

// Sort
Array.Sort(numbers);
Console.WriteLine(string.Join(", ", numbers));  // 1, 2, 3, 4, 5, 6, 7, 8, 9

// Sort with Comparison
string[] names = { "Charlie", "Alice", "Bob" };
Array.Sort(names, (a, b) => string.Compare(a, b, StringComparison.Ordinal));
Console.WriteLine(string.Join(", ", names));  // Alice, Bob, Charlie

// Reverse
Array.Reverse(numbers);
Console.WriteLine(string.Join(", ", numbers));  // 9, 8, 7, 6, 5, 4, 3, 2, 1

// Search
Array.Sort(numbers);  // ต้อง sort ก่อน BinarySearch
int index = Array.BinarySearch(numbers, 5);
Console.WriteLine($"พบ 5 ที่ index {index}");  // 4

// IndexOf (ไม่ต้อง sort)
int linearSearch = Array.IndexOf(numbers, 7);
Console.WriteLine($"พบ 7 ที่ index {linearSearch}");

// Copy
int[] source = { 1, 2, 3, 4, 5 };
int[] dest = new int[5];
Array.Copy(source, dest, source.Length);

// Clone
int[] cloned = (int[])source.Clone();

// Fill
int[] filled = new int[5];
Array.Fill(filled, 42);
Console.WriteLine(string.Join(", ", filled));  // 42, 42, 42, 42, 42

// Clear
Array.Clear(filled, 1, 3);  // Clear index 1-3
Console.WriteLine(string.Join(", ", filled));  // 42, 0, 0, 0, 42

// Exists, TrueForAll, Find
bool hasLarge = Array.Exists(numbers, n => n > 7);       // True
bool allPositive = Array.TrueForAll(numbers, n => n > 0); // True
int firstLarge = Array.Find(numbers, n => n > 7);         // 8
int[] allLarge = Array.FindAll(numbers, n => n > 7);      // [8, 9]
```

### Multi-dimensional Arrays
```csharp
// 2D Array (Matrix)
int[,] matrix = new int[3, 4];  // 3 rows, 4 columns

// Initialization
int[,] grid = {
    { 1, 2, 3, 4 },
    { 5, 6, 7, 8 },
    { 9, 10, 11, 12 }
};

// Access
Console.WriteLine(grid[1, 2]);   // 7 (row 1, col 2)
Console.WriteLine(grid.GetLength(0));  // 3 (rows)
Console.WriteLine(grid.GetLength(1));  // 4 (cols)
Console.WriteLine(grid.Length);        // 12 (total elements)

// Loop through 2D array
for (int row = 0; row < grid.GetLength(0); row++)
{
    for (int col = 0; col < grid.GetLength(1); col++)
    {
        Console.Write($"{grid[row, col],4}");
    }
    Console.WriteLine();
}

// 3D Array
int[,,] cube = new int[2, 3, 4];  // 2x3x4
```

### Jagged Arrays (Array ของ Arrays)
```csharp
// Jagged arrays - แต่ละแถวมีขนาดต่างกัน
int[][] jagged = new int[3][];
jagged[0] = new int[] { 1, 2 };
jagged[1] = new int[] { 3, 4, 5, 6 };
jagged[2] = new int[] { 7, 8, 9 };

// หรือ initialize พร้อมกัน
int[][] triangle = new int[][]
{
    new int[] { 1 },
    new int[] { 1, 2 },
    new int[] { 1, 2, 3 },
    new int[] { 1, 2, 3, 4 }
};

// Access
Console.WriteLine(jagged[1][2]);  // 5

// Loop
foreach (int[] row in jagged)
{
    foreach (int val in row)
        Console.Write($"{val} ");
    Console.WriteLine();
}
```

---

## ขั้นตอนที่ 52: List<T>

### List<T> พื้นฐาน
```csharp
// สร้าง List
var numbers = new List<int>();
var names = new List<string> { "Alice", "Bob", "Charlie" };
var scores = new List<double>(capacity: 100);  // Initial capacity

// Add elements
numbers.Add(10);
numbers.Add(20);
numbers.Add(30);
numbers.AddRange(new[] { 40, 50, 60 });

// Insert
numbers.Insert(2, 25);  // Insert 25 at index 2
Console.WriteLine(string.Join(", ", numbers));  // 10, 20, 25, 30, 40, 50, 60

// Remove
numbers.Remove(25);           // Remove first occurrence of 25
numbers.RemoveAt(0);          // Remove at index 0
numbers.RemoveRange(1, 2);    // Remove 2 elements starting at index 1
numbers.RemoveAll(n => n > 40); // Remove all where condition

// Properties and Methods
Console.WriteLine(numbers.Count);        // จำนวน elements
Console.WriteLine(numbers.Capacity);     // ขนาด buffer ปัจจุบัน
Console.WriteLine(numbers.Contains(30)); // True
Console.WriteLine(numbers.IndexOf(30));  // หา index
Console.WriteLine(numbers[0]);           // Access by index

// Sort
names.Sort();
names.Sort((a, b) => b.CompareTo(a));  // Descending
names.Reverse();

// Search
string? found = names.Find(n => n.StartsWith("B"));
List<string> allFound = names.FindAll(n => n.Length > 3);
int idx = names.FindIndex(n => n == "Alice");

// Convert to Array
int[] arr = numbers.ToArray();

// LINQ on List
var evenNumbers = numbers.Where(n => n % 2 == 0).ToList();
int sum = numbers.Sum();
double avg = numbers.Average();
```

### List Operations ขั้นสูง
```csharp
// ForEach method
var items = new List<string> { "apple", "banana", "cherry" };
items.ForEach(item => Console.WriteLine(item.ToUpper()));

// Exists, TrueForAll
bool hasLong = items.Exists(s => s.Length > 6);   // True (banana, cherry)
bool allShort = items.TrueForAll(s => s.Length < 10); // True

// Sorting with Comparer
var students = new List<(string name, int score)>
{
    ("Alice", 85),
    ("Bob", 92),
    ("Charlie", 78),
    ("Diana", 92)
};

// Sort by score (desc), then by name (asc)
students.Sort((a, b) => 
    b.score != a.score ? b.score.CompareTo(a.score) : 
    string.Compare(a.name, b.name, StringComparison.Ordinal));

foreach (var (name, score) in students)
    Console.WriteLine($"{name}: {score}");
// Bob: 92, Diana: 92, Alice: 85, Charlie: 78

// Binary Search (list ต้อง sorted)
var sortedNumbers = new List<int> { 1, 3, 5, 7, 9, 11 };
int pos = sortedNumbers.BinarySearch(7);  // 3
```

---

## ขั้นตอนที่ 53: Dictionary<TKey, TValue>

### Dictionary พื้นฐาน
```csharp
// สร้าง Dictionary
var ages = new Dictionary<string, int>();
var config = new Dictionary<string, string>
{
    { "server", "localhost" },
    { "port", "5432" },
    { "database", "mydb" }
};

// เพิ่ม/แก้ไข
ages["Alice"] = 25;
ages["Bob"] = 30;
ages.Add("Charlie", 28);      // ถ้า key ซ้ำ = ArgumentException

// อ่านค่า
Console.WriteLine(ages["Alice"]);     // 25
// ages["Unknown"]  ← KeyNotFoundException!

// ปลอดภัยกว่า
if (ages.TryGetValue("Bob", out int bobAge))
    Console.WriteLine($"Bob's age: {bobAge}");

int age = ages.GetValueOrDefault("Unknown", 0);  // 0 ถ้าไม่พบ

// Check existence
Console.WriteLine(ages.ContainsKey("Alice"));    // True
Console.WriteLine(ages.ContainsValue(30));        // True

// Remove
ages.Remove("Bob");

// Properties
Console.WriteLine(ages.Count);                   // 2
Console.WriteLine(string.Join(", ", ages.Keys));  // Alice, Charlie
Console.WriteLine(string.Join(", ", ages.Values)); // 25, 28

// Loop
foreach (KeyValuePair<string, int> pair in ages)
    Console.WriteLine($"{pair.Key}: {pair.Value}");

// Deconstruct
foreach (var (name, a) in ages)
    Console.WriteLine($"{name}: {a}");

// Clear
ages.Clear();
```

### Dictionary ขั้นสูง
```csharp
// Dictionary กับ complex values
var studentScores = new Dictionary<string, List<int>>
{
    { "Alice", new List<int> { 85, 90, 78 } },
    { "Bob", new List<int> { 72, 88, 95 } },
    { "Charlie", new List<int> { 90, 85, 92 } }
};

foreach (var (name, scores) in studentScores)
{
    double avg = scores.Average();
    Console.WriteLine($"{name}: {avg:F1}");
}

// Nested Dictionary
var cityDistricts = new Dictionary<string, Dictionary<string, int>>
{
    { "กรุงเทพ", new Dictionary<string, int> { {"บางรัก", 50000}, {"สาทร", 40000} } },
    { "เชียงใหม่", new Dictionary<string, int> { {"เมือง", 30000} } }
};

// GroupBy สร้าง Dictionary
var words = new[] { "apple", "ant", "bear", "bee", "cat", "car" };
var grouped = words.GroupBy(w => w[0])
    .ToDictionary(g => g.Key, g => g.ToList());

foreach (var (letter, wordList) in grouped)
    Console.WriteLine($"{letter}: {string.Join(", ", wordList)}");
// a: apple, ant
// b: bear, bee
// c: cat, car

// ConcurrentDictionary (thread-safe)
// var concurrent = new System.Collections.Concurrent.ConcurrentDictionary<string, int>();
// concurrent.AddOrUpdate("key", 1, (k, v) => v + 1);
```

---

## ขั้นตอนที่ 54: HashSet<T>

### HashSet - unique elements เท่านั้น
```csharp
// HashSet - ไม่มีค่าซ้ำ, ค้นหาเร็ว O(1)
var uniqueNumbers = new HashSet<int> { 1, 2, 3, 4, 5 };

// Add
uniqueNumbers.Add(3);   // ไม่ถูกเพิ่ม (มีอยู่แล้ว)
uniqueNumbers.Add(6);   // ถูกเพิ่ม
Console.WriteLine(uniqueNumbers.Count);  // 6

// Contains - O(1)
Console.WriteLine(uniqueNumbers.Contains(3));   // True
Console.WriteLine(uniqueNumbers.Contains(10));  // False

// Set Operations
var set1 = new HashSet<int> { 1, 2, 3, 4, 5 };
var set2 = new HashSet<int> { 4, 5, 6, 7, 8 };

// Union (รวม)
var union = new HashSet<int>(set1);
union.UnionWith(set2);
Console.WriteLine(string.Join(", ", union));  // 1, 2, 3, 4, 5, 6, 7, 8

// Intersection (ส่วนที่ซ้ำกัน)
var intersection = new HashSet<int>(set1);
intersection.IntersectWith(set2);
Console.WriteLine(string.Join(", ", intersection));  // 4, 5

// Difference (ส่วนที่ต่าง)
var difference = new HashSet<int>(set1);
difference.ExceptWith(set2);
Console.WriteLine(string.Join(", ", difference));  // 1, 2, 3

// Symmetric Difference (ส่วนที่ไม่ซ้ำกัน)
var symDiff = new HashSet<int>(set1);
symDiff.SymmetricExceptWith(set2);
Console.WriteLine(string.Join(", ", symDiff));  // 1, 2, 3, 6, 7, 8

// IsSubsetOf, IsSupersetOf
var small = new HashSet<int> { 1, 2 };
Console.WriteLine(small.IsSubsetOf(set1));    // True
Console.WriteLine(set1.IsSupersetOf(small));  // True

// Remove duplicates from list
var withDups = new List<int> { 1, 2, 2, 3, 3, 3, 4 };
var unique = new HashSet<int>(withDups);
Console.WriteLine(string.Join(", ", unique));  // 1, 2, 3, 4
```

---

## ขั้นตอนที่ 55: Stack<T> และ Queue<T>

### Stack<T> - LIFO (Last In, First Out)
```csharp
// Stack - เหมือนกองจาน (ใส่ด้านบน เอาจากด้านบน)
var stack = new Stack<int>();

// Push - เพิ่มด้านบน
stack.Push(1);
stack.Push(2);
stack.Push(3);
Console.WriteLine(stack.Count);  // 3

// Peek - ดูด้านบน (ไม่เอาออก)
Console.WriteLine(stack.Peek());  // 3

// Pop - เอาออกจากด้านบน
Console.WriteLine(stack.Pop());   // 3
Console.WriteLine(stack.Pop());   // 2
Console.WriteLine(stack.Count);   // 1

// TryPeek, TryPop (safe versions)
if (stack.TryPop(out int value))
    Console.WriteLine($"Popped: {value}");

// ตัวอย่าง: ตรวจสอบ brackets
bool IsBalanced(string expression)
{
    var stack = new Stack<char>();
    var pairs = new Dictionary<char, char>
    {
        { ')', '(' }, { ']', '[' }, { '}', '{' }
    };
    
    foreach (char c in expression)
    {
        if (c is '(' or '[' or '{')
        {
            stack.Push(c);
        }
        else if (pairs.ContainsKey(c))
        {
            if (stack.Count == 0 || stack.Pop() != pairs[c])
                return false;
        }
    }
    
    return stack.Count == 0;
}

Console.WriteLine(IsBalanced("(a + [b * {c}])"));  // True
Console.WriteLine(IsBalanced("(a + [b * c)"));     // False

// ตัวอย่าง: Undo/Redo
class TextEditor
{
    private string text = "";
    private Stack<string> undoStack = new();
    private Stack<string> redoStack = new();
    
    public void Type(string addition)
    {
        undoStack.Push(text);
        redoStack.Clear();
        text += addition;
    }
    
    public void Undo()
    {
        if (undoStack.Count > 0)
        {
            redoStack.Push(text);
            text = undoStack.Pop();
        }
    }
    
    public void Redo()
    {
        if (redoStack.Count > 0)
        {
            undoStack.Push(text);
            text = redoStack.Pop();
        }
    }
    
    public string GetText() => text;
}
```

### Queue<T> - FIFO (First In, First Out)
```csharp
// Queue - เหมือนแถวคิว (เข้าด้านหลัง ออกด้านหน้า)
var queue = new Queue<string>();

// Enqueue - เพิ่มด้านหลัง
queue.Enqueue("First");
queue.Enqueue("Second");
queue.Enqueue("Third");

// Peek - ดูด้านหน้า (ไม่เอาออก)
Console.WriteLine(queue.Peek());    // First

// Dequeue - เอาออกจากด้านหน้า
Console.WriteLine(queue.Dequeue()); // First
Console.WriteLine(queue.Dequeue()); // Second
Console.WriteLine(queue.Count);     // 1

// TryDequeue, TryPeek (safe versions)
if (queue.TryDequeue(out string? item))
    Console.WriteLine($"Dequeued: {item}");

// ตัวอย่าง: Print Queue
class PrintQueue
{
    private Queue<string> _jobs = new();
    
    public void AddJob(string document)
    {
        _jobs.Enqueue(document);
        Console.WriteLine($"เพิ่มงาน: {document} (ลำดับที่ {_jobs.Count})");
    }
    
    public void ProcessNext()
    {
        if (_jobs.TryDequeue(out string? job))
            Console.WriteLine($"พิมพ์: {job} ({_jobs.Count} งานที่เหลือ)");
        else
            Console.WriteLine("ไม่มีงานในคิว");
    }
    
    public int JobCount => _jobs.Count;
}

var printer = new PrintQueue();
printer.AddJob("report.pdf");
printer.AddJob("invoice.pdf");
printer.AddJob("photo.png");
printer.ProcessNext();
printer.ProcessNext();
```

---

## ขั้นตอนที่ 56: LinkedList<T> และ SortedXxx

### LinkedList<T>
```csharp
// LinkedList - ดีสำหรับ insert/delete บ่อย
var linkedList = new LinkedList<int>();

// AddFirst, AddLast
linkedList.AddLast(3);
linkedList.AddLast(5);
linkedList.AddFirst(1);  // [1, 3, 5]

// AddAfter, AddBefore
var node3 = linkedList.Find(3)!;
linkedList.AddAfter(node3, 4);   // [1, 3, 4, 5]
linkedList.AddBefore(node3, 2);  // [1, 2, 3, 4, 5]

// Traverse
foreach (int val in linkedList)
    Console.Write($"{val} ");
// 1 2 3 4 5

// First, Last
Console.WriteLine(linkedList.First!.Value);  // 1
Console.WriteLine(linkedList.Last!.Value);   // 5

// Remove
linkedList.Remove(3);
linkedList.RemoveFirst();
linkedList.RemoveLast();
```

### SortedList<TKey, TValue> และ SortedDictionary
```csharp
// SortedList - เรียงลำดับตาม key อัตโนมัติ
var sortedList = new SortedList<string, int>
{
    { "Charlie", 85 },
    { "Alice", 92 },
    { "Bob", 78 }
};

// แสดงผลเรียงตาม key
foreach (var (name, score) in sortedList)
    Console.WriteLine($"{name}: {score}");
// Alice: 92, Bob: 78, Charlie: 85

// Access by index
Console.WriteLine(sortedList.Keys[0]);   // Alice
Console.WriteLine(sortedList.Values[0]); // 92

// SortedDictionary - ดีกว่าสำหรับ insert/delete
var sortedDict = new SortedDictionary<int, string>
{
    { 3, "Three" },
    { 1, "One" },
    { 2, "Two" }
};

foreach (var (key, value) in sortedDict)
    Console.Write($"{key}:{value} ");
// 1:One 2:Two 3:Three
```

---

## ขั้นตอนที่ 57: IEnumerable<T> และ Custom Collections

### IEnumerable<T>
```csharp
// IEnumerable<T> - Interface สำหรับ iteration
// ทุก collection (Array, List, Dictionary) implement IEnumerable<T>

void PrintAll<T>(IEnumerable<T> items)
{
    foreach (var item in items)
        Console.WriteLine(item);
}

// ใช้กับทุก collection
PrintAll(new int[] { 1, 2, 3 });
PrintAll(new List<string> { "a", "b", "c" });
PrintAll(new HashSet<double> { 1.1, 2.2, 3.3 });

// สร้าง Custom IEnumerable
class NumberRange : IEnumerable<int>
{
    private int _start, _end, _step;
    
    public NumberRange(int start, int end, int step = 1)
    {
        _start = start;
        _end = end;
        _step = step;
    }
    
    public IEnumerator<int> GetEnumerator()
    {
        for (int i = _start; i <= _end; i += _step)
            yield return i;
    }
    
    System.Collections.IEnumerator System.Collections.IEnumerable.GetEnumerator()
        => GetEnumerator();
}

// ใช้งาน
foreach (int n in new NumberRange(1, 20, 3))
    Console.Write($"{n} ");  // 1 4 7 10 13 16 19

// LINQ ใช้ได้เลย
var doubled = new NumberRange(1, 10).Select(n => n * 2).ToList();
```

---

## ขั้นตอนที่ 58: Collection Expressions (C# 12)

### Collection Expressions
```csharp
// C# 12 - Collection expressions
int[] array1 = [1, 2, 3, 4, 5];
List<int> list1 = [1, 2, 3, 4, 5];
HashSet<string> set1 = ["a", "b", "c"];
Span<int> span1 = [1, 2, 3];

// Spread operator (..)
int[] first = [1, 2, 3];
int[] second = [4, 5, 6];
int[] combined = [..first, ..second];  // [1, 2, 3, 4, 5, 6]
int[] withExtra = [0, ..first, ..second, 7];  // [0, 1, 2, 3, 4, 5, 6, 7]

// กับ List
List<string> fruits = ["apple", "banana"];
List<string> veggies = ["carrot", "potato"];
List<string> food = [..fruits, ..veggies, "chocolate"];

Console.WriteLine(string.Join(", ", food));
// apple, banana, carrot, potato, chocolate
```

---

## ขั้นตอนที่ 59: Performance Comparison

### เมื่อใช้ Collection ไหน?
```csharp
/*
┌──────────────────────────────────────────────────────────────────┐
│                    Collection Comparison                          │
├──────────────────┬────────────┬────────────┬────────────┬────────┤
│ Collection       │ Access     │ Search     │ Insert     │ Delete │
├──────────────────┼────────────┼────────────┼────────────┼────────┤
│ Array            │ O(1)       │ O(n)       │ N/A        │ N/A   │
│ List<T>          │ O(1)       │ O(n)       │ O(n)*/O(1) │ O(n)  │
│ LinkedList<T>    │ O(n)       │ O(n)       │ O(1)       │ O(1)  │
│ Dictionary       │ O(1)       │ O(1)       │ O(1)       │ O(1)  │
│ SortedDict       │ O(log n)   │ O(log n)   │ O(log n)   │ O(log n)│
│ HashSet          │ -          │ O(1)       │ O(1)       │ O(1)  │
│ SortedSet        │ -          │ O(log n)   │ O(log n)   │ O(log n)│
│ Stack            │ O(1) top   │ O(n)       │ O(1)       │ O(1)  │
│ Queue            │ O(1) front │ O(n)       │ O(1)       │ O(1)  │
└──────────────────┴────────────┴────────────┴────────────┴────────┘
*/

// ตัวอย่างการเลือก Collection ที่เหมาะสม

// ✅ ใช้ Array เมื่อ: ขนาดคงที่, access เร็ว, performance critical
double[] coordinates = new double[1000000];

// ✅ ใช้ List<T> เมื่อ: ขนาดไม่แน่นอน, access by index บ่อย
var items = new List<string>();

// ✅ ใช้ Dictionary เมื่อ: lookup by key, mapping relationships
var cache = new Dictionary<string, object>();

// ✅ ใช้ HashSet เมื่อ: ต้องการ unique values, set operations
var visitedUrls = new HashSet<string>();

// ✅ ใช้ Queue เมื่อ: task queue, message processing
var messageQueue = new Queue<Message>();

// ✅ ใช้ Stack เมื่อ: undo/redo, DFS, expression evaluation
var undoHistory = new Stack<Action>();

class Message { public string Content { get; set; } = ""; }
class Action { }
```

---

## ขั้นตอนที่ 60: โปรแกรมตัวอย่าง - Inventory Management

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace InventoryManagement
{
    record Product(
        string Id,
        string Name,
        string Category,
        decimal Price,
        int Quantity
    );
    
    class Inventory
    {
        private Dictionary<string, Product> _products = new();
        private Queue<string> _recentlyViewed = new();
        private HashSet<string> _lowStockAlerts = new();
        private Stack<(string action, Product product)> _history = new();
        
        const int LOW_STOCK_THRESHOLD = 5;
        const int RECENT_VIEW_LIMIT = 10;
        
        public void AddProduct(Product product)
        {
            if (_products.ContainsKey(product.Id))
                throw new InvalidOperationException($"Product {product.Id} already exists");
            
            _products[product.Id] = product;
            _history.Push(("ADD", product));
            CheckLowStock(product);
            
            Console.WriteLine($"✅ เพิ่ม: {product.Name} (ID: {product.Id})");
        }
        
        public Product? GetProduct(string id)
        {
            if (!_products.TryGetValue(id, out var product))
                return null;
            
            // Track recently viewed
            TrackView(id);
            return product;
        }
        
        public void UpdateStock(string id, int quantity)
        {
            if (!_products.TryGetValue(id, out var product))
                throw new KeyNotFoundException($"Product {id} not found");
            
            var oldProduct = product;
            var newProduct = product with { Quantity = product.Quantity + quantity };
            _products[id] = newProduct;
            _history.Push(("UPDATE_STOCK", oldProduct));
            CheckLowStock(newProduct);
            
            string action = quantity >= 0 ? "เพิ่ม" : "ลด";
            Console.WriteLine($"📦 {action}สต็อก {product.Name}: {product.Quantity} → {newProduct.Quantity}");
        }
        
        public bool RemoveProduct(string id)
        {
            if (!_products.TryGetValue(id, out var product))
                return false;
            
            _products.Remove(id);
            _lowStockAlerts.Remove(id);
            _history.Push(("REMOVE", product));
            
            Console.WriteLine($"🗑️ ลบ: {product.Name}");
            return true;
        }
        
        public void UndoLastAction()
        {
            if (!_history.TryPop(out var lastAction))
            {
                Console.WriteLine("ไม่มีการกระทำให้ Undo");
                return;
            }
            
            var (action, product) = lastAction;
            
            switch (action)
            {
                case "ADD":
                    _products.Remove(product.Id);
                    Console.WriteLine($"Undo: ลบ {product.Name}");
                    break;
                case "REMOVE":
                    _products[product.Id] = product;
                    Console.WriteLine($"Undo: คืน {product.Name}");
                    break;
                case "UPDATE_STOCK":
                    _products[product.Id] = product;
                    Console.WriteLine($"Undo: คืนสต็อก {product.Name} เป็น {product.Quantity}");
                    break;
            }
        }
        
        public IEnumerable<Product> Search(
            string? name = null,
            string? category = null,
            decimal? minPrice = null,
            decimal? maxPrice = null,
            bool inStockOnly = false)
        {
            var query = _products.Values.AsEnumerable();
            
            if (name != null)
                query = query.Where(p => p.Name.Contains(name, 
                    StringComparison.OrdinalIgnoreCase));
            if (category != null)
                query = query.Where(p => p.Category.Equals(category, 
                    StringComparison.OrdinalIgnoreCase));
            if (minPrice.HasValue)
                query = query.Where(p => p.Price >= minPrice.Value);
            if (maxPrice.HasValue)
                query = query.Where(p => p.Price <= maxPrice.Value);
            if (inStockOnly)
                query = query.Where(p => p.Quantity > 0);
            
            return query.OrderBy(p => p.Name);
        }
        
        public Dictionary<string, List<Product>> GetByCategory()
        {
            return _products.Values
                .GroupBy(p => p.Category)
                .ToDictionary(g => g.Key, g => g.OrderBy(p => p.Name).ToList());
        }
        
        public IEnumerable<Product> GetLowStock()
        {
            return _lowStockAlerts
                .Where(id => _products.ContainsKey(id))
                .Select(id => _products[id])
                .OrderBy(p => p.Quantity);
        }
        
        public IEnumerable<string> GetRecentlyViewed()
        {
            return _recentlyViewed
                .Where(id => _products.ContainsKey(id))
                .Select(id => _products[id].Name);
        }
        
        public (decimal totalValue, int totalItems, int categories) GetStats()
        {
            decimal totalValue = _products.Values.Sum(p => p.Price * p.Quantity);
            int totalItems = _products.Values.Sum(p => p.Quantity);
            int categories = _products.Values.Select(p => p.Category).Distinct().Count();
            return (totalValue, totalItems, categories);
        }
        
        private void CheckLowStock(Product product)
        {
            if (product.Quantity <= LOW_STOCK_THRESHOLD)
                _lowStockAlerts.Add(product.Id);
            else
                _lowStockAlerts.Remove(product.Id);
        }
        
        private void TrackView(string id)
        {
            // Remove if already in queue (to move to front)
            var temp = _recentlyViewed.Where(x => x != id).ToList();
            _recentlyViewed.Clear();
            
            _recentlyViewed.Enqueue(id);  // Add to front (conceptually)
            foreach (var item in temp.Take(RECENT_VIEW_LIMIT - 1))
                _recentlyViewed.Enqueue(item);
        }
        
        public void DisplayProducts(IEnumerable<Product>? products = null)
        {
            var list = (products ?? _products.Values).ToList();
            
            if (!list.Any())
            {
                Console.WriteLine("ไม่พบสินค้า");
                return;
            }
            
            Console.WriteLine($"\n{"ID",-10} {"ชื่อสินค้า",-25} {"หมวดหมู่",-15} {"ราคา",10} {"สต็อก",8}");
            Console.WriteLine(new string('─', 73));
            
            foreach (var p in list)
            {
                bool isLow = _lowStockAlerts.Contains(p.Id);
                Console.ForegroundColor = p.Quantity == 0 ? ConsoleColor.Red :
                    isLow ? ConsoleColor.Yellow : ConsoleColor.White;
                
                string name = p.Name.Length > 23 ? p.Name[..23] + ".." : p.Name;
                string stockStr = p.Quantity == 0 ? "หมด" : 
                    (isLow ? $"{p.Quantity}⚠️" : p.Quantity.ToString());
                
                Console.WriteLine($"{p.Id,-10} {name,-25} {p.Category,-15} {p.Price,10:N2} {stockStr,8}");
                Console.ResetColor();
            }
            
            Console.WriteLine(new string('─', 73));
            Console.WriteLine($"รวม {list.Count} รายการ");
        }
    }
    
    class Program
    {
        static Inventory inventory = new Inventory();
        
        static void Main(string[] args)
        {
            Console.OutputEncoding = System.Text.Encoding.UTF8;
            
            // เพิ่มสินค้าตัวอย่าง
            SeedData();
            
            RunMenu();
        }
        
        static void SeedData()
        {
            inventory.AddProduct(new Product("P001", "Apple MacBook Pro 14", "Laptop", 65000m, 10));
            inventory.AddProduct(new Product("P002", "Dell XPS 15", "Laptop", 55000m, 8));
            inventory.AddProduct(new Product("P003", "Sony WH-1000XM5", "Headphones", 12000m, 25));
            inventory.AddProduct(new Product("P004", "iPad Air 5", "Tablet", 22000m, 3));  // Low stock
            inventory.AddProduct(new Product("P005", "Logitech MX Master 3", "Mouse", 3500m, 2));  // Low stock
            inventory.AddProduct(new Product("P006", "Samsung 27\" Monitor", "Monitor", 15000m, 0));  // Out of stock
            inventory.AddProduct(new Product("P007", "Mechanical Keyboard", "Keyboard", 4500m, 15));
            inventory.AddProduct(new Product("P008", "USB-C Hub 7-in-1", "Accessories", 1500m, 50));
        }
        
        static void RunMenu()
        {
            bool running = true;
            while (running)
            {
                Console.Clear();
                Console.ForegroundColor = ConsoleColor.Cyan;
                Console.WriteLine("╔══════════════════════════════════╗");
                Console.WriteLine("║    📦 ระบบจัดการคลังสินค้า       ║");
                Console.WriteLine("╚══════════════════════════════════╝");
                Console.ResetColor();
                
                // แสดง low stock warning
                var lowStock = inventory.GetLowStock().ToList();
                if (lowStock.Any())
                {
                    Console.ForegroundColor = ConsoleColor.Yellow;
                    Console.WriteLine($"⚠️  สินค้าใกล้หมด: {string.Join(", ", lowStock.Select(p => p.Name))}");
                    Console.ResetColor();
                }
                
                Console.WriteLine("\n1. ดูสินค้าทั้งหมด");
                Console.WriteLine("2. ค้นหาสินค้า");
                Console.WriteLine("3. ดูตามหมวดหมู่");
                Console.WriteLine("4. อัปเดตสต็อก");
                Console.WriteLine("5. เพิ่มสินค้าใหม่");
                Console.WriteLine("6. แสดงสถิติ");
                Console.WriteLine("7. Undo");
                Console.WriteLine("0. ออก");
                
                Console.Write("\nเลือก: ");
                
                switch (Console.ReadLine())
                {
                    case "1":
                        inventory.DisplayProducts();
                        break;
                    case "2":
                        SearchProducts();
                        break;
                    case "3":
                        ShowByCategory();
                        break;
                    case "4":
                        UpdateStock();
                        break;
                    case "5":
                        AddNewProduct();
                        break;
                    case "6":
                        ShowStats();
                        break;
                    case "7":
                        inventory.UndoLastAction();
                        break;
                    case "0":
                        running = false;
                        break;
                }
                
                if (running)
                {
                    Console.WriteLine("\nกด Enter เพื่อดำเนินการต่อ...");
                    Console.ReadLine();
                }
            }
        }
        
        static void SearchProducts()
        {
            Console.Write("ค้นหาชื่อสินค้า: ");
            string? name = Console.ReadLine();
            Console.Write("หมวดหมู่ (เว้นว่างถ้าไม่ระบุ): ");
            string? category = Console.ReadLine();
            Console.Write("เฉพาะสินค้ามีสต็อก? (Y/N): ");
            bool inStock = Console.ReadLine()?.ToUpper() == "Y";
            
            var results = inventory.Search(
                name: string.IsNullOrWhiteSpace(name) ? null : name,
                category: string.IsNullOrWhiteSpace(category) ? null : category,
                inStockOnly: inStock
            );
            
            inventory.DisplayProducts(results);
        }
        
        static void ShowByCategory()
        {
            var byCategory = inventory.GetByCategory();
            
            foreach (var (category, products) in byCategory.OrderBy(x => x.Key))
            {
                Console.ForegroundColor = ConsoleColor.Cyan;
                Console.WriteLine($"\n📁 {category} ({products.Count} รายการ)");
                Console.ResetColor();
                inventory.DisplayProducts(products);
            }
        }
        
        static void UpdateStock()
        {
            Console.Write("ID สินค้า: ");
            string? id = Console.ReadLine();
            if (string.IsNullOrWhiteSpace(id)) return;
            
            var product = inventory.GetProduct(id);
            if (product == null)
            {
                Console.WriteLine("ไม่พบสินค้า");
                return;
            }
            
            Console.WriteLine($"สินค้า: {product.Name} (สต็อกปัจจุบัน: {product.Quantity})");
            Console.Write("จำนวนที่เปลี่ยนแปลง (+/-): ");
            
            if (int.TryParse(Console.ReadLine(), out int change))
                inventory.UpdateStock(id, change);
        }
        
        static void AddNewProduct()
        {
            Console.Write("ID: ");
            string? id = Console.ReadLine();
            Console.Write("ชื่อสินค้า: ");
            string? name = Console.ReadLine();
            Console.Write("หมวดหมู่: ");
            string? category = Console.ReadLine();
            Console.Write("ราคา: ");
            decimal.TryParse(Console.ReadLine(), out decimal price);
            Console.Write("จำนวนสต็อก: ");
            int.TryParse(Console.ReadLine(), out int qty);
            
            if (id != null && name != null)
                inventory.AddProduct(new Product(id, name, category ?? "ทั่วไป", price, qty));
        }
        
        static void ShowStats()
        {
            var (totalValue, totalItems, categories) = inventory.GetStats();
            var lowStock = inventory.GetLowStock().ToList();
            
            Console.WriteLine("\n=== สถิติคลังสินค้า ===");
            Console.WriteLine($"มูลค่ารวม: {totalValue:N2} บาท");
            Console.WriteLine($"จำนวนสินค้ารวม: {totalItems:N0} ชิ้น");
            Console.WriteLine($"หมวดหมู่: {categories} หมวด");
            Console.WriteLine($"สินค้าใกล้หมด: {lowStock.Count} รายการ");
            
            var recent = inventory.GetRecentlyViewed().ToList();
            if (recent.Any())
                Console.WriteLine($"ดูล่าสุด: {string.Join(", ", recent.Take(5))}");
        }
    }
}
```

---

## 📝 สรุป Part 06

| Collection | ใช้เมื่อ | Access | Search |
|------------|---------|--------|--------|
| Array | ขนาดคงที่, performance | O(1) | O(n) |
| List<T> | ขนาดเปลี่ยนได้, index access | O(1) | O(n) |
| Dictionary | Key-value, lookup เร็ว | O(1) | O(1) |
| HashSet | Unique values, set ops | - | O(1) |
| Stack | LIFO, undo/redo | O(1) top | O(n) |
| Queue | FIFO, task queue | O(1) front | O(n) |
| LinkedList | Insert/Delete บ่อย | O(n) | O(n) |
| SortedDict | Sorted key-value | O(log n) | O(log n) |

---

## 🏋️ แบบฝึกหัด Part 06

### แบบฝึกหัดที่ 1: Student Grade Book
ใช้ Dictionary และ List สร้างระบบ grade book:
- เพิ่มนักเรียนและคะแนนหลายวิชา
- คำนวณ GPA
- จัดอันดับนักเรียน

### แบบฝึกหัดที่ 2: Browser History
ใช้ Stack สร้างระบบ browser history:
- Back, Forward, AddPage
- แสดง history ทั้งหมด

### แบบฝึกหัดที่ 3: Word Frequency Counter
ใช้ Dictionary นับความถี่คำในข้อความ:
- อ่านข้อความ
- นับแต่ละคำ
- แสดง top 10 คำที่พบบ่อยสุด

---

**ก่อนหน้า → [Part 05: Methods](part05-methods.md)**  
**ต่อไป → [Part 07: OOP - Classes](part07-oop-classes.md)**
