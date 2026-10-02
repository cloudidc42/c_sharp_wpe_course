# Part 86: Capstone - Payment & Financial Systems
## ขั้นตอนที่ 851-860: ระบบการชำระเงินใน ShopThai

---

## 🎯 เป้าหมายของ Part นี้
- Stripe integration
- Omise (payment gateway ยอดนิยมในไทย)
- Payment state machine
- Webhook security
- Refund processing
- Financial reporting
- PCI-DSS considerations

---

## ขั้นตอนที่ 851: Payment Gateway Architecture

```
Payment Flow:
─────────────────────────────────────────────────────────

  Customer ─► Checkout ─► [ Payment Service ]
                                │
                      ┌─────────┼─────────┐
                      ▼         ▼         ▼
                   Stripe    Omise    TrueMoney
                      │         │         │
                      └─────────┼─────────┘
                                │
                           [ Webhook ]
                                │
                         [ Order Service ]
                         (confirm/cancel)
```

```csharp
// Payment gateway abstraction
public interface IPaymentGateway
{
    string Name { get; }
    Task<PaymentIntent> CreatePaymentIntentAsync(CreatePaymentRequest request);
    Task<PaymentResult> CaptureAsync(string paymentIntentId);
    Task<RefundResult> RefundAsync(string chargeId, decimal amount, string reason);
    bool ValidateWebhookSignature(string payload, string signature, string secret);
}

public record CreatePaymentRequest(
    decimal Amount,
    string Currency,
    string OrderId,
    string CustomerEmail,
    Dictionary<string, string>? Metadata = null);

public record PaymentIntent(
    string Id,
    string ClientSecret,
    string Status,
    decimal Amount,
    string Currency);

public record PaymentResult(bool Success, string TransactionId, string? Error);
public record RefundResult(bool Success, string RefundId, string? Error);
```

---

## ขั้นตอนที่ 852: Stripe Integration

```csharp
// dotnet add package Stripe.net

public class StripePaymentGateway : IPaymentGateway
{
    private readonly StripeOptions _options;
    
    public StripePaymentGateway(IOptions<StripeOptions> options)
    {
        _options = options.Value;
        StripeConfiguration.ApiKey = options.Value.SecretKey;
    }
    
    public string Name => "Stripe";
    
    public async Task<PaymentIntent> CreatePaymentIntentAsync(CreatePaymentRequest request)
    {
        var service = new PaymentIntentService();
        var options = new PaymentIntentCreateOptions
        {
            Amount = (long)(request.Amount * 100), // Stripe uses cents
            Currency = request.Currency.ToLower(),
            AutomaticPaymentMethods = new PaymentIntentAutomaticPaymentMethodsOptions
            {
                Enabled = true
            },
            Metadata = new Dictionary<string, string>
            {
                ["order_id"] = request.OrderId,
                ["customer_email"] = request.CustomerEmail
            }
        };
        
        var intent = await service.CreateAsync(options);
        return new PaymentIntent(intent.Id, intent.ClientSecret, intent.Status, 
            request.Amount, request.Currency);
    }
    
    public async Task<PaymentResult> CaptureAsync(string paymentIntentId)
    {
        var service = new PaymentIntentService();
        var intent = await service.CaptureAsync(paymentIntentId);
        return intent.Status == "succeeded"
            ? new PaymentResult(true, intent.LatestChargeId!, null)
            : new PaymentResult(false, "", $"Payment failed: {intent.Status}");
    }
    
    public async Task<RefundResult> RefundAsync(string chargeId, decimal amount, string reason)
    {
        var service = new RefundService();
        var refund = await service.CreateAsync(new RefundCreateOptions
        {
            Charge = chargeId,
            Amount = (long)(amount * 100),
            Reason = reason switch
            {
                "duplicate" => "duplicate",
                "fraudulent" => "fraudulent",
                _ => "requested_by_customer"
            }
        });
        
        return refund.Status == "succeeded"
            ? new RefundResult(true, refund.Id, null)
            : new RefundResult(false, "", refund.FailureReason);
    }
    
    public bool ValidateWebhookSignature(string payload, string signature, string secret)
    {
        try
        {
            EventUtility.ConstructEvent(payload, signature, secret);
            return true;
        }
        catch (StripeException)
        {
            return false;
        }
    }
}
```

---

## ขั้นตอนที่ 853: Omise Integration (ไทย)

```csharp
// Omise: payment gateway ยอดนิยมในประเทศไทย
// dotnet add package Omise.Net

public class OmisePaymentGateway : IPaymentGateway
{
    private readonly OmiseOptions _options;
    private readonly Client _omiseClient;
    
    public OmisePaymentGateway(IOptions<OmiseOptions> options)
    {
        _options = options.Value;
        _omiseClient = new Client(_options.SecretKey, _options.PublicKey);
    }
    
    public string Name => "Omise";
    
    public async Task<PaymentIntent> CreatePaymentIntentAsync(CreatePaymentRequest request)
    {
        // สร้าง source สำหรับ PromptPay
        var source = await _omiseClient.Sources.Create(new CreateSourceRequest
        {
            Amount = (long)(request.Amount * 100), // สตางค์
            Currency = "thb",
            Type = SourceType.PromptPay
        });
        
        // สร้าง charge
        var charge = await _omiseClient.Charges.Create(new CreateChargeRequest
        {
            Amount = (long)(request.Amount * 100),
            Currency = "thb",
            Source = source.Id,
            ReturnUri = $"{_options.WebhookBaseUrl}/payment/callback",
            Metadata = new Dictionary<string, object>
            {
                ["order_id"] = request.OrderId
            }
        });
        
        return new PaymentIntent(
            charge.Id,
            charge.Source?.ScannableCode?.Image?.Download?.Uri ?? "",
            charge.Status.ToString(),
            request.Amount,
            "THB");
    }
    
    public async Task<PaymentResult> CaptureAsync(string chargeId)
    {
        var charge = await _omiseClient.Charges.Retrieve(chargeId);
        return charge.Paid
            ? new PaymentResult(true, charge.Transaction?.Id ?? chargeId, null)
            : new PaymentResult(false, "", charge.FailureMessage);
    }
    
    public async Task<RefundResult> RefundAsync(string chargeId, decimal amount, string reason)
    {
        var refund = await _omiseClient.Charges.CreateRefund(chargeId, new CreateRefundRequest
        {
            Amount = (long)(amount * 100)
        });
        
        return new RefundResult(true, refund.Id, null);
    }
    
    public bool ValidateWebhookSignature(string payload, string signature, string secret)
    {
        // Omise uses HMAC-SHA256
        using var hmac = new HMACSHA256(Encoding.UTF8.GetBytes(secret));
        var computed = Convert.ToHexString(hmac.ComputeHash(Encoding.UTF8.GetBytes(payload))).ToLower();
        return CryptographicOperations.FixedTimeEquals(
            Encoding.UTF8.GetBytes(computed),
            Encoding.UTF8.GetBytes(signature));
    }
}
```

---

## ขั้นตอนที่ 854: Webhook Handler

```csharp
// Secure webhook handling
[ApiController]
[Route("api/webhooks")]
public class WebhookController : ControllerBase
{
    private readonly IPaymentWebhookService _webhookService;
    private readonly ILogger<WebhookController> _logger;
    
    [HttpPost("stripe")]
    public async Task<IActionResult> StripeWebhook(
        [FromServices] IOptions<StripeOptions> stripeOptions)
    {
        // Read raw body for signature validation
        using var reader = new StreamReader(Request.Body);
        var payload = await reader.ReadToEndAsync();
        
        var signature = Request.Headers["Stripe-Signature"].FirstOrDefault();
        if (string.IsNullOrEmpty(signature))
            return BadRequest("Missing signature");
        
        // Validate signature BEFORE processing
        if (!_webhookService.ValidateStripeSignature(payload, signature, stripeOptions.Value.WebhookSecret))
        {
            _logger.LogWarning("Invalid Stripe webhook signature");
            return Unauthorized();
        }
        
        // Parse event
        var stripeEvent = EventUtility.ParseEvent(payload);
        
        switch (stripeEvent.Type)
        {
            case Events.PaymentIntentSucceeded:
                var intent = stripeEvent.Data.Object as PaymentIntent;
                await _webhookService.HandlePaymentSucceededAsync(intent!.Metadata["order_id"], intent.Id);
                break;
            
            case Events.PaymentIntentPaymentFailed:
                var failedIntent = stripeEvent.Data.Object as PaymentIntent;
                await _webhookService.HandlePaymentFailedAsync(
                    failedIntent!.Metadata["order_id"],
                    failedIntent.LastPaymentError?.Message ?? "Payment failed");
                break;
            
            case Events.ChargeRefunded:
                var charge = stripeEvent.Data.Object as Charge;
                await _webhookService.HandleRefundAsync(charge!.Id, charge.AmountRefunded / 100m);
                break;
        }
        
        return Ok(new { received = true });
    }
}

// Payment webhook service
public class PaymentWebhookService : IPaymentWebhookService
{
    private readonly IOrderRepository _orders;
    private readonly IPublishEndpoint _events;
    
    public async Task HandlePaymentSucceededAsync(string orderId, string transactionId)
    {
        var order = await _orders.GetByIdAsync(Guid.Parse(orderId));
        if (order == null) return;
        
        order.ConfirmPayment(transactionId);
        await _orders.SaveAsync(order);
        
        await _events.Publish(new PaymentConfirmed(Guid.Parse(orderId), transactionId, DateTime.UtcNow));
    }
    
    public async Task HandlePaymentFailedAsync(string orderId, string reason)
    {
        var order = await _orders.GetByIdAsync(Guid.Parse(orderId));
        if (order == null) return;
        
        order.CancelPayment(reason);
        await _orders.SaveAsync(order);
        
        await _events.Publish(new PaymentFailed(Guid.Parse(orderId), reason, DateTime.UtcNow));
    }
}
```

---

## ขั้นตอนที่ 855: Payment State Machine

```csharp
// Payment goes through well-defined states
public enum PaymentStatus
{
    Pending,        // สร้างแล้ว รอชำระ
    Processing,     // กำลังดำเนินการ
    Authorized,     // อนุมัติแล้ว ยังไม่ capture
    Captured,       // เรียกเก็บเงินแล้ว
    Failed,         // ล้มเหลว
    Cancelled,      // ยกเลิก
    Refunding,      // กำลังคืนเงิน
    Refunded,       // คืนเงินแล้ว
    PartialRefunded // คืนเงินบางส่วน
}

public class Payment : AggregateRoot<PaymentId>
{
    public OrderId OrderId { get; private set; }
    public decimal Amount { get; private set; }
    public string Currency { get; private set; }
    public PaymentStatus Status { get; private set; }
    public string? TransactionId { get; private set; }
    public string? FailureReason { get; private set; }
    public decimal RefundedAmount { get; private set; }
    
    private Payment() { }
    
    public static Payment Create(PaymentId id, OrderId orderId, decimal amount, string currency)
    {
        var payment = new Payment
        {
            Id = id,
            OrderId = orderId,
            Amount = amount,
            Currency = currency,
            Status = PaymentStatus.Pending
        };
        payment.AddDomainEvent(new PaymentCreated(id, orderId, amount, currency));
        return payment;
    }
    
    public void Capture(string transactionId)
    {
        EnsureStatus(PaymentStatus.Authorized);
        TransactionId = transactionId;
        Status = PaymentStatus.Captured;
        AddDomainEvent(new PaymentCaptured(Id, OrderId, transactionId));
    }
    
    public void Fail(string reason)
    {
        EnsureStatus(PaymentStatus.Pending, PaymentStatus.Processing, PaymentStatus.Authorized);
        FailureReason = reason;
        Status = PaymentStatus.Failed;
        AddDomainEvent(new PaymentFailed(Id, OrderId, reason));
    }
    
    public void Refund(decimal amount)
    {
        if (Status != PaymentStatus.Captured && Status != PaymentStatus.PartialRefunded)
            throw new DomainException("INVALID_REFUND", "Can only refund captured payments");
        
        if (amount > Amount - RefundedAmount)
            throw new DomainException("REFUND_EXCEEDS_AMOUNT", "Refund exceeds remaining amount");
        
        RefundedAmount += amount;
        Status = RefundedAmount >= Amount ? PaymentStatus.Refunded : PaymentStatus.PartialRefunded;
        AddDomainEvent(new PaymentRefunded(Id, OrderId, amount, RefundedAmount));
    }
    
    private void EnsureStatus(params PaymentStatus[] allowed)
    {
        if (!allowed.Contains(Status))
            throw new DomainException("INVALID_PAYMENT_STATE", 
                $"Cannot perform action on payment in {Status} state");
    }
}
```

---

## ขั้นตอนที่ 856-860: Financial Reporting

```csharp
// Financial report queries
public class RevenueReportQuery : IRequest<RevenueReport>
{
    public DateOnly From { get; init; }
    public DateOnly To { get; init; }
    public string? Currency { get; init; }
}

public record RevenueReport(
    decimal GrossRevenue,
    decimal Refunds,
    decimal NetRevenue,
    int TotalOrders,
    int SuccessfulOrders,
    Dictionary<string, decimal> RevenueByDay,
    Dictionary<string, decimal> RevenueByPaymentMethod);

public class RevenueReportHandler : IRequestHandler<RevenueReportQuery, RevenueReport>
{
    private readonly AppDbContext _db;
    
    public async Task<RevenueReport> Handle(RevenueReportQuery q, CancellationToken ct)
    {
        var payments = await _db.Payments
            .Where(p => p.CreatedAt.Date >= q.From.ToDateTime(TimeOnly.MinValue)
                     && p.CreatedAt.Date <= q.To.ToDateTime(TimeOnly.MaxValue)
                     && p.Status == PaymentStatus.Captured
                     && (q.Currency == null || p.Currency == q.Currency))
            .Select(p => new
            {
                p.Amount,
                p.RefundedAmount,
                p.Currency,
                p.PaymentMethod,
                Date = p.CreatedAt.Date
            })
            .ToListAsync(ct);
        
        var gross = payments.Sum(p => p.Amount);
        var refunds = payments.Sum(p => p.RefundedAmount);
        
        return new RevenueReport(
            GrossRevenue: gross,
            Refunds: refunds,
            NetRevenue: gross - refunds,
            TotalOrders: payments.Count,
            SuccessfulOrders: payments.Count(p => p.Amount > p.RefundedAmount),
            RevenueByDay: payments.GroupBy(p => p.Date.ToString("yyyy-MM-dd"))
                .ToDictionary(g => g.Key, g => g.Sum(p => p.Amount - p.RefundedAmount)),
            RevenueByPaymentMethod: payments.GroupBy(p => p.PaymentMethod)
                .ToDictionary(g => g.Key, g => g.Sum(p => p.Amount - p.RefundedAmount)));
    }
}
```

---

## 📝 สรุป Part 86

| Payment Provider | ใช้กับ |
|-----------------|--------|
| Stripe | สากล (Visa, MC, AmEx, Apple Pay) |
| Omise | ไทย (PromptPay, TrueMoney, SCB) |
| 2C2P | อาเซียน (PromptPay, GrabPay) |

PCI-DSS considerations:
- **ห้าม** เก็บ card number ในฐานข้อมูลเอง
- ใช้ tokenization จาก payment gateway
- เก็บเฉพาะ token + last 4 digits
- HTTPS everywhere
- Audit log ทุก payment operation

---

**ก่อนหน้า → [Part 85: Capstone Search](part85-capstone-search.md)**  
**ต่อไป → [Part 87: Capstone - Performance & Scalability](part87-capstone-performance.md)**
