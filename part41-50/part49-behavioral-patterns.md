# Part 49: Behavioral Design Patterns ใน C#

Behavioral Design Patterns เป็นกลุ่ม Design Pattern ที่มุ่งเน้นการจัดการการสื่อสารและการทำงานร่วมกันระหว่าง Objects ช่วยให้โค้ดมีความยืดหยุ่น บำรุงรักษาได้ง่าย และลด coupling ระหว่าง components

---

## ขั้นตอนที่ 481: Observer Pattern — การแจ้งเตือนเหตุการณ์

**Observer Pattern** คือ Pattern ที่กำหนดความสัมพันธ์แบบ one-to-many ระหว่าง Objects เมื่อ Object หนึ่ง (Subject) เปลี่ยนสถานะ ก็จะแจ้งเตือน Objects ที่สนใจ (Observers) ทั้งหมดโดยอัตโนมัติ

### แนวคิดหลัก
- **Subject (Observable)**: Object ที่ถูกสังเกตการณ์ มีข้อมูลที่เปลี่ยนแปลง
- **Observer**: Object ที่ต้องการรับข้อมูลเมื่อ Subject เปลี่ยนแปลง
- **Event**: กลไกใน C# ที่ทำให้ Observer Pattern สะดวกและปลอดภัยยิ่งขึ้น

### ตัวอย่าง: ระบบติดตามราคาหุ้น (Stock Price Monitor)

```csharp
using System;
using System.Collections.Generic;
using System.Reactive.Linq;
using System.Reactive.Subjects;

// ==========================================
// Observer Pattern - Stock Price Monitor
// ==========================================

// Event Args สำหรับข้อมูลราคาหุ้น
public class StockPriceEventArgs : EventArgs
{
    public string Symbol { get; set; }
    public decimal OldPrice { get; set; }
    public decimal NewPrice { get; set; }
    public DateTime Timestamp { get; set; }
    public decimal ChangePercent => OldPrice == 0 ? 0 : ((NewPrice - OldPrice) / OldPrice) * 100;
}

// Interface สำหรับ Observer
public interface IStockObserver
{
    string ObserverName { get; }
    void OnPriceChanged(object sender, StockPriceEventArgs e);
}

// Subject: ระบบติดตามราคาหุ้น
public class StockPriceMonitor
{
    private readonly Dictionary<string, decimal> _prices = new();
    
    // ใช้ C# Events สำหรับ Observer Pattern
    public event EventHandler<StockPriceEventArgs> PriceChanged;
    public event EventHandler<StockPriceEventArgs> PriceDropped;
    public event EventHandler<StockPriceEventArgs> PriceRisen;

    private readonly List<IStockObserver> _observers = new();

    public string Name { get; }

    public StockPriceMonitor(string name)
    {
        Name = name;
    }

    // เพิ่ม Observer เข้าระบบ
    public void Subscribe(IStockObserver observer)
    {
        _observers.Add(observer);
        PriceChanged += observer.OnPriceChanged;
        Console.WriteLine($"[Monitor] {observer.ObserverName} subscribed to {Name}");
    }

    // ลบ Observer ออกจากระบบ
    public void Unsubscribe(IStockObserver observer)
    {
        _observers.Remove(observer);
        PriceChanged -= observer.OnPriceChanged;
        Console.WriteLine($"[Monitor] {observer.ObserverName} unsubscribed from {Name}");
    }

    // อัปเดตราคาหุ้นและแจ้งเตือน Observers
    public void UpdatePrice(string symbol, decimal newPrice)
    {
        var oldPrice = _prices.GetValueOrDefault(symbol, 0);
        _prices[symbol] = newPrice;

        var args = new StockPriceEventArgs
        {
            Symbol = symbol,
            OldPrice = oldPrice,
            NewPrice = newPrice,
            Timestamp = DateTime.Now
        };

        // แจ้งเตือน Observers ทั้งหมดผ่าน Event
        OnPriceChanged(args);

        if (newPrice < oldPrice && oldPrice > 0)
            PriceDropped?.Invoke(this, args);
        else if (newPrice > oldPrice)
            PriceRisen?.Invoke(this, args);
    }

    protected virtual void OnPriceChanged(StockPriceEventArgs args)
    {
        PriceChanged?.Invoke(this, args);
    }

    public decimal GetCurrentPrice(string symbol) =>
        _prices.GetValueOrDefault(symbol, 0);
}

// Observer 1: แจ้งเตือนราคา
public class PriceAlert : IStockObserver
{
    public string ObserverName => "PriceAlert";
    private readonly decimal _threshold;

    public PriceAlert(decimal threshold)
    {
        _threshold = threshold;
    }

    public void OnPriceChanged(object sender, StockPriceEventArgs e)
    {
        if (Math.Abs(e.ChangePercent) >= _threshold)
        {
            var direction = e.ChangePercent >= 0 ? "เพิ่มขึ้น" : "ลดลง";
            Console.WriteLine($"[ALERT] {e.Symbol} ราคา{direction} {Math.Abs(e.ChangePercent):F2}% " +
                            $"({e.OldPrice:F2} → {e.NewPrice:F2})");
        }
    }
}

// Observer 2: บันทึก Log ราคา
public class PriceLogger : IStockObserver
{
    public string ObserverName => "PriceLogger";
    private readonly List<string> _log = new();

    public void OnPriceChanged(object sender, StockPriceEventArgs e)
    {
        var entry = $"[{e.Timestamp:HH:mm:ss}] {e.Symbol}: {e.OldPrice:F2} → {e.NewPrice:F2}";
        _log.Add(entry);
        Console.WriteLine($"[LOG] {entry}");
    }

    public IReadOnlyList<string> GetLog() => _log.AsReadOnly();
}

// Observer 3: อัปเดตกราฟ
public class ChartUpdater : IStockObserver
{
    public string ObserverName => "ChartUpdater";
    private readonly Dictionary<string, List<decimal>> _chartData = new();

    public void OnPriceChanged(object sender, StockPriceEventArgs e)
    {
        if (!_chartData.ContainsKey(e.Symbol))
            _chartData[e.Symbol] = new List<decimal>();

        _chartData[e.Symbol].Add(e.NewPrice);
        Console.WriteLine($"[CHART] อัปเดตกราฟ {e.Symbol}: {_chartData[e.Symbol].Count} จุดข้อมูล");
    }

    public IReadOnlyList<decimal> GetChartData(string symbol) =>
        _chartData.GetValueOrDefault(symbol, new List<decimal>()).AsReadOnly();
}

// ตัวอย่างการใช้ IObservable<T> (Reactive Extensions)
public class ReactiveStockMonitor
{
    private readonly Subject<StockPriceEventArgs> _priceSubject = new();
    
    public IObservable<StockPriceEventArgs> PriceStream => _priceSubject.AsObservable();

    public void UpdatePrice(string symbol, decimal newPrice, decimal oldPrice)
    {
        _priceSubject.OnNext(new StockPriceEventArgs
        {
            Symbol = symbol,
            OldPrice = oldPrice,
            NewPrice = newPrice,
            Timestamp = DateTime.Now
        });
    }

    public IDisposable SubscribeToLargeChanges(decimal threshold, Action<StockPriceEventArgs> handler)
    {
        return PriceStream
            .Where(e => Math.Abs(e.ChangePercent) >= threshold)
            .Subscribe(handler);
    }
}

// โปรแกรมทดสอบ Observer Pattern
class ObserverDemo
{
    static void Main()
    {
        Console.WriteLine("=== Observer Pattern Demo ===\n");

        var monitor = new StockPriceMonitor("SET Stock Exchange");

        // สร้าง Observers
        var alert = new PriceAlert(threshold: 2.0m);
        var logger = new PriceLogger();
        var chart = new ChartUpdater();

        // Subscribe Observers
        monitor.Subscribe(alert);
        monitor.Subscribe(logger);
        monitor.Subscribe(chart);

        // อัปเดตราคาหุ้น
        Console.WriteLine("\n--- อัปเดตราคาหุ้น ---");
        monitor.UpdatePrice("PTT", 35.50m);
        monitor.UpdatePrice("PTT", 36.00m);
        monitor.UpdatePrice("ADVANC", 200.00m);
        monitor.UpdatePrice("ADVANC", 195.00m);  // ลดลง 2.5%
        monitor.UpdatePrice("PTT", 34.00m);       // ลดลง ~5.5%

        // Unsubscribe Observer หนึ่งตัว
        Console.WriteLine("\n--- Unsubscribe ChartUpdater ---");
        monitor.Unsubscribe(chart);
        monitor.UpdatePrice("PTT", 35.00m);

        Console.WriteLine("\n--- Log ทั้งหมด ---");
        foreach (var entry in logger.GetLog())
            Console.WriteLine(entry);
    }
}
```

---

## ขั้นตอนที่ 482: Strategy Pattern — อัลกอริทึมที่สับเปลี่ยนได้

**Strategy Pattern** ช่วยให้เราสามารถกำหนดกลุ่มของอัลกอริทึม (Strategies) แยกแต่ละตัวออกมา และทำให้สามารถสับเปลี่ยนกันได้โดยไม่ต้องแก้ไข Client Code

### แนวคิดหลัก
- **Context**: Object ที่ใช้ Strategy ในการทำงาน
- **Strategy Interface**: กำหนด Contract ที่ Strategies ทั้งหมดต้องปฏิบัติตาม
- **Concrete Strategies**: การ Implement อัลกอริทึมแต่ละแบบ

### ตัวอย่าง: SortingContext และ PaymentStrategy

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

// ==========================================
// Strategy Pattern - Sorting Algorithms
// ==========================================

// Strategy Interface สำหรับการเรียงลำดับ
public interface ISortStrategy<T> where T : IComparable<T>
{
    string Name { get; }
    IList<T> Sort(IList<T> data);
    int ComparisonCount { get; }
}

// Concrete Strategy: Bubble Sort
public class BubbleSort<T> : ISortStrategy<T> where T : IComparable<T>
{
    public string Name => "Bubble Sort";
    public int ComparisonCount { get; private set; }

    public IList<T> Sort(IList<T> data)
    {
        ComparisonCount = 0;
        var result = new List<T>(data);
        int n = result.Count;

        for (int i = 0; i < n - 1; i++)
        {
            for (int j = 0; j < n - i - 1; j++)
            {
                ComparisonCount++;
                if (result[j].CompareTo(result[j + 1]) > 0)
                {
                    (result[j], result[j + 1]) = (result[j + 1], result[j]);
                }
            }
        }
        return result;
    }
}

// Concrete Strategy: Quick Sort
public class QuickSort<T> : ISortStrategy<T> where T : IComparable<T>
{
    public string Name => "Quick Sort";
    public int ComparisonCount { get; private set; }

    public IList<T> Sort(IList<T> data)
    {
        ComparisonCount = 0;
        var result = new List<T>(data);
        QuickSortImpl(result, 0, result.Count - 1);
        return result;
    }

    private void QuickSortImpl(List<T> arr, int low, int high)
    {
        if (low < high)
        {
            int pivot = Partition(arr, low, high);
            QuickSortImpl(arr, low, pivot - 1);
            QuickSortImpl(arr, pivot + 1, high);
        }
    }

    private int Partition(List<T> arr, int low, int high)
    {
        var pivot = arr[high];
        int i = low - 1;

        for (int j = low; j < high; j++)
        {
            ComparisonCount++;
            if (arr[j].CompareTo(pivot) <= 0)
            {
                i++;
                (arr[i], arr[j]) = (arr[j], arr[i]);
            }
        }
        (arr[i + 1], arr[high]) = (arr[high], arr[i + 1]);
        return i + 1;
    }
}

// Concrete Strategy: Merge Sort
public class MergeSort<T> : ISortStrategy<T> where T : IComparable<T>
{
    public string Name => "Merge Sort";
    public int ComparisonCount { get; private set; }

    public IList<T> Sort(IList<T> data)
    {
        ComparisonCount = 0;
        return MergeSortImpl(new List<T>(data));
    }

    private List<T> MergeSortImpl(List<T> arr)
    {
        if (arr.Count <= 1) return arr;

        int mid = arr.Count / 2;
        var left = MergeSortImpl(arr.Take(mid).ToList());
        var right = MergeSortImpl(arr.Skip(mid).ToList());
        return Merge(left, right);
    }

    private List<T> Merge(List<T> left, List<T> right)
    {
        var result = new List<T>();
        int i = 0, j = 0;

        while (i < left.Count && j < right.Count)
        {
            ComparisonCount++;
            if (left[i].CompareTo(right[j]) <= 0)
                result.Add(left[i++]);
            else
                result.Add(right[j++]);
        }

        result.AddRange(left.Skip(i));
        result.AddRange(right.Skip(j));
        return result;
    }
}

// Context: ใช้ Strategy สำหรับการเรียงลำดับ
public class SortingContext<T> where T : IComparable<T>
{
    private ISortStrategy<T> _strategy;

    public SortingContext(ISortStrategy<T> strategy)
    {
        _strategy = strategy;
    }

    // เปลี่ยน Strategy ได้ที่ Runtime
    public void SetStrategy(ISortStrategy<T> strategy)
    {
        Console.WriteLine($"[Context] เปลี่ยนกลยุทธ์เป็น: {strategy.Name}");
        _strategy = strategy;
    }

    public IList<T> Sort(IList<T> data)
    {
        Console.WriteLine($"[Context] กำลังเรียงลำดับด้วย {_strategy.Name}...");
        var result = _strategy.Sort(data);
        Console.WriteLine($"[Context] เสร็จสิ้น! จำนวนการเปรียบเทียบ: {_strategy.ComparisonCount}");
        return result;
    }
}

// ==========================================
// Strategy Pattern - Payment System
// ==========================================

public class PaymentContext
{
    public string OrderId { get; set; }
    public decimal Amount { get; set; }
    public string CustomerName { get; set; }
}

public interface IPaymentStrategy
{
    string MethodName { get; }
    bool ValidatePayment(PaymentContext context);
    bool ProcessPayment(PaymentContext context);
    decimal CalculateFee(decimal amount);
}

// Payment Strategy: บัตรเครดิต
public class CreditCardPayment : IPaymentStrategy
{
    public string MethodName => "Credit Card";
    private readonly string _cardNumber;
    private readonly string _cvv;

    public CreditCardPayment(string cardNumber, string cvv)
    {
        _cardNumber = cardNumber;
        _cvv = cvv;
    }

    public bool ValidatePayment(PaymentContext context)
    {
        // ตรวจสอบความถูกต้องของบัตร (ตัวอย่างง่ายๆ)
        bool valid = _cardNumber.Length == 16 && _cvv.Length == 3 && context.Amount > 0;
        Console.WriteLine($"[CreditCard] ตรวจสอบบัตร: {(valid ? "ผ่าน" : "ไม่ผ่าน")}");
        return valid;
    }

    public bool ProcessPayment(PaymentContext context)
    {
        Console.WriteLine($"[CreditCard] ชำระเงิน {context.Amount + CalculateFee(context.Amount):F2} บาท " +
                         $"สำหรับคำสั่งซื้อ {context.OrderId}");
        return true;
    }

    public decimal CalculateFee(decimal amount) => amount * 0.015m; // ค่าธรรมเนียม 1.5%
}

// Payment Strategy: PromptPay
public class PromptPayPayment : IPaymentStrategy
{
    public string MethodName => "PromptPay";
    private readonly string _phoneNumber;

    public PromptPayPayment(string phoneNumber)
    {
        _phoneNumber = phoneNumber;
    }

    public bool ValidatePayment(PaymentContext context)
    {
        bool valid = _phoneNumber.Length == 10 && context.Amount > 0 && context.Amount <= 1000000;
        Console.WriteLine($"[PromptPay] ตรวจสอบหมายเลข {_phoneNumber}: {(valid ? "ผ่าน" : "ไม่ผ่าน")}");
        return valid;
    }

    public bool ProcessPayment(PaymentContext context)
    {
        Console.WriteLine($"[PromptPay] โอนเงิน {context.Amount:F2} บาท ไปยัง {_phoneNumber} " +
                         $"สำหรับคำสั่งซื้อ {context.OrderId}");
        return true;
    }

    public decimal CalculateFee(decimal amount) => 0; // ฟรี
}

// Payment Strategy: Cryptocurrency
public class CryptoPayment : IPaymentStrategy
{
    public string MethodName => "Cryptocurrency";
    private readonly string _walletAddress;
    private readonly string _currency;

    public CryptoPayment(string walletAddress, string currency = "BTC")
    {
        _walletAddress = walletAddress;
        _currency = currency;
    }

    public bool ValidatePayment(PaymentContext context)
    {
        bool valid = _walletAddress.Length >= 26 && context.Amount > 0;
        Console.WriteLine($"[Crypto] ตรวจสอบ Wallet: {(valid ? "ผ่าน" : "ไม่ผ่าน")}");
        return valid;
    }

    public bool ProcessPayment(PaymentContext context)
    {
        decimal cryptoAmount = context.Amount / 1500000; // สมมติ BTC rate
        Console.WriteLine($"[Crypto] ชำระเงิน {cryptoAmount:F8} {_currency} " +
                         $"({context.Amount:F2} THB) ไปยัง {_walletAddress[..8]}...");
        return true;
    }

    public decimal CalculateFee(decimal amount) => amount * 0.005m; // ค่าธรรมเนียม 0.5%
}

// Payment Processor Context
public class PaymentProcessor
{
    private IPaymentStrategy _strategy;

    public PaymentProcessor(IPaymentStrategy strategy)
    {
        _strategy = strategy;
    }

    public void ChangePaymentMethod(IPaymentStrategy strategy)
    {
        Console.WriteLine($"\n[Processor] เปลี่ยนวิธีชำระเงินเป็น: {strategy.MethodName}");
        _strategy = strategy;
    }

    public bool ProcessOrder(PaymentContext context)
    {
        Console.WriteLine($"\n[Processor] ประมวลผลคำสั่งซื้อ {context.OrderId}");
        Console.WriteLine($"[Processor] ลูกค้า: {context.CustomerName}, จำนวนเงิน: {context.Amount:F2} บาท");
        Console.WriteLine($"[Processor] วิธีชำระเงิน: {_strategy.MethodName}");
        Console.WriteLine($"[Processor] ค่าธรรมเนียม: {_strategy.CalculateFee(context.Amount):F2} บาท");

        if (!_strategy.ValidatePayment(context))
        {
            Console.WriteLine("[Processor] การชำระเงินล้มเหลว: การตรวจสอบไม่ผ่าน");
            return false;
        }

        bool success = _strategy.ProcessPayment(context);
        Console.WriteLine($"[Processor] ผลการชำระเงิน: {(success ? "สำเร็จ" : "ล้มเหลว")}");
        return success;
    }
}

// โปรแกรมทดสอบ Strategy Pattern
class StrategyDemo
{
    static void Main()
    {
        Console.WriteLine("=== Strategy Pattern Demo ===\n");

        // ทดสอบ Sorting Strategies
        Console.WriteLine("--- Sorting Strategies ---");
        var data = new List<int> { 64, 25, 12, 22, 11, 90, 33, 47, 5, 78 };
        Console.WriteLine($"ข้อมูลเริ่มต้น: [{string.Join(", ", data)}]");

        var context = new SortingContext<int>(new BubbleSort<int>());
        var sorted = context.Sort(data);
        Console.WriteLine($"ผลลัพธ์: [{string.Join(", ", sorted)}]\n");

        context.SetStrategy(new QuickSort<int>());
        sorted = context.Sort(data);
        Console.WriteLine($"ผลลัพธ์: [{string.Join(", ", sorted)}]\n");

        context.SetStrategy(new MergeSort<int>());
        sorted = context.Sort(data);
        Console.WriteLine($"ผลลัพธ์: [{string.Join(", ", sorted)}]\n");

        // ทดสอบ Payment Strategies
        Console.WriteLine("\n--- Payment Strategies ---");
        var order = new PaymentContext
        {
            OrderId = "ORD-2024-001",
            CustomerName = "สมชาย ใจดี",
            Amount = 5000m
        };

        var processor = new PaymentProcessor(new CreditCardPayment("1234567890123456", "123"));
        processor.ProcessOrder(order);

        processor.ChangePaymentMethod(new PromptPayPayment("0891234567"));
        processor.ProcessOrder(order);

        processor.ChangePaymentMethod(new CryptoPayment("1A1zP1eP5QGefi2DMPTfTL5SLmv7Divf", "BTC"));
        processor.ProcessOrder(order);
    }
}
```

---

## ขั้นตอนที่ 483: Command Pattern — การห่อหุ้ม Request

**Command Pattern** แปลง Request หรือการทำงานให้เป็น Object อิสระ ทำให้สามารถส่งต่อ, บันทึก, และ Undo/Redo การทำงานได้

### แนวคิดหลัก
- **Command Interface**: กำหนดเมธอด Execute และ Undo
- **Concrete Commands**: การ Implement คำสั่งแต่ละแบบ
- **Invoker**: เรียกใช้และจัดการ Commands
- **Receiver**: Object ที่ได้รับการดำเนินการจาก Command

### ตัวอย่าง: Text Editor พร้อม Undo/Redo

```csharp
using System;
using System.Collections.Generic;
using System.Text;

// ==========================================
// Command Pattern - Text Editor
// ==========================================

// Command Interface
public interface ICommand
{
    string Description { get; }
    void Execute();
    void Undo();
}

// Receiver: Text Document
public class TextDocument
{
    private readonly StringBuilder _content = new();
    public string Content => _content.ToString();
    public int CursorPosition { get; set; }

    public void InsertText(int position, string text)
    {
        position = Math.Clamp(position, 0, _content.Length);
        _content.Insert(position, text);
        CursorPosition = position + text.Length;
    }

    public string DeleteText(int position, int length)
    {
        position = Math.Clamp(position, 0, _content.Length);
        length = Math.Min(length, _content.Length - position);
        var deleted = _content.ToString(position, length);
        _content.Remove(position, length);
        CursorPosition = position;
        return deleted;
    }

    public void SetText(int position, int length, string newText)
    {
        position = Math.Clamp(position, 0, _content.Length);
        length = Math.Min(length, _content.Length - position);
        _content.Remove(position, length);
        _content.Insert(position, newText);
    }

    public string GetText(int position, int length)
    {
        position = Math.Clamp(position, 0, _content.Length);
        length = Math.Min(length, _content.Length - position);
        return _content.ToString(position, length);
    }

    public override string ToString() => _content.ToString();
}

// Concrete Command: แทรกข้อความ
public class InsertTextCommand : ICommand
{
    private readonly TextDocument _document;
    private readonly int _position;
    private readonly string _text;

    public string Description => $"แทรก '{_text}' ที่ตำแหน่ง {_position}";

    public InsertTextCommand(TextDocument document, int position, string text)
    {
        _document = document;
        _position = position;
        _text = text;
    }

    public void Execute()
    {
        _document.InsertText(_position, _text);
    }

    public void Undo()
    {
        _document.DeleteText(_position, _text.Length);
    }
}

// Concrete Command: ลบข้อความ
public class DeleteTextCommand : ICommand
{
    private readonly TextDocument _document;
    private readonly int _position;
    private readonly int _length;
    private string _deletedText;

    public string Description => $"ลบข้อความ {_length} ตัวอักษร ที่ตำแหน่ง {_position}";

    public DeleteTextCommand(TextDocument document, int position, int length)
    {
        _document = document;
        _position = position;
        _length = length;
    }

    public void Execute()
    {
        _deletedText = _document.DeleteText(_position, _length);
    }

    public void Undo()
    {
        _document.InsertText(_position, _deletedText);
    }
}

// Concrete Command: จัดรูปแบบข้อความ (ใส่ Tag)
public class FormatCommand : ICommand
{
    private readonly TextDocument _document;
    private readonly int _position;
    private readonly int _length;
    private readonly string _formatTag;
    private string _originalText;

    public string Description => $"จัดรูปแบบด้วย <{_formatTag}> ที่ตำแหน่ง {_position}";

    public FormatCommand(TextDocument document, int position, int length, string formatTag)
    {
        _document = document;
        _position = position;
        _length = length;
        _formatTag = formatTag;
    }

    public void Execute()
    {
        _originalText = _document.GetText(_position, _length);
        string formatted = $"<{_formatTag}>{_originalText}</{_formatTag}>";
        _document.SetText(_position, _length, formatted);
    }

    public void Undo()
    {
        var currentText = $"<{_formatTag}>{_originalText}</{_formatTag}>";
        _document.SetText(_position, currentText.Length, _originalText);
    }
}

// Invoker: Undo/Redo Manager
public class UndoRedoManager
{
    private readonly Stack<ICommand> _undoStack = new();
    private readonly Stack<ICommand> _redoStack = new();
    private readonly int _maxHistory;

    public UndoRedoManager(int maxHistory = 50)
    {
        _maxHistory = maxHistory;
    }

    public bool CanUndo => _undoStack.Count > 0;
    public bool CanRedo => _redoStack.Count > 0;
    public int UndoCount => _undoStack.Count;
    public int RedoCount => _redoStack.Count;

    // ดำเนินการคำสั่งและบันทึกใน Undo Stack
    public void Execute(ICommand command)
    {
        command.Execute();
        _undoStack.Push(command);
        _redoStack.Clear(); // ล้าง Redo Stack เมื่อมีคำสั่งใหม่

        // จำกัดขนาด History
        if (_undoStack.Count > _maxHistory)
        {
            var temp = new Stack<ICommand>();
            int keep = _maxHistory;
            foreach (var cmd in _undoStack)
            {
                if (keep-- > 0) temp.Push(cmd);
            }
            _undoStack.Clear();
            foreach (var cmd in temp) _undoStack.Push(cmd);
        }

        Console.WriteLine($"[Manager] ดำเนินการ: {command.Description}");
    }

    // ยกเลิกคำสั่งล่าสุด
    public void Undo()
    {
        if (!CanUndo)
        {
            Console.WriteLine("[Manager] ไม่มีคำสั่งที่สามารถยกเลิกได้");
            return;
        }

        var command = _undoStack.Pop();
        command.Undo();
        _redoStack.Push(command);
        Console.WriteLine($"[Manager] ยกเลิก: {command.Description}");
    }

    // ทำคำสั่งที่ยกเลิกไปอีกครั้ง
    public void Redo()
    {
        if (!CanRedo)
        {
            Console.WriteLine("[Manager] ไม่มีคำสั่งที่สามารถทำซ้ำได้");
            return;
        }

        var command = _redoStack.Pop();
        command.Execute();
        _undoStack.Push(command);
        Console.WriteLine($"[Manager] ทำซ้ำ: {command.Description}");
    }

    // แสดง History
    public void PrintHistory()
    {
        Console.WriteLine("\n[History] รายการคำสั่ง:");
        var list = _undoStack.ToList();
        list.Reverse();
        for (int i = 0; i < list.Count; i++)
            Console.WriteLine($"  {i + 1}. {list[i].Description}");
    }
}

// Text Editor ที่ใช้ Command Pattern
public class TextEditor
{
    private readonly TextDocument _document = new();
    private readonly UndoRedoManager _manager = new();

    public string Content => _document.Content;

    public void TypeText(string text)
    {
        var command = new InsertTextCommand(_document, _document.CursorPosition, text);
        _manager.Execute(command);
    }

    public void InsertAt(int position, string text)
    {
        var command = new InsertTextCommand(_document, position, text);
        _manager.Execute(command);
    }

    public void DeleteAt(int position, int length)
    {
        var command = new DeleteTextCommand(_document, position, length);
        _manager.Execute(command);
    }

    public void Bold(int position, int length)
    {
        var command = new FormatCommand(_document, position, length, "b");
        _manager.Execute(command);
    }

    public void Undo() => _manager.Undo();
    public void Redo() => _manager.Redo();
    public void PrintHistory() => _manager.PrintHistory();

    public void PrintContent()
    {
        Console.WriteLine($"\n[Editor] เนื้อหา: \"{_document.Content}\"");
        Console.WriteLine($"[Editor] Undo: {_manager.UndoCount}, Redo: {_manager.RedoCount}");
    }
}

// โปรแกรมทดสอบ Command Pattern
class CommandDemo
{
    static void Main()
    {
        Console.WriteLine("=== Command Pattern Demo ===\n");

        var editor = new TextEditor();

        editor.TypeText("สวัสดี");
        editor.TypeText(" ชาวโลก");
        editor.TypeText("!");
        editor.PrintContent();

        Console.WriteLine("\n--- ทดสอบ Bold ---");
        editor.Bold(0, 7);
        editor.PrintContent();

        Console.WriteLine("\n--- ทดสอบ Delete ---");
        editor.DeleteAt(0, 3);
        editor.PrintContent();

        Console.WriteLine("\n--- ทดสอบ Undo ---");
        editor.Undo();
        editor.PrintContent();

        editor.Undo();
        editor.PrintContent();

        Console.WriteLine("\n--- ทดสอบ Redo ---");
        editor.Redo();
        editor.PrintContent();

        editor.PrintHistory();
    }
}
```

---

## ขั้นตอนที่ 484: Iterator Pattern — การท่องผ่าน Collection

**Iterator Pattern** ให้วิธีการเข้าถึง Elements ของ Collection ตามลำดับโดยไม่ต้องเปิดเผยโครงสร้างภายใน

### แนวคิดหลัก
- **Iterator Interface**: กำหนดวิธีการท่องผ่านข้อมูล (HasNext, Next)
- **IEnumerable<T>**: Interface ใน C# ที่รองรับ foreach loop
- **Custom Collections**: การสร้าง Collection ที่มีการท่องผ่านแบบพิเศษ

### ตัวอย่าง: Binary Tree พร้อม InOrder/PreOrder/PostOrder Traversal

```csharp
using System;
using System.Collections;
using System.Collections.Generic;
using System.Linq;

// ==========================================
// Iterator Pattern - Binary Tree
// ==========================================

// Node ของ Binary Tree
public class TreeNode<T>
{
    public T Value { get; set; }
    public TreeNode<T> Left { get; set; }
    public TreeNode<T> Right { get; set; }

    public TreeNode(T value)
    {
        Value = value;
    }
}

// Enum สำหรับประเภทการท่องผ่าน
public enum TraversalOrder
{
    InOrder,    // ซ้าย → รูท → ขวา
    PreOrder,   // รูท → ซ้าย → ขวา
    PostOrder   // ซ้าย → ขวา → รูท
}

// Binary Tree ที่รองรับการท่องผ่านหลายแบบ
public class BinaryTree<T> : IEnumerable<T> where T : IComparable<T>
{
    private TreeNode<T> _root;
    private TraversalOrder _traversalOrder = TraversalOrder.InOrder;

    // แทรก Node ใหม่ (BST)
    public void Insert(T value)
    {
        _root = InsertNode(_root, value);
    }

    private TreeNode<T> InsertNode(TreeNode<T> node, T value)
    {
        if (node == null) return new TreeNode<T>(value);

        int cmp = value.CompareTo(node.Value);
        if (cmp < 0)
            node.Left = InsertNode(node.Left, value);
        else if (cmp > 0)
            node.Right = InsertNode(node.Right, value);

        return node;
    }

    // กำหนดประเภทการท่องผ่าน
    public BinaryTree<T> WithTraversal(TraversalOrder order)
    {
        _traversalOrder = order;
        return this;
    }

    // IEnumerable<T> Implementation
    public IEnumerator<T> GetEnumerator()
    {
        return _traversalOrder switch
        {
            TraversalOrder.InOrder => GetInOrderEnumerator(),
            TraversalOrder.PreOrder => GetPreOrderEnumerator(),
            TraversalOrder.PostOrder => GetPostOrderEnumerator(),
            _ => GetInOrderEnumerator()
        };
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();

    // Iterator สำหรับ InOrder Traversal (ใช้ yield return)
    private IEnumerator<T> GetInOrderEnumerator()
    {
        foreach (var value in InOrder(_root))
            yield return value;
    }

    private IEnumerable<T> InOrder(TreeNode<T> node)
    {
        if (node == null) yield break;
        foreach (var v in InOrder(node.Left)) yield return v;
        yield return node.Value;
        foreach (var v in InOrder(node.Right)) yield return v;
    }

    // Iterator สำหรับ PreOrder Traversal
    private IEnumerator<T> GetPreOrderEnumerator()
    {
        foreach (var value in PreOrder(_root))
            yield return value;
    }

    private IEnumerable<T> PreOrder(TreeNode<T> node)
    {
        if (node == null) yield break;
        yield return node.Value;
        foreach (var v in PreOrder(node.Left)) yield return v;
        foreach (var v in PreOrder(node.Right)) yield return v;
    }

    // Iterator สำหรับ PostOrder Traversal
    private IEnumerator<T> GetPostOrderEnumerator()
    {
        foreach (var value in PostOrder(_root))
            yield return value;
    }

    private IEnumerable<T> PostOrder(TreeNode<T> node)
    {
        if (node == null) yield break;
        foreach (var v in PostOrder(node.Left)) yield return v;
        foreach (var v in PostOrder(node.Right)) yield return v;
        yield return node.Value;
    }

    // Iterator แบบ Level-Order (BFS) โดยใช้ Queue
    public IEnumerable<T> LevelOrder()
    {
        if (_root == null) yield break;

        var queue = new Queue<TreeNode<T>>();
        queue.Enqueue(_root);

        while (queue.Count > 0)
        {
            var node = queue.Dequeue();
            yield return node.Value;

            if (node.Left != null) queue.Enqueue(node.Left);
            if (node.Right != null) queue.Enqueue(node.Right);
        }
    }
}

// Custom Iterator: Page-Based Collection
public class PagedCollection<T> : IEnumerable<IEnumerable<T>>
{
    private readonly List<T> _items;
    private readonly int _pageSize;

    public PagedCollection(IEnumerable<T> items, int pageSize)
    {
        _items = new List<T>(items);
        _pageSize = pageSize;
    }

    public int TotalPages => (int)Math.Ceiling(_items.Count / (double)_pageSize);
    public int TotalItems => _items.Count;

    // ดึงข้อมูลทีละหน้า
    public IEnumerable<T> GetPage(int pageNumber)
    {
        int skip = (pageNumber - 1) * _pageSize;
        return _items.Skip(skip).Take(_pageSize);
    }

    // IEnumerable สำหรับ foreach ที่วนผ่านทีละหน้า
    public IEnumerator<IEnumerable<T>> GetEnumerator()
    {
        for (int page = 1; page <= TotalPages; page++)
        {
            yield return GetPage(page);
        }
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

// โปรแกรมทดสอบ Iterator Pattern
class IteratorDemo
{
    static void Main()
    {
        Console.WriteLine("=== Iterator Pattern Demo ===\n");

        var tree = new BinaryTree<int>();
        int[] values = { 50, 30, 70, 20, 40, 60, 80, 10, 25 };

        Console.WriteLine("แทรกค่า: " + string.Join(", ", values));
        foreach (var v in values) tree.Insert(v);

        Console.WriteLine("\n--- InOrder Traversal (เรียงลำดับ) ---");
        tree.WithTraversal(TraversalOrder.InOrder);
        Console.Write("ผลลัพธ์: ");
        Console.WriteLine(string.Join(" → ", tree));

        Console.WriteLine("\n--- PreOrder Traversal ---");
        tree.WithTraversal(TraversalOrder.PreOrder);
        Console.Write("ผลลัพธ์: ");
        Console.WriteLine(string.Join(" → ", tree));

        Console.WriteLine("\n--- PostOrder Traversal ---");
        tree.WithTraversal(TraversalOrder.PostOrder);
        Console.Write("ผลลัพธ์: ");
        Console.WriteLine(string.Join(" → ", tree));

        Console.WriteLine("\n--- Level-Order Traversal (BFS) ---");
        Console.Write("ผลลัพธ์: ");
        Console.WriteLine(string.Join(" → ", tree.LevelOrder()));

        Console.WriteLine("\n--- Paged Collection ---");
        var numbers = Enumerable.Range(1, 25).ToList();
        var paged = new PagedCollection<int>(numbers, 5);

        Console.WriteLine($"จำนวนทั้งหมด: {paged.TotalItems}, จำนวนหน้า: {paged.TotalPages}");
        int pageNum = 1;
        foreach (var page in paged)
        {
            Console.WriteLine($"หน้า {pageNum++}: [{string.Join(", ", page)}]");
        }
    }
}
```

---

## ขั้นตอนที่ 485: State Pattern — พฤติกรรมตามสถานะ

**State Pattern** ให้ Object เปลี่ยนพฤติกรรมเมื่อสถานะภายในเปลี่ยนแปลง เสมือนว่า Object นั้นเปลี่ยน Class ไปเลย

### แนวคิดหลัก
- **Context**: Object ที่มีสถานะและมอบหมายพฤติกรรมให้ State
- **State Interface**: กำหนด Behavior ที่ต้องมีในแต่ละสถานะ
- **Concrete States**: การ Implement พฤติกรรมในแต่ละสถานะ

### ตัวอย่าง: Order State Machine และ Traffic Light

```csharp
using System;
using System.Collections.Generic;

// ==========================================
// State Pattern - Order State Machine
// ==========================================

// Context: คำสั่งซื้อ
public class Order
{
    public string OrderId { get; }
    public string CustomerName { get; }
    public decimal TotalAmount { get; }
    public DateTime CreatedAt { get; } = DateTime.Now;
    public List<string> History { get; } = new();

    // Current State
    private IOrderState _currentState;

    public Order(string orderId, string customerName, decimal amount)
    {
        OrderId = orderId;
        CustomerName = customerName;
        TotalAmount = amount;
        _currentState = new PendingState();
        History.Add($"[{DateTime.Now:HH:mm:ss}] สร้างคำสั่งซื้อ - สถานะ: {_currentState.StateName}");
    }

    // เปลี่ยนสถานะ
    public void TransitionTo(IOrderState newState)
    {
        Console.WriteLine($"[Order {OrderId}] {_currentState.StateName} → {newState.StateName}");
        _currentState = newState;
        History.Add($"[{DateTime.Now:HH:mm:ss}] เปลี่ยนสถานะเป็น: {newState.StateName}");
    }

    public string CurrentStateName => _currentState.StateName;

    // มอบหมายการทำงานให้ State
    public void Confirm() => _currentState.Confirm(this);
    public void Process() => _currentState.Process(this);
    public void Ship() => _currentState.Ship(this);
    public void Deliver() => _currentState.Deliver(this);
    public void Cancel(string reason) => _currentState.Cancel(this, reason);

    public void PrintStatus()
    {
        Console.WriteLine($"\n[Order] ID: {OrderId}");
        Console.WriteLine($"[Order] ลูกค้า: {CustomerName}");
        Console.WriteLine($"[Order] ยอดรวม: {TotalAmount:F2} บาท");
        Console.WriteLine($"[Order] สถานะปัจจุบัน: {CurrentStateName}");
    }

    public void PrintHistory()
    {
        Console.WriteLine("\n[History] ประวัติคำสั่งซื้อ:");
        foreach (var entry in History)
            Console.WriteLine($"  {entry}");
    }
}

// State Interface
public interface IOrderState
{
    string StateName { get; }
    void Confirm(Order order);
    void Process(Order order);
    void Ship(Order order);
    void Deliver(Order order);
    void Cancel(Order order, string reason);
}

// Base State: ป้องกัน State ที่ไม่ถูกต้อง
public abstract class OrderStateBase : IOrderState
{
    public abstract string StateName { get; }

    public virtual void Confirm(Order order) =>
        Console.WriteLine($"[{StateName}] ไม่สามารถยืนยันคำสั่งซื้อได้ในสถานะนี้");

    public virtual void Process(Order order) =>
        Console.WriteLine($"[{StateName}] ไม่สามารถประมวลผลได้ในสถานะนี้");

    public virtual void Ship(Order order) =>
        Console.WriteLine($"[{StateName}] ไม่สามารถจัดส่งได้ในสถานะนี้");

    public virtual void Deliver(Order order) =>
        Console.WriteLine($"[{StateName}] ไม่สามารถยืนยันการจัดส่งได้ในสถานะนี้");

    public virtual void Cancel(Order order, string reason) =>
        Console.WriteLine($"[{StateName}] ไม่สามารถยกเลิกได้ในสถานะนี้");
}

// Concrete State: รอดำเนินการ
public class PendingState : OrderStateBase
{
    public override string StateName => "รอดำเนินการ";

    public override void Confirm(Order order)
    {
        Console.WriteLine($"[{StateName}] ยืนยันคำสั่งซื้อ {order.OrderId} แล้ว");
        order.TransitionTo(new ProcessingState());
    }

    public override void Cancel(Order order, string reason)
    {
        Console.WriteLine($"[{StateName}] ยกเลิกคำสั่งซื้อ: {reason}");
        order.TransitionTo(new CancelledState());
    }
}

// Concrete State: กำลังดำเนินการ
public class ProcessingState : OrderStateBase
{
    public override string StateName => "กำลังดำเนินการ";

    public override void Process(Order order)
    {
        Console.WriteLine($"[{StateName}] กำลังเตรียมสินค้า...");
    }

    public override void Ship(Order order)
    {
        Console.WriteLine($"[{StateName}] ส่งมอบให้บริษัทขนส่งแล้ว");
        order.TransitionTo(new ShippedState());
    }

    public override void Cancel(Order order, string reason)
    {
        Console.WriteLine($"[{StateName}] ยกเลิกคำสั่งซื้อและคืนเงิน: {reason}");
        order.TransitionTo(new CancelledState());
    }
}

// Concrete State: จัดส่งแล้ว
public class ShippedState : OrderStateBase
{
    public override string StateName => "จัดส่งแล้ว";

    public override void Deliver(Order order)
    {
        Console.WriteLine($"[{StateName}] ลูกค้าได้รับสินค้าแล้ว");
        order.TransitionTo(new DeliveredState());
    }
}

// Concrete State: ส่งมอบแล้ว
public class DeliveredState : OrderStateBase
{
    public override string StateName => "ส่งมอบแล้ว";
    // สถานะสุดท้าย ไม่สามารถเปลี่ยนได้อีก
}

// Concrete State: ยกเลิก
public class CancelledState : OrderStateBase
{
    public override string StateName => "ยกเลิก";
    // สถานะสุดท้าย ไม่สามารถเปลี่ยนได้อีก
}

// ==========================================
// State Pattern - Traffic Light
// ==========================================

public class TrafficLight
{
    private ITrafficLightState _state;
    private int _cycleCount = 0;

    public TrafficLight()
    {
        _state = new RedLightState();
    }

    public void ChangeState(ITrafficLightState newState)
    {
        _state = newState;
    }

    public void Tick()
    {
        _state.Handle(this);
        _cycleCount++;
    }

    public void PrintStatus()
    {
        Console.WriteLine($"[TrafficLight] รอบที่ {_cycleCount}: {_state.Color} - {_state.Action}");
    }
}

public interface ITrafficLightState
{
    string Color { get; }
    string Action { get; }
    void Handle(TrafficLight light);
}

public class RedLightState : ITrafficLightState
{
    public string Color => "แดง";
    public string Action => "หยุด!";
    public void Handle(TrafficLight light)
    {
        Console.WriteLine("[TrafficLight] ไฟแดง → เปลี่ยนเป็นไฟเขียว");
        light.ChangeState(new GreenLightState());
    }
}

public class GreenLightState : ITrafficLightState
{
    public string Color => "เขียว";
    public string Action => "ไปได้!";
    public void Handle(TrafficLight light)
    {
        Console.WriteLine("[TrafficLight] ไฟเขียว → เปลี่ยนเป็นไฟเหลือง");
        light.ChangeState(new YellowLightState());
    }
}

public class YellowLightState : ITrafficLightState
{
    public string Color => "เหลือง";
    public string Action => "ระวัง!";
    public void Handle(TrafficLight light)
    {
        Console.WriteLine("[TrafficLight] ไฟเหลือง → เปลี่ยนเป็นไฟแดง");
        light.ChangeState(new RedLightState());
    }
}

// โปรแกรมทดสอบ State Pattern
class StateDemo
{
    static void Main()
    {
        Console.WriteLine("=== State Pattern Demo ===\n");

        // ทดสอบ Order State Machine
        Console.WriteLine("--- Order State Machine ---");
        var order = new Order("ORD-2024-002", "วิชัย มีสุข", 15000m);
        order.PrintStatus();

        order.Confirm();
        order.Process();
        order.Ship();
        order.Deliver();
        order.PrintStatus();
        order.PrintHistory();

        // ทดสอบการยกเลิก
        Console.WriteLine("\n--- Order ที่ถูกยกเลิก ---");
        var order2 = new Order("ORD-2024-003", "สมหญิง ดีงาม", 5000m);
        order2.Confirm();
        order2.Cancel("ลูกค้าขอยกเลิก");
        order2.PrintStatus();

        // ทดสอบ Traffic Light
        Console.WriteLine("\n--- Traffic Light ---");
        var light = new TrafficLight();
        for (int i = 0; i < 6; i++)
        {
            light.PrintStatus();
            light.Tick();
        }
    }
}
```

---

## ขั้นตอนที่ 486: Template Method Pattern — โครงร่างอัลกอริทึม

**Template Method Pattern** กำหนดโครงร่างของอัลกอริทึมใน Base Class และให้ Subclasses Override ขั้นตอนบางส่วนได้โดยไม่เปลี่ยนโครงสร้างรวม

### แนวคิดหลัก
- **Abstract Class**: กำหนด Template Method และ Abstract Steps
- **Template Method**: เมธอดที่กำหนดลำดับขั้นตอน
- **Hook Methods**: เมธอดที่ Subclass สามารถ Override ได้ (optional)

### ตัวอย่าง: DataExporter สำหรับ CSV, JSON, XML

```csharp
using System;
using System.Collections.Generic;
using System.Text;
using System.Text.Json;

// ==========================================
// Template Method Pattern - Data Exporter
// ==========================================

public class ExportRecord
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
    public decimal Amount { get; set; }
    public DateTime Date { get; set; }
}

// Abstract Base: Template Method Pattern
public abstract class DataExporter
{
    // Template Method - กำหนดลำดับขั้นตอนการ Export
    public string Export(IEnumerable<ExportRecord> records)
    {
        Console.WriteLine($"[Exporter] เริ่ม Export ด้วยรูปแบบ {FormatName}");

        ValidateData(records);
        var preprocessed = PreprocessData(records);
        var header = GenerateHeader();
        var body = GenerateBody(preprocessed);
        var footer = GenerateFooter();
        var result = AssembleOutput(header, body, footer);

        Console.WriteLine($"[Exporter] Export เสร็จสิ้น: {result.Length} ตัวอักษร");
        return result;
    }

    // Abstract Methods ที่ต้อง Implement
    protected abstract string FormatName { get; }
    protected abstract string GenerateHeader();
    protected abstract string GenerateBody(IEnumerable<ExportRecord> records);
    protected abstract string GenerateFooter();

    // Hook Methods ที่ Subclass สามารถ Override ได้
    protected virtual void ValidateData(IEnumerable<ExportRecord> records)
    {
        Console.WriteLine("[Exporter] ตรวจสอบข้อมูล...");
    }

    protected virtual IEnumerable<ExportRecord> PreprocessData(IEnumerable<ExportRecord> records)
    {
        return records; // Default: ไม่มีการ preprocess
    }

    protected virtual string AssembleOutput(string header, string body, string footer)
    {
        return header + body + footer;
    }
}

// Concrete Exporter: CSV
public class CsvExporter : DataExporter
{
    protected override string FormatName => "CSV";
    private readonly string _delimiter;

    public CsvExporter(string delimiter = ",")
    {
        _delimiter = delimiter;
    }

    protected override string GenerateHeader()
    {
        return $"Id{_delimiter}Name{_delimiter}Email{_delimiter}Amount{_delimiter}Date\n";
    }

    protected override string GenerateBody(IEnumerable<ExportRecord> records)
    {
        var sb = new StringBuilder();
        foreach (var r in records)
        {
            sb.AppendLine($"{r.Id}{_delimiter}{EscapeCsv(r.Name)}{_delimiter}" +
                         $"{r.Email}{_delimiter}{r.Amount:F2}{_delimiter}{r.Date:yyyy-MM-dd}");
        }
        return sb.ToString();
    }

    protected override string GenerateFooter() => "";

    private string EscapeCsv(string value)
    {
        if (value.Contains(_delimiter) || value.Contains('"'))
            return $"\"{value.Replace("\"", "\"\"")}\"";
        return value;
    }
}

// Concrete Exporter: JSON
public class JsonExporter : DataExporter
{
    protected override string FormatName => "JSON";
    private readonly bool _prettyPrint;

    public JsonExporter(bool prettyPrint = true)
    {
        _prettyPrint = prettyPrint;
    }

    protected override string GenerateHeader() => "";
    protected override string GenerateFooter() => "";

    protected override string GenerateBody(IEnumerable<ExportRecord> records)
    {
        var options = new JsonSerializerOptions
        {
            WriteIndented = _prettyPrint,
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase
        };

        var data = new
        {
            ExportedAt = DateTime.Now,
            TotalRecords = records.Count(),
            Records = records.Select(r => new
            {
                r.Id,
                r.Name,
                r.Email,
                Amount = r.Amount,
                Date = r.Date.ToString("yyyy-MM-dd")
            })
        };

        return JsonSerializer.Serialize(data, options);
    }

    private int Count<T>(IEnumerable<T> items) => items.Count();
}

// Concrete Exporter: XML
public class XmlExporter : DataExporter
{
    protected override string FormatName => "XML";

    protected override string GenerateHeader()
    {
        return "<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n<Records>\n";
    }

    protected override string GenerateBody(IEnumerable<ExportRecord> records)
    {
        var sb = new StringBuilder();
        foreach (var r in records)
        {
            sb.AppendLine($"  <Record id=\"{r.Id}\">");
            sb.AppendLine($"    <Name>{EscapeXml(r.Name)}</Name>");
            sb.AppendLine($"    <Email>{r.Email}</Email>");
            sb.AppendLine($"    <Amount currency=\"THB\">{r.Amount:F2}</Amount>");
            sb.AppendLine($"    <Date>{r.Date:yyyy-MM-dd}</Date>");
            sb.AppendLine($"  </Record>");
        }
        return sb.ToString();
    }

    protected override string GenerateFooter() => "</Records>";

    private string EscapeXml(string value) =>
        value.Replace("&", "&amp;").Replace("<", "&lt;").Replace(">", "&gt;");

    // Override Hook: เพิ่ม namespace
    protected override string AssembleOutput(string header, string body, string footer)
    {
        return header + body + footer + "\n";
    }
}

// โปรแกรมทดสอบ Template Method Pattern
class TemplateMethodDemo
{
    static void Main()
    {
        Console.WriteLine("=== Template Method Pattern Demo ===\n");

        var records = new List<ExportRecord>
        {
            new() { Id = 1, Name = "สมชาย ใจดี", Email = "somchai@example.com", Amount = 1500.00m, Date = new DateTime(2024, 1, 15) },
            new() { Id = 2, Name = "สมหญิง, ดีงาม", Email = "somying@example.com", Amount = 3200.50m, Date = new DateTime(2024, 1, 16) },
            new() { Id = 3, Name = "วิชัย มีสุข", Email = "vichai@example.com", Amount = 750.75m, Date = new DateTime(2024, 1, 17) }
        };

        // Export เป็น CSV
        Console.WriteLine("=== Export CSV ===");
        var csvExporter = new CsvExporter();
        var csvOutput = csvExporter.Export(records);
        Console.WriteLine(csvOutput);

        // Export เป็น JSON
        Console.WriteLine("=== Export JSON ===");
        var jsonExporter = new JsonExporter(prettyPrint: true);
        var jsonOutput = jsonExporter.Export(records);
        Console.WriteLine(jsonOutput);

        // Export เป็น XML
        Console.WriteLine("=== Export XML ===");
        var xmlExporter = new XmlExporter();
        var xmlOutput = xmlExporter.Export(records);
        Console.WriteLine(xmlOutput);
    }
}
```

---

## ขั้นตอนที่ 487: Chain of Responsibility Pattern — ส่งต่อ Request

**Chain of Responsibility Pattern** ส่งต่อ Request ผ่านห่วงโซ่ของ Handlers โดยแต่ละ Handler สามารถประมวลผลหรือส่งต่อให้ Handler ถัดไปได้

### แนวคิดหลัก
- **Handler Interface**: กำหนด Interface สำหรับการจัดการ Request
- **Chain Setup**: เชื่อมต่อ Handlers เป็นลำดับ
- **Middleware Pipeline**: การนำไปใช้จริงในเว็บ Application

### ตัวอย่าง: HTTP Middleware Pipeline และ Validation Chain

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text.RegularExpressions;

// ==========================================
// Chain of Responsibility - Validation Chain
// ==========================================

public class FormData
{
    public string Name { get; set; }
    public string Email { get; set; }
    public string Phone { get; set; }
    public string Password { get; set; }
    public int Age { get; set; }
    public List<string> ValidationErrors { get; } = new();
    public bool IsValid => ValidationErrors.Count == 0;
}

// Handler Interface
public abstract class ValidationHandler
{
    private ValidationHandler _nextHandler;

    public ValidationHandler SetNext(ValidationHandler next)
    {
        _nextHandler = next;
        return next; // ส่งคืน next เพื่อให้ Chain ต่อกันได้
    }

    // Template Method: ตรวจสอบและส่งต่อ
    public void Validate(FormData data)
    {
        DoValidate(data);
        _nextHandler?.Validate(data); // ส่งต่อให้ Handler ถัดไปเสมอ
    }

    protected abstract void DoValidate(FormData data);
    public abstract string RuleName { get; }
}

// Concrete Handler: ตรวจสอบชื่อ
public class NameValidator : ValidationHandler
{
    public override string RuleName => "Name Validation";

    protected override void DoValidate(FormData data)
    {
        if (string.IsNullOrWhiteSpace(data.Name))
            data.ValidationErrors.Add("กรุณากรอกชื่อ");
        else if (data.Name.Length < 2)
            data.ValidationErrors.Add("ชื่อต้องมีอย่างน้อย 2 ตัวอักษร");
        else if (data.Name.Length > 100)
            data.ValidationErrors.Add("ชื่อต้องไม่เกิน 100 ตัวอักษร");

        Console.WriteLine($"[{RuleName}] ตรวจสอบชื่อ: {(data.ValidationErrors.Count == 0 ? "ผ่าน" : "ไม่ผ่าน")}");
    }
}

// Concrete Handler: ตรวจสอบ Email
public class EmailValidator : ValidationHandler
{
    public override string RuleName => "Email Validation";

    protected override void DoValidate(FormData data)
    {
        int errorsBefore = data.ValidationErrors.Count;

        if (string.IsNullOrWhiteSpace(data.Email))
            data.ValidationErrors.Add("กรุณากรอก Email");
        else if (!Regex.IsMatch(data.Email, @"^[^@\s]+@[^@\s]+\.[^@\s]+$"))
            data.ValidationErrors.Add($"รูปแบบ Email ไม่ถูกต้อง: {data.Email}");

        bool passed = data.ValidationErrors.Count == errorsBefore;
        Console.WriteLine($"[{RuleName}] ตรวจสอบ Email: {(passed ? "ผ่าน" : "ไม่ผ่าน")}");
    }
}

// Concrete Handler: ตรวจสอบเบอร์โทรศัพท์
public class PhoneValidator : ValidationHandler
{
    public override string RuleName => "Phone Validation";

    protected override void DoValidate(FormData data)
    {
        int errorsBefore = data.ValidationErrors.Count;

        if (!string.IsNullOrEmpty(data.Phone))
        {
            var cleaned = Regex.Replace(data.Phone, @"[\s\-\(\)]", "");
            if (!Regex.IsMatch(cleaned, @"^[0-9]{9,10}$"))
                data.ValidationErrors.Add($"รูปแบบเบอร์โทรศัพท์ไม่ถูกต้อง: {data.Phone}");
        }

        bool passed = data.ValidationErrors.Count == errorsBefore;
        Console.WriteLine($"[{RuleName}] ตรวจสอบเบอร์โทร: {(passed ? "ผ่าน" : "ไม่ผ่าน")}");
    }
}

// Concrete Handler: ตรวจสอบรหัสผ่าน
public class PasswordValidator : ValidationHandler
{
    public override string RuleName => "Password Validation";

    protected override void DoValidate(FormData data)
    {
        int errorsBefore = data.ValidationErrors.Count;

        if (string.IsNullOrEmpty(data.Password))
        {
            data.ValidationErrors.Add("กรุณากรอกรหัสผ่าน");
        }
        else
        {
            if (data.Password.Length < 8)
                data.ValidationErrors.Add("รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร");
            if (!data.Password.Any(char.IsUpper))
                data.ValidationErrors.Add("รหัสผ่านต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว");
            if (!data.Password.Any(char.IsDigit))
                data.ValidationErrors.Add("รหัสผ่านต้องมีตัวเลขอย่างน้อย 1 ตัว");
        }

        bool passed = data.ValidationErrors.Count == errorsBefore;
        Console.WriteLine($"[{RuleName}] ตรวจสอบรหัสผ่าน: {(passed ? "ผ่าน" : "ไม่ผ่าน")}");
    }
}

// Concrete Handler: ตรวจสอบอายุ
public class AgeValidator : ValidationHandler
{
    public override string RuleName => "Age Validation";
    private readonly int _minAge;
    private readonly int _maxAge;

    public AgeValidator(int minAge = 18, int maxAge = 120)
    {
        _minAge = minAge;
        _maxAge = maxAge;
    }

    protected override void DoValidate(FormData data)
    {
        int errorsBefore = data.ValidationErrors.Count;

        if (data.Age < _minAge)
            data.ValidationErrors.Add($"อายุต้องไม่น้อยกว่า {_minAge} ปี");
        else if (data.Age > _maxAge)
            data.ValidationErrors.Add($"กรุณากรอกอายุที่ถูกต้อง");

        bool passed = data.ValidationErrors.Count == errorsBefore;
        Console.WriteLine($"[{RuleName}] ตรวจสอบอายุ {data.Age}: {(passed ? "ผ่าน" : "ไม่ผ่าน")}");
    }
}

// ValidationChain Builder
public class ValidationChainBuilder
{
    private readonly List<ValidationHandler> _handlers = new();

    public ValidationChainBuilder AddHandler(ValidationHandler handler)
    {
        _handlers.Add(handler);
        return this;
    }

    public ValidationHandler Build()
    {
        if (_handlers.Count == 0) throw new InvalidOperationException("ต้องมี Handler อย่างน้อย 1 ตัว");

        for (int i = 0; i < _handlers.Count - 1; i++)
            _handlers[i].SetNext(_handlers[i + 1]);

        return _handlers[0];
    }
}

// โปรแกรมทดสอบ Chain of Responsibility
class ChainDemo
{
    static void Main()
    {
        Console.WriteLine("=== Chain of Responsibility Demo ===\n");

        // สร้าง Validation Chain
        var chain = new ValidationChainBuilder()
            .AddHandler(new NameValidator())
            .AddHandler(new EmailValidator())
            .AddHandler(new PhoneValidator())
            .AddHandler(new PasswordValidator())
            .AddHandler(new AgeValidator(minAge: 18))
            .Build();

        // ทดสอบข้อมูลที่ถูกต้อง
        Console.WriteLine("--- ทดสอบข้อมูลที่ถูกต้อง ---");
        var validData = new FormData
        {
            Name = "สมชาย ใจดี",
            Email = "somchai@example.com",
            Phone = "0891234567",
            Password = "MyPass1234",
            Age = 25
        };
        chain.Validate(validData);
        Console.WriteLine($"\nผลการตรวจสอบ: {(validData.IsValid ? "ผ่านทั้งหมด" : "ไม่ผ่าน")}");

        // ทดสอบข้อมูลที่ไม่ถูกต้อง
        Console.WriteLine("\n--- ทดสอบข้อมูลที่ไม่ถูกต้อง ---");
        var invalidData = new FormData
        {
            Name = "A",
            Email = "not-an-email",
            Phone = "abc",
            Password = "weak",
            Age = 15
        };
        chain.Validate(invalidData);
        Console.WriteLine($"\nผลการตรวจสอบ: ไม่ผ่าน ({invalidData.ValidationErrors.Count} ข้อผิดพลาด)");
        foreach (var error in invalidData.ValidationErrors)
            Console.WriteLine($"  ❌ {error}");
    }
}
```

---

## ขั้นตอนที่ 488-490: ตัวอย่างรวม — Workflow Engine

ในส่วนสุดท้ายนี้ เราจะนำ Pattern ต่างๆ มารวมกันสร้าง **Workflow Approval Engine** ที่รองรับกระบวนการอนุมัติแบบหลายขั้นตอน โดยใช้:
- **Observer**: แจ้งเตือนเมื่อสถานะ Workflow เปลี่ยนแปลง
- **Command**: บันทึกและ Undo การกระทำใน Workflow
- **State**: จัดการสถานะของ Workflow Request
- **Chain of Responsibility**: ส่งต่อการอนุมัติผ่านลำดับชั้น

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

// ==========================================
// Step 488: Workflow Engine - Core Components
// ==========================================

// ข้อมูล Workflow Request
public class WorkflowRequest
{
    public string RequestId { get; } = Guid.NewGuid().ToString()[..8].ToUpper();
    public string Title { get; set; }
    public string Description { get; set; }
    public string RequestedBy { get; set; }
    public decimal Amount { get; set; }
    public string Category { get; set; }
    public DateTime CreatedAt { get; } = DateTime.Now;
    public List<ApprovalRecord> ApprovalHistory { get; } = new();
    public List<string> Comments { get; } = new();

    public void AddApproval(string approver, bool approved, string comment)
    {
        ApprovalHistory.Add(new ApprovalRecord
        {
            Approver = approver,
            Approved = approved,
            Comment = comment,
            Timestamp = DateTime.Now
        });
        if (!string.IsNullOrEmpty(comment))
            Comments.Add($"[{approver}]: {comment}");
    }
}

public class ApprovalRecord
{
    public string Approver { get; set; }
    public bool Approved { get; set; }
    public string Comment { get; set; }
    public DateTime Timestamp { get; set; }
}

// ==========================================
// Step 489: State Pattern สำหรับ Workflow
// ==========================================

public interface IWorkflowState
{
    string StateName { get; }
    string StateIcon { get; }
    bool CanApprove { get; }
    bool CanReject { get; }
    bool CanCancel { get; }
    void Approve(WorkflowEngine engine, string approver, string comment);
    void Reject(WorkflowEngine engine, string approver, string comment);
    void Cancel(WorkflowEngine engine, string reason);
}

public abstract class WorkflowStateBase : IWorkflowState
{
    public abstract string StateName { get; }
    public abstract string StateIcon { get; }
    public virtual bool CanApprove => false;
    public virtual bool CanReject => false;
    public virtual bool CanCancel => false;

    public virtual void Approve(WorkflowEngine engine, string approver, string comment) =>
        Console.WriteLine($"[{StateName}] ไม่สามารถอนุมัติได้ในสถานะนี้");

    public virtual void Reject(WorkflowEngine engine, string approver, string comment) =>
        Console.WriteLine($"[{StateName}] ไม่สามารถปฏิเสธได้ในสถานะนี้");

    public virtual void Cancel(WorkflowEngine engine, string reason) =>
        Console.WriteLine($"[{StateName}] ไม่สามารถยกเลิกได้ในสถานะนี้");
}

public class DraftState : WorkflowStateBase
{
    public override string StateName => "ร่าง";
    public override string StateIcon => "📝";
    public override bool CanCancel => true;

    public override void Approve(WorkflowEngine engine, string approver, string comment)
    {
        Console.WriteLine($"[Workflow] ส่งคำขออนุมัติโดย {approver}");
        engine.TransitionTo(new PendingApprovalState(), approver, comment, true);
    }

    public override void Cancel(WorkflowEngine engine, string reason)
    {
        Console.WriteLine($"[Workflow] ยกเลิกร่าง: {reason}");
        engine.TransitionTo(new CancelledWorkflowState(), "System", reason, false);
    }
}

public class PendingApprovalState : WorkflowStateBase
{
    public override string StateName => "รอการอนุมัติ";
    public override string StateIcon => "⏳";
    public override bool CanApprove => true;
    public override bool CanReject => true;
    public override bool CanCancel => true;

    public override void Approve(WorkflowEngine engine, string approver, string comment)
    {
        engine.Request.AddApproval(approver, true, comment);
        Console.WriteLine($"[Workflow] {approver} อนุมัติแล้ว");

        // ตรวจสอบว่าต้องผ่านการอนุมัติกี่ขั้น
        if (engine.IsFullyApproved())
        {
            engine.TransitionTo(new ApprovedState(), approver, comment, true);
        }
        else
        {
            Console.WriteLine($"[Workflow] รอการอนุมัติจากระดับถัดไป");
        }
    }

    public override void Reject(WorkflowEngine engine, string approver, string comment)
    {
        engine.Request.AddApproval(approver, false, comment);
        Console.WriteLine($"[Workflow] {approver} ปฏิเสธ: {comment}");
        engine.TransitionTo(new RejectedState(), approver, comment, false);
    }

    public override void Cancel(WorkflowEngine engine, string reason)
    {
        engine.TransitionTo(new CancelledWorkflowState(), "System", reason, false);
    }
}

public class ApprovedState : WorkflowStateBase
{
    public override string StateName => "อนุมัติแล้ว";
    public override string StateIcon => "✅";
}

public class RejectedState : WorkflowStateBase
{
    public override string StateName => "ถูกปฏิเสธ";
    public override string StateIcon => "❌";
}

public class CancelledWorkflowState : WorkflowStateBase
{
    public override string StateName => "ยกเลิก";
    public override string StateIcon => "🚫";
}

// ==========================================
// Step 490: Observer + Command + Chain
// ==========================================

// Observer: แจ้งเตือนการเปลี่ยนแปลง Workflow
public interface IWorkflowObserver
{
    string Name { get; }
    void OnStateChanged(WorkflowRequest request, string oldState, string newState);
    void OnActionTaken(WorkflowRequest request, string actor, string action, string comment);
}

// Observer: Email Notifier
public class EmailNotifier : IWorkflowObserver
{
    public string Name => "EmailNotifier";

    public void OnStateChanged(WorkflowRequest request, string oldState, string newState)
    {
        Console.WriteLine($"[EMAIL] ส่งแจ้งเตือนถึง {request.RequestedBy}: " +
                         $"คำขอ {request.RequestId} เปลี่ยนสถานะ {oldState} → {newState}");
    }

    public void OnActionTaken(WorkflowRequest request, string actor, string action, string comment)
    {
        Console.WriteLine($"[EMAIL] แจ้งเตือน: {actor} ได้ดำเนินการ '{action}' " +
                         $"กับคำขอ {request.RequestId}");
    }
}

// Observer: Audit Logger
public class AuditLogger : IWorkflowObserver
{
    public string Name => "AuditLogger";
    private readonly List<string> _auditLog = new();

    public void OnStateChanged(WorkflowRequest request, string oldState, string newState)
    {
        var entry = $"[AUDIT {DateTime.Now:yyyy-MM-dd HH:mm:ss}] " +
                   $"Request {request.RequestId}: {oldState} → {newState}";
        _auditLog.Add(entry);
        Console.WriteLine(entry);
    }

    public void OnActionTaken(WorkflowRequest request, string actor, string action, string comment)
    {
        var entry = $"[AUDIT {DateTime.Now:yyyy-MM-dd HH:mm:ss}] " +
                   $"Actor: {actor}, Action: {action}, Comment: {comment}";
        _auditLog.Add(entry);
        Console.WriteLine(entry);
    }

    public void PrintAuditLog()
    {
        Console.WriteLine("\n[AuditLog] รายการทั้งหมด:");
        foreach (var entry in _auditLog) Console.WriteLine($"  {entry}");
    }
}

// Chain of Responsibility: Approval Handlers
public abstract class ApprovalHandler
{
    private ApprovalHandler _next;
    public abstract string HandlerName { get; }
    public abstract decimal ApprovalLimit { get; }

    public ApprovalHandler SetNext(ApprovalHandler next)
    {
        _next = next;
        return next;
    }

    public bool Handle(WorkflowRequest request, string approver)
    {
        if (CanHandle(request))
        {
            Console.WriteLine($"[Chain] {HandlerName} ({approver}) จัดการคำขอ " +
                            $"มูลค่า {request.Amount:F0} บาท");
            return DoApprove(request, approver);
        }
        else if (_next != null)
        {
            Console.WriteLine($"[Chain] {HandlerName}: เกินวงเงิน ส่งต่อให้ {_next.HandlerName}");
            return _next.Handle(request, approver);
        }
        else
        {
            Console.WriteLine($"[Chain] ไม่มี Handler ที่สามารถจัดการได้");
            return false;
        }
    }

    protected virtual bool CanHandle(WorkflowRequest request) =>
        request.Amount <= ApprovalLimit;

    protected abstract bool DoApprove(WorkflowRequest request, string approver);
}

public class TeamLeadApproval : ApprovalHandler
{
    public override string HandlerName => "Team Lead";
    public override decimal ApprovalLimit => 10000m;

    protected override bool DoApprove(WorkflowRequest request, string approver)
    {
        Console.WriteLine($"[TeamLead] อนุมัติงบประมาณ {request.Amount:F0} บาท");
        return true;
    }
}

public class ManagerApproval : ApprovalHandler
{
    public override string HandlerName => "Manager";
    public override decimal ApprovalLimit => 100000m;

    protected override bool DoApprove(WorkflowRequest request, string approver)
    {
        Console.WriteLine($"[Manager] อนุมัติงบประมาณ {request.Amount:F0} บาท");
        return true;
    }
}

public class DirectorApproval : ApprovalHandler
{
    public override string HandlerName => "Director";
    public override decimal ApprovalLimit => 1000000m;

    protected override bool DoApprove(WorkflowRequest request, string approver)
    {
        Console.WriteLine($"[Director] อนุมัติงบประมาณ {request.Amount:F0} บาท");
        return true;
    }
}

// Workflow Engine: รวมทุก Pattern
public class WorkflowEngine
{
    public WorkflowRequest Request { get; }
    private IWorkflowState _state;
    private readonly List<IWorkflowObserver> _observers = new();
    private readonly ApprovalHandler _approvalChain;
    private readonly int _requiredApprovals;

    public string CurrentState => _state.StateName;

    public WorkflowEngine(WorkflowRequest request, int requiredApprovals = 2)
    {
        Request = request;
        _state = new DraftState();
        _requiredApprovals = requiredApprovals;

        // สร้าง Approval Chain
        var teamLead = new TeamLeadApproval();
        var manager = new ManagerApproval();
        var director = new DirectorApproval();
        teamLead.SetNext(manager).SetNext(director);
        _approvalChain = teamLead;
    }

    public void AddObserver(IWorkflowObserver observer)
    {
        _observers.Add(observer);
        Console.WriteLine($"[Engine] เพิ่ม Observer: {observer.Name}");
    }

    public void TransitionTo(IWorkflowState newState, string actor, string comment, bool approved)
    {
        string oldStateName = _state.StateName;
        _state = newState;

        // แจ้งเตือน Observers (Observer Pattern)
        foreach (var observer in _observers)
        {
            observer.OnStateChanged(Request, oldStateName, newState.StateName);
            observer.OnActionTaken(Request, actor, approved ? "อนุมัติ" : "ปฏิเสธ/ยกเลิก", comment);
        }
    }

    public bool IsFullyApproved() =>
        Request.ApprovalHistory.Count(a => a.Approved) >= _requiredApprovals;

    // ส่งคำขออนุมัติ
    public void Submit(string submitter, string comment = "")
    {
        Console.WriteLine($"\n[Engine] {submitter} ส่งคำขออนุมัติ: {Request.Title}");
        _state.Approve(this, submitter, comment);
    }

    // อนุมัติผ่าน Chain of Responsibility
    public void Approve(string approver, string comment = "")
    {
        Console.WriteLine($"\n[Engine] {approver} กำลังอนุมัติ...");
        bool chainApproved = _approvalChain.Handle(Request, approver);
        if (chainApproved)
        {
            _state.Approve(this, approver, comment);
        }
    }

    // ปฏิเสธ
    public void Reject(string rejector, string comment)
    {
        Console.WriteLine($"\n[Engine] {rejector} ปฏิเสธคำขอ");
        _state.Reject(this, rejector, comment);
    }

    // ยกเลิก
    public void Cancel(string reason)
    {
        Console.WriteLine($"\n[Engine] ยกเลิกคำขอ: {reason}");
        _state.Cancel(this, reason);
    }

    // แสดงสรุป
    public void PrintSummary()
    {
        Console.WriteLine($"\n{'=' * 50}");
        Console.WriteLine($"[Summary] คำขอ: {Request.RequestId}");
        Console.WriteLine($"[Summary] หัวข้อ: {Request.Title}");
        Console.WriteLine($"[Summary] ผู้ขอ: {Request.RequestedBy}");
        Console.WriteLine($"[Summary] มูลค่า: {Request.Amount:F2} บาท");
        Console.WriteLine($"[Summary] สถานะ: {_state.StateIcon} {_state.StateName}");
        Console.WriteLine($"[Summary] ประวัติการอนุมัติ:");

        foreach (var record in Request.ApprovalHistory)
        {
            string action = record.Approved ? "✅ อนุมัติ" : "❌ ปฏิเสธ";
            Console.WriteLine($"  [{record.Timestamp:HH:mm:ss}] {record.Approver}: {action}" +
                            $"{(string.IsNullOrEmpty(record.Comment) ? "" : $" - {record.Comment}")}");
        }
        Console.WriteLine($"{'=' * 50}");
    }
}

// โปรแกรมทดสอบ Workflow Engine
class WorkflowDemo
{
    static void Main()
    {
        Console.WriteLine("=== Workflow Engine Demo ===");
        Console.WriteLine("รวม Observer + Command + State + Chain of Responsibility\n");

        // สร้าง Request
        var request = new WorkflowRequest
        {
            Title = "ขอซื้อ MacBook Pro สำหรับทีม Dev",
            Description = "ต้องการซื้อ MacBook Pro 14\" จำนวน 3 เครื่อง",
            RequestedBy = "สมชาย ใจดี",
            Amount = 90000m,
            Category = "IT Equipment"
        };

        // สร้าง Workflow Engine
        var engine = new WorkflowEngine(request, requiredApprovals: 2);

        // เพิ่ม Observers
        var emailNotifier = new EmailNotifier();
        var auditLogger = new AuditLogger();
        engine.AddObserver(emailNotifier);
        engine.AddObserver(auditLogger);

        // Workflow Process
        Console.WriteLine("\n=== เริ่มกระบวนการอนุมัติ ===");

        // ขั้นที่ 1: ส่งคำขอ
        engine.Submit("สมชาย ใจดี", "จำเป็นสำหรับโปรเจกต์ใหม่");

        // ขั้นที่ 2: Team Lead อนุมัติ
        engine.Approve("นายสมศักดิ์ หัวหน้าทีม", "เห็นด้วย ทีมต้องการจริงๆ");

        // ขั้นที่ 3: Manager อนุมัติ (ครบ 2 คน = approved)
        engine.Approve("นางสาววิภา ผู้จัดการ", "อนุมัติงบประมาณ");

        // แสดงสรุป
        engine.PrintSummary();

        // ทดสอบ Reject
        Console.WriteLine("\n\n=== ทดสอบการปฏิเสธ ===");
        var request2 = new WorkflowRequest
        {
            Title = "ขอเดินทางไปสัมมนาต่างประเทศ",
            RequestedBy = "วิชัย มีสุข",
            Amount = 150000m,
            Category = "Travel"
        };

        var engine2 = new WorkflowEngine(request2, requiredApprovals: 2);
        engine2.AddObserver(auditLogger);
        engine2.Submit("วิชัย มีสุข");
        engine2.Reject("นางสาววิภา ผู้จัดการ", "งบประมาณไม่เพียงพอในไตรมาสนี้");
        engine2.PrintSummary();

        // แสดง Audit Log ทั้งหมด
        auditLogger.PrintAuditLog();
    }
}
```

---

## สรุปตาราง Behavioral Design Patterns

| Pattern | จุดประสงค์หลัก | ปัญหาที่แก้ | ตัวอย่างใน .NET |
|---------|--------------|------------|----------------|
| **Observer** | แจ้งเตือน Object ที่สนใจ | Tight coupling ระหว่าง Subject และ Observer | `IObservable<T>`, Events, Reactive Extensions |
| **Strategy** | สับเปลี่ยน Algorithm ได้ | การ Hard-code Algorithm ใน Class | `IComparer<T>`, `IEqualityComparer<T>` |
| **Command** | ห่อหุ้ม Request เป็น Object | ต้องการ Undo/Redo, Queue, Log | `ICommand` ใน WPF/MVVM |
| **Iterator** | ท่องผ่าน Collection | เปิดเผยโครงสร้างภายใน | `IEnumerable<T>`, `yield return` |
| **State** | เปลี่ยนพฤติกรรมตามสถานะ | การใช้ if/switch มากเกินไป | State Machine, Workflow |
| **Template Method** | กำหนดโครงร่างอัลกอริทึม | Code Duplication ใน Subclasses | `HttpMessageHandler`, `DbContext` |
| **Chain of Responsibility** | ส่งต่อ Request ตามลำดับ | ไม่ต้องการ Coupling ระหว่าง Sender และ Receiver | ASP.NET Middleware Pipeline |

---

## แนวทางการเลือก Behavioral Pattern

**ใช้ Observer เมื่อ:**
- ต้องการแจ้งเตือน Objects หลายตัวเมื่อมีการเปลี่ยนแปลง
- ไม่ต้องการให้ Subject รู้จัก Observer โดยตรง
- ต้องการ Event-Driven Architecture

**ใช้ Strategy เมื่อ:**
- มีหลาย Algorithm ที่ทำงานแบบเดียวกันแต่ต่างกันในรายละเอียด
- ต้องการสับเปลี่ยน Algorithm ที่ Runtime
- ต้องการขจัด Conditional Logic

**ใช้ Command เมื่อ:**
- ต้องการ Undo/Redo
- ต้องการ Queue งาน
- ต้องการ Transaction และ Rollback

**ใช้ Iterator เมื่อ:**
- ต้องการท่องผ่าน Collection แบบต่างๆ
- ต้องการซ่อนโครงสร้างของ Collection
- ต้องการ Lazy Evaluation ด้วย `yield return`

**ใช้ State เมื่อ:**
- Object มีพฤติกรรมขึ้นอยู่กับสถานะ
- มีการเปลี่ยนสถานะที่ชัดเจน (State Machine)
- มี if/switch จำนวนมากที่ตรวจสอบสถานะ

**ใช้ Template Method เมื่อ:**
- มีขั้นตอนหลักเหมือนกัน แต่รายละเอียดต่างกัน
- ต้องการ Code Reuse ใน Base Class
- ต้องการบังคับให้ Subclass Override บางเมธอด

**ใช้ Chain of Responsibility เมื่อ:**
- มีหลาย Handler ที่อาจจัดการ Request
- ต้องการ Pipeline Processing
- ต้องการ Middleware Pattern

---

## Navigation

- [← Part 48: Structural Patterns](part48-structural-patterns.md)
- [→ Part 50: Design Patterns Final](part50-design-patterns-final.md)
