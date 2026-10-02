# Part 63: Event Sourcing
## ขั้นตอนที่ 621-630: จัดเก็บ State ด้วย Events

---

## 🎯 เป้าหมายของ Part นี้
- Event Sourcing คืออะไร
- Event Store
- Aggregate reconstruction จาก events
- Snapshots
- Event Projections
- Eventual Consistency
- Marten (PostgreSQL Event Store for .NET)

---

## ขั้นตอนที่ 621: Event Sourcing Concepts

```
Traditional (State-based):
┌─────────┐     update     ┌─────────────────────┐
│ Command │ ──────────────→ │ Current State in DB │
└─────────┘                └─────────────────────┘
→ ข้อมูลเก่าหายไป (ไม่รู้ว่าเคยเกิดอะไรขึ้น)

Event Sourcing:
┌─────────┐     append     ┌─────────────────────────────────────┐
│ Command │ ──────────────→ │ Event Store (append-only log)       │
└─────────┘                │  1. AccountCreated($1000)           │
                           │  2. MoneyDeposited($500)            │
                           │  3. MoneyWithdrawn($200)            │
                           │  4. AccountFrozen                   │
                           └─────────────────────────────────────┘
                                         ↓ replay
                                   Current State: $1300 (frozen)
→ ประวัติทุกอย่างยังอยู่ สามารถ replay เพื่อสร้าง state ใหม่
```

---

## ขั้นตอนที่ 622: Defining Events

```csharp
// Base event interface
public interface IDomainEvent
{
    Guid StreamId { get; }   // Aggregate ID
    int Version { get; }     // Event number in stream
    DateTime OccurredAt { get; }
    string EventType { get; }
}

// Base event record
public abstract record DomainEvent(Guid StreamId, int Version) : IDomainEvent
{
    public DateTime OccurredAt { get; } = DateTime.UtcNow;
    public string EventType => GetType().Name;
}

// Bank Account events
public record AccountOpenedEvent(Guid StreamId, int Version, string OwnerId, decimal InitialBalance) 
    : DomainEvent(StreamId, Version);

public record MoneyDepositedEvent(Guid StreamId, int Version, decimal Amount, string Description)
    : DomainEvent(StreamId, Version);

public record MoneyWithdrawnEvent(Guid StreamId, int Version, decimal Amount, string Description)
    : DomainEvent(StreamId, Version);

public record MoneyTransferredEvent(Guid StreamId, int Version, Guid ToAccountId, decimal Amount)
    : DomainEvent(StreamId, Version);

public record AccountFrozenEvent(Guid StreamId, int Version, string Reason)
    : DomainEvent(StreamId, Version);

public record AccountClosedEvent(Guid StreamId, int Version)
    : DomainEvent(StreamId, Version);
```

---

## ขั้นตอนที่ 623: Event-Sourced Aggregate

```csharp
// BankAccount.cs - reconstructed from events
public class BankAccount
{
    // Current state (rebuilt from events)
    public Guid Id { get; private set; }
    public string OwnerId { get; private set; } = "";
    public decimal Balance { get; private set; }
    public bool IsFrozen { get; private set; }
    public bool IsClosed { get; private set; }
    public int Version { get; private set; }
    
    // Uncommitted events (raised this session)
    private readonly List<IDomainEvent> _uncommitted = new();
    public IReadOnlyList<IDomainEvent> UncommittedEvents => _uncommitted.AsReadOnly();
    
    // Factory: create new account
    public static BankAccount Open(string ownerId, decimal initialBalance)
    {
        if (initialBalance < 0) throw new DomainException("Initial balance cannot be negative");
        
        var account = new BankAccount();
        account.Apply(new AccountOpenedEvent(Guid.NewGuid(), 1, ownerId, initialBalance));
        return account;
    }
    
    // Reconstruct from event stream (replay)
    public static BankAccount FromEvents(IEnumerable<IDomainEvent> events)
    {
        var account = new BankAccount();
        foreach (var evt in events)
            account.Apply(evt, isNew: false);
        return account;
    }
    
    // Business operations
    public void Deposit(decimal amount, string description = "")
    {
        if (IsClosed) throw new DomainException("Account is closed");
        if (IsFrozen) throw new DomainException("Account is frozen");
        if (amount <= 0) throw new DomainException("Amount must be positive");
        
        Apply(new MoneyDepositedEvent(Id, Version + 1, amount, description));
    }
    
    public void Withdraw(decimal amount, string description = "")
    {
        if (IsClosed) throw new DomainException("Account is closed");
        if (IsFrozen) throw new DomainException("Account is frozen");
        if (amount <= 0) throw new DomainException("Amount must be positive");
        if (Balance < amount) throw new DomainException($"Insufficient funds. Balance: {Balance}, Requested: {amount}");
        
        Apply(new MoneyWithdrawnEvent(Id, Version + 1, amount, description));
    }
    
    public void Freeze(string reason)
    {
        if (IsClosed || IsFrozen) return;
        Apply(new AccountFrozenEvent(Id, Version + 1, reason));
    }
    
    public void Close()
    {
        if (IsClosed) return;
        if (Balance != 0) throw new DomainException("Cannot close account with non-zero balance");
        Apply(new AccountClosedEvent(Id, Version + 1));
    }
    
    // Apply event (mutate state)
    private void Apply(IDomainEvent evt, bool isNew = true)
    {
        // Mutate state based on event type
        When((dynamic)evt);
        Version = evt.Version;
        
        if (isNew)
            _uncommitted.Add(evt);
    }
    
    // State mutators - pure functions, no validation
    private void When(AccountOpenedEvent e)
    {
        Id = e.StreamId;
        OwnerId = e.OwnerId;
        Balance = e.InitialBalance;
    }
    
    private void When(MoneyDepositedEvent e)
        => Balance += e.Amount;
    
    private void When(MoneyWithdrawnEvent e)
        => Balance -= e.Amount;
    
    private void When(MoneyTransferredEvent e)
        => Balance -= e.Amount;
    
    private void When(AccountFrozenEvent e)
        => IsFrozen = true;
    
    private void When(AccountClosedEvent e)
        => IsClosed = true;
    
    public void ClearUncommitted() => _uncommitted.Clear();
}
```

---

## ขั้นตอนที่ 624: Event Store Implementation

```csharp
// IEventStore.cs
public interface IEventStore
{
    Task AppendAsync(Guid streamId, IEnumerable<IDomainEvent> events, int expectedVersion, CancellationToken ct = default);
    Task<IReadOnlyList<IDomainEvent>> LoadAsync(Guid streamId, CancellationToken ct = default);
    Task<IReadOnlyList<IDomainEvent>> LoadFromVersionAsync(Guid streamId, int fromVersion, CancellationToken ct = default);
}

// Simple SQLite Event Store
public class SqliteEventStore : IEventStore
{
    private readonly string _connectionString;
    
    public SqliteEventStore(string connectionString)
    {
        _connectionString = connectionString;
        InitializeSchema();
    }
    
    private void InitializeSchema()
    {
        using var conn = new SqliteConnection(_connectionString);
        conn.Open();
        using var cmd = conn.CreateCommand();
        cmd.CommandText = """
            CREATE TABLE IF NOT EXISTS events (
                stream_id TEXT NOT NULL,
                version INTEGER NOT NULL,
                event_type TEXT NOT NULL,
                event_data TEXT NOT NULL,
                occurred_at TEXT NOT NULL,
                PRIMARY KEY (stream_id, version)
            );
            CREATE INDEX IF NOT EXISTS idx_events_stream ON events(stream_id, version);
            """;
        cmd.ExecuteNonQuery();
    }
    
    public async Task AppendAsync(Guid streamId, IEnumerable<IDomainEvent> events, int expectedVersion, CancellationToken ct = default)
    {
        using var conn = new SqliteConnection(_connectionString);
        await conn.OpenAsync(ct);
        using var tx = await conn.BeginTransactionAsync(ct);
        
        try
        {
            // Optimistic concurrency check
            var currentVersion = await GetCurrentVersionAsync(conn, streamId);
            if (currentVersion != expectedVersion)
                throw new ConcurrencyException($"Stream {streamId}: expected version {expectedVersion}, got {currentVersion}");
            
            foreach (var evt in events)
            {
                using var cmd = conn.CreateCommand();
                cmd.Transaction = (SqliteTransaction)tx;
                cmd.CommandText = """
                    INSERT INTO events (stream_id, version, event_type, event_data, occurred_at)
                    VALUES ($streamId, $version, $type, $data, $ts)
                    """;
                cmd.Parameters.AddWithValue("$streamId", streamId.ToString());
                cmd.Parameters.AddWithValue("$version", evt.Version);
                cmd.Parameters.AddWithValue("$type", evt.EventType);
                cmd.Parameters.AddWithValue("$data", JsonSerializer.Serialize(evt, evt.GetType()));
                cmd.Parameters.AddWithValue("$ts", evt.OccurredAt.ToString("o"));
                await cmd.ExecuteNonQueryAsync(ct);
            }
            
            await tx.CommitAsync(ct);
        }
        catch
        {
            await tx.RollbackAsync(ct);
            throw;
        }
    }
    
    public async Task<IReadOnlyList<IDomainEvent>> LoadAsync(Guid streamId, CancellationToken ct = default)
    {
        using var conn = new SqliteConnection(_connectionString);
        await conn.OpenAsync(ct);
        using var cmd = conn.CreateCommand();
        cmd.CommandText = "SELECT event_type, event_data FROM events WHERE stream_id = $id ORDER BY version";
        cmd.Parameters.AddWithValue("$id", streamId.ToString());
        
        var events = new List<IDomainEvent>();
        using var reader = await cmd.ExecuteReaderAsync(ct);
        while (await reader.ReadAsync(ct))
        {
            var type = reader.GetString(0);
            var data = reader.GetString(1);
            var evt = DeserializeEvent(type, data);
            if (evt != null) events.Add(evt);
        }
        return events.AsReadOnly();
    }
    
    private IDomainEvent? DeserializeEvent(string eventType, string data)
    {
        return eventType switch
        {
            nameof(AccountOpenedEvent) => JsonSerializer.Deserialize<AccountOpenedEvent>(data),
            nameof(MoneyDepositedEvent) => JsonSerializer.Deserialize<MoneyDepositedEvent>(data),
            nameof(MoneyWithdrawnEvent) => JsonSerializer.Deserialize<MoneyWithdrawnEvent>(data),
            nameof(AccountFrozenEvent) => JsonSerializer.Deserialize<AccountFrozenEvent>(data),
            nameof(AccountClosedEvent) => JsonSerializer.Deserialize<AccountClosedEvent>(data),
            _ => null
        };
    }
    
    private async Task<int> GetCurrentVersionAsync(SqliteConnection conn, Guid streamId)
    {
        using var cmd = conn.CreateCommand();
        cmd.CommandText = "SELECT COALESCE(MAX(version), 0) FROM events WHERE stream_id = $id";
        cmd.Parameters.AddWithValue("$id", streamId.ToString());
        var result = await cmd.ExecuteScalarAsync();
        return Convert.ToInt32(result);
    }
    
    public async Task<IReadOnlyList<IDomainEvent>> LoadFromVersionAsync(Guid streamId, int fromVersion, CancellationToken ct = default)
    {
        using var conn = new SqliteConnection(_connectionString);
        await conn.OpenAsync(ct);
        using var cmd = conn.CreateCommand();
        cmd.CommandText = "SELECT event_type, event_data FROM events WHERE stream_id = $id AND version > $v ORDER BY version";
        cmd.Parameters.AddWithValue("$id", streamId.ToString());
        cmd.Parameters.AddWithValue("$v", fromVersion);
        
        var events = new List<IDomainEvent>();
        using var reader = await cmd.ExecuteReaderAsync(ct);
        while (await reader.ReadAsync(ct))
            events.Add(DeserializeEvent(reader.GetString(0), reader.GetString(1))!);
        return events.AsReadOnly();
    }
}
```

---

## ขั้นตอนที่ 625: Repository with Event Store

```csharp
// IBankAccountRepository.cs
public interface IBankAccountRepository
{
    Task<BankAccount?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task SaveAsync(BankAccount account, CancellationToken ct = default);
}

// BankAccountRepository.cs
public class BankAccountRepository : IBankAccountRepository
{
    private readonly IEventStore _store;
    
    public BankAccountRepository(IEventStore store) => _store = store;
    
    public async Task<BankAccount?> GetByIdAsync(Guid id, CancellationToken ct = default)
    {
        var events = await _store.LoadAsync(id, ct);
        if (!events.Any()) return null;
        return BankAccount.FromEvents(events);
    }
    
    public async Task SaveAsync(BankAccount account, CancellationToken ct = default)
    {
        var events = account.UncommittedEvents;
        if (!events.Any()) return;
        
        var expectedVersion = account.Version - events.Count;
        await _store.AppendAsync(account.Id, events, expectedVersion, ct);
        account.ClearUncommitted();
    }
}
```

---

## ขั้นตอนที่ 626: Projections - Read Models from Events

```csharp
// Projection: build read model from event stream
public class AccountSummaryProjection
{
    private readonly Dictionary<Guid, AccountSummaryReadModel> _accounts = new();
    
    // Process each event type
    public void Apply(AccountOpenedEvent e)
    {
        _accounts[e.StreamId] = new AccountSummaryReadModel
        {
            Id = e.StreamId,
            OwnerId = e.OwnerId,
            Balance = e.InitialBalance,
            Status = "Active",
            TransactionCount = 0,
            OpenedAt = e.OccurredAt
        };
    }
    
    public void Apply(MoneyDepositedEvent e)
    {
        if (_accounts.TryGetValue(e.StreamId, out var acc))
        {
            acc.Balance += e.Amount;
            acc.TransactionCount++;
            acc.LastTransactionAt = e.OccurredAt;
        }
    }
    
    public void Apply(MoneyWithdrawnEvent e)
    {
        if (_accounts.TryGetValue(e.StreamId, out var acc))
        {
            acc.Balance -= e.Amount;
            acc.TransactionCount++;
            acc.LastTransactionAt = e.OccurredAt;
        }
    }
    
    public void Apply(AccountFrozenEvent e)
    {
        if (_accounts.TryGetValue(e.StreamId, out var acc))
            acc.Status = "Frozen";
    }
    
    public void Apply(AccountClosedEvent e)
    {
        if (_accounts.TryGetValue(e.StreamId, out var acc))
            acc.Status = "Closed";
    }
    
    // Process any event using dynamic dispatch
    public void Process(IDomainEvent evt) => Apply((dynamic)evt);
    
    public AccountSummaryReadModel? Get(Guid id) => _accounts.GetValueOrDefault(id);
    public IEnumerable<AccountSummaryReadModel> GetAll() => _accounts.Values;
    public IEnumerable<AccountSummaryReadModel> GetByOwner(string ownerId)
        => _accounts.Values.Where(a => a.OwnerId == ownerId);
}

public class AccountSummaryReadModel
{
    public Guid Id { get; set; }
    public string OwnerId { get; set; } = "";
    public decimal Balance { get; set; }
    public string Status { get; set; } = "";
    public int TransactionCount { get; set; }
    public DateTime OpenedAt { get; set; }
    public DateTime? LastTransactionAt { get; set; }
}
```

---

## ขั้นตอนที่ 627-630: Complete Demo

```csharp
// Program.cs - Event Sourcing Demo
var store = new SqliteEventStore("Data Source=eventstore.db");
var repo = new BankAccountRepository(store);
var projection = new AccountSummaryProjection();

Console.WriteLine("=== Event Sourcing Demo ===\n");

// Open account
var account = BankAccount.Open("user123", initialBalance: 5000);
await repo.SaveAsync(account);
Console.WriteLine($"✅ เปิดบัญชี: {account.Id}");
Console.WriteLine($"   ยอดเงิน: ฿{account.Balance:N0}");

// Operations
account.Deposit(1500, "โอนเงินจากธนาคาร");
account.Deposit(2000, "เงินเดือน");
account.Withdraw(800, "ค่าอาหาร");
account.Withdraw(1200, "ค่าเช่า");
await repo.SaveAsync(account);
Console.WriteLine($"\n📊 หลังธุรกรรม:");
Console.WriteLine($"   ยอดเงิน: ฿{account.Balance:N0} (คาดหวัง: ฿6,500)");
Console.WriteLine($"   Version: {account.Version}");

// Reload from events (prove state reconstruction)
var reloaded = await repo.GetByIdAsync(account.Id);
Console.WriteLine($"\n🔄 โหลดจาก Event Store:");
Console.WriteLine($"   ยอดเงิน: ฿{reloaded!.Balance:N0} (ตรงกัน: {reloaded.Balance == account.Balance})");

// Build projection (read model)
var allEvents = await store.LoadAsync(account.Id);
foreach (var evt in allEvents)
    projection.Process(evt);

var summary = projection.Get(account.Id)!;
Console.WriteLine($"\n📈 Read Model:");
Console.WriteLine($"   เจ้าของ: {summary.OwnerId}");
Console.WriteLine($"   ยอดเงิน: ฿{summary.Balance:N0}");
Console.WriteLine($"   จำนวนธุรกรรม: {summary.TransactionCount}");
Console.WriteLine($"   สถานะ: {summary.Status}");

// Show all events (audit trail)
Console.WriteLine($"\n📜 Event History ({allEvents.Count} events):");
foreach (var evt in allEvents)
    Console.WriteLine($"  v{evt.Version}: {evt.EventType} @ {evt.OccurredAt:HH:mm:ss}");

// Freeze account
account.Freeze("ยื่นเอกสารยืนยันตัวตนไม่ครบ");
await repo.SaveAsync(account);

// Try to withdraw from frozen account
try
{
    account.Withdraw(100, "test");
}
catch (DomainException ex)
{
    Console.WriteLine($"\n⛔ {ex.Message}");
}

Console.WriteLine("\n=== Demo สำเร็จ ===");
```

---

## 📝 สรุป Part 63

| Concept | อธิบาย |
|---------|--------|
| Event Store | Append-only log ของ events |
| Aggregate | สร้าง state จาก events |
| Projection | สร้าง read model จาก events |
| Eventual Consistency | Read model อาจล่าช้าเล็กน้อย |
| Optimistic Concurrency | ป้องกัน race condition ด้วย version |
| Audit Trail | ประวัติทุก operation ยังอยู่ |

ข้อดี Event Sourcing:
- Full audit trail ไม่มีข้อมูลหาย
- สร้าง read model ใหม่ได้ตลอดเวลา
- Debug ง่ายเพราะเห็น history

ข้อเสีย:
- Complexity สูงกว่า CRUD ธรรมดา
- Query ยากกว่า (ต้องมี projection)
- Event schema evolution ต้องระวัง

---

**ก่อนหน้า → [Part 62: CQRS](part62-cqrs.md)**  
**ต่อไป → [Part 64: Microservices Introduction](part64-microservices.md)**
