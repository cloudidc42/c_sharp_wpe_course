# Part 10: Exception Handling
## ขั้นตอนที่ 91-100: การจัดการ Exceptions อย่างมืออาชีพ

---

## 🎯 เป้าหมายของ Part นี้
- เข้าใจ Exception hierarchy ใน .NET
- ใช้ try/catch/finally ได้อย่างถูกต้อง
- สร้าง Custom Exceptions
- รู้จัก Exception filters (C# 6+)
- ใช้ Global Exception Handling
- เข้าใจ best practices การจัดการ errors

---

## ขั้นตอนที่ 91: Exception พื้นฐาน

### Exception Hierarchy
```csharp
/*
System.Object
└── System.Exception
    ├── System.SystemException
    │   ├── System.ArgumentException
    │   │   ├── System.ArgumentNullException
    │   │   └── System.ArgumentOutOfRangeException
    │   ├── System.InvalidOperationException
    │   ├── System.NullReferenceException
    │   ├── System.IndexOutOfRangeException
    │   ├── System.OverflowException
    │   ├── System.DivideByZeroException
    │   ├── System.FormatException
    │   ├── System.NotImplementedException
    │   ├── System.NotSupportedException
    │   ├── System.OutOfMemoryException
    │   └── System.IO.IOException
    │       ├── System.IO.FileNotFoundException
    │       └── System.IO.DirectoryNotFoundException
    └── System.ApplicationException (deprecated, use Exception directly)
*/

// Properties ของ Exception
try
{
    throw new InvalidOperationException("Cannot perform this operation");
}
catch (InvalidOperationException ex)
{
    Console.WriteLine($"Message: {ex.Message}");
    Console.WriteLine($"Source: {ex.Source}");
    Console.WriteLine($"StackTrace: {ex.StackTrace}");
    Console.WriteLine($"HResult: {ex.HResult:X8}");
}
```

### try/catch/finally
```csharp
string ReadFile(string path)
{
    FileStream? stream = null;
    try
    {
        stream = new FileStream(path, FileMode.Open);
        // อ่านไฟล์...
        byte[] buffer = new byte[stream.Length];
        stream.Read(buffer, 0, buffer.Length);
        return System.Text.Encoding.UTF8.GetString(buffer);
    }
    catch (FileNotFoundException ex)
    {
        Console.WriteLine($"ไม่พบไฟล์: {ex.FileName}");
        return string.Empty;
    }
    catch (UnauthorizedAccessException ex)
    {
        Console.WriteLine($"ไม่มีสิทธิ์เข้าถึงไฟล์: {ex.Message}");
        return string.Empty;
    }
    catch (Exception ex)
    {
        Console.WriteLine($"เกิดข้อผิดพลาด: {ex.Message}");
        throw;  // Re-throw เพื่อให้ caller จัดการ
    }
    finally
    {
        // Always executed - ทำ cleanup เสมอ
        stream?.Close();
        Console.WriteLine("ปิดไฟล์แล้ว");
    }
}

// Catch multiple exception types (C# 6+)
void Process(string input)
{
    try
    {
        int value = int.Parse(input);
        int result = 100 / value;
        Console.WriteLine($"Result: {result}");
    }
    catch (FormatException ex) when (input.Length > 10)
    {
        // Exception filter - จับเฉพาะเมื่อ input ยาวเกิน 10
        Console.WriteLine($"Input ยาวเกินไปและ format ผิด: {ex.Message}");
    }
    catch (FormatException or DivideByZeroException)
    {
        // Catch multiple types (C# 9+)
        Console.WriteLine("Format error หรือ Division by zero");
    }
}
```

---

## ขั้นตอนที่ 92: Custom Exceptions

### สร้าง Custom Exception Classes
```csharp
// Base custom exception
public class AppException : Exception
{
    public string ErrorCode { get; }
    public DateTime OccurredAt { get; } = DateTime.UtcNow;
    
    public AppException(string errorCode, string message) 
        : base(message)
    {
        ErrorCode = errorCode;
    }
    
    public AppException(string errorCode, string message, Exception innerException) 
        : base(message, innerException)
    {
        ErrorCode = errorCode;
    }
}

// Specific exceptions
public class ValidationException : AppException
{
    public IReadOnlyDictionary<string, string[]> Errors { get; }
    
    public ValidationException(Dictionary<string, string[]> errors)
        : base("VALIDATION_ERROR", "One or more validation errors occurred")
    {
        Errors = errors.AsReadOnly();
    }
    
    public ValidationException(string field, string error)
        : base("VALIDATION_ERROR", $"Validation failed for {field}: {error}")
    {
        Errors = new Dictionary<string, string[]> 
        { 
            { field, new[] { error } } 
        }.AsReadOnly();
    }
}

public class NotFoundException : AppException
{
    public string EntityType { get; }
    public object EntityId { get; }
    
    public NotFoundException(string entityType, object entityId)
        : base("NOT_FOUND", $"{entityType} with id '{entityId}' not found")
    {
        EntityType = entityType;
        EntityId = entityId;
    }
}

public class BusinessRuleException : AppException
{
    public string RuleName { get; }
    
    public BusinessRuleException(string ruleName, string message)
        : base("BUSINESS_RULE_VIOLATION", message)
    {
        RuleName = ruleName;
    }
}

// การใช้งาน
void CreateOrder(string customerId, int productId, int quantity)
{
    if (string.IsNullOrEmpty(customerId))
        throw new ValidationException("customerId", "Customer ID is required");
    
    if (quantity <= 0)
        throw new ValidationException("quantity", "Quantity must be greater than 0");
    
    // สมมติว่าไม่พบ customer
    bool customerExists = false;
    if (!customerExists)
        throw new NotFoundException("Customer", customerId);
    
    // Business rule check
    int maxOrderQuantity = 100;
    if (quantity > maxOrderQuantity)
        throw new BusinessRuleException(
            "MAX_ORDER_QUANTITY",
            $"Cannot order more than {maxOrderQuantity} items at once"
        );
}

try
{
    CreateOrder("", 1, 5);
}
catch (ValidationException ex)
{
    Console.WriteLine($"Validation Error [{ex.ErrorCode}]:");
    foreach (var (field, errors) in ex.Errors)
        Console.WriteLine($"  {field}: {string.Join(", ", errors)}");
}
catch (NotFoundException ex)
{
    Console.WriteLine($"Not Found [{ex.ErrorCode}]: {ex.Message}");
}
catch (BusinessRuleException ex)
{
    Console.WriteLine($"Business Rule [{ex.RuleName}]: {ex.Message}");
}
catch (AppException ex)
{
    Console.WriteLine($"App Error [{ex.ErrorCode}]: {ex.Message}");
}
```

---

## ขั้นตอนที่ 93: Exception Chaining และ Re-throwing

### การ chain exceptions
```csharp
void LoadConfiguration(string configPath)
{
    try
    {
        string json = File.ReadAllText(configPath);
        // Parse JSON...
        var invalid = int.Parse("not-a-number");
    }
    catch (FileNotFoundException ex)
    {
        // Wrap ใน AppException พร้อมเก็บ inner exception
        throw new AppException(
            "CONFIG_NOT_FOUND",
            $"Configuration file not found: {configPath}",
            ex  // innerException - เก็บ original exception
        );
    }
    catch (FormatException ex)
    {
        throw new AppException(
            "CONFIG_INVALID",
            "Configuration file has invalid format",
            ex
        );
    }
}

// ดู exception chain
try
{
    LoadConfiguration("config.json");
}
catch (AppException ex)
{
    Console.WriteLine($"Error: {ex.Message}");
    
    // ดู full chain
    Exception? current = ex;
    int depth = 0;
    while (current != null)
    {
        Console.WriteLine($"{"  ".PadLeft(depth * 2)}Caused by: [{current.GetType().Name}] {current.Message}");
        current = current.InnerException;
        depth++;
    }
}

// Re-throw patterns
void ProcessData(string data)
{
    try
    {
        // ...
    }
    catch (Exception ex)
    {
        // ❌ Bad - loses original stack trace
        // throw ex;
        
        // ✅ Good - preserves stack trace
        throw;
        
        // ✅ Also good - wrap with context
        // throw new AppException("PROCESS_ERROR", "Failed to process data", ex);
    }
}
```

---

## ขั้นตอนที่ 94: Exception Filters

### When clause
```csharp
// Exception filter ด้วย when keyword
void HandleHttpException(HttpRequestException ex)
{
    // จัดการต่างกันตาม status code
}

async Task FetchDataAsync(string url, int retryCount = 3)
{
    for (int attempt = 1; attempt <= retryCount; attempt++)
    {
        try
        {
            Console.WriteLine($"Attempt {attempt}...");
            await Task.Delay(100);  // จำลอง HTTP request
            
            // จำลอง exception
            if (attempt < 3) throw new TimeoutException($"Timeout on attempt {attempt}");
            
            Console.WriteLine("Success!");
            return;
        }
        catch (TimeoutException ex) when (attempt < retryCount)
        {
            // Retry - ไม่ re-throw
            Console.WriteLine($"Timeout, retrying... ({ex.Message})");
            await Task.Delay(attempt * 1000);  // Exponential backoff
        }
        catch (TimeoutException)
        {
            // Last attempt failed
            throw;
        }
    }
}

await FetchDataAsync("https://api.example.com/data");

// Exception filter สำหรับ logging (ไม่ swallow exception)
bool LogException(Exception ex)
{
    Console.ForegroundColor = ConsoleColor.Red;
    Console.WriteLine($"[LOG] Exception: {ex.GetType().Name}: {ex.Message}");
    Console.ResetColor();
    return false;  // ไม่จับ exception (จะ propagate ต่อ)
}

try
{
    throw new InvalidOperationException("Test error");
}
catch (Exception ex) when (LogException(ex))
{
    // นี่จะไม่ถูก execute เพราะ LogException return false
    Console.WriteLine("This won't execute");
}
// Exception ถูก log แล้ว propagate ต่อ
```

---

## ขั้นตอนที่ 95: Result Pattern

### Result<T> Pattern แทน Exceptions
```csharp
// Result pattern - ใช้ในกรณีที่ error เป็นส่วนหนึ่งของ normal flow
readonly struct Result<T>
{
    private readonly T? _value;
    private readonly string? _error;
    
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    
    private Result(T value)
    {
        _value = value;
        _error = null;
        IsSuccess = true;
    }
    
    private Result(string error)
    {
        _value = default;
        _error = error;
        IsSuccess = false;
    }
    
    public T Value => IsSuccess ? _value! : throw new InvalidOperationException(_error);
    public string Error => IsFailure ? _error! : throw new InvalidOperationException("No error");
    
    public static Result<T> Success(T value) => new(value);
    public static Result<T> Failure(string error) => new(error);
    
    public Result<TNext> Map<TNext>(Func<T, TNext> transform)
        => IsSuccess ? Result<TNext>.Success(transform(Value)) : Result<TNext>.Failure(Error);
    
    public Result<TNext> Bind<TNext>(Func<T, Result<TNext>> func)
        => IsSuccess ? func(Value) : Result<TNext>.Failure(Error);
    
    public T GetValueOrDefault(T defaultValue = default!)
        => IsSuccess ? _value! : defaultValue;
    
    public override string ToString()
        => IsSuccess ? $"Success({_value})" : $"Failure({_error})";
}

// Static helper class
static class Result
{
    public static Result<T> Of<T>(Func<T> func)
    {
        try { return Result<T>.Success(func()); }
        catch (Exception ex) { return Result<T>.Failure(ex.Message); }
    }
}

// การใช้งาน
Result<int> Divide(int a, int b)
{
    if (b == 0) return Result<int>.Failure("Division by zero");
    return Result<int>.Success(a / b);
}

Result<double> Sqrt(int n)
{
    if (n < 0) return Result<double>.Failure("Cannot sqrt negative number");
    return Result<double>.Success(Math.Sqrt(n));
}

// Chaining results
var result = Divide(100, 4)
    .Bind(x => Sqrt(x));

if (result.IsSuccess)
    Console.WriteLine($"Result: {result.Value:F4}");
else
    Console.WriteLine($"Error: {result.Error}");
```

---

## ขั้นตอนที่ 96-100: โปรแกรมตัวอย่าง - Robust Banking System

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

namespace RobustBanking
{
    // Custom Exceptions
    public class BankingException : Exception
    {
        public string ErrorCode { get; }
        public DateTime Timestamp { get; } = DateTime.UtcNow;
        
        public BankingException(string errorCode, string message) : base(message)
            => ErrorCode = errorCode;
        
        public BankingException(string errorCode, string message, Exception inner) 
            : base(message, inner)
            => ErrorCode = errorCode;
    }
    
    public class AccountNotFoundException : BankingException
    {
        public string AccountNumber { get; }
        
        public AccountNotFoundException(string accountNumber)
            : base("ACCOUNT_NOT_FOUND", $"Account {accountNumber} not found")
            => AccountNumber = accountNumber;
    }
    
    public class InsufficientFundsException : BankingException
    {
        public decimal RequiredAmount { get; }
        public decimal AvailableBalance { get; }
        
        public InsufficientFundsException(decimal required, decimal available)
            : base("INSUFFICIENT_FUNDS", 
                $"Insufficient funds. Required: {required:N2}, Available: {available:N2}")
        {
            RequiredAmount = required;
            AvailableBalance = available;
        }
    }
    
    public class AccountLockedException : BankingException
    {
        public string Reason { get; }
        
        public AccountLockedException(string reason)
            : base("ACCOUNT_LOCKED", $"Account is locked: {reason}")
            => Reason = reason;
    }
    
    public class TransactionLimitException : BankingException
    {
        public decimal AttemptedAmount { get; }
        public decimal DailyLimit { get; }
        
        public TransactionLimitException(decimal attempted, decimal dailyLimit)
            : base("TRANSACTION_LIMIT", 
                $"Transaction exceeds daily limit. Attempted: {attempted:N2}, Limit: {dailyLimit:N2}")
        {
            AttemptedAmount = attempted;
            DailyLimit = dailyLimit;
        }
    }
    
    // Domain Models
    enum AccountStatus { Active, Locked, Closed }
    
    enum TransactionType { Deposit, Withdrawal, Transfer }
    
    class Transaction
    {
        public Guid Id { get; } = Guid.NewGuid();
        public DateTime Timestamp { get; } = DateTime.UtcNow;
        public TransactionType Type { get; init; }
        public decimal Amount { get; init; }
        public string Description { get; init; } = "";
        public decimal BalanceAfter { get; init; }
        
        public override string ToString()
            => $"[{Timestamp:HH:mm:ss}] {Type}: {Amount:N2} บาท | ยอด: {BalanceAfter:N2} บาท | {Description}";
    }
    
    class BankAccount
    {
        private decimal _balance;
        private List<Transaction> _transactions = new();
        private int _failedAttempts = 0;
        private const int MaxFailedAttempts = 3;
        private const decimal DailyTransactionLimit = 100_000m;
        
        public string AccountNumber { get; }
        public string OwnerName { get; }
        public AccountStatus Status { get; private set; } = AccountStatus.Active;
        public decimal Balance => _balance;
        public IReadOnlyList<Transaction> Transactions => _transactions.AsReadOnly();
        
        // Daily transaction tracking
        private decimal TodayTransactionTotal => _transactions
            .Where(t => t.Timestamp.Date == DateTime.Today 
                && t.Type != TransactionType.Deposit)
            .Sum(t => t.Amount);
        
        public BankAccount(string accountNumber, string ownerName, decimal initialBalance = 0)
        {
            AccountNumber = accountNumber;
            OwnerName = ownerName;
            _balance = initialBalance;
        }
        
        public void Deposit(decimal amount, string description = "Deposit")
        {
            ValidateAccountActive();
            
            if (amount <= 0)
                throw new ArgumentOutOfRangeException(nameof(amount), "Amount must be positive");
            
            if (amount > 10_000_000m)
                throw new BankingException("DEPOSIT_LIMIT", 
                    "Single deposit cannot exceed 10,000,000 บาท");
            
            _balance += amount;
            RecordTransaction(TransactionType.Deposit, amount, description);
            _failedAttempts = 0;
        }
        
        public void Withdraw(decimal amount, string description = "Withdrawal")
        {
            ValidateAccountActive();
            
            if (amount <= 0)
                throw new ArgumentOutOfRangeException(nameof(amount), "Amount must be positive");
            
            // Check daily limit
            decimal todayTotal = TodayTransactionTotal + amount;
            if (todayTotal > DailyTransactionLimit)
                throw new TransactionLimitException(todayTotal, DailyTransactionLimit);
            
            // Check balance
            if (_balance < amount)
            {
                _failedAttempts++;
                if (_failedAttempts >= MaxFailedAttempts)
                {
                    Status = AccountStatus.Locked;
                    throw new AccountLockedException($"Too many failed attempts ({MaxFailedAttempts})");
                }
                throw new InsufficientFundsException(amount, _balance);
            }
            
            _balance -= amount;
            RecordTransaction(TransactionType.Withdrawal, amount, description);
            _failedAttempts = 0;
        }
        
        public void Unlock()
        {
            if (Status != AccountStatus.Locked)
                throw new BankingException("NOT_LOCKED", "Account is not locked");
            
            Status = AccountStatus.Active;
            _failedAttempts = 0;
            Console.WriteLine($"Account {AccountNumber} unlocked");
        }
        
        private void ValidateAccountActive()
        {
            switch (Status)
            {
                case AccountStatus.Locked:
                    throw new AccountLockedException("Account has been locked");
                case AccountStatus.Closed:
                    throw new BankingException("ACCOUNT_CLOSED", "Account is closed");
            }
        }
        
        private void RecordTransaction(TransactionType type, decimal amount, string description)
        {
            _transactions.Add(new Transaction
            {
                Type = type,
                Amount = amount,
                Description = description,
                BalanceAfter = _balance
            });
        }
        
        public void PrintStatement(int lastN = 10)
        {
            Console.WriteLine($"\n=== Statement: {AccountNumber} ({OwnerName}) ===");
            Console.WriteLine($"Status: {Status} | Balance: {_balance:N2} บาท");
            Console.WriteLine(new string('-', 60));
            
            var recent = _transactions.TakeLast(lastN).ToList();
            if (!recent.Any())
            {
                Console.WriteLine("ไม่มีรายการ");
                return;
            }
            
            foreach (var tx in recent)
                Console.WriteLine(tx);
        }
    }
    
    class Bank
    {
        private Dictionary<string, BankAccount> _accounts = new();
        private static int _accountCounter = 1000;
        
        public BankAccount CreateAccount(string ownerName, decimal initialDeposit = 0)
        {
            string accountNumber = $"ACC{++_accountCounter:0000}";
            var account = new BankAccount(accountNumber, ownerName, initialDeposit);
            _accounts[accountNumber] = account;
            Console.WriteLine($"✅ เปิดบัญชี {accountNumber} สำหรับ {ownerName}");
            return account;
        }
        
        public BankAccount GetAccount(string accountNumber)
        {
            return _accounts.TryGetValue(accountNumber, out var account)
                ? account
                : throw new AccountNotFoundException(accountNumber);
        }
        
        public void Transfer(string fromAccountNo, string toAccountNo, decimal amount, string description = "Transfer")
        {
            // ดึง accounts พร้อม validate
            var fromAccount = GetAccount(fromAccountNo);
            var toAccount = GetAccount(toAccountNo);
            
            // ใช้ transaction-like behavior
            decimal originalFromBalance = fromAccount.Balance;
            
            try
            {
                fromAccount.Withdraw(amount, $"Transfer to {toAccountNo}: {description}");
                
                try
                {
                    toAccount.Deposit(amount, $"Transfer from {fromAccountNo}: {description}");
                }
                catch (Exception depositEx)
                {
                    // Rollback withdrawal if deposit fails
                    fromAccount.Deposit(amount, $"Rollback transfer to {toAccountNo}");
                    throw new BankingException("TRANSFER_FAILED", 
                        "Transfer failed, withdrawal has been reversed", depositEx);
                }
                
                Console.WriteLine($"✅ โอนเงิน {amount:N2} บาท จาก {fromAccountNo} ไป {toAccountNo}");
            }
            catch (BankingException)
            {
                throw;  // Re-throw banking exceptions
            }
            catch (Exception ex)
            {
                throw new BankingException("TRANSFER_ERROR", 
                    "Unexpected error during transfer", ex);
            }
        }
        
        public void SafeOperation(string accountNumber, Action<BankAccount> operation)
        {
            try
            {
                var account = GetAccount(accountNumber);
                operation(account);
            }
            catch (AccountNotFoundException ex)
            {
                Console.ForegroundColor = ConsoleColor.Red;
                Console.WriteLine($"❌ [{ex.ErrorCode}] {ex.Message}");
                Console.ResetColor();
            }
            catch (InsufficientFundsException ex)
            {
                Console.ForegroundColor = ConsoleColor.Yellow;
                Console.WriteLine($"⚠️ [{ex.ErrorCode}] {ex.Message}");
                Console.WriteLine($"   ต้องการ: {ex.RequiredAmount:N2} | มี: {ex.AvailableBalance:N2}");
                Console.ResetColor();
            }
            catch (AccountLockedException ex)
            {
                Console.ForegroundColor = ConsoleColor.Red;
                Console.WriteLine($"🔒 [{ex.ErrorCode}] {ex.Message}");
                Console.ResetColor();
            }
            catch (TransactionLimitException ex)
            {
                Console.ForegroundColor = ConsoleColor.Yellow;
                Console.WriteLine($"⚠️ [{ex.ErrorCode}] {ex.Message}");
                Console.ResetColor();
            }
            catch (BankingException ex)
            {
                Console.ForegroundColor = ConsoleColor.Red;
                Console.WriteLine($"❌ [{ex.ErrorCode}] {ex.Message}");
                Console.ResetColor();
            }
        }
    }
    
    class Program
    {
        static void Main(string[] args)
        {
            Console.OutputEncoding = System.Text.Encoding.UTF8;
            Console.Title = "Robust Banking System";
            
            var bank = new Bank();
            
            // สร้างบัญชี
            var alice = bank.CreateAccount("Alice Smith", 50_000m);
            var bob = bank.CreateAccount("Bob Jones", 10_000m);
            
            Console.WriteLine("\n=== ทดสอบ Operations ปกติ ===");
            bank.SafeOperation(alice.AccountNumber, acc => {
                acc.Deposit(20_000m, "เงินเดือน");
                acc.Withdraw(5_000m, "ค่าน้ำ ค่าไฟ");
            });
            
            Console.WriteLine("\n=== ทดสอบ Transfer ===");
            bank.Transfer(alice.AccountNumber, bob.AccountNumber, 15_000m, "ค่าสินค้า");
            
            Console.WriteLine("\n=== ทดสอบ Error Cases ===");
            
            // Insufficient funds
            bank.SafeOperation(bob.AccountNumber, acc => 
                acc.Withdraw(999_999m, "ถอนเงินเกินบัญชี"));
            
            // Account not found
            bank.SafeOperation("ACC9999", acc => acc.Deposit(1000m));
            
            // Force lock account
            Console.WriteLine("\n--- ทดสอบ Account Lock ---");
            for (int i = 0; i < 4; i++)
            {
                bank.SafeOperation(bob.AccountNumber, acc =>
                    acc.Withdraw(999_999m, "พยายามถอนเกิน"));
            }
            
            // Try to use locked account
            bank.SafeOperation(bob.AccountNumber, acc => acc.Deposit(1000m));
            
            // Unlock and retry
            bank.GetAccount(bob.AccountNumber).Unlock();
            bank.SafeOperation(bob.AccountNumber, acc => acc.Deposit(1000m, "ฝากหลัง unlock"));
            
            // Print statements
            alice.PrintStatement();
            bob.PrintStatement();
            
            Console.WriteLine("\nกด Enter เพื่อออก...");
            Console.ReadLine();
        }
    }
}
```

---

## 📝 สรุป Part 10

| หัวข้อ | รายละเอียดสำคัญ |
|--------|----------------|
| try/catch/finally | จัดการ exceptions อย่างถูกต้อง |
| Custom Exception | สร้าง exceptions ที่มีข้อมูลเพียงพอ |
| Exception Chaining | เก็บ InnerException เพื่อดู root cause |
| Re-throwing | ใช้ `throw;` ไม่ใช่ `throw ex;` |
| Exception Filter | `when` clause สำหรับ conditional catch |
| Result Pattern | แทน exceptions สำหรับ expected errors |
| Error Codes | ใช้ string codes เพื่อจัดกลุ่มและแปล |

---

## 🏋️ แบบฝึกหัด Part 10

### แบบฝึกหัดที่ 1: Retry Policy
สร้าง `RetryPolicy` class ที่:
- retry ได้ N ครั้ง
- มี exponential backoff
- สามารถระบุ exception types ที่ควร retry

### แบบฝึกหัดที่ 2: Global Error Handler
สร้าง middleware ที่ catch exceptions ทุกประเภทและ log อย่างเป็นระบบ

### แบบฝึกหัดที่ 3: Result Monad
ขยาย Result<T> ให้มี:
- `Match(onSuccess, onFailure)` method
- `Combine(results)` - รวม multiple results
- Async versions (`MapAsync`, `BindAsync`)

---

**ก่อนหน้า → [Part 09: Interfaces](part09-interfaces.md)**  
**ต่อไป → [Part 11: String Manipulation](../part11-20/part11-strings.md)**
