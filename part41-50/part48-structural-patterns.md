# Part 48: Structural Design Patterns (โครงสร้างดีไซน์แพทเทิร์น)

## ภาพรวม (Overview)

Structural Design Patterns คือแพทเทิร์นที่ช่วยในการจัดวางโครงสร้างของคลาสและออบเจกต์ให้สามารถทำงานร่วมกันได้อย่างมีประสิทธิภาพ แพทเทิร์นเหล่านี้เน้นการรวมกลุ่มของคลาสและออบเจกต์เพื่อสร้างโครงสร้างที่ใหญ่ขึ้น

### แพทเทิร์นที่จะเรียนในส่วนนี้:
- **Adapter** - แปลง interface ที่ไม่เข้ากันให้ทำงานร่วมกันได้
- **Decorator** - เพิ่มฟังก์ชันการทำงานโดยไม่ต้องแก้ไขคลาสเดิม
- **Facade** - ให้ interface ที่เรียบง่ายสำหรับระบบที่ซับซ้อน
- **Proxy** - ควบคุมการเข้าถึงออบเจกต์
- **Composite** - จัดการกลุ่มออบเจกต์แบบ tree structure
- **Bridge** - แยก abstraction ออกจาก implementation

---

## ขั้นตอนที่ 471: Adapter Pattern - พื้นฐาน

### คืออะไร?

Adapter Pattern คือแพทเทิร์นที่ทำหน้าที่เหมือน "อแดปเตอร์" ในชีวิตจริง เช่น อแดปเตอร์ปลั๊กไฟที่แปลงปลั๊กแบบยุโรปให้ใช้กับเต้ารับแบบไทยได้ ในโปรแกรมมิ่ง Adapter ช่วยให้คลาสที่มี interface ไม่เข้ากันสามารถทำงานร่วมกันได้

### เมื่อไหร่ควรใช้?
- เมื่อต้องการใช้ไลบรารีเก่า (legacy code) แต่ interface ไม่ตรงกับที่ต้องการ
- เมื่อต้องการรวม third-party library เข้ากับระบบ
- เมื่อต้องการ reuse คลาสที่มีอยู่แล้วแต่ interface ไม่เข้ากัน

### ตัวอย่าง: Legacy Payment Gateway Adapter

สมมติว่าระบบใหม่ของเราต้องการ interface `IPaymentProcessor` แต่ระบบเก่ามี `LegacyPaymentGateway` ที่มี method ต่างชื่อกัน

```csharp
using System;
using System.Collections.Generic;

namespace AdapterPattern
{
    // === Interface ใหม่ที่ระบบของเราต้องการ ===
    public interface IPaymentProcessor
    {
        bool ProcessPayment(string customerId, decimal amount, string currency);
        bool RefundPayment(string transactionId, decimal amount);
        string GetTransactionStatus(string transactionId);
    }

    // === ระบบ Payment เก่า (Legacy) ที่เราไม่สามารถแก้ไขได้ ===
    // สมมติว่านี่คือ third-party library หรือ legacy code
    public class LegacyPaymentGateway
    {
        public string InitiateCharge(string userId, double amountInCents, string currencyCode)
        {
            // จำลองการชำระเงิน - คืน transaction ID
            Console.WriteLine($"[Legacy] กำลังชำระเงิน {amountInCents} cents สกุลเงิน {currencyCode} สำหรับ user {userId}");
            string transactionId = $"TXN-{Guid.NewGuid().ToString().Substring(0, 8).ToUpper()}";
            Console.WriteLine($"[Legacy] Transaction ID: {transactionId}");
            return transactionId;
        }

        public bool ReverseCharge(string txnRef, double refundAmountInCents)
        {
            // จำลองการคืนเงิน
            Console.WriteLine($"[Legacy] กำลังคืนเงิน {refundAmountInCents} cents สำหรับ transaction {txnRef}");
            return true;
        }

        public int QueryTransactionState(string txnRef)
        {
            // คืนค่า: 0=Pending, 1=Success, 2=Failed, 3=Refunded
            Console.WriteLine($"[Legacy] กำลังตรวจสอบสถานะ transaction {txnRef}");
            return 1; // จำลองว่า Success
        }
    }

    // === Adapter: แปลง LegacyPaymentGateway ให้ใช้ IPaymentProcessor ได้ ===
    public class LegacyPaymentAdapter : IPaymentProcessor
    {
        private readonly LegacyPaymentGateway _legacyGateway;
        
        // เก็บ transaction ID ที่สร้างระหว่าง ProcessPayment เพื่อใช้ใน GetTransactionStatus
        private readonly Dictionary<string, string> _customerTransactions = new Dictionary<string, string>();

        public LegacyPaymentAdapter(LegacyPaymentGateway legacyGateway)
        {
            _legacyGateway = legacyGateway;
        }

        public bool ProcessPayment(string customerId, decimal amount, string currency)
        {
            // แปลง decimal เป็น double และ amount เป็น cents
            double amountInCents = (double)(amount * 100);
            
            // เรียก method ของ legacy system ที่ชื่อต่างกัน
            string transactionId = _legacyGateway.InitiateCharge(customerId, amountInCents, currency);
            
            // เก็บ transaction ID สำหรับใช้ทีหลัง
            if (!string.IsNullOrEmpty(transactionId))
            {
                _customerTransactions[customerId] = transactionId;
                return true;
            }
            
            return false;
        }

        public bool RefundPayment(string transactionId, decimal amount)
        {
            // แปลง decimal เป็น double และ amount เป็น cents
            double amountInCents = (double)(amount * 100);
            
            // เรียก method คืนเงินของ legacy system
            return _legacyGateway.ReverseCharge(transactionId, amountInCents);
        }

        public string GetTransactionStatus(string transactionId)
        {
            // แปลงค่า int จาก legacy เป็น string ที่อ่านได้
            int statusCode = _legacyGateway.QueryTransactionState(transactionId);
            
            return statusCode switch
            {
                0 => "Pending",
                1 => "Success",
                2 => "Failed",
                3 => "Refunded",
                _ => "Unknown"
            };
        }
    }

    // === ระบบ Payment ใหม่ที่ใช้ interface ใหม่ ===
    public class ModernPaymentService : IPaymentProcessor
    {
        public bool ProcessPayment(string customerId, decimal amount, string currency)
        {
            Console.WriteLine($"[Modern] ชำระเงิน {amount} {currency} สำหรับ customer {customerId}");
            return true;
        }

        public bool RefundPayment(string transactionId, decimal amount)
        {
            Console.WriteLine($"[Modern] คืนเงิน {amount} สำหรับ transaction {transactionId}");
            return true;
        }

        public string GetTransactionStatus(string transactionId)
        {
            Console.WriteLine($"[Modern] ตรวจสอบสถานะ {transactionId}");
            return "Success";
        }
    }

    // === Client Code ที่ใช้งาน IPaymentProcessor ===
    public class OrderService
    {
        private readonly IPaymentProcessor _paymentProcessor;

        // รับ IPaymentProcessor ผ่าน Dependency Injection
        // ไม่ต้องรู้ว่าเป็น Legacy หรือ Modern
        public OrderService(IPaymentProcessor paymentProcessor)
        {
            _paymentProcessor = paymentProcessor;
        }

        public void ProcessOrder(string customerId, decimal totalAmount)
        {
            Console.WriteLine($"\n=== กำลังประมวลผลคำสั่งซื้อ ===");
            Console.WriteLine($"Customer: {customerId}, ยอดรวม: {totalAmount} THB");
            
            bool success = _paymentProcessor.ProcessPayment(customerId, totalAmount, "THB");
            
            if (success)
            {
                Console.WriteLine("✓ ชำระเงินสำเร็จ!");
            }
            else
            {
                Console.WriteLine("✗ ชำระเงินล้มเหลว!");
            }
        }
    }

    class Program471
    {
        static void Main471()
        {
            Console.WriteLine("=== Adapter Pattern: Legacy Payment Gateway ===\n");

            // === ใช้ระบบ Legacy ผ่าน Adapter ===
            Console.WriteLine("--- ใช้ Legacy Gateway ผ่าน Adapter ---");
            var legacyGateway = new LegacyPaymentGateway();
            IPaymentProcessor legacyAdapter = new LegacyPaymentAdapter(legacyGateway);
            
            var orderService1 = new OrderService(legacyAdapter);
            orderService1.ProcessOrder("CUST-001", 1500.00m);

            // ตรวจสอบสถานะ
            string status = legacyAdapter.GetTransactionStatus("TXN-12345678");
            Console.WriteLine($"สถานะ: {status}");

            Console.WriteLine();

            // === ใช้ระบบ Modern โดยตรง ===
            Console.WriteLine("--- ใช้ Modern Payment Service โดยตรง ---");
            IPaymentProcessor modernService = new ModernPaymentService();
            
            var orderService2 = new OrderService(modernService);
            orderService2.ProcessOrder("CUST-002", 2500.00m);

            // Client code ใช้งานเหมือนกันทั้งคู่ เพราะ Adapter ทำให้ interface เหมือนกัน
            Console.WriteLine("\n✓ Adapter Pattern ช่วยให้ใช้ Legacy และ Modern System ด้วย interface เดียวกัน");
        }
    }
}
```

---

## ขั้นตอนที่ 472: Adapter Pattern - ILogger Adapter

### ตัวอย่างเพิ่มเติม: แปลง Logging Library

ในโลกจริง เราอาจต้องการเปลี่ยน logging library แต่ไม่อยากแก้โค้ดทุกที่ที่ใช้งาน

```csharp
using System;
using Microsoft.Extensions.Logging;

namespace LoggerAdapterPattern
{
    // === Interface ที่ระบบของเราใช้ (Custom Logging Interface) ===
    public interface IAppLogger
    {
        void LogInfo(string message);
        void LogWarning(string message);
        void LogError(string message, Exception ex = null);
        void LogDebug(string message);
    }

    // === Logger เก่าของเรา (Custom Implementation) ===
    public class ConsoleLogger : IAppLogger
    {
        private readonly string _componentName;

        public ConsoleLogger(string componentName)
        {
            _componentName = componentName;
        }

        public void LogInfo(string message)
        {
            Console.ForegroundColor = ConsoleColor.Green;
            Console.WriteLine($"[INFO] [{_componentName}] {DateTime.Now:HH:mm:ss} - {message}");
            Console.ResetColor();
        }

        public void LogWarning(string message)
        {
            Console.ForegroundColor = ConsoleColor.Yellow;
            Console.WriteLine($"[WARN] [{_componentName}] {DateTime.Now:HH:mm:ss} - {message}");
            Console.ResetColor();
        }

        public void LogError(string message, Exception ex = null)
        {
            Console.ForegroundColor = ConsoleColor.Red;
            Console.WriteLine($"[ERROR] [{_componentName}] {DateTime.Now:HH:mm:ss} - {message}");
            if (ex != null)
                Console.WriteLine($"  Exception: {ex.Message}");
            Console.ResetColor();
        }

        public void LogDebug(string message)
        {
            Console.ForegroundColor = ConsoleColor.Gray;
            Console.WriteLine($"[DEBUG] [{_componentName}] {DateTime.Now:HH:mm:ss} - {message}");
            Console.ResetColor();
        }
    }

    // === Adapter: แปลง Microsoft.Extensions.Logging.ILogger ให้ใช้ IAppLogger ได้ ===
    // ใช้เมื่อต้องการเปลี่ยนมาใช้ Microsoft logging framework
    public class MicrosoftLoggerAdapter : IAppLogger
    {
        private readonly ILogger _microsoftLogger;

        public MicrosoftLoggerAdapter(ILogger microsoftLogger)
        {
            _microsoftLogger = microsoftLogger;
        }

        public void LogInfo(string message)
        {
            // แปลง LogInfo ของเราเป็น LogInformation ของ Microsoft
            _microsoftLogger.LogInformation(message);
        }

        public void LogWarning(string message)
        {
            // แปลง LogWarning ของเราเป็น LogWarning ของ Microsoft (ชื่อเหมือนกัน)
            _microsoftLogger.LogWarning(message);
        }

        public void LogError(string message, Exception ex = null)
        {
            // แปลง LogError พร้อม Exception
            _microsoftLogger.LogError(ex, message);
        }

        public void LogDebug(string message)
        {
            // แปลง LogDebug (ชื่อเหมือนกัน)
            _microsoftLogger.LogDebug(message);
        }
    }

    // === Null Logger: ใช้สำหรับ Testing (No-op implementation) ===
    public class NullLogger : IAppLogger
    {
        public static readonly NullLogger Instance = new NullLogger();
        
        private NullLogger() { }
        
        public void LogInfo(string message) { }
        public void LogWarning(string message) { }
        public void LogError(string message, Exception ex = null) { }
        public void LogDebug(string message) { }
    }

    // === Service ที่ใช้ IAppLogger ===
    public class UserService
    {
        private readonly IAppLogger _logger;

        public UserService(IAppLogger logger)
        {
            _logger = logger;
        }

        public bool CreateUser(string username, string email)
        {
            _logger.LogInfo($"กำลังสร้าง user: {username}");
            
            if (string.IsNullOrWhiteSpace(username))
            {
                _logger.LogWarning("Username ไม่ถูกต้อง - ไม่สามารถสร้าง user ได้");
                return false;
            }

            try
            {
                // จำลองการสร้าง user
                _logger.LogDebug($"กำลัง validate email: {email}");
                
                if (!email.Contains("@"))
                    throw new ArgumentException("Email ไม่ถูกต้อง");

                _logger.LogInfo($"สร้าง user '{username}' สำเร็จ");
                return true;
            }
            catch (Exception ex)
            {
                _logger.LogError($"ไม่สามารถสร้าง user '{username}' ได้", ex);
                return false;
            }
        }
    }

    class Program472
    {
        static void Main472()
        {
            Console.WriteLine("=== Adapter Pattern: ILogger Adapter ===\n");

            // === ใช้ ConsoleLogger โดยตรง ===
            Console.WriteLine("--- ใช้ ConsoleLogger ---");
            var consoleLogger = new ConsoleLogger("UserService");
            var userService1 = new UserService(consoleLogger);
            
            userService1.CreateUser("สมชาย", "somchai@example.com");
            userService1.CreateUser("", "invalid-email");
            userService1.CreateUser("สมหญิง", "no-at-sign");

            Console.WriteLine();

            // === ใช้ NullLogger สำหรับ Testing ===
            Console.WriteLine("--- ใช้ NullLogger (ไม่แสดง log ใดๆ) ---");
            var userService2 = new UserService(NullLogger.Instance);
            bool result = userService2.CreateUser("ทดสอบ", "test@test.com");
            Console.WriteLine($"ผลการสร้าง user: {(result ? "สำเร็จ" : "ล้มเหลว")}");

            Console.WriteLine("\n✓ ILogger Adapter ช่วยให้เปลี่ยน logging library ได้โดยไม่ต้องแก้ไข Service");
        }
    }
}
```

---

## ขั้นตอนที่ 473: Decorator Pattern

### คืออะไร?

Decorator Pattern ช่วยให้เพิ่มฟังก์ชันการทำงานให้กับออบเจกต์โดยไม่ต้องแก้ไขคลาสเดิม โดยการ "ห่อ" (wrap) ออบเจกต์ด้วยออบเจกต์อีกอัน ทำให้สามารถเพิ่มฟังก์ชันได้แบบ dynamic และสามารถผสมกันได้หลายชั้น

### เมื่อไหร่ควรใช้?
- เมื่อต้องการเพิ่มฟังก์ชันให้กับออบเจกต์แบบ dynamic โดยไม่แก้ไขคลาสเดิม
- เมื่อการใช้ inheritance จะทำให้เกิดคลาสมากเกินไป
- เมื่อต้องการผสมฟังก์ชันต่างๆ ได้อย่างยืดหยุ่น

### ตัวอย่าง: Stream Decorators และ Logging/Caching Decorators

```csharp
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Text;

namespace DecoratorPattern
{
    // ============================================================
    // ส่วนที่ 1: Data Service Decorators (Logging + Caching)
    // ============================================================

    // === Interface หลักสำหรับดึงข้อมูล ===
    public interface IDataService
    {
        string GetData(string key);
        void SaveData(string key, string value);
    }

    // === Implementation จริงที่ดึงข้อมูลจาก Database ===
    public class DatabaseDataService : IDataService
    {
        private readonly Dictionary<string, string> _database = new Dictionary<string, string>
        {
            { "user:1", "สมชาย ใจดี" },
            { "user:2", "สมหญิง รักเรียน" },
            { "product:1", "แล็ปท็อป ASUS" },
            { "product:2", "โทรศัพท์ Samsung" }
        };

        public string GetData(string key)
        {
            // จำลองการดึงข้อมูลช้าๆ จาก database
            System.Threading.Thread.Sleep(100); // delay 100ms
            
            if (_database.TryGetValue(key, out string value))
            {
                return value;
            }
            return null;
        }

        public void SaveData(string key, string value)
        {
            System.Threading.Thread.Sleep(50); // delay 50ms
            _database[key] = value;
            Console.WriteLine($"[Database] บันทึก key='{key}' สำเร็จ");
        }
    }

    // === Abstract Decorator Base Class ===
    // Decorator ทุกตัวต้อง inherit จากนี้
    public abstract class DataServiceDecorator : IDataService
    {
        // เก็บ reference ไปยัง IDataService ที่จะถูก wrap
        protected readonly IDataService _wrappedService;

        protected DataServiceDecorator(IDataService dataService)
        {
            _wrappedService = dataService;
        }

        // โดย default จะ delegate ไปยัง wrapped service
        public virtual string GetData(string key)
        {
            return _wrappedService.GetData(key);
        }

        public virtual void SaveData(string key, string value)
        {
            _wrappedService.SaveData(key, value);
        }
    }

    // === Logging Decorator: บันทึก log ทุกครั้งที่มีการเรียกใช้ ===
    public class LoggingDataServiceDecorator : DataServiceDecorator
    {
        private readonly string _serviceName;

        public LoggingDataServiceDecorator(IDataService dataService, string serviceName = "DataService")
            : base(dataService)
        {
            _serviceName = serviceName;
        }

        public override string GetData(string key)
        {
            Console.WriteLine($"[LOG] {_serviceName}.GetData() เรียกด้วย key='{key}' เวลา {DateTime.Now:HH:mm:ss.fff}");
            
            var stopwatch = Stopwatch.StartNew();
            string result = _wrappedService.GetData(key);
            stopwatch.Stop();
            
            Console.WriteLine($"[LOG] {_serviceName}.GetData() เสร็จสิ้น ใช้เวลา {stopwatch.ElapsedMilliseconds}ms, result={(result ?? "null")}");
            
            return result;
        }

        public override void SaveData(string key, string value)
        {
            Console.WriteLine($"[LOG] {_serviceName}.SaveData() เรียกด้วย key='{key}', value='{value}'");
            
            var stopwatch = Stopwatch.StartNew();
            _wrappedService.SaveData(key, value);
            stopwatch.Stop();
            
            Console.WriteLine($"[LOG] {_serviceName}.SaveData() เสร็จสิ้น ใช้เวลา {stopwatch.ElapsedMilliseconds}ms");
        }
    }

    // === Caching Decorator: Cache ผลลัพธ์เพื่อลดการเรียก database ===
    public class CachingDataServiceDecorator : DataServiceDecorator
    {
        private readonly Dictionary<string, (string Value, DateTime Expiry)> _cache 
            = new Dictionary<string, (string, DateTime)>();
        private readonly TimeSpan _cacheDuration;

        public CachingDataServiceDecorator(IDataService dataService, TimeSpan? cacheDuration = null)
            : base(dataService)
        {
            _cacheDuration = cacheDuration ?? TimeSpan.FromMinutes(5);
        }

        public override string GetData(string key)
        {
            // ตรวจสอบว่ามีข้อมูลใน cache และยังไม่หมดอายุ
            if (_cache.TryGetValue(key, out var cached) && cached.Expiry > DateTime.Now)
            {
                Console.WriteLine($"[CACHE] Cache HIT สำหรับ key='{key}' (หมดอายุ: {cached.Expiry:HH:mm:ss})");
                return cached.Value;
            }

            Console.WriteLine($"[CACHE] Cache MISS สำหรับ key='{key}' - ดึงข้อมูลจาก source");
            
            // ดึงข้อมูลจาก wrapped service
            string value = _wrappedService.GetData(key);
            
            if (value != null)
            {
                // บันทึกลง cache
                _cache[key] = (value, DateTime.Now.Add(_cacheDuration));
                Console.WriteLine($"[CACHE] บันทึก cache สำหรับ key='{key}'");
            }
            
            return value;
        }

        public override void SaveData(string key, string value)
        {
            // เมื่อบันทึกข้อมูล ให้ invalidate cache
            if (_cache.ContainsKey(key))
            {
                _cache.Remove(key);
                Console.WriteLine($"[CACHE] Invalidated cache สำหรับ key='{key}'");
            }
            
            _wrappedService.SaveData(key, value);
        }

        public void ClearCache()
        {
            _cache.Clear();
            Console.WriteLine("[CACHE] ล้าง cache ทั้งหมดแล้ว");
        }
    }

    // === Retry Decorator: ลองใหม่อัตโนมัติเมื่อเกิดข้อผิดพลาด ===
    public class RetryDataServiceDecorator : DataServiceDecorator
    {
        private readonly int _maxRetries;
        private readonly TimeSpan _retryDelay;

        public RetryDataServiceDecorator(IDataService dataService, int maxRetries = 3, int retryDelayMs = 500)
            : base(dataService)
        {
            _maxRetries = maxRetries;
            _retryDelay = TimeSpan.FromMilliseconds(retryDelayMs);
        }

        public override string GetData(string key)
        {
            for (int attempt = 1; attempt <= _maxRetries; attempt++)
            {
                try
                {
                    return _wrappedService.GetData(key);
                }
                catch (Exception ex)
                {
                    Console.WriteLine($"[RETRY] ครั้งที่ {attempt}/{_maxRetries} ล้มเหลว: {ex.Message}");
                    
                    if (attempt < _maxRetries)
                    {
                        System.Threading.Thread.Sleep(_retryDelay);
                    }
                    else
                    {
                        Console.WriteLine($"[RETRY] เกิน retry limit แล้ว - throw exception");
                        throw;
                    }
                }
            }
            return null;
        }

        public override void SaveData(string key, string value)
        {
            for (int attempt = 1; attempt <= _maxRetries; attempt++)
            {
                try
                {
                    _wrappedService.SaveData(key, value);
                    return;
                }
                catch (Exception ex)
                {
                    Console.WriteLine($"[RETRY] ครั้งที่ {attempt}/{_maxRetries} ล้มเหลว: {ex.Message}");
                    if (attempt >= _maxRetries) throw;
                    System.Threading.Thread.Sleep(_retryDelay);
                }
            }
        }
    }

    class Program473
    {
        static void Main473()
        {
            Console.WriteLine("=== Decorator Pattern: Data Service Decorators ===\n");

            // === ขั้นที่ 1: ใช้ DatabaseDataService โดยตรง ===
            Console.WriteLine("--- ขั้นที่ 1: ใช้ Database โดยตรง ---");
            IDataService dbService = new DatabaseDataService();
            Console.WriteLine(dbService.GetData("user:1"));
            Console.WriteLine();

            // === ขั้นที่ 2: เพิ่ม Logging Decorator ===
            Console.WriteLine("--- ขั้นที่ 2: เพิ่ม Logging ---");
            IDataService withLogging = new LoggingDataServiceDecorator(dbService, "MyApp");
            withLogging.GetData("user:1");
            Console.WriteLine();

            // === ขั้นที่ 3: เพิ่ม Caching + Logging ===
            Console.WriteLine("--- ขั้นที่ 3: เพิ่ม Caching + Logging ---");
            // Stack decorators: Database -> Cache -> Log
            // การเรียก GetData จะผ่าน Log -> Cache -> Database
            IDataService withCachingAndLogging = new LoggingDataServiceDecorator(
                new CachingDataServiceDecorator(dbService, TimeSpan.FromMinutes(1)),
                "CachedService"
            );

            Console.WriteLine("เรียกครั้งที่ 1 (ควร miss cache):");
            withCachingAndLogging.GetData("product:1");
            
            Console.WriteLine("\nเรียกครั้งที่ 2 (ควร hit cache):");
            withCachingAndLogging.GetData("product:1");
            
            Console.WriteLine("\nบันทึกข้อมูล (ควร invalidate cache):");
            withCachingAndLogging.SaveData("product:1", "แล็ปท็อป ASUS ROG");
            
            Console.WriteLine("\nเรียกครั้งที่ 3 หลังบันทึก (ควร miss cache อีกครั้ง):");
            withCachingAndLogging.GetData("product:1");

            Console.WriteLine("\n✓ Decorator Pattern ช่วยให้เพิ่ม Logging, Caching, Retry ได้โดยไม่แก้ไข DatabaseDataService");
        }
    }
}
```

---

## ขั้นตอนที่ 474: Facade Pattern

### คืออะไร?

Facade Pattern ให้ interface ที่เรียบง่ายและใช้งานง่ายสำหรับระบบที่ซับซ้อน เหมือนรีโมทคอนโทรลที่มีปุ่มเดียวสำหรับ "ดูหนัง" แต่ข้างหลังมันสั่ง TV, DVD Player, Sound System, และ Dimmer หลายตัว

### เมื่อไหร่ควรใช้?
- เมื่อต้องการซ่อนความซับซ้อนของระบบจาก client
- เมื่อต้องการสร้าง entry point เดียวสำหรับ subsystem ที่มีหลาย component
- เมื่อต้องการ layer ระหว่าง client และ complex subsystem

### ตัวอย่าง: HomeTheatre Facade และ Order Facade

```csharp
using System;
using System.Collections.Generic;

namespace FacadePattern
{
    // ============================================================
    // ส่วนที่ 1: HomeTheatre Facade
    // ============================================================

    // === Subsystem Components ===
    public class Television
    {
        public void TurnOn() => Console.WriteLine("[TV] เปิดโทรทัศน์");
        public void TurnOff() => Console.WriteLine("[TV] ปิดโทรทัศน์");
        public void SetInput(string input) => Console.WriteLine($"[TV] เปลี่ยน input เป็น {input}");
        public void SetVolume(int level) => Console.WriteLine($"[TV] ตั้งระดับเสียง TV เป็น {level}");
    }

    public class BluRayPlayer
    {
        public void TurnOn() => Console.WriteLine("[BluRay] เปิด Blu-ray Player");
        public void TurnOff() => Console.WriteLine("[BluRay] ปิด Blu-ray Player");
        public void LoadDisc(string movie) => Console.WriteLine($"[BluRay] ใส่แผ่น: {movie}");
        public void Play() => Console.WriteLine("[BluRay] เริ่มเล่นหนัง");
        public void Pause() => Console.WriteLine("[BluRay] หยุดชั่วคราว");
        public void Eject() => Console.WriteLine("[BluRay] นำแผ่นออก");
    }

    public class SoundSystem
    {
        public void TurnOn() => Console.WriteLine("[Sound] เปิดระบบเสียง");
        public void TurnOff() => Console.WriteLine("[Sound] ปิดระบบเสียง");
        public void SetMode(string mode) => Console.WriteLine($"[Sound] ตั้ง mode เป็น {mode}");
        public void SetVolume(int level) => Console.WriteLine($"[Sound] ตั้งระดับเสียงเป็น {level}");
        public void SetSurroundSound(bool enabled) => Console.WriteLine($"[Sound] Surround Sound: {(enabled ? "เปิด" : "ปิด")}");
    }

    public class LightDimmer
    {
        public void Dim(int percentage) => Console.WriteLine($"[Light] ลดความสว่างเป็น {percentage}%");
        public void BrightenUp() => Console.WriteLine("[Light] เพิ่มความสว่างเต็มที่");
    }

    public class Projector
    {
        public void TurnOn() => Console.WriteLine("[Projector] เปิดโปรเจคเตอร์");
        public void TurnOff() => Console.WriteLine("[Projector] ปิดโปรเจคเตอร์");
        public void SetInput(string input) => Console.WriteLine($"[Projector] ตั้ง input เป็น {input}");
        public void SetMode(string mode) => Console.WriteLine($"[Projector] ตั้ง mode เป็น {mode}");
    }

    // === Facade: ซ่อนความซับซ้อนทั้งหมด ===
    public class HomeTheatreFacade
    {
        private readonly Television _tv;
        private readonly BluRayPlayer _bluRay;
        private readonly SoundSystem _sound;
        private readonly LightDimmer _lights;
        private readonly Projector _projector;

        public HomeTheatreFacade()
        {
            _tv = new Television();
            _bluRay = new BluRayPlayer();
            _sound = new SoundSystem();
            _lights = new LightDimmer();
            _projector = new Projector();
        }

        // === ปุ่ม "ดูหนัง" - ทำทุกอย่างด้วยคำสั่งเดียว ===
        public void WatchMovie(string movie)
        {
            Console.WriteLine($"\n=== เตรียมดูหนัง: {movie} ===");
            
            _lights.Dim(20);           // หรี่ไฟให้มืด
            _projector.TurnOn();        // เปิดโปรเจคเตอร์
            _projector.SetMode("Movie");
            _projector.SetInput("HDMI1");
            _sound.TurnOn();            // เปิดระบบเสียง
            _sound.SetMode("Movie");
            _sound.SetSurroundSound(true);
            _sound.SetVolume(60);
            _bluRay.TurnOn();           // เปิด Blu-ray
            _bluRay.LoadDisc(movie);    // ใส่แผ่น
            _bluRay.Play();             // เริ่มเล่น
            
            Console.WriteLine($"✓ เริ่มดู {movie} แล้ว สนุกกับการดูหนังครับ!");
        }

        // === ปุ่ม "หยุดดูหนัง" ===
        public void EndMovie()
        {
            Console.WriteLine("\n=== จบการดูหนัง ===");
            
            _bluRay.Eject();
            _bluRay.TurnOff();
            _sound.TurnOff();
            _projector.TurnOff();
            _lights.BrightenUp();       // เปิดไฟเต็มที่
            
            Console.WriteLine("✓ ปิดอุปกรณ์ทั้งหมดเรียบร้อยแล้ว");
        }

        // === โหมดฟังเพลง ===
        public void ListenToMusic()
        {
            Console.WriteLine("\n=== โหมดฟังเพลง ===");
            
            _lights.Dim(50);
            _tv.TurnOn();
            _tv.SetInput("AUX");
            _sound.TurnOn();
            _sound.SetMode("Music");
            _sound.SetSurroundSound(false);
            _sound.SetVolume(40);
            
            Console.WriteLine("✓ พร้อมฟังเพลงแล้ว");
        }
    }

    // ============================================================
    // ส่วนที่ 2: Order Processing Facade
    // ============================================================

    // === Subsystem: ส่วนต่างๆ ของระบบ E-commerce ===
    public class InventoryService
    {
        private readonly Dictionary<string, int> _stock = new Dictionary<string, int>
        {
            { "PROD-001", 50 },
            { "PROD-002", 3 },
            { "PROD-003", 0 }
        };

        public bool CheckAvailability(string productId, int quantity)
        {
            if (_stock.TryGetValue(productId, out int available))
            {
                bool isAvailable = available >= quantity;
                Console.WriteLine($"[Inventory] {productId}: สต็อก {available} ชิ้น, ต้องการ {quantity} ชิ้น - {(isAvailable ? "มีสินค้า" : "สินค้าหมด")}");
                return isAvailable;
            }
            Console.WriteLine($"[Inventory] ไม่พบสินค้า {productId}");
            return false;
        }

        public void ReserveStock(string productId, int quantity)
        {
            if (_stock.ContainsKey(productId))
            {
                _stock[productId] -= quantity;
                Console.WriteLine($"[Inventory] จองสต็อก {productId}: {quantity} ชิ้น (เหลือ {_stock[productId]} ชิ้น)");
            }
        }
    }

    public class PaymentService
    {
        public bool ProcessPayment(string customerId, decimal amount, string method)
        {
            Console.WriteLine($"[Payment] ชำระเงิน {amount:C2} ด้วย {method} สำหรับ customer {customerId}");
            // จำลองการชำระเงิน
            return true;
        }

        public string GenerateReceipt(string orderId, decimal amount)
        {
            string receiptNo = $"RCP-{orderId}-{DateTime.Now:yyyyMMddHHmm}";
            Console.WriteLine($"[Payment] สร้างใบเสร็จ: {receiptNo} ยอด {amount:C2}");
            return receiptNo;
        }
    }

    public class ShippingService
    {
        public string CalculateShipping(string address, decimal weight)
        {
            decimal shippingCost = weight * 20; // 20 บาท/กิโลกรัม
            Console.WriteLine($"[Shipping] คำนวณค่าจัดส่งไปที่ {address}: {shippingCost:C2}");
            return shippingCost.ToString("C2");
        }

        public string CreateShipment(string orderId, string address)
        {
            string trackingNo = $"TRK-{orderId}-{DateTime.Now:yyyyMMdd}";
            Console.WriteLine($"[Shipping] สร้างการจัดส่ง: tracking {trackingNo} ไปที่ {address}");
            return trackingNo;
        }
    }

    public class NotificationService
    {
        public void SendEmail(string email, string subject, string body)
        {
            Console.WriteLine($"[Email] ส่ง email ถึง {email}: {subject}");
        }

        public void SendSMS(string phone, string message)
        {
            Console.WriteLine($"[SMS] ส่ง SMS ถึง {phone}: {message}");
        }
    }

    // === Order Facade: รวม subsystem ทั้งหมดเข้าด้วยกัน ===
    public class OrderFacade
    {
        private readonly InventoryService _inventory;
        private readonly PaymentService _payment;
        private readonly ShippingService _shipping;
        private readonly NotificationService _notification;

        public OrderFacade()
        {
            _inventory = new InventoryService();
            _payment = new PaymentService();
            _shipping = new ShippingService();
            _notification = new NotificationService();
        }

        // === สั่งซื้อสินค้าด้วยขั้นตอนเดียว ===
        public OrderResult PlaceOrder(OrderRequest request)
        {
            Console.WriteLine($"\n=== ประมวลผลคำสั่งซื้อของ {request.CustomerName} ===");

            // 1. ตรวจสอบสต็อก
            if (!_inventory.CheckAvailability(request.ProductId, request.Quantity))
            {
                return new OrderResult { Success = false, Message = "สินค้าหมดสต็อก" };
            }

            // 2. ชำระเงิน
            decimal total = request.UnitPrice * request.Quantity;
            if (!_payment.ProcessPayment(request.CustomerId, total, request.PaymentMethod))
            {
                return new OrderResult { Success = false, Message = "ชำระเงินล้มเหลว" };
            }

            // 3. จองสต็อก
            _inventory.ReserveStock(request.ProductId, request.Quantity);

            // 4. สร้างการจัดส่ง
            string orderId = $"ORD-{DateTime.Now:yyyyMMddHHmmss}";
            string trackingNo = _shipping.CreateShipment(orderId, request.ShippingAddress);

            // 5. สร้างใบเสร็จ
            string receiptNo = _payment.GenerateReceipt(orderId, total);

            // 6. แจ้งเตือน customer
            _notification.SendEmail(
                request.CustomerEmail,
                $"ยืนยันคำสั่งซื้อ {orderId}",
                $"คำสั่งซื้อของคุณสำเร็จแล้ว tracking: {trackingNo}"
            );
            _notification.SendSMS(request.CustomerPhone, $"สั่งซื้อสำเร็จ order: {orderId}");

            Console.WriteLine($"✓ คำสั่งซื้อสำเร็จ! Order ID: {orderId}");
            return new OrderResult 
            { 
                Success = true, 
                OrderId = orderId,
                TrackingNumber = trackingNo,
                ReceiptNumber = receiptNo,
                Message = "สั่งซื้อสำเร็จ"
            };
        }
    }

    public class OrderRequest
    {
        public string CustomerId { get; set; }
        public string CustomerName { get; set; }
        public string CustomerEmail { get; set; }
        public string CustomerPhone { get; set; }
        public string ProductId { get; set; }
        public int Quantity { get; set; }
        public decimal UnitPrice { get; set; }
        public string PaymentMethod { get; set; }
        public string ShippingAddress { get; set; }
    }

    public class OrderResult
    {
        public bool Success { get; set; }
        public string OrderId { get; set; }
        public string TrackingNumber { get; set; }
        public string ReceiptNumber { get; set; }
        public string Message { get; set; }
    }

    class Program474
    {
        static void Main474()
        {
            Console.WriteLine("=== Facade Pattern Demo ===\n");

            // === HomeTheatre Facade ===
            Console.WriteLine("--- Home Theatre Facade ---");
            var homeTheatre = new HomeTheatreFacade();
            homeTheatre.WatchMovie("Avengers: Endgame");
            homeTheatre.EndMovie();

            Console.WriteLine();

            // === Order Facade ===
            Console.WriteLine("--- Order Processing Facade ---");
            var orderFacade = new OrderFacade();
            
            var order = new OrderRequest
            {
                CustomerId = "CUST-001",
                CustomerName = "สมชาย ใจดี",
                CustomerEmail = "somchai@example.com",
                CustomerPhone = "081-234-5678",
                ProductId = "PROD-001",
                Quantity = 2,
                UnitPrice = 15000m,
                PaymentMethod = "Credit Card",
                ShippingAddress = "123/4 ถนนสีลม กรุงเทพฯ"
            };

            var result = orderFacade.PlaceOrder(order);
            Console.WriteLine($"\nผลลัพธ์: {result.Message}");
            if (result.Success)
            {
                Console.WriteLine($"Order: {result.OrderId}");
                Console.WriteLine($"Tracking: {result.TrackingNumber}");
                Console.WriteLine($"Receipt: {result.ReceiptNumber}");
            }

            Console.WriteLine("\n✓ Facade Pattern ซ่อนความซับซ้อนของ Subsystems ทั้งหมด");
        }
    }
}
```

---

## ขั้นตอนที่ 475: Proxy Pattern

### คืออะไร?

Proxy Pattern ให้ออบเจกต์ตัวแทน (surrogate) แทนออบเจกต์อีกตัว เพื่อควบคุมการเข้าถึง มี 3 ประเภทหลัก:
- **Virtual Proxy**: Lazy loading - สร้างออบเจกต์เมื่อต้องการจริงๆ เท่านั้น
- **Caching Proxy**: Cache ผลลัพธ์เพื่อลดการคำนวณซ้ำ
- **Protection Proxy**: ควบคุมสิทธิ์การเข้าถึง

### ตัวอย่าง: Virtual Proxy สำหรับ Images และ Caching Proxy

```csharp
using System;
using System.Collections.Generic;
using System.Threading;

namespace ProxyPattern
{
    // ============================================================
    // ส่วนที่ 1: Virtual Proxy - Lazy Loading Image
    // ============================================================

    public interface IImage
    {
        void Display();
        string FileName { get; }
        long FileSizeBytes { get; }
    }

    // === Real Image: โหลดไฟล์จาก disk จริงๆ ===
    public class RealImage : IImage
    {
        public string FileName { get; }
        public long FileSizeBytes { get; private set; }
        private byte[] _imageData; // จำลองข้อมูลภาพ

        public RealImage(string fileName)
        {
            FileName = fileName;
            LoadFromDisk(); // โหลดทันทีที่สร้าง
        }

        private void LoadFromDisk()
        {
            Console.WriteLine($"[RealImage] กำลังโหลด '{FileName}' จาก disk...");
            Thread.Sleep(200); // จำลองการโหลดช้าๆ
            FileSizeBytes = new Random().Next(1024 * 512, 1024 * 1024 * 10); // 512KB - 10MB
            _imageData = new byte[100]; // จำลองข้อมูล
            Console.WriteLine($"[RealImage] โหลด '{FileName}' เสร็จแล้ว ({FileSizeBytes / 1024:N0} KB)");
        }

        public void Display()
        {
            Console.WriteLine($"[RealImage] แสดงภาพ: {FileName} ({FileSizeBytes / 1024:N0} KB)");
        }
    }

    // === Virtual Proxy: โหลดภาพเฉพาะเมื่อต้องการแสดงผลเท่านั้น ===
    public class LazyImageProxy : IImage
    {
        private RealImage _realImage; // null จนกว่าจะมีการเรียกใช้งาน
        private readonly string _fileName;
        private long _fileSizeBytes;

        public string FileName => _fileName;
        public long FileSizeBytes
        {
            get
            {
                EnsureLoaded();
                return _realImage.FileSizeBytes;
            }
        }

        public LazyImageProxy(string fileName)
        {
            _fileName = fileName;
            Console.WriteLine($"[LazyProxy] สร้าง Proxy สำหรับ '{fileName}' (ยังไม่โหลด)");
        }

        private void EnsureLoaded()
        {
            if (_realImage == null)
            {
                Console.WriteLine($"[LazyProxy] โหลด '{_fileName}' เป็นครั้งแรก...");
                _realImage = new RealImage(_fileName);
            }
        }

        public void Display()
        {
            EnsureLoaded();
            _realImage.Display();
        }
    }

    // ============================================================
    // ส่วนที่ 2: Protection Proxy - Access Control
    // ============================================================

    public interface IDocumentService
    {
        string ReadDocument(string docId);
        void WriteDocument(string docId, string content);
        void DeleteDocument(string docId);
    }

    // === Real Document Service ===
    public class DocumentService : IDocumentService
    {
        private readonly Dictionary<string, string> _documents = new Dictionary<string, string>
        {
            { "doc:1", "เอกสารนโยบายบริษัท" },
            { "doc:2", "รายงานประจำปี 2024" },
            { "doc:3", "ข้อมูลเงินเดือนพนักงาน" }
        };

        public string ReadDocument(string docId)
        {
            if (_documents.TryGetValue(docId, out string content))
            {
                Console.WriteLine($"[DocService] อ่านเอกสาร {docId}: {content}");
                return content;
            }
            Console.WriteLine($"[DocService] ไม่พบเอกสาร {docId}");
            return null;
        }

        public void WriteDocument(string docId, string content)
        {
            _documents[docId] = content;
            Console.WriteLine($"[DocService] บันทึกเอกสาร {docId}: {content}");
        }

        public void DeleteDocument(string docId)
        {
            if (_documents.Remove(docId))
                Console.WriteLine($"[DocService] ลบเอกสาร {docId} แล้ว");
            else
                Console.WriteLine($"[DocService] ไม่พบเอกสาร {docId} สำหรับลบ");
        }
    }

    // === User และ Role สำหรับ Access Control ===
    public enum UserRole { Guest, Employee, Manager, Admin }

    public class User
    {
        public string Name { get; set; }
        public UserRole Role { get; set; }
    }

    // === Protection Proxy: ตรวจสอบสิทธิ์ก่อนทุกการเรียกใช้ ===
    public class SecureDocumentProxy : IDocumentService
    {
        private readonly DocumentService _realService;
        private readonly User _currentUser;

        // กำหนดสิทธิ์ขั้นต่ำสำหรับแต่ละ operation
        private readonly Dictionary<string, UserRole> _sensitiveDocRequirements = new Dictionary<string, UserRole>
        {
            { "doc:3", UserRole.Manager } // เอกสารเงินเดือนต้องเป็น Manager ขึ้นไป
        };

        public SecureDocumentProxy(DocumentService service, User user)
        {
            _realService = service;
            _currentUser = user;
        }

        public string ReadDocument(string docId)
        {
            // ตรวจสอบสิทธิ์อ่าน
            if (!CanAccess(docId, UserRole.Employee))
            {
                Console.WriteLine($"[SecureProxy] ปฏิเสธการเข้าถึง: {_currentUser.Name} ({_currentUser.Role}) ไม่มีสิทธิ์อ่าน {docId}");
                return null;
            }

            Console.WriteLine($"[SecureProxy] อนุญาต: {_currentUser.Name} อ่าน {docId}");
            return _realService.ReadDocument(docId);
        }

        public void WriteDocument(string docId, string content)
        {
            // ต้องเป็น Manager ขึ้นไปถึงจะเขียนได้
            if (!CanAccess(docId, UserRole.Manager))
            {
                Console.WriteLine($"[SecureProxy] ปฏิเสธ: {_currentUser.Name} ({_currentUser.Role}) ไม่มีสิทธิ์เขียน {docId}");
                return;
            }

            Console.WriteLine($"[SecureProxy] อนุญาต: {_currentUser.Name} เขียน {docId}");
            _realService.WriteDocument(docId, content);
        }

        public void DeleteDocument(string docId)
        {
            // ต้องเป็น Admin เท่านั้นถึงจะลบได้
            if (_currentUser.Role < UserRole.Admin)
            {
                Console.WriteLine($"[SecureProxy] ปฏิเสธ: {_currentUser.Name} ({_currentUser.Role}) ไม่มีสิทธิ์ลบเอกสาร");
                return;
            }

            Console.WriteLine($"[SecureProxy] อนุญาต: {_currentUser.Name} ลบ {docId}");
            _realService.DeleteDocument(docId);
        }

        private bool CanAccess(string docId, UserRole requiredRole)
        {
            // ตรวจสอบ role ทั่วไปก่อน
            if (_currentUser.Role < requiredRole)
                return false;

            // ตรวจสอบ document-specific requirements
            if (_sensitiveDocRequirements.TryGetValue(docId, out UserRole docRole))
            {
                return _currentUser.Role >= docRole;
            }

            return true;
        }
    }

    class Program475
    {
        static void Main475()
        {
            Console.WriteLine("=== Proxy Pattern Demo ===\n");

            // === Virtual Proxy Demo ===
            Console.WriteLine("--- Virtual Proxy: Lazy Loading Images ---");
            
            // สร้าง gallery ด้วย lazy proxies - ยังไม่โหลดรูปภาพ
            var gallery = new List<IImage>
            {
                new LazyImageProxy("photo1.jpg"),
                new LazyImageProxy("photo2.jpg"),
                new LazyImageProxy("photo3.jpg")
            };

            Console.WriteLine("\nสร้าง gallery แล้ว (ยังไม่โหลดรูปภาพ)");
            Console.WriteLine("กดดูเฉพาะรูปที่ 1...");
            gallery[0].Display(); // โหลดเฉพาะรูปนี้

            Console.WriteLine("\nดูรูปที่ 1 อีกครั้ง...");
            gallery[0].Display(); // ใช้ภาพที่โหลดแล้ว ไม่โหลดซ้ำ

            Console.WriteLine("\nรูปที่ 2 และ 3 ยังไม่ถูกโหลด (ประหยัด memory)");

            Console.WriteLine();

            // === Protection Proxy Demo ===
            Console.WriteLine("--- Protection Proxy: Access Control ---");
            
            var docService = new DocumentService();

            // Employee พยายามเข้าถึงเอกสาร
            var employee = new User { Name = "พนักงาน ก.", Role = UserRole.Employee };
            var employeeProxy = new SecureDocumentProxy(docService, employee);
            
            Console.WriteLine($"\nการเข้าถึงในฐานะ {employee.Name} ({employee.Role}):");
            employeeProxy.ReadDocument("doc:1");    // อ่านได้
            employeeProxy.ReadDocument("doc:3");    // ไม่ได้ - ต้อง Manager
            employeeProxy.WriteDocument("doc:1", "แก้ไข");  // ไม่ได้ - ต้อง Manager
            employeeProxy.DeleteDocument("doc:1");  // ไม่ได้ - ต้อง Admin

            Console.WriteLine();

            // Manager เข้าถึง
            var manager = new User { Name = "ผู้จัดการ ข.", Role = UserRole.Manager };
            var managerProxy = new SecureDocumentProxy(docService, manager);
            
            Console.WriteLine($"\nการเข้าถึงในฐานะ {manager.Name} ({manager.Role}):");
            managerProxy.ReadDocument("doc:3");     // อ่านได้ - เป็น Manager
            managerProxy.WriteDocument("doc:1", "อัพเดทนโยบาย");  // เขียนได้
            managerProxy.DeleteDocument("doc:1");   // ไม่ได้ - ต้อง Admin

            Console.WriteLine("\n✓ Proxy Pattern ควบคุม access และ lazy loading ได้อย่างมีประสิทธิภาพ");
        }
    }
}
```

---

## ขั้นตอนที่ 476: Composite Pattern

### คืออะไร?

Composite Pattern ช่วยจัดการกลุ่มของออบเจกต์แบบ tree structure โดยทำให้ client ใช้งาน single object และ group ได้เหมือนกัน เหมาะสำหรับ hierarchical structures เช่น File System, UI Components, Organization Chart

### ตัวอย่าง: FileSystem (File/Folder) และ UI Component Tree

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;

namespace CompositePattern
{
    // ============================================================
    // ส่วนที่ 1: FileSystem Composite
    // ============================================================

    // === Component Interface: ทั้ง File และ Folder implement ===
    public abstract class FileSystemItem
    {
        public string Name { get; protected set; }
        protected string _path;

        protected FileSystemItem(string name, string path)
        {
            Name = name;
            _path = path;
        }

        public abstract long GetSize(); // ขนาดไฟล์หรือโฟลเดอร์
        public abstract void Display(int depth = 0);
        public abstract int GetItemCount(); // จำนวนไฟล์/โฟลเดอร์

        protected string GetIndent(int depth) => new string(' ', depth * 4);
    }

    // === Leaf: ไฟล์เดี่ยวๆ ===
    public class FileItem : FileSystemItem
    {
        private readonly long _sizeBytes;
        private readonly string _extension;

        public FileItem(string name, string path, long sizeBytes)
            : base(name, path)
        {
            _sizeBytes = sizeBytes;
            _extension = System.IO.Path.GetExtension(name).ToLower();
        }

        public override long GetSize() => _sizeBytes;
        public override int GetItemCount() => 1;

        public override void Display(int depth = 0)
        {
            string icon = GetIcon();
            Console.WriteLine($"{GetIndent(depth)}{icon} {Name} ({FormatSize(_sizeBytes)})");
        }

        private string GetIcon() => _extension switch
        {
            ".cs" => "📄",
            ".jpg" or ".png" or ".gif" => "🖼",
            ".mp4" or ".avi" => "🎬",
            ".mp3" or ".wav" => "🎵",
            ".pdf" => "📕",
            ".zip" or ".rar" => "📦",
            _ => "📄"
        };

        private string FormatSize(long bytes)
        {
            if (bytes < 1024) return $"{bytes} B";
            if (bytes < 1024 * 1024) return $"{bytes / 1024.0:F1} KB";
            if (bytes < 1024 * 1024 * 1024) return $"{bytes / 1024.0 / 1024.0:F1} MB";
            return $"{bytes / 1024.0 / 1024.0 / 1024.0:F1} GB";
        }
    }

    // === Composite: โฟลเดอร์ที่มี children ===
    public class FolderItem : FileSystemItem
    {
        private readonly List<FileSystemItem> _children = new List<FileSystemItem>();

        public FolderItem(string name, string path) : base(name, path) { }

        public void Add(FileSystemItem item) => _children.Add(item);
        public void Remove(FileSystemItem item) => _children.Remove(item);

        public override long GetSize() => _children.Sum(c => c.GetSize());
        public override int GetItemCount() => _children.Sum(c => c.GetItemCount());

        public override void Display(int depth = 0)
        {
            long totalSize = GetSize();
            int count = GetItemCount();
            Console.WriteLine($"{GetIndent(depth)}📁 {Name}/ ({FormatSize(totalSize)}, {count} รายการ)");
            
            // แสดง children
            foreach (var child in _children.OrderBy(c => c is FolderItem ? 0 : 1).ThenBy(c => c.Name))
            {
                child.Display(depth + 1);
            }
        }

        // ค้นหาไฟล์ใน tree
        public List<FileSystemItem> Search(string pattern)
        {
            var results = new List<FileSystemItem>();
            
            foreach (var child in _children)
            {
                if (child.Name.Contains(pattern, StringComparison.OrdinalIgnoreCase))
                    results.Add(child);
                
                if (child is FolderItem folder)
                    results.AddRange(folder.Search(pattern));
            }
            
            return results;
        }

        private string FormatSize(long bytes)
        {
            if (bytes < 1024) return $"{bytes} B";
            if (bytes < 1024 * 1024) return $"{bytes / 1024.0:F1} KB";
            if (bytes < 1024 * 1024 * 1024) return $"{bytes / 1024.0 / 1024.0:F1} MB";
            return $"{bytes / 1024.0 / 1024.0 / 1024.0:F1} GB";
        }
    }

    // ============================================================
    // ส่วนที่ 2: UI Component Tree
    // ============================================================

    // === Abstract UI Component ===
    public abstract class UIComponent
    {
        public string Id { get; protected set; }
        public string Name { get; protected set; }
        public bool IsVisible { get; set; } = true;
        public bool IsEnabled { get; set; } = true;

        protected UIComponent(string id, string name)
        {
            Id = id;
            Name = name;
        }

        public abstract void Render(int depth = 0);
        public abstract void SetEnabled(bool enabled);

        protected string GetIndent(int depth) => new string(' ', depth * 2);
    }

    // === Leaf UI Components ===
    public class Button : UIComponent
    {
        private string _text;
        private string _color;

        public Button(string id, string text, string color = "Blue")
            : base(id, $"Button[{text}]")
        {
            _text = text;
            _color = color;
        }

        public override void Render(int depth = 0)
        {
            if (!IsVisible) return;
            string state = IsEnabled ? "enabled" : "disabled";
            Console.WriteLine($"{GetIndent(depth)}🔘 <Button id='{Id}' text='{_text}' color='{_color}' {state} />");
        }

        public override void SetEnabled(bool enabled) => IsEnabled = enabled;
    }

    public class TextBox : UIComponent
    {
        private string _placeholder;

        public TextBox(string id, string placeholder) : base(id, $"TextBox[{placeholder}]")
        {
            _placeholder = placeholder;
        }

        public override void Render(int depth = 0)
        {
            if (!IsVisible) return;
            string state = IsEnabled ? "" : " disabled";
            Console.WriteLine($"{GetIndent(depth)}📝 <TextBox id='{Id}' placeholder='{_placeholder}'{state} />");
        }

        public override void SetEnabled(bool enabled) => IsEnabled = enabled;
    }

    public class Label : UIComponent
    {
        private string _text;

        public Label(string id, string text) : base(id, $"Label[{text}]")
        {
            _text = text;
        }

        public override void Render(int depth = 0)
        {
            if (!IsVisible) return;
            Console.WriteLine($"{GetIndent(depth)}🏷 <Label id='{Id}'>{_text}</Label>");
        }

        public override void SetEnabled(bool enabled) => IsEnabled = enabled;
    }

    // === Composite UI Container ===
    public class Panel : UIComponent
    {
        private readonly List<UIComponent> _children = new List<UIComponent>();
        private string _layout;

        public Panel(string id, string name, string layout = "Vertical")
            : base(id, name)
        {
            _layout = layout;
        }

        public void Add(UIComponent component) => _children.Add(component);
        public void Remove(UIComponent component) => _children.Remove(component);

        public UIComponent FindById(string id)
        {
            if (Id == id) return this;
            
            foreach (var child in _children)
            {
                if (child.Id == id) return child;
                if (child is Panel panel)
                {
                    var found = panel.FindById(id);
                    if (found != null) return found;
                }
            }
            return null;
        }

        public override void Render(int depth = 0)
        {
            if (!IsVisible) return;
            
            Console.WriteLine($"{GetIndent(depth)}📦 <Panel id='{Id}' name='{Name}' layout='{_layout}'>");
            
            foreach (var child in _children)
                child.Render(depth + 1);
            
            Console.WriteLine($"{GetIndent(depth)}</Panel>");
        }

        // ส่ง SetEnabled ไปยัง children ทั้งหมดด้วย
        public override void SetEnabled(bool enabled)
        {
            IsEnabled = enabled;
            foreach (var child in _children)
                child.SetEnabled(enabled);
        }
    }

    class Program476
    {
        static void Main476()
        {
            Console.WriteLine("=== Composite Pattern Demo ===\n");

            // === FileSystem Composite ===
            Console.WriteLine("--- FileSystem Tree ---");
            
            // สร้าง structure แบบ tree
            var root = new FolderItem("MyProject", "/");
            
            var src = new FolderItem("src", "/src");
            src.Add(new FileItem("Program.cs", "/src/Program.cs", 15360));
            src.Add(new FileItem("Startup.cs", "/src/Startup.cs", 8192));
            
            var models = new FolderItem("Models", "/src/Models");
            models.Add(new FileItem("User.cs", "/src/Models/User.cs", 4096));
            models.Add(new FileItem("Product.cs", "/src/Models/Product.cs", 6144));
            models.Add(new FileItem("Order.cs", "/src/Models/Order.cs", 5120));
            src.Add(models);
            
            var services = new FolderItem("Services", "/src/Services");
            services.Add(new FileItem("UserService.cs", "/src/Services/UserService.cs", 12288));
            services.Add(new FileItem("OrderService.cs", "/src/Services/OrderService.cs", 9216));
            src.Add(services);
            
            root.Add(src);
            
            var docs = new FolderItem("docs", "/docs");
            docs.Add(new FileItem("README.md", "/docs/README.md", 2048));
            docs.Add(new FileItem("API.pdf", "/docs/API.pdf", 512000));
            root.Add(docs);
            
            root.Add(new FileItem(".gitignore", "/.gitignore", 512));
            root.Add(new FileItem("README.md", "/README.md", 3072));

            root.Display();
            Console.WriteLine($"\nรวมทั้งหมด: {root.GetItemCount()} รายการ, ขนาด: {root.GetSize() / 1024.0:F1} KB");

            Console.WriteLine("\nค้นหาไฟล์ที่มีชื่อว่า 'Service':");
            var searchResults = root.Search("Service");
            foreach (var item in searchResults)
                Console.WriteLine($"  พบ: {item.Name}");

            Console.WriteLine();

            // === UI Component Tree ===
            Console.WriteLine("--- UI Component Tree ---");
            
            var loginForm = new Panel("loginForm", "Login Form", "Vertical");
            
            var headerPanel = new Panel("headerPanel", "Header", "Horizontal");
            headerPanel.Add(new Label("titleLabel", "เข้าสู่ระบบ"));
            loginForm.Add(headerPanel);
            
            var inputPanel = new Panel("inputPanel", "Input Fields", "Vertical");
            inputPanel.Add(new Label("userLabel", "ชื่อผู้ใช้:"));
            inputPanel.Add(new TextBox("usernameInput", "กรอกชื่อผู้ใช้"));
            inputPanel.Add(new Label("passLabel", "รหัสผ่าน:"));
            inputPanel.Add(new TextBox("passwordInput", "กรอกรหัสผ่าน"));
            loginForm.Add(inputPanel);
            
            var buttonPanel = new Panel("buttonPanel", "Buttons", "Horizontal");
            buttonPanel.Add(new Button("loginBtn", "เข้าสู่ระบบ", "Blue"));
            buttonPanel.Add(new Button("cancelBtn", "ยกเลิก", "Gray"));
            loginForm.Add(buttonPanel);
            
            loginForm.Render();

            Console.WriteLine("\nหลังจาก disable form ทั้งหมด:");
            loginForm.SetEnabled(false);
            loginForm.Render();

            Console.WriteLine("\n✓ Composite Pattern จัดการ Tree Structure ได้อย่างสม่ำเสมอ");
        }
    }
}
```

---

## ขั้นตอนที่ 477: Bridge Pattern

### คืออะไร?

Bridge Pattern แยก abstraction (สิ่งที่ทำ) ออกจาก implementation (วิธีทำ) เพื่อให้ทั้งสองสามารถเปลี่ยนแปลงได้อย่างอิสระ ป้องกัน "Inheritance Explosion" ที่เกิดจากการ subclass หลายมิติ

### ตัวอย่าง: Shape + Renderer Bridge

```csharp
using System;
using System.Collections.Generic;

namespace BridgePattern
{
    // ============================================================
    // Bridge Pattern: Shape/Renderer
    // ============================================================

    // === Implementation Interface: วิธีการ render ===
    public interface IRenderer
    {
        void RenderCircle(double x, double y, double radius);
        void RenderRectangle(double x, double y, double width, double height);
        void RenderTriangle(double x1, double y1, double x2, double y2, double x3, double y3);
        void SetColor(string color);
        void SetOpacity(double opacity);
        string GetRendererName();
    }

    // === Concrete Implementations ===

    // Vector Renderer: สำหรับ SVG/Vector output
    public class VectorRenderer : IRenderer
    {
        private string _currentColor = "black";
        private double _opacity = 1.0;

        public void RenderCircle(double x, double y, double radius)
        {
            Console.WriteLine($"[SVG] <circle cx='{x}' cy='{y}' r='{radius}' fill='{_currentColor}' opacity='{_opacity}' />");
        }

        public void RenderRectangle(double x, double y, double width, double height)
        {
            Console.WriteLine($"[SVG] <rect x='{x}' y='{y}' width='{width}' height='{height}' fill='{_currentColor}' opacity='{_opacity}' />");
        }

        public void RenderTriangle(double x1, double y1, double x2, double y2, double x3, double y3)
        {
            Console.WriteLine($"[SVG] <polygon points='{x1},{y1} {x2},{y2} {x3},{y3}' fill='{_currentColor}' opacity='{_opacity}' />");
        }

        public void SetColor(string color) => _currentColor = color;
        public void SetOpacity(double opacity) => _opacity = opacity;
        public string GetRendererName() => "SVG Vector Renderer";
    }

    // Raster Renderer: สำหรับ Pixel/Bitmap output
    public class RasterRenderer : IRenderer
    {
        private string _currentColor = "black";
        private double _opacity = 1.0;

        public void RenderCircle(double x, double y, double radius)
        {
            Console.WriteLine($"[Canvas] drawCircle(center=({x},{y}), radius={radius}, color={_currentColor}, alpha={_opacity})");
        }

        public void RenderRectangle(double x, double y, double width, double height)
        {
            Console.WriteLine($"[Canvas] fillRect(x={x}, y={y}, w={width}, h={height}, color={_currentColor}, alpha={_opacity})");
        }

        public void RenderTriangle(double x1, double y1, double x2, double y2, double x3, double y3)
        {
            Console.WriteLine($"[Canvas] drawTriangle(({x1},{y1})->({x2},{y2})->({x3},{y3}), color={_currentColor}, alpha={_opacity})");
        }

        public void SetColor(string color) => _currentColor = color;
        public void SetOpacity(double opacity) => _opacity = opacity;
        public string GetRendererName() => "HTML5 Canvas Raster Renderer";
    }

    // Console Renderer: สำหรับ debug/text output
    public class ConsoleRenderer : IRenderer
    {
        private string _currentColor = "white";
        private double _opacity = 1.0;

        public void RenderCircle(double x, double y, double radius)
        {
            Console.WriteLine($"[Console] ○ วงกลม: ศูนย์กลาง({x},{y}) รัศมี={radius} สี={_currentColor}");
        }

        public void RenderRectangle(double x, double y, double width, double height)
        {
            Console.WriteLine($"[Console] □ สี่เหลี่ยม: ({x},{y}) กว้าง={width} สูง={height} สี={_currentColor}");
        }

        public void RenderTriangle(double x1, double y1, double x2, double y2, double x3, double y3)
        {
            Console.WriteLine($"[Console] △ สามเหลี่ยม: ({x1},{y1})-({x2},{y2})-({x3},{y3}) สี={_currentColor}");
        }

        public void SetColor(string color) => _currentColor = color;
        public void SetOpacity(double opacity) => _opacity = opacity;
        public string GetRendererName() => "Console Text Renderer";
    }

    // === Abstraction: Shape ===
    public abstract class Shape
    {
        // Bridge: เก็บ reference ไปยัง IRenderer
        protected IRenderer Renderer;
        public string Color { get; set; } = "black";
        public double Opacity { get; set; } = 1.0;

        protected Shape(IRenderer renderer)
        {
            Renderer = renderer;
        }

        // เปลี่ยน renderer ได้แบบ dynamic
        public void SwitchRenderer(IRenderer newRenderer)
        {
            Console.WriteLine($"เปลี่ยน renderer จาก {Renderer.GetRendererName()} เป็น {newRenderer.GetRendererName()}");
            Renderer = newRenderer;
        }

        public abstract void Draw();
        public abstract void Resize(double factor);
        public abstract double GetArea();
    }

    // === Refined Abstractions ===
    public class Circle : Shape
    {
        public double X { get; set; }
        public double Y { get; set; }
        public double Radius { get; set; }

        public Circle(double x, double y, double radius, IRenderer renderer)
            : base(renderer)
        {
            X = x; Y = y; Radius = radius;
        }

        public override void Draw()
        {
            Renderer.SetColor(Color);
            Renderer.SetOpacity(Opacity);
            Renderer.RenderCircle(X, Y, Radius);
        }

        public override void Resize(double factor) => Radius *= factor;
        public override double GetArea() => Math.PI * Radius * Radius;
    }

    public class Rectangle : Shape
    {
        public double X { get; set; }
        public double Y { get; set; }
        public double Width { get; set; }
        public double Height { get; set; }

        public Rectangle(double x, double y, double width, double height, IRenderer renderer)
            : base(renderer)
        {
            X = x; Y = y; Width = width; Height = height;
        }

        public override void Draw()
        {
            Renderer.SetColor(Color);
            Renderer.SetOpacity(Opacity);
            Renderer.RenderRectangle(X, Y, Width, Height);
        }

        public override void Resize(double factor) { Width *= factor; Height *= factor; }
        public override double GetArea() => Width * Height;
    }

    public class Triangle : Shape
    {
        private double _x1, _y1, _x2, _y2, _x3, _y3;

        public Triangle(double x1, double y1, double x2, double y2, double x3, double y3, IRenderer renderer)
            : base(renderer)
        {
            _x1 = x1; _y1 = y1; _x2 = x2; _y2 = y2; _x3 = x3; _y3 = y3;
        }

        public override void Draw()
        {
            Renderer.SetColor(Color);
            Renderer.SetOpacity(Opacity);
            Renderer.RenderTriangle(_x1, _y1, _x2, _y2, _x3, _y3);
        }

        public override void Resize(double factor)
        {
            _x1 *= factor; _y1 *= factor;
            _x2 *= factor; _y2 *= factor;
            _x3 *= factor; _y3 *= factor;
        }

        public override double GetArea()
        {
            return Math.Abs((_x1 * (_y2 - _y3) + _x2 * (_y3 - _y1) + _x3 * (_y1 - _y2)) / 2);
        }
    }

    class Program477
    {
        static void Main477()
        {
            Console.WriteLine("=== Bridge Pattern: Shape + Renderer ===\n");

            // สร้าง renderers
            IRenderer svgRenderer = new VectorRenderer();
            IRenderer canvasRenderer = new RasterRenderer();
            IRenderer consoleRenderer = new ConsoleRenderer();

            // วาดรูปด้วย SVG Renderer
            Console.WriteLine("--- วาดด้วย SVG Vector Renderer ---");
            var circle = new Circle(100, 100, 50, svgRenderer) { Color = "red" };
            var rect = new Rectangle(10, 10, 200, 150, svgRenderer) { Color = "blue", Opacity = 0.8 };
            var triangle = new Triangle(50, 0, 100, 100, 0, 100, svgRenderer) { Color = "green" };

            circle.Draw();
            rect.Draw();
            triangle.Draw();

            Console.WriteLine();
            Console.WriteLine("--- เปลี่ยนเป็น Console Renderer ---");
            // เปลี่ยน renderer โดยไม่ต้องสร้าง shape ใหม่
            circle.SwitchRenderer(consoleRenderer);
            rect.SwitchRenderer(consoleRenderer);
            triangle.SwitchRenderer(consoleRenderer);

            circle.Draw();
            rect.Draw();
            triangle.Draw();

            Console.WriteLine();
            Console.WriteLine("--- เปลี่ยนเป็น Canvas Renderer และ Resize ---");
            circle.SwitchRenderer(canvasRenderer);
            circle.Resize(2.0); // ขยายเป็น 2 เท่า
            circle.Draw();
            Console.WriteLine($"พื้นที่วงกลม: {circle.GetArea():F2} px²");

            Console.WriteLine("\n✓ Bridge Pattern ช่วยให้เปลี่ยน Renderer ได้โดยไม่กระทบ Shape");
        }
    }
}
```

---

## ขั้นตอนที่ 478-480: ตัวอย่างรวม - Plugin System

ในส่วนนี้เราจะสร้างระบบ Plugin ที่ใช้ Structural Patterns หลายตัวรวมกัน:
- **Adapter**: แปลง third-party plugins ให้ใช้ interface ของเรา
- **Decorator**: เพิ่ม logging, validation, authorization ให้ plugins
- **Facade**: สร้าง API เรียบง่ายสำหรับจัดการ plugins

```csharp
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Linq;

namespace PluginSystem
{
    // ============================================================
    // ขั้นตอนที่ 478: Core Plugin Infrastructure
    // ============================================================

    // === Plugin Interface ที่ระบบของเราต้องการ ===
    public interface IPlugin
    {
        string Name { get; }
        string Version { get; }
        string Description { get; }
        bool IsInitialized { get; }
        
        void Initialize(IPluginContext context);
        object Execute(string command, Dictionary<string, object> parameters);
        void Shutdown();
    }

    // === Plugin Context: ข้อมูล context สำหรับ plugin ===
    public interface IPluginContext
    {
        string AppVersion { get; }
        string Environment { get; }
        IPluginLogger Logger { get; }
        T GetService<T>() where T : class;
    }

    // === Plugin Logger ===
    public interface IPluginLogger
    {
        void Info(string message);
        void Warning(string message);
        void Error(string message, Exception ex = null);
    }

    // === Implementation ของ Context และ Logger ===
    public class PluginLogger : IPluginLogger
    {
        private readonly string _pluginName;

        public PluginLogger(string pluginName) => _pluginName = pluginName;

        public void Info(string message) => Console.WriteLine($"  [INFO:{_pluginName}] {message}");
        public void Warning(string message) 
        {
            Console.ForegroundColor = ConsoleColor.Yellow;
            Console.WriteLine($"  [WARN:{_pluginName}] {message}");
            Console.ResetColor();
        }
        public void Error(string message, Exception ex = null)
        {
            Console.ForegroundColor = ConsoleColor.Red;
            Console.WriteLine($"  [ERROR:{_pluginName}] {message}");
            if (ex != null) Console.WriteLine($"    Exception: {ex.Message}");
            Console.ResetColor();
        }
    }

    public class PluginContext : IPluginContext
    {
        private readonly Dictionary<Type, object> _services = new Dictionary<Type, object>();
        
        public string AppVersion { get; }
        public string Environment { get; }
        public IPluginLogger Logger { get; }

        public PluginContext(string appVersion, string environment, string pluginName)
        {
            AppVersion = appVersion;
            Environment = environment;
            Logger = new PluginLogger(pluginName);
        }

        public void RegisterService<T>(T service) where T : class
        {
            _services[typeof(T)] = service;
        }

        public T GetService<T>() where T : class
        {
            if (_services.TryGetValue(typeof(T), out object service))
                return service as T;
            return null;
        }
    }

    // ============================================================
    // ขั้นตอนที่ 479: Adapter - แปลง Third-Party Plugins
    // ============================================================

    // === Third-party Plugin A (ไม่ compatible กับ IPlugin ของเรา) ===
    public class ThirdPartyImagePlugin
    {
        public string PluginName => "ImageProcessor v2.1";
        
        public void Setup(string config)
        {
            Console.WriteLine($"  [ThirdParty-Image] Setup ด้วย config: {config}");
        }

        public string ProcessImage(string imagePath, string operation, int quality)
        {
            Console.WriteLine($"  [ThirdParty-Image] ประมวลผล {imagePath} ด้วย {operation} quality={quality}");
            return $"processed_{imagePath}";
        }

        public void Cleanup()
        {
            Console.WriteLine("  [ThirdParty-Image] Cleanup");
        }
    }

    // === Third-party Plugin B (อีกแบบที่ไม่ compatible) ===
    public class ThirdPartyEmailPlugin
    {
        public string GetName() => "EmailSender Pro";
        public string GetVersion() => "3.0.1";
        
        public bool Connect(string smtpServer, int port)
        {
            Console.WriteLine($"  [ThirdParty-Email] เชื่อมต่อ {smtpServer}:{port}");
            return true;
        }

        public bool SendEmail(string to, string subject, string body)
        {
            Console.WriteLine($"  [ThirdParty-Email] ส่ง email ถึง {to}: {subject}");
            return true;
        }

        public void Disconnect()
        {
            Console.WriteLine("  [ThirdParty-Email] ตัดการเชื่อมต่อ");
        }
    }

    // === Adapter สำหรับ ThirdPartyImagePlugin ===
    public class ImagePluginAdapter : IPlugin
    {
        private readonly ThirdPartyImagePlugin _imagePlugin;
        private bool _initialized = false;

        public string Name => "Image Processor Plugin";
        public string Version => "2.1";
        public string Description => "ประมวลผลรูปภาพ (adapted from ThirdParty)";
        public bool IsInitialized => _initialized;

        public ImagePluginAdapter(ThirdPartyImagePlugin imagePlugin)
        {
            _imagePlugin = imagePlugin;
        }

        public void Initialize(IPluginContext context)
        {
            context.Logger.Info("กำลัง initialize Image Plugin...");
            _imagePlugin.Setup($"env={context.Environment}");
            _initialized = true;
        }

        public object Execute(string command, Dictionary<string, object> parameters)
        {
            if (!_initialized) throw new InvalidOperationException("Plugin ยังไม่ได้ initialize");

            return command switch
            {
                "compress" => _imagePlugin.ProcessImage(
                    parameters.GetValueOrDefault("path", "/unknown").ToString(),
                    "compress",
                    int.Parse(parameters.GetValueOrDefault("quality", "80").ToString())
                ),
                "resize" => _imagePlugin.ProcessImage(
                    parameters.GetValueOrDefault("path", "/unknown").ToString(),
                    "resize",
                    100
                ),
                _ => throw new ArgumentException($"ไม่รู้จัก command: {command}")
            };
        }

        public void Shutdown()
        {
            _imagePlugin.Cleanup();
            _initialized = false;
        }
    }

    // === Adapter สำหรับ ThirdPartyEmailPlugin ===
    public class EmailPluginAdapter : IPlugin
    {
        private readonly ThirdPartyEmailPlugin _emailPlugin;
        private bool _initialized = false;

        public string Name => "Email Sender Plugin";
        public string Version => "3.0.1";
        public string Description => "ส่ง email (adapted from ThirdParty)";
        public bool IsInitialized => _initialized;

        public EmailPluginAdapter(ThirdPartyEmailPlugin emailPlugin)
        {
            _emailPlugin = emailPlugin;
        }

        public void Initialize(IPluginContext context)
        {
            context.Logger.Info("กำลัง initialize Email Plugin...");
            _emailPlugin.Connect("smtp.example.com", 587);
            _initialized = true;
        }

        public object Execute(string command, Dictionary<string, object> parameters)
        {
            if (!_initialized) throw new InvalidOperationException("Plugin ยังไม่ได้ initialize");

            if (command == "send")
            {
                return _emailPlugin.SendEmail(
                    parameters.GetValueOrDefault("to", "").ToString(),
                    parameters.GetValueOrDefault("subject", "").ToString(),
                    parameters.GetValueOrDefault("body", "").ToString()
                );
            }

            throw new ArgumentException($"ไม่รู้จัก command: {command}");
        }

        public void Shutdown()
        {
            _emailPlugin.Disconnect();
            _initialized = false;
        }
    }

    // === Built-in Report Plugin (ใช้ IPlugin โดยตรง) ===
    public class ReportPlugin : IPlugin
    {
        public string Name => "Report Generator";
        public string Version => "1.5";
        public string Description => "สร้างรายงานในรูปแบบต่างๆ";
        public bool IsInitialized { get; private set; }

        private IPluginContext _context;

        public void Initialize(IPluginContext context)
        {
            _context = context;
            context.Logger.Info("Report Plugin พร้อมใช้งาน");
            IsInitialized = true;
        }

        public object Execute(string command, Dictionary<string, object> parameters)
        {
            string format = parameters.GetValueOrDefault("format", "pdf").ToString();
            string title = parameters.GetValueOrDefault("title", "รายงาน").ToString();

            _context.Logger.Info($"สร้างรายงาน '{title}' ในรูปแบบ {format}");
            return $"report_{title.Replace(" ", "_")}.{format}";
        }

        public void Shutdown()
        {
            _context.Logger.Info("Report Plugin ปิดการทำงาน");
            IsInitialized = false;
        }
    }

    // === Decorator: เพิ่ม Logging ให้ทุก Plugin ===
    public class LoggingPluginDecorator : IPlugin
    {
        private readonly IPlugin _innerPlugin;
        private readonly IPluginLogger _logger;

        public string Name => _innerPlugin.Name;
        public string Version => _innerPlugin.Version;
        public string Description => _innerPlugin.Description + " [+Logging]";
        public bool IsInitialized => _innerPlugin.IsInitialized;

        public LoggingPluginDecorator(IPlugin plugin, IPluginLogger logger)
        {
            _innerPlugin = plugin;
            _logger = logger;
        }

        public void Initialize(IPluginContext context)
        {
            _logger.Info($"เริ่ม Initialize plugin: {Name}");
            var stopwatch = Stopwatch.StartNew();
            _innerPlugin.Initialize(context);
            stopwatch.Stop();
            _logger.Info($"Initialize สำเร็จ ใช้เวลา {stopwatch.ElapsedMilliseconds}ms");
        }

        public object Execute(string command, Dictionary<string, object> parameters)
        {
            _logger.Info($"Execute: {Name}.{command}()");
            var stopwatch = Stopwatch.StartNew();
            
            try
            {
                var result = _innerPlugin.Execute(command, parameters);
                stopwatch.Stop();
                _logger.Info($"Execute สำเร็จ ใช้เวลา {stopwatch.ElapsedMilliseconds}ms, result={result}");
                return result;
            }
            catch (Exception ex)
            {
                stopwatch.Stop();
                _logger.Error($"Execute ล้มเหลว ใช้เวลา {stopwatch.ElapsedMilliseconds}ms", ex);
                throw;
            }
        }

        public void Shutdown()
        {
            _logger.Info($"กำลัง Shutdown plugin: {Name}");
            _innerPlugin.Shutdown();
            _logger.Info($"Shutdown สำเร็จ");
        }
    }

    // === Decorator: เพิ่ม Authorization ให้ทุก Plugin ===
    public class AuthorizedPluginDecorator : IPlugin
    {
        private readonly IPlugin _innerPlugin;
        private readonly HashSet<string> _allowedCommands;
        private readonly string _requiredPermission;

        public string Name => _innerPlugin.Name;
        public string Version => _innerPlugin.Version;
        public string Description => _innerPlugin.Description + " [+Auth]";
        public bool IsInitialized => _innerPlugin.IsInitialized;

        private bool _isAuthorized = false;

        public AuthorizedPluginDecorator(IPlugin plugin, string permission, params string[] allowedCommands)
        {
            _innerPlugin = plugin;
            _requiredPermission = permission;
            _allowedCommands = new HashSet<string>(allowedCommands, StringComparer.OrdinalIgnoreCase);
        }

        public void Authorize()
        {
            Console.WriteLine($"  [Auth] ตรวจสอบสิทธิ์ '{_requiredPermission}' - อนุญาต");
            _isAuthorized = true;
        }

        public void Initialize(IPluginContext context) => _innerPlugin.Initialize(context);

        public object Execute(string command, Dictionary<string, object> parameters)
        {
            if (!_isAuthorized)
                throw new UnauthorizedAccessException($"ยังไม่ได้ authorize สำหรับ '{_requiredPermission}'");

            if (!_allowedCommands.Contains(command))
                throw new UnauthorizedAccessException($"command '{command}' ไม่ได้รับอนุญาต");

            return _innerPlugin.Execute(command, parameters);
        }

        public void Shutdown() => _innerPlugin.Shutdown();
    }

    // ============================================================
    // ขั้นตอนที่ 480: Facade - Plugin Manager
    // ============================================================

    // === Plugin Manager Facade: จัดการ plugins ทั้งหมด ===
    public class PluginManager
    {
        private readonly Dictionary<string, IPlugin> _plugins = new Dictionary<string, IPlugin>();
        private readonly PluginContext _baseContext;
        private readonly IPluginLogger _managerLogger;

        public PluginManager(string appVersion, string environment)
        {
            _managerLogger = new PluginLogger("PluginManager");
            _baseContext = new PluginContext(appVersion, environment, "System");
        }

        // === ลงทะเบียน plugin (พร้อม decorators อัตโนมัติ) ===
        public void RegisterPlugin(IPlugin plugin, bool withLogging = true)
        {
            IPlugin finalPlugin = plugin;

            // เพิ่ม Logging Decorator อัตโนมัติ
            if (withLogging)
            {
                var pluginLogger = new PluginLogger(plugin.Name);
                finalPlugin = new LoggingPluginDecorator(finalPlugin, pluginLogger);
            }

            _plugins[plugin.Name] = finalPlugin;
            _managerLogger.Info($"ลงทะเบียน plugin '{plugin.Name}' v{plugin.Version}");
        }

        // === Initialize plugins ทั้งหมด ===
        public void InitializeAll()
        {
            Console.WriteLine("\n[PluginManager] กำลัง initialize plugins ทั้งหมด...");
            
            foreach (var (name, plugin) in _plugins)
            {
                try
                {
                    var context = new PluginContext(_baseContext.AppVersion, _baseContext.Environment, name);
                    plugin.Initialize(context);
                }
                catch (Exception ex)
                {
                    _managerLogger.Error($"ไม่สามารถ initialize '{name}'", ex);
                }
            }
        }

        // === รัน command บน plugin ===
        public object RunPlugin(string pluginName, string command, Dictionary<string, object> parameters = null)
        {
            if (!_plugins.TryGetValue(pluginName, out IPlugin plugin))
            {
                _managerLogger.Warning($"ไม่พบ plugin '{pluginName}'");
                return null;
            }

            if (!plugin.IsInitialized)
            {
                _managerLogger.Warning($"Plugin '{pluginName}' ยังไม่ได้ initialize");
                return null;
            }

            return plugin.Execute(command, parameters ?? new Dictionary<string, object>());
        }

        // === Shutdown plugins ทั้งหมด ===
        public void ShutdownAll()
        {
            Console.WriteLine("\n[PluginManager] กำลัง shutdown plugins ทั้งหมด...");
            
            foreach (var (name, plugin) in _plugins)
            {
                try
                {
                    plugin.Shutdown();
                }
                catch (Exception ex)
                {
                    _managerLogger.Error($"เกิดข้อผิดพลาดระหว่าง shutdown '{name}'", ex);
                }
            }
        }

        // === แสดงรายการ plugins ที่ลงทะเบียนแล้ว ===
        public void ListPlugins()
        {
            Console.WriteLine("\n[PluginManager] รายการ Plugins ที่ลงทะเบียน:");
            Console.WriteLine($"  {'ชื่อ',-30} {'Version',-10} {'สถานะ',-12} คำอธิบาย");
            Console.WriteLine(new string('-', 80));

            foreach (var (name, plugin) in _plugins)
            {
                string status = plugin.IsInitialized ? "✓ พร้อม" : "✗ ยังไม่พร้อม";
                Console.WriteLine($"  {plugin.Name,-30} {plugin.Version,-10} {status,-12} {plugin.Description}");
            }
        }
    }

    class Program478_480
    {
        static void Main478_480()
        {
            Console.WriteLine("=== Plugin System: Adapter + Decorator + Facade ===\n");

            // === สร้าง Plugin Manager (Facade) ===
            var manager = new PluginManager("2.0.0", "Production");

            // === สร้าง Plugins โดยใช้ Adapter ===
            Console.WriteLine("--- ลงทะเบียน Plugins ---");

            // Third-party plugins ผ่าน Adapter
            var imagePlugin = new ImagePluginAdapter(new ThirdPartyImagePlugin());
            var emailPlugin = new EmailPluginAdapter(new ThirdPartyEmailPlugin());
            
            // Built-in plugin
            var reportPlugin = new ReportPlugin();

            // ลงทะเบียนผ่าน Facade (จะเพิ่ม Logging Decorator อัตโนมัติ)
            manager.RegisterPlugin(imagePlugin);
            manager.RegisterPlugin(emailPlugin);
            manager.RegisterPlugin(reportPlugin);

            // === Initialize ทั้งหมด ===
            manager.InitializeAll();

            // === แสดงรายการ ===
            manager.ListPlugins();

            // === รัน Commands ===
            Console.WriteLine("\n--- รัน Plugin Commands ---");

            // ใช้ Image Plugin
            Console.WriteLine("\n[1] ประมวลผลรูปภาพ:");
            manager.RunPlugin("Image Processor Plugin", "compress", new Dictionary<string, object>
            {
                { "path", "/uploads/photo.jpg" },
                { "quality", "85" }
            });

            // ใช้ Email Plugin
            Console.WriteLine("\n[2] ส่ง Email:");
            manager.RunPlugin("Email Sender Plugin", "send", new Dictionary<string, object>
            {
                { "to", "customer@example.com" },
                { "subject", "ยืนยันการสั่งซื้อ" },
                { "body", "ขอบคุณสำหรับการสั่งซื้อ" }
            });

            // ใช้ Report Plugin
            Console.WriteLine("\n[3] สร้างรายงาน:");
            var reportFile = manager.RunPlugin("Report Generator", "generate", new Dictionary<string, object>
            {
                { "title", "รายงานยอดขาย เดือนตุลาคม" },
                { "format", "pdf" }
            });
            Console.WriteLine($"  ไฟล์รายงาน: {reportFile}");

            // === Shutdown ===
            manager.ShutdownAll();

            Console.WriteLine("\n=== สรุป: Plugin System ใช้ Structural Patterns ===");
            Console.WriteLine("  Adapter  → แปลง ThirdParty plugins ให้ compatible กับ IPlugin");
            Console.WriteLine("  Decorator → เพิ่ม Logging/Authorization โดยไม่แก้ไข Plugin เดิม");
            Console.WriteLine("  Facade   → PluginManager ซ่อน complexity ของ plugin lifecycle");
        }
    }
}
```

---

## สรุป Structural Design Patterns

### ตารางเปรียบเทียบ

| Pattern | ปัญหาที่แก้ | ใช้เมื่อ |
|---------|-------------|----------|
| **Adapter** | Interface ไม่ตรงกัน | ต้องใช้ legacy code หรือ third-party library ที่มี interface ต่างกัน |
| **Decorator** | ต้องการเพิ่มฟังก์ชันแบบ dynamic | ต้องการเพิ่ม logging, caching, retry โดยไม่แก้ไขคลาสเดิม |
| **Facade** | ระบบซับซ้อนเกินไป | ต้องการ API เรียบง่ายสำหรับ subsystem หลายส่วน |
| **Proxy** | ควบคุมการเข้าถึง | ต้องการ lazy loading, access control, หรือ caching |
| **Composite** | โครงสร้างแบบ tree | จัดการ hierarchical objects เช่น file system, UI tree |
| **Bridge** | Abstraction + Implementation เปลี่ยนแปลงอิสระ | หลีกเลี่ยง inheritance explosion จากหลายมิติ |

### หลักการสำคัญที่ได้เรียน

1. **Adapter**: ใช้ Adapter เมื่อต้องทำให้ interface ที่ไม่เข้ากันทำงานร่วมกัน ไม่ว่าจะเป็น Legacy Payment System หรือ Logging Libraries
2. **Decorator**: สามารถ stack decorators ได้หลายชั้น เช่น Database → Cache → Logging เพื่อเพิ่มฟังก์ชันได้อย่างยืดหยุ่น
3. **Facade**: ซ่อน complexity ไว้ข้างหลัง ให้ client เห็นแค่ interface เรียบง่าย เช่น HomeTheatre.WatchMovie() หรือ OrderFacade.PlaceOrder()
4. **Proxy**: มี 3 แบบหลัก - Virtual (lazy loading), Caching (ลด computation), Protection (access control)
5. **Composite**: ทำให้ single object และ group ใช้งานได้เหมือนกัน ผ่าน common interface
6. **Bridge**: แยก "สิ่งที่ทำ" (Shape) ออกจาก "วิธีทำ" (Renderer) ให้เปลี่ยนแปลงได้อิสระ

### Pattern Combinations ในโลกจริง

```
Plugin System:
  Adapter     → แปลง third-party plugins
  Decorator   → เพิ่ม logging/auth ให้ plugins
  Facade      → PluginManager จัดการ lifecycle

E-commerce:
  Facade      → OrderFacade รวม subsystems
  Decorator   → DataService + Caching + Logging
  Proxy       → Virtual Proxy สำหรับ lazy loading resources

UI Framework:
  Composite   → Component tree (Panel > Button, TextBox)
  Decorator   → เพิ่ม validation, styling
  Bridge      → Shape/Renderer สำหรับ cross-platform rendering
```

---

## การนำไปใช้ใน .NET และ C# จริง

### Adapter ที่พบบ่อยใน .NET
- `IEnumerable<T>` adapters ใน LINQ
- `DbDataAdapter` ใน ADO.NET
- `HttpMessageHandler` adapters

### Decorator ที่พบบ่อยใน .NET
- `Stream` decorators: `GZipStream`, `BufferedStream`, `CryptoStream`
- `ILogger` decorators ใน Microsoft.Extensions.Logging
- Middleware pipeline ใน ASP.NET Core

### Facade ที่พบบ่อยใน .NET
- `DbContext` ใน Entity Framework (facade สำหรับ database operations)
- `HttpClient` (facade สำหรับ HTTP operations)
- `SmtpClient` (facade สำหรับ email sending)

### Proxy ที่พบบ่อยใน .NET
- Lazy<T> (Virtual Proxy)
- DispatchProxy ใน System.Reflection
- Castle DynamicProxy (ใช้ใน ORM และ DI frameworks)

---

## การนำทาง (Navigation)

- **ก่อนหน้า**: [Part 47 - Creational Patterns](part47-creational-patterns.md)
- **ถัดไป**: [Part 49 - Behavioral Patterns](part49-behavioral-patterns.md)
