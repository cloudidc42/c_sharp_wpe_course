# Part 12: File I/O
## ขั้นตอนที่ 111-120: การอ่านเขียนไฟล์และ Streams

---

## 🎯 เป้าหมายของ Part นี้
- อ่านและเขียนไฟล์ด้วย File class
- ทำงานกับ Streams (FileStream, MemoryStream)
- อ่านเขียน CSV, JSON
- จัดการ Directories และ Paths
- Async File Operations
- สร้างระบบ File Manager จริง

---

## ขั้นตอนที่ 111: File class

### File และ Directory Operations
```csharp
using System.IO;

// ตรวจสอบ existence
bool fileExists = File.Exists(@"data.txt");
bool dirExists = Directory.Exists(@"C:\Users");

// สร้าง directory
Directory.CreateDirectory(@"output\logs\2026");

// Path operations
string fullPath = Path.GetFullPath("data.txt");
string dir = Path.GetDirectoryName(fullPath) ?? "";
string fileName = Path.GetFileName(fullPath);
string fileNameNoExt = Path.GetFileNameWithoutExtension(fullPath);
string extension = Path.GetExtension(fullPath);

Console.WriteLine($"Full: {fullPath}");
Console.WriteLine($"Dir: {dir}");
Console.WriteLine($"File: {fileName}");
Console.WriteLine($"Name: {fileNameNoExt}");
Console.WriteLine($"Ext: {extension}");

// Combine paths (safe, cross-platform)
string logPath = Path.Combine("logs", "2026", "app.log");
Console.WriteLine(logPath);  // logs/2026/app.log or logs\2026\app.log

// Temp files
string tempFile = Path.GetTempFileName();
string tempDir = Path.GetTempPath();
```

### Read / Write Text
```csharp
// Write text (creates or overwrites)
File.WriteAllText("output.txt", "Hello, World!");

// Write multiple lines
string[] lines = { "Line 1", "Line 2", "Line 3" };
File.WriteAllLines("output.txt", lines);

// Append
File.AppendAllText("output.txt", "\nNew content");
File.AppendAllLines("output.txt", new[] { "Line 4", "Line 5" });

// Read all at once
string content = File.ReadAllText("output.txt");
string[] readLines = File.ReadAllLines("output.txt");

// Read line by line (memory efficient for large files)
foreach (string line in File.ReadLines("output.txt"))
{
    Console.WriteLine(line);
    // process each line without loading entire file
}

// Encoding
File.WriteAllText("thai.txt", "สวัสดีชาวโลก", System.Text.Encoding.UTF8);
string thaiContent = File.ReadAllText("thai.txt", System.Text.Encoding.UTF8);
```

### File Operations
```csharp
// Copy
File.Copy("source.txt", "dest.txt");
File.Copy("source.txt", "dest.txt", overwrite: true);

// Move / Rename
File.Move("old.txt", "new.txt");

// Delete
if (File.Exists("temp.txt"))
    File.Delete("temp.txt");

// Get file info
var info = new FileInfo("data.txt");
Console.WriteLine($"Name: {info.Name}");
Console.WriteLine($"Size: {info.Length} bytes");
Console.WriteLine($"Created: {info.CreationTime}");
Console.WriteLine($"Modified: {info.LastWriteTime}");
Console.WriteLine($"ReadOnly: {info.IsReadOnly}");
```

---

## ขั้นตอนที่ 112: StreamReader และ StreamWriter

### StreamReader สำหรับ Large Files
```csharp
// StreamReader - อ่านทีละบรรทัด (memory efficient)
using (var reader = new StreamReader("largefile.txt", System.Text.Encoding.UTF8))
{
    string? line;
    int lineNumber = 0;
    
    while ((line = reader.ReadLine()) != null)
    {
        lineNumber++;
        // Process line...
        if (lineNumber % 10000 == 0)
            Console.WriteLine($"Processed {lineNumber} lines...");
    }
    
    Console.WriteLine($"Total: {lineNumber} lines");
}

// StreamWriter - เขียนทีละบรรทัด
using (var writer = new StreamWriter("output.txt", append: false, System.Text.Encoding.UTF8))
{
    writer.WriteLine("Header Line");
    
    for (int i = 1; i <= 100; i++)
    {
        writer.WriteLine($"Line {i}: {DateTime.Now:HH:mm:ss.fff}");
    }
    
    writer.Flush();  // Flush buffer ก่อนปิด
}

// StreamWriter ด้วย auto-flush
var writer2 = new StreamWriter("log.txt", append: true)
{
    AutoFlush = true  // flush ทุก write
};
writer2.WriteLine($"[{DateTime.Now:s}] Application started");
writer2.Dispose();
```

---

## ขั้นตอนที่ 113: Binary Files

### FileStream และ BinaryReader/Writer
```csharp
// เขียน binary
using (var stream = new FileStream("data.bin", FileMode.Create))
using (var writer = new BinaryWriter(stream))
{
    writer.Write(42);          // int (4 bytes)
    writer.Write(3.14f);       // float (4 bytes)
    writer.Write(true);        // bool (1 byte)
    writer.Write("Hello");     // string (length-prefixed)
    writer.Write(new byte[] { 1, 2, 3, 4, 5 });  // bytes
}

// อ่าน binary
using (var stream = new FileStream("data.bin", FileMode.Open))
using (var reader = new BinaryReader(stream))
{
    int number = reader.ReadInt32();
    float value = reader.ReadSingle();
    bool flag = reader.ReadBoolean();
    string text = reader.ReadString();
    byte[] bytes = reader.ReadBytes(5);
    
    Console.WriteLine($"Int: {number}, Float: {value}, Bool: {flag}");
    Console.WriteLine($"String: {text}");
    Console.WriteLine($"Bytes: {string.Join(", ", bytes)}");
}

// MemoryStream - เหมือน FileStream แต่อยู่ใน memory
using var memStream = new MemoryStream();
using var writer3 = new BinaryWriter(memStream);

writer3.Write("Data in memory");
writer3.Write(12345);

// Reset position เพื่ออ่าน
memStream.Position = 0;

using var reader3 = new BinaryReader(memStream);
Console.WriteLine(reader3.ReadString());
Console.WriteLine(reader3.ReadInt32());
```

---

## ขั้นตอนที่ 114: CSV Processing

### อ่านเขียน CSV
```csharp
using System.Text;

class CsvProcessor
{
    // Write CSV
    public static void WriteCsv<T>(string path, IEnumerable<T> data, 
        Func<T, string[]> rowSelector, string[]? headers = null)
    {
        using var writer = new StreamWriter(path, false, Encoding.UTF8);
        
        // BOM for Excel compatibility
        // writer.Write('\uFEFF');
        
        if (headers != null)
            writer.WriteLine(string.Join(",", headers.Select(EscapeCsv)));
        
        foreach (var item in data)
        {
            var row = rowSelector(item);
            writer.WriteLine(string.Join(",", row.Select(EscapeCsv)));
        }
    }
    
    // Read CSV
    public static IEnumerable<Dictionary<string, string>> ReadCsv(string path)
    {
        string[]? headers = null;
        
        foreach (string line in File.ReadLines(path, Encoding.UTF8))
        {
            string[] fields = ParseCsvLine(line);
            
            if (headers == null)
            {
                headers = fields;
                continue;
            }
            
            var row = new Dictionary<string, string>();
            for (int i = 0; i < Math.Min(headers.Length, fields.Length); i++)
                row[headers[i]] = fields[i];
            
            yield return row;
        }
    }
    
    private static string EscapeCsv(string field)
    {
        if (field.Contains(',') || field.Contains('"') || field.Contains('\n'))
            return $"\"{field.Replace("\"", "\"\"")}\"";
        return field;
    }
    
    private static string[] ParseCsvLine(string line)
    {
        var fields = new List<string>();
        var current = new StringBuilder();
        bool inQuotes = false;
        
        for (int i = 0; i < line.Length; i++)
        {
            char c = line[i];
            
            if (c == '"')
            {
                if (inQuotes && i + 1 < line.Length && line[i + 1] == '"')
                {
                    current.Append('"');
                    i++;
                }
                else
                {
                    inQuotes = !inQuotes;
                }
            }
            else if (c == ',' && !inQuotes)
            {
                fields.Add(current.ToString());
                current.Clear();
            }
            else
            {
                current.Append(c);
            }
        }
        
        fields.Add(current.ToString());
        return fields.ToArray();
    }
}

// การใช้งาน
record Product(int Id, string Name, decimal Price, int Stock);

var products = new[]
{
    new Product(1, "Laptop", 35000m, 10),
    new Product(2, "Phone, Smart", 15000m, 25),
    new Product(3, "Tablet \"Pro\"", 20000m, 15)
};

// เขียน CSV
CsvProcessor.WriteCsv(
    "products.csv",
    products,
    p => new[] { p.Id.ToString(), p.Name, p.Price.ToString("F2"), p.Stock.ToString() },
    new[] { "ID", "Name", "Price", "Stock" }
);

// อ่าน CSV
Console.WriteLine("Products from CSV:");
foreach (var row in CsvProcessor.ReadCsv("products.csv"))
{
    Console.WriteLine($"  ID:{row["ID"]} | {row["Name"]} | {row["Price"]}");
}
```

---

## ขั้นตอนที่ 115: JSON File Operations

### อ่านเขียน JSON
```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

// Data models
class AppConfig
{
    public string DatabaseUrl { get; set; } = "";
    public int Port { get; set; } = 8080;
    public bool EnableCache { get; set; } = true;
    public List<string> AllowedOrigins { get; set; } = new();
    
    [JsonPropertyName("api_key")]
    public string ApiKey { get; set; } = "";
    
    [JsonIgnore]
    public string SensitiveData { get; set; } = "";
}

// Write JSON
var config = new AppConfig
{
    DatabaseUrl = "mongodb://localhost:27017",
    Port = 3000,
    EnableCache = false,
    AllowedOrigins = new List<string> { "http://localhost", "https://example.com" },
    ApiKey = "sk-1234567890"
};

var options = new JsonSerializerOptions
{
    WriteIndented = true,  // pretty print
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
};

string json = JsonSerializer.Serialize(config, options);
await File.WriteAllTextAsync("config.json", json);
Console.WriteLine("Config saved:");
Console.WriteLine(json);

// Read JSON
string loadedJson = await File.ReadAllTextAsync("config.json");
var loadedConfig = JsonSerializer.Deserialize<AppConfig>(loadedJson, options);
Console.WriteLine($"\nLoaded: Port={loadedConfig?.Port}, Cache={loadedConfig?.EnableCache}");

// Work with JSON directly
using var doc = JsonDocument.Parse(json);
var root = doc.RootElement;
Console.WriteLine($"Port: {root.GetProperty("port").GetInt32()}");
Console.WriteLine($"Origins count: {root.GetProperty("allowedOrigins").GetArrayLength()}");
```

---

## ขั้นตอนที่ 116-120: โปรแกรมตัวอย่าง - File Manager System

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Text;
using System.Text.Json;
using System.Threading.Tasks;

namespace FileManagerSystem
{
    record FileItem(string Path, string Name, long Size, DateTime Modified, bool IsDirectory);
    
    record SearchResult(string Path, int LineNumber, string LineContent, int MatchCount);
    
    class FileManager
    {
        private readonly string _workingDirectory;
        private Stack<string> _history = new();
        
        public string CurrentDirectory { get; private set; }
        
        public FileManager(string startDirectory)
        {
            _workingDirectory = startDirectory;
            CurrentDirectory = startDirectory;
            
            if (!Directory.Exists(startDirectory))
                Directory.CreateDirectory(startDirectory);
        }
        
        // List files and directories
        public IEnumerable<FileItem> List(string? pattern = null)
        {
            var items = new List<FileItem>();
            
            try
            {
                // Directories first
                foreach (string dir in Directory.GetDirectories(CurrentDirectory))
                {
                    var info = new DirectoryInfo(dir);
                    items.Add(new FileItem(dir, info.Name, 0, info.LastWriteTime, true));
                }
                
                // Then files
                string searchPattern = pattern ?? "*";
                foreach (string file in Directory.GetFiles(CurrentDirectory, searchPattern))
                {
                    var info = new FileInfo(file);
                    items.Add(new FileItem(file, info.Name, info.Length, info.LastWriteTime, false));
                }
            }
            catch (UnauthorizedAccessException)
            {
                Console.WriteLine("⚠️ ไม่มีสิทธิ์เข้าถึงบางไฟล์/โฟลเดอร์");
            }
            
            return items;
        }
        
        // Navigate
        public bool NavigateTo(string path)
        {
            string fullPath = Path.IsPathRooted(path) 
                ? path 
                : Path.Combine(CurrentDirectory, path);
            
            fullPath = Path.GetFullPath(fullPath);
            
            if (!Directory.Exists(fullPath))
            {
                Console.WriteLine($"❌ ไม่พบ directory: {path}");
                return false;
            }
            
            _history.Push(CurrentDirectory);
            CurrentDirectory = fullPath;
            return true;
        }
        
        public bool NavigateBack()
        {
            if (!_history.Any())
            {
                Console.WriteLine("ไม่มี history");
                return false;
            }
            
            CurrentDirectory = _history.Pop();
            return true;
        }
        
        // Search in files
        public async Task<List<SearchResult>> SearchAsync(string searchText, string pattern = "*.txt",
            bool caseSensitive = false, int maxResults = 100)
        {
            var results = new List<SearchResult>();
            var comparison = caseSensitive 
                ? StringComparison.Ordinal 
                : StringComparison.OrdinalIgnoreCase;
            
            var files = Directory.GetFiles(CurrentDirectory, pattern, SearchOption.AllDirectories);
            
            foreach (string file in files)
            {
                if (results.Count >= maxResults) break;
                
                try
                {
                    int lineNum = 0;
                    int fileMatches = 0;
                    
                    await foreach (string line in ReadLinesAsync(file))
                    {
                        lineNum++;
                        if (line.Contains(searchText, comparison))
                        {
                            fileMatches++;
                            if (fileMatches <= 5)  // max 5 results per file
                            {
                                results.Add(new SearchResult(file, lineNum, line.Trim(), fileMatches));
                            }
                        }
                    }
                }
                catch (Exception ex)
                {
                    Console.WriteLine($"⚠️ ข้ามไฟล์ {Path.GetFileName(file)}: {ex.Message}");
                }
            }
            
            return results;
        }
        
        private async IAsyncEnumerable<string> ReadLinesAsync(string path)
        {
            using var reader = new StreamReader(path);
            string? line;
            while ((line = await reader.ReadLineAsync()) != null)
                yield return line;
        }
        
        // Copy with progress
        public async Task CopyFileAsync(string sourceName, string destName,
            IProgress<double>? progress = null)
        {
            string sourcePath = Path.Combine(CurrentDirectory, sourceName);
            string destPath = Path.Combine(CurrentDirectory, destName);
            
            if (!File.Exists(sourcePath))
                throw new FileNotFoundException($"ไม่พบไฟล์: {sourceName}");
            
            var sourceInfo = new FileInfo(sourcePath);
            long totalBytes = sourceInfo.Length;
            long copiedBytes = 0;
            const int bufferSize = 81920;  // 80KB buffer
            
            using var source = new FileStream(sourcePath, FileMode.Open, FileAccess.Read);
            using var dest = new FileStream(destPath, FileMode.Create, FileAccess.Write);
            
            byte[] buffer = new byte[bufferSize];
            int bytesRead;
            
            while ((bytesRead = await source.ReadAsync(buffer)) > 0)
            {
                await dest.WriteAsync(buffer.AsMemory(0, bytesRead));
                copiedBytes += bytesRead;
                
                if (totalBytes > 0)
                    progress?.Report((double)copiedBytes / totalBytes * 100);
            }
        }
        
        // Save index as JSON
        public async Task SaveIndexAsync(string indexFile)
        {
            var index = new
            {
                Path = CurrentDirectory,
                GeneratedAt = DateTime.Now,
                Files = List()
                    .Select(f => new
                    {
                        f.Name,
                        f.Size,
                        f.Modified,
                        f.IsDirectory
                    })
                    .ToList()
            };
            
            string json = JsonSerializer.Serialize(index, new JsonSerializerOptions { WriteIndented = true });
            await File.WriteAllTextAsync(indexFile, json);
        }
        
        // Get directory stats
        public (long totalSize, int fileCount, int dirCount) GetStats()
        {
            long totalSize = 0;
            int fileCount = 0;
            int dirCount = 0;
            
            try
            {
                foreach (string file in Directory.GetFiles(CurrentDirectory, "*", SearchOption.AllDirectories))
                {
                    totalSize += new FileInfo(file).Length;
                    fileCount++;
                }
                
                dirCount = Directory.GetDirectories(CurrentDirectory, "*", SearchOption.AllDirectories).Length;
            }
            catch (UnauthorizedAccessException) { }
            
            return (totalSize, fileCount, dirCount);
        }
        
        // Format file size
        public static string FormatSize(long bytes)
        {
            string[] sizes = { "B", "KB", "MB", "GB", "TB" };
            double size = bytes;
            int order = 0;
            
            while (size >= 1024 && order < sizes.Length - 1)
            {
                order++;
                size /= 1024;
            }
            
            return $"{size:F2} {sizes[order]}";
        }
        
        // Display listing
        public void DisplayListing(string? pattern = null)
        {
            Console.WriteLine($"\n📁 {CurrentDirectory}");
            Console.WriteLine(new string('─', 70));
            Console.WriteLine($"{"Name",40} {"Size",10} {"Modified",20}");
            Console.WriteLine(new string('─', 70));
            
            int count = 0;
            foreach (var item in List(pattern))
            {
                string icon = item.IsDirectory ? "📁" : "📄";
                string size = item.IsDirectory ? "<DIR>" : FormatSize(item.Size);
                string name = item.Name.Length > 37 ? item.Name[..37] + "..." : item.Name;
                
                Console.WriteLine($"{icon} {name,-38} {size,10} {item.Modified:dd/MM/yyyy HH:mm}");
                count++;
            }
            
            Console.WriteLine(new string('─', 70));
            Console.WriteLine($"รวม {count} รายการ");
        }
    }
    
    class Program
    {
        static async Task Main()
        {
            Console.OutputEncoding = Encoding.UTF8;
            Console.Title = "File Manager System";
            
            // สร้าง test directory structure
            string testDir = Path.Combine(Path.GetTempPath(), "FileManagerDemo");
            
            var fm = new FileManager(testDir);
            
            // สร้างไฟล์ตัวอย่าง
            Console.WriteLine("กำลังสร้างไฟล์ตัวอย่าง...");
            
            await File.WriteAllTextAsync(Path.Combine(testDir, "readme.txt"), 
                "Welcome to File Manager Demo\nThis is a test file\nWith multiple lines");
            
            await File.WriteAllTextAsync(Path.Combine(testDir, "notes.txt"),
                "Meeting notes\nDiscuss project timeline\nAssign tasks to team");
            
            Directory.CreateDirectory(Path.Combine(testDir, "documents"));
            await File.WriteAllTextAsync(Path.Combine(testDir, "documents", "report.txt"),
                "Monthly Report\nTotal sales: 1,500,000\nNew customers: 150");
            
            // แสดง listing
            fm.DisplayListing();
            
            // Get stats
            var (totalSize, fileCount, dirCount) = fm.GetStats();
            Console.WriteLine($"\nStats: {fileCount} files, {dirCount} dirs, {FileManager.FormatSize(totalSize)}");
            
            // Search
            Console.WriteLine("\n🔍 กำลังค้นหา 'test'...");
            var results = await fm.SearchAsync("test", "*.txt");
            foreach (var result in results)
            {
                Console.WriteLine($"  {Path.GetFileName(result.Path)}:{result.LineNumber} → {result.LineContent}");
            }
            
            // Copy file with progress
            Console.WriteLine("\n📋 กำลัง copy ไฟล์...");
            var progress = new Progress<double>(p => 
                Console.Write($"\rProgress: {p:F1}%"));
            
            try
            {
                await fm.CopyFileAsync("readme.txt", "readme_copy.txt", progress);
                Console.WriteLine("\n✅ Copy สำเร็จ");
            }
            catch (Exception ex)
            {
                Console.WriteLine($"\n❌ Error: {ex.Message}");
            }
            
            // Save index
            string indexPath = Path.Combine(testDir, "index.json");
            await fm.SaveIndexAsync(indexPath);
            Console.WriteLine($"\n💾 บันทึก index ที่: {indexPath}");
            
            // Navigate into subdirectory
            fm.NavigateTo("documents");
            fm.DisplayListing();
            
            fm.NavigateBack();
            Console.WriteLine($"\n⬅️ กลับมาที่: {fm.CurrentDirectory}");
            
            Console.WriteLine("\nกด Enter เพื่อออก...");
            Console.ReadLine();
            
            // Cleanup
            Directory.Delete(testDir, recursive: true);
        }
    }
}
```

---

## 📝 สรุป Part 12

| หัวข้อ | Key Points |
|--------|-----------|
| File class | ReadAllText, WriteAllText, Copy, Move, Delete |
| StreamReader/Writer | สำหรับไฟล์ขนาดใหญ่ |
| BinaryReader/Writer | อ่านเขียนข้อมูล binary |
| MemoryStream | Stream ใน memory |
| CSV Processing | Parse/write CSV manually |
| JSON | System.Text.Json สำหรับ JSON |
| Async I/O | ReadAllTextAsync, WriteAllTextAsync |
| Path | Combine, GetFullPath, GetExtension |

---

**ก่อนหน้า → [Part 11: String Manipulation](part11-strings.md)**  
**ต่อไป → [Part 13: LINQ Basics](part13-linq.md)**
