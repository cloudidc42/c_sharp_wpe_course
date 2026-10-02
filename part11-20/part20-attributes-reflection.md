# Part 20: Attributes & Reflection
## ขั้นตอนที่ 191-200: Metadata Programming

---

## 🎯 เป้าหมายของ Part นี้
- สร้างและใช้ Custom Attributes
- Reflection: ตรวจสอบ types และ members
- Dynamic object creation
- AOP (Aspect-Oriented Programming) ด้วย Attributes
- Source Generators เบื้องต้น
- สร้าง ORM เล็กๆ ด้วย Reflection

---

## ขั้นตอนที่ 191: Built-in Attributes

### Common C# Attributes
```csharp
// [Obsolete] - mark deprecated code
[Obsolete("Use NewMethod() instead", error: false)]
public void OldMethod() { }

[Obsolete("REMOVED", error: true)]  // Compile error if used
public void RemovedMethod() { }

// [Conditional] - compile conditionally
using System.Diagnostics;

[Conditional("DEBUG")]
static void DebugLog(string message)
{
    Console.WriteLine($"[DEBUG] {message}");
}

// Only executes in Debug build
DebugLog("This only runs in debug");

// [Flags] - enum as bit flags
[Flags]
enum Permission
{
    None    = 0b0000,
    Read    = 0b0001,
    Write   = 0b0010,
    Delete  = 0b0100,
    Admin   = 0b1000,
    
    ReadWrite = Read | Write,
    All = Read | Write | Delete | Admin
}

var userPerms = Permission.Read | Permission.Write;
Console.WriteLine(userPerms);                              // Read, Write
Console.WriteLine(userPerms.HasFlag(Permission.Read));     // True
Console.WriteLine(userPerms.HasFlag(Permission.Delete));   // False
Console.WriteLine((int)userPerms);                         // 3

// [StructLayout] - control memory layout
using System.Runtime.InteropServices;

[StructLayout(LayoutKind.Sequential, Pack = 1)]
struct NetworkPacket
{
    public byte Version;
    public byte Type;
    public short Length;
    public int Checksum;
}

// [DllImport] - P/Invoke
[DllImport("kernel32.dll")]
static extern bool Beep(int frequency, int duration);

// [JsonPropertyName], [JsonIgnore] - JSON serialization
using System.Text.Json.Serialization;

class ApiResponse
{
    [JsonPropertyName("user_id")]
    public int UserId { get; set; }
    
    [JsonIgnore]
    public string Password { get; set; } = "";
    
    [JsonPropertyName("full_name")]
    public string FullName { get; set; } = "";
}
```

---

## ขั้นตอนที่ 192: Custom Attributes

### สร้าง Custom Attributes
```csharp
// Custom Attribute
[AttributeUsage(
    AttributeTargets.Class | AttributeTargets.Method | AttributeTargets.Property,
    AllowMultiple = false,
    Inherited = true
)]
class ValidateAttribute : Attribute
{
    public bool Required { get; set; } = true;
    public string? ErrorMessage { get; set; }
    
    public ValidateAttribute() { }
    public ValidateAttribute(string errorMessage) { ErrorMessage = errorMessage; }
}

[AttributeUsage(AttributeTargets.Property)]
class RangeAttribute : Attribute
{
    public double Min { get; }
    public double Max { get; }
    
    public RangeAttribute(double min, double max)
    {
        Min = min;
        Max = max;
    }
}

[AttributeUsage(AttributeTargets.Property)]
class RegexAttribute : Attribute
{
    public string Pattern { get; }
    public string ErrorMessage { get; set; } = "Invalid format";
    
    public RegexAttribute(string pattern) => Pattern = pattern;
}

// Using custom attributes
[Validate]
class UserRegistration
{
    [Validate(Required = true)]
    [RegexAttribute(@"^[a-zA-Z0-9_]{3,20}$", ErrorMessage = "Invalid username format")]
    public string Username { get; set; } = "";
    
    [Validate]
    [RegexAttribute(@"^[^@]+@[^@]+\.[^@]+$", ErrorMessage = "Invalid email")]
    public string Email { get; set; } = "";
    
    [Validate]
    [RangeAttribute(18, 120)]
    public int Age { get; set; }
    
    public string? OptionalNote { get; set; }
}
```

---

## ขั้นตอนที่ 193: Reflection Basics

### ตรวจสอบ Types ด้วย Reflection
```csharp
using System.Reflection;

// Get Type
Type type = typeof(string);
Type type2 = "Hello".GetType();
Type? type3 = Type.GetType("System.String");

Console.WriteLine($"Name: {type.Name}");
Console.WriteLine($"FullName: {type.FullName}");
Console.WriteLine($"IsClass: {type.IsClass}");
Console.WriteLine($"IsSealed: {type.IsSealed}");

// Properties
PropertyInfo[] props = type.GetProperties();
Console.WriteLine($"\nProperties of string ({props.Length}):");
foreach (var prop in props.Take(5))
    Console.WriteLine($"  {prop.PropertyType.Name} {prop.Name} [get:{prop.CanRead}, set:{prop.CanWrite}]");

// Methods
MethodInfo[] methods = type.GetMethods(BindingFlags.Public | BindingFlags.Instance)
    .Where(m => !m.IsSpecialName)  // exclude get_/set_
    .Take(10)
    .ToArray();

Console.WriteLine($"\nMethods (sample):");
foreach (var method in methods)
{
    var parms = method.GetParameters().Select(p => $"{p.ParameterType.Name} {p.Name}");
    Console.WriteLine($"  {method.ReturnType.Name} {method.Name}({string.Join(", ", parms)})");
}

// Constructor
ConstructorInfo[] ctors = type.GetConstructors();
Console.WriteLine($"\nConstructors: {ctors.Length}");
```

### Dynamic Object Creation
```csharp
// สร้าง object แบบ dynamic
Type listType = typeof(List<>).MakeGenericType(typeof(int));
object list = Activator.CreateInstance(listType)!;

// Invoke method
MethodInfo addMethod = listType.GetMethod("Add")!;
addMethod.Invoke(list, new object[] { 1 });
addMethod.Invoke(list, new object[] { 2 });
addMethod.Invoke(list, new object[] { 3 });

PropertyInfo countProp = listType.GetProperty("Count")!;
int count = (int)countProp.GetValue(list)!;
Console.WriteLine($"Count: {count}");  // 3

// Get/Set property values
class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public decimal Price { get; set; }
}

var product = new Product { Id = 1, Name = "Laptop", Price = 35000m };
Type productType = product.GetType();

// Read property values
foreach (var prop in productType.GetProperties())
{
    object? value = prop.GetValue(product);
    Console.WriteLine($"{prop.Name}: {value}");
}

// Set property value dynamically
PropertyInfo nameProp = productType.GetProperty("Name")!;
nameProp.SetValue(product, "MacBook Pro");
Console.WriteLine($"Updated name: {product.Name}");
```

---

## ขั้นตอนที่ 194-200: โปรแกรมตัวอย่าง - Mini ORM

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Reflection;
using System.Text;
using System.Text.RegularExpressions;

namespace MiniOrm
{
    // ORM Attributes
    [AttributeUsage(AttributeTargets.Class)]
    class TableAttribute : Attribute
    {
        public string Name { get; }
        public TableAttribute(string name) => Name = name;
    }
    
    [AttributeUsage(AttributeTargets.Property)]
    class ColumnAttribute : Attribute
    {
        public string? Name { get; set; }
        public bool IsPrimaryKey { get; set; }
        public bool IsNullable { get; set; } = true;
        public int MaxLength { get; set; } = -1;
    }
    
    [AttributeUsage(AttributeTargets.Property)]
    class NotMappedAttribute : Attribute { }
    
    // Entity base class
    abstract class Entity
    {
        [Column(IsPrimaryKey = true)]
        public int Id { get; set; }
    }
    
    // Domain Entities
    [Table("users")]
    class User : Entity
    {
        [Column(Name = "username", MaxLength = 50)]
        public string Username { get; set; } = "";
        
        [Column(Name = "email", MaxLength = 200)]
        public string Email { get; set; } = "";
        
        [Column(Name = "created_at")]
        public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
        
        [Column(Name = "is_active")]
        public bool IsActive { get; set; } = true;
        
        [NotMapped]
        public string DisplayName => $"{Username} <{Email}>";
    }
    
    [Table("products")]
    class Product : Entity
    {
        [Column(Name = "product_name", MaxLength = 200)]
        public string Name { get; set; } = "";
        
        [Column(Name = "unit_price")]
        public decimal Price { get; set; }
        
        [Column(Name = "stock_qty")]
        public int Stock { get; set; }
        
        [Column(Name = "category", MaxLength = 100)]
        public string Category { get; set; } = "";
    }
    
    // Schema Generator
    class SchemaGenerator
    {
        public string GenerateCreateTable<T>() where T : Entity
        {
            var type = typeof(T);
            var tableAttr = type.GetCustomAttribute<TableAttribute>()
                ?? throw new InvalidOperationException($"Type {type.Name} has no [Table] attribute");
            
            var sb = new StringBuilder();
            sb.AppendLine($"CREATE TABLE IF NOT EXISTS {tableAttr.Name} (");
            
            var columns = new List<string>();
            
            foreach (var prop in type.GetProperties())
            {
                if (prop.GetCustomAttribute<NotMappedAttribute>() != null) continue;
                
                var colAttr = prop.GetCustomAttribute<ColumnAttribute>();
                string colName = colAttr?.Name ?? ToSnakeCase(prop.Name);
                string colType = GetSqlType(prop.PropertyType, colAttr);
                
                bool isPrimaryKey = colAttr?.IsPrimaryKey == true || prop.Name == "Id";
                bool isNullable = colAttr?.IsNullable ?? true;
                
                string colDef = $"    {colName} {colType}";
                
                if (isPrimaryKey)
                    colDef += " PRIMARY KEY AUTOINCREMENT";
                else if (!isNullable)
                    colDef += " NOT NULL";
                
                columns.Add(colDef);
            }
            
            sb.AppendLine(string.Join(",\n", columns));
            sb.AppendLine(");");
            
            return sb.ToString();
        }
        
        private string GetSqlType(Type type, ColumnAttribute? attr)
        {
            // Handle nullable types
            if (type.IsGenericType && type.GetGenericTypeDefinition() == typeof(Nullable<>))
                type = Nullable.GetUnderlyingType(type)!;
            
            return type.Name switch
            {
                "Int32" or "Int16" => "INTEGER",
                "Int64" => "BIGINT",
                "String" => attr?.MaxLength > 0 ? $"VARCHAR({attr.MaxLength})" : "TEXT",
                "Decimal" or "Double" or "Single" => "DECIMAL(18,2)",
                "Boolean" => "BOOLEAN",
                "DateTime" => "DATETIME",
                "Guid" => "VARCHAR(36)",
                _ => "TEXT"
            };
        }
        
        private string ToSnakeCase(string name)
            => Regex.Replace(name, @"([A-Z])", "_$1").ToLower().TrimStart('_');
    }
    
    // SQL Builder using Reflection
    class SqlBuilder<T> where T : Entity
    {
        private readonly Type _type;
        private readonly string _tableName;
        private readonly List<PropertyInfo> _mappedProps;
        
        public SqlBuilder()
        {
            _type = typeof(T);
            var tableAttr = _type.GetCustomAttribute<TableAttribute>();
            _tableName = tableAttr?.Name ?? _type.Name.ToLower() + "s";
            
            _mappedProps = _type.GetProperties()
                .Where(p => p.GetCustomAttribute<NotMappedAttribute>() == null)
                .ToList();
        }
        
        private string GetColumnName(PropertyInfo prop)
        {
            var attr = prop.GetCustomAttribute<ColumnAttribute>();
            return attr?.Name ?? ToSnakeCase(prop.Name);
        }
        
        private string ToSnakeCase(string name)
            => Regex.Replace(name, @"([A-Z])", "_$1").ToLower().TrimStart('_');
        
        public string GenerateInsert(T entity)
        {
            var nonPkProps = _mappedProps
                .Where(p => p.GetCustomAttribute<ColumnAttribute>()?.IsPrimaryKey != true && p.Name != "Id")
                .ToList();
            
            var columns = nonPkProps.Select(GetColumnName);
            var values = nonPkProps.Select(p => FormatValue(p.GetValue(entity)));
            
            return $"INSERT INTO {_tableName} ({string.Join(", ", columns)}) " +
                   $"VALUES ({string.Join(", ", values)});";
        }
        
        public string GenerateUpdate(T entity)
        {
            var nonPkProps = _mappedProps
                .Where(p => p.GetCustomAttribute<ColumnAttribute>()?.IsPrimaryKey != true && p.Name != "Id")
                .ToList();
            
            var sets = nonPkProps.Select(p => $"{GetColumnName(p)} = {FormatValue(p.GetValue(p))}");
            
            return $"UPDATE {_tableName} SET {string.Join(", ", sets)} WHERE id = {entity.Id};";
        }
        
        public string GenerateSelect(string? whereClause = null, int? limit = null)
        {
            var columns = string.Join(", ", _mappedProps.Select(GetColumnName));
            var sql = $"SELECT {columns} FROM {_tableName}";
            
            if (whereClause != null) sql += $" WHERE {whereClause}";
            if (limit.HasValue) sql += $" LIMIT {limit}";
            
            return sql + ";";
        }
        
        public string GenerateDelete(int id)
            => $"DELETE FROM {_tableName} WHERE id = {id};";
        
        private string FormatValue(object? value) => value switch
        {
            null => "NULL",
            string s => $"'{s.Replace("'", "''")}'",
            bool b => b ? "1" : "0",
            DateTime dt => $"'{dt:yyyy-MM-dd HH:mm:ss}'",
            _ => value.ToString() ?? "NULL"
        };
        
        // Create T from property-value dictionary (mock of DB read)
        public T? Hydrate(Dictionary<string, object?> data)
        {
            var entity = (T?)Activator.CreateInstance(typeof(T));
            if (entity == null) return null;
            
            foreach (var prop in _mappedProps)
            {
                string colName = GetColumnName(prop);
                if (!data.TryGetValue(colName, out var value)) continue;
                
                try
                {
                    var converted = Convert.ChangeType(value, prop.PropertyType);
                    prop.SetValue(entity, converted);
                }
                catch { /* Skip conversion errors */ }
            }
            
            return entity;
        }
    }
    
    // Attribute-based Validator using Reflection
    class ReflectionValidator
    {
        public List<string> Validate<T>(T obj) where T : class
        {
            var errors = new List<string>();
            var type = obj.GetType();
            
            foreach (var prop in type.GetProperties())
            {
                object? value = prop.GetValue(obj);
                
                // Check MaxLength
                var colAttr = prop.GetCustomAttribute<ColumnAttribute>();
                if (colAttr?.MaxLength > 0 && value is string str)
                {
                    if (str.Length > colAttr.MaxLength)
                        errors.Add($"{prop.Name}: exceeds max length ({str.Length}/{colAttr.MaxLength})");
                }
                
                // Check nullable
                if (colAttr?.IsNullable == false && value is null)
                    errors.Add($"{prop.Name}: cannot be null");
                
                if (value is string s2 && !colAttr?.IsNullable != false && string.IsNullOrEmpty(s2))
                    errors.Add($"{prop.Name}: cannot be empty");
            }
            
            return errors;
        }
    }
    
    class Program
    {
        static void Main()
        {
            Console.OutputEncoding = Encoding.UTF8;
            Console.Title = "Mini ORM Demo";
            
            var schemaGen = new SchemaGenerator();
            var userBuilder = new SqlBuilder<User>();
            var productBuilder = new SqlBuilder<Product>();
            var validator = new ReflectionValidator();
            
            // Generate schemas
            Console.WriteLine("=== Schema Generation ===");
            Console.WriteLine(schemaGen.GenerateCreateTable<User>());
            Console.WriteLine(schemaGen.GenerateCreateTable<Product>());
            
            // Generate SQL statements
            Console.WriteLine("=== SQL Generation ===");
            
            var user = new User { Username = "alice", Email = "alice@example.com" };
            Console.WriteLine(userBuilder.GenerateInsert(user));
            Console.WriteLine(userBuilder.GenerateSelect("is_active = 1", 10));
            Console.WriteLine(userBuilder.GenerateDelete(1));
            
            var product = new Product { Name = "Laptop", Price = 45000m, Stock = 10, Category = "Electronics" };
            Console.WriteLine(productBuilder.GenerateInsert(product));
            
            // Validation using reflection
            Console.WriteLine("\n=== Reflection Validation ===");
            var invalidUser = new User { Username = "", Email = "a".PadRight(201, 'x') + "@x.com" };
            var errors = validator.Validate(invalidUser);
            
            if (errors.Any())
            {
                Console.WriteLine("Validation errors:");
                errors.ForEach(e => Console.WriteLine($"  - {e}"));
            }
            
            // Hydrate objects from "DB" data (simulated)
            Console.WriteLine("\n=== Object Hydration ===");
            var userData = new Dictionary<string, object?>
            {
                { "id", 1 },
                { "username", "bob" },
                { "email", "bob@example.com" },
                { "created_at", DateTime.Now },
                { "is_active", true }
            };
            
            var hydratedUser = userBuilder.Hydrate(userData);
            if (hydratedUser != null)
            {
                Console.WriteLine($"Hydrated user: {hydratedUser.Username} ({hydratedUser.Email})");
                Console.WriteLine($"Display name: {hydratedUser.DisplayName}");
            }
            
            // Inspect type with reflection
            Console.WriteLine("\n=== Type Inspection ===");
            Type userType = typeof(User);
            Console.WriteLine($"Type: {userType.FullName}");
            Console.WriteLine($"Base: {userType.BaseType?.Name}");
            Console.WriteLine("Properties:");
            
            foreach (var prop in userType.GetProperties())
            {
                var attrs = prop.GetCustomAttributes().Select(a => a.GetType().Name.Replace("Attribute", ""));
                string attrStr = attrs.Any() ? $"[{string.Join(", ", attrs)}] " : "";
                Console.WriteLine($"  {attrStr}{prop.PropertyType.Name} {prop.Name}");
            }
            
            Console.WriteLine("\nกด Enter เพื่อออก...");
            Console.ReadLine();
        }
    }
}
```

---

## 📝 สรุป Part 20

| หัวข้อ | Key Points |
|--------|-----------|
| Attributes | Metadata decorators บน code elements |
| `[AttributeUsage]` | ควบคุม target และ multiplicity |
| Custom Attribute | สืบทอดจาก `Attribute` class |
| `typeof(T)` | รับ Type object |
| `GetType()` | รับ runtime type |
| `GetProperties()` | ดู properties ด้วย reflection |
| `GetValue/SetValue` | อ่านเขียน property dynamically |
| `Activator.CreateInstance` | สร้าง object dynamically |
| `GetCustomAttribute<T>()` | อ่าน attribute จาก member |

---

**ก่อนหน้า → [Part 19: Extension Methods](part19-extension-methods.md)**  
**ต่อไป → [Part 21: WinForms Basics](../part21-30/part21-winforms-intro.md)**
