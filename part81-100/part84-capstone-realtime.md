# Part 84: Capstone - Real-time Features (ฟีเจอร์แบบเรียลไทม์)

## ภาพรวม

ในส่วนนี้เราจะพัฒนาฟีเจอร์แบบเรียลไทม์สำหรับโปรเจกต์ ShopThai โดยใช้ SignalR, WebPush, และเทคโนโลยีอื่นๆ เพื่อสร้างประสบการณ์ที่ดีให้กับผู้ใช้ในรูปแบบ interactive และ live

ฟีเจอร์เรียลไทม์ที่เราจะพัฒนาในส่วนนี้:
- การติดตามออเดอร์แบบสด (Live Order Tracking)
- แดชบอร์ดผู้ดูแลระบบแบบเรียลไทม์ (Admin Real-time Dashboard)
- การอัพเดทสต็อกสินค้าแบบสด (Live Inventory Updates)
- ระบบแชทสนับสนุนลูกค้า (Chat Support System)
- การแจ้งเตือนผ่าน Browser (Push Notifications)
- การค้นหาแบบเรียลไทม์ (Real-time Search Suggestions)
- ระบบประมูล/ฟลากเซล (Auction/Flash Sale)
- การอัพเดทราคาสินค้าแบบสด (Live Price Updates)
- ตะกร้าสินค้าร่วมกัน (Multi-user Cart Collaboration)
- การเพิ่มประสิทธิภาพ SignalR (Performance Optimization)

---

## Step 831: Live Order Tracking with SignalR

### อธิบาย

การติดตามออเดอร์แบบสดเป็นฟีเจอร์สำคัญที่ช่วยให้ลูกค้าสามารถเห็นสถานะออเดอร์ของตนเองแบบ real-time โดยไม่ต้องรีเฟรชหน้าเว็บ เราจะใช้ SignalR เพื่อสร้างการเชื่อมต่อแบบ persistent ระหว่าง server และ client

### การสร้าง Typed Hub Interface

```csharp
// Hubs/IOrderTrackingClient.cs
public interface IOrderTrackingClient
{
    Task OrderStatusUpdated(OrderStatusUpdate update);
    Task OrderLocationUpdated(LocationUpdate location);
    Task OrderDelivered(OrderDeliveryConfirmation confirmation);
    Task EstimatedTimeUpdated(TimeEstimate estimate);
    Task OrderCancelled(OrderCancellationInfo info);
}

public class OrderStatusUpdate
{
    public Guid OrderId { get; set; }
    public string OrderNumber { get; set; } = string.Empty;
    public OrderStatus NewStatus { get; set; }
    public OrderStatus PreviousStatus { get; set; }
    public string StatusMessage { get; set; } = string.Empty;
    public DateTime UpdatedAt { get; set; }
    public string? UpdatedBy { get; set; }
}

public class LocationUpdate
{
    public Guid OrderId { get; set; }
    public double Latitude { get; set; }
    public double Longitude { get; set; }
    public string? DeliveryPersonName { get; set; }
    public string? DeliveryPersonPhone { get; set; }
    public DateTime UpdatedAt { get; set; }
}

public enum OrderStatus
{
    Pending,
    Confirmed,
    Processing,
    ReadyForPickup,
    OutForDelivery,
    Delivered,
    Cancelled,
    Refunded
}
```

### OrderTrackingHub

```csharp
// Hubs/OrderTrackingHub.cs
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.SignalR;

[Authorize]
public class OrderTrackingHub : Hub<IOrderTrackingClient>
{
    private readonly IOrderRepository _orderRepository;
    private readonly ILogger<OrderTrackingHub> _logger;
    private readonly IConnectionTracker _connectionTracker;

    public OrderTrackingHub(
        IOrderRepository orderRepository,
        ILogger<OrderTrackingHub> logger,
        IConnectionTracker connectionTracker)
    {
        _orderRepository = orderRepository;
        _logger = logger;
        _connectionTracker = connectionTracker;
    }

    public override async Task OnConnectedAsync()
    {
        var userId = Context.UserIdentifier;
        if (userId != null)
        {
            await _connectionTracker.AddConnectionAsync(userId, Context.ConnectionId);
            _logger.LogInformation("User {UserId} connected to OrderTrackingHub", userId);
        }
        await base.OnConnectedAsync();
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        var userId = Context.UserIdentifier;
        if (userId != null)
        {
            await _connectionTracker.RemoveConnectionAsync(userId, Context.ConnectionId);
        }
        await base.OnDisconnectedAsync(exception);
    }

    // ลูกค้าสมัครรับการแจ้งเตือนออเดอร์เฉพาะ
    public async Task SubscribeToOrder(Guid orderId)
    {
        var userId = Context.UserIdentifier;
        if (userId == null) return;

        // ตรวจสอบว่าออเดอร์นี้เป็นของลูกค้าคนนี้หรือไม่
        var order = await _orderRepository.GetByIdAsync(orderId);
        if (order == null || order.CustomerId.ToString() != userId)
        {
            throw new HubException("ไม่มีสิทธิ์ติดตามออเดอร์นี้");
        }

        var groupName = GetOrderGroupName(orderId);
        await Groups.AddToGroupAsync(Context.ConnectionId, groupName);
        
        _logger.LogInformation("User {UserId} subscribed to order {OrderId}", userId, orderId);
        
        // ส่งสถานะปัจจุบันกลับไป
        await Clients.Caller.OrderStatusUpdated(new OrderStatusUpdate
        {
            OrderId = orderId,
            OrderNumber = order.OrderNumber,
            NewStatus = order.Status,
            PreviousStatus = order.Status,
            StatusMessage = GetStatusMessage(order.Status),
            UpdatedAt = order.UpdatedAt
        });
    }

    public async Task UnsubscribeFromOrder(Guid orderId)
    {
        var groupName = GetOrderGroupName(orderId);
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, groupName);
    }

    private static string GetOrderGroupName(Guid orderId) => $"order-{orderId}";

    private static string GetStatusMessage(OrderStatus status) => status switch
    {
        OrderStatus.Pending => "รอการยืนยัน",
        OrderStatus.Confirmed => "ยืนยันออเดอร์แล้ว",
        OrderStatus.Processing => "กำลังเตรียมสินค้า",
        OrderStatus.ReadyForPickup => "พร้อมส่ง",
        OrderStatus.OutForDelivery => "กำลังจัดส่ง",
        OrderStatus.Delivered => "ส่งสำเร็จ",
        OrderStatus.Cancelled => "ยกเลิกออเดอร์",
        OrderStatus.Refunded => "คืนเงินแล้ว",
        _ => "ไม่ทราบสถานะ"
    };
}
```

### Order Status Service

```csharp
// Services/OrderStatusService.cs
public class OrderStatusService : IOrderStatusService
{
    private readonly IHubContext<OrderTrackingHub, IOrderTrackingClient> _hubContext;
    private readonly IOrderRepository _orderRepository;
    private readonly ILogger<OrderStatusService> _logger;

    public OrderStatusService(
        IHubContext<OrderTrackingHub, IOrderTrackingClient> hubContext,
        IOrderRepository orderRepository,
        ILogger<OrderStatusService> logger)
    {
        _hubContext = hubContext;
        _orderRepository = orderRepository;
        _logger = logger;
    }

    public async Task UpdateOrderStatusAsync(Guid orderId, OrderStatus newStatus, string? updatedBy = null)
    {
        var order = await _orderRepository.GetByIdAsync(orderId);
        if (order == null) throw new NotFoundException($"Order {orderId} not found");

        var previousStatus = order.Status;
        order.Status = newStatus;
        order.UpdatedAt = DateTime.UtcNow;
        
        await _orderRepository.UpdateAsync(order);

        // แจ้งเตือนผ่าน SignalR ไปยัง group ของออเดอร์
        var groupName = $"order-{orderId}";
        await _hubContext.Clients.Group(groupName).OrderStatusUpdated(new OrderStatusUpdate
        {
            OrderId = orderId,
            OrderNumber = order.OrderNumber,
            NewStatus = newStatus,
            PreviousStatus = previousStatus,
            StatusMessage = GetStatusMessage(newStatus),
            UpdatedAt = DateTime.UtcNow,
            UpdatedBy = updatedBy
        });

        _logger.LogInformation(
            "Order {OrderId} status updated from {PreviousStatus} to {NewStatus}",
            orderId, previousStatus, newStatus);
    }

    public async Task UpdateDeliveryLocationAsync(Guid orderId, double lat, double lng, string? deliveryPersonName = null)
    {
        var groupName = $"order-{orderId}";
        await _hubContext.Clients.Group(groupName).OrderLocationUpdated(new LocationUpdate
        {
            OrderId = orderId,
            Latitude = lat,
            Longitude = lng,
            DeliveryPersonName = deliveryPersonName,
            UpdatedAt = DateTime.UtcNow
        });
    }

    private static string GetStatusMessage(OrderStatus status) => status switch
    {
        OrderStatus.Pending => "รอการยืนยันจากร้านค้า",
        OrderStatus.Confirmed => "ร้านค้ายืนยันออเดอร์แล้ว",
        OrderStatus.Processing => "กำลังเตรียมสินค้าให้คุณ",
        OrderStatus.ReadyForPickup => "สินค้าพร้อมส่งแล้ว",
        OrderStatus.OutForDelivery => "คนขนส่งกำลังมาส่งสินค้า",
        OrderStatus.Delivered => "ได้รับสินค้าแล้ว ขอบคุณที่ใช้บริการ",
        OrderStatus.Cancelled => "ออเดอร์ถูกยกเลิก",
        OrderStatus.Refunded => "ดำเนินการคืนเงินแล้ว",
        _ => string.Empty
    };
}
```

---

## Step 832: Admin Real-time Dashboard

### อธิบาย

แดชบอร์ดผู้ดูแลระบบแบบเรียลไทม์ช่วยให้ admin สามารถเห็นข้อมูลสำคัญต่างๆ แบบ live เช่น ยอดขาย, จำนวนผู้ใช้ที่ active, การแจ้งเตือนสต็อก และอื่นๆ เราจะใช้ Background Service เพื่อ broadcast ข้อมูลไปยัง admin clients

### AdminDashboard Hub Interface

```csharp
// Hubs/IAdminDashboardClient.cs
public interface IAdminDashboardClient
{
    Task MetricsUpdated(DashboardMetrics metrics);
    Task NewOrderReceived(OrderSummary order);
    Task LowStockAlert(LowStockItem item);
    Task UserActivityUpdated(UserActivityMetrics activity);
    Task RevenueUpdated(RevenueData revenue);
    Task SystemAlertReceived(SystemAlert alert);
}

public class DashboardMetrics
{
    public int TotalOrdersToday { get; set; }
    public decimal TotalRevenueToday { get; set; }
    public int ActiveUsers { get; set; }
    public int PendingOrders { get; set; }
    public int LowStockItems { get; set; }
    public decimal AverageOrderValue { get; set; }
    public DateTime LastUpdated { get; set; }
}

public class UserActivityMetrics
{
    public int OnlineUsers { get; set; }
    public int UsersViewingProducts { get; set; }
    public int UsersInCheckout { get; set; }
    public List<PageViewStat> TopPages { get; set; } = new();
}

public class PageViewStat
{
    public string PageName { get; set; } = string.Empty;
    public int ViewCount { get; set; }
}
```

### AdminDashboardHub

```csharp
// Hubs/AdminDashboardHub.cs
[Authorize(Roles = "Admin")]
public class AdminDashboardHub : Hub<IAdminDashboardClient>
{
    private readonly ILogger<AdminDashboardHub> _logger;

    public AdminDashboardHub(ILogger<AdminDashboardHub> logger)
    {
        _logger = logger;
    }

    public override async Task OnConnectedAsync()
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, "admins");
        _logger.LogInformation("Admin {UserId} connected to dashboard", Context.UserIdentifier);
        await base.OnConnectedAsync();
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, "admins");
        await base.OnDisconnectedAsync(exception);
    }

    // Admin สามารถกรองข้อมูลตามช่วงเวลาได้
    public async Task SetMetricsFilter(MetricsFilter filter)
    {
        var groupName = $"admin-filter-{filter.TimeRange}";
        await Groups.AddToGroupAsync(Context.ConnectionId, groupName);
    }
}
```

### Background Service สำหรับ Broadcast Metrics

```csharp
// Services/DashboardMetricsBroadcastService.cs
public class DashboardMetricsBroadcastService : BackgroundService
{
    private readonly IHubContext<AdminDashboardHub, IAdminDashboardClient> _hubContext;
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<DashboardMetricsBroadcastService> _logger;
    private readonly TimeSpan _broadcastInterval = TimeSpan.FromSeconds(5);

    public DashboardMetricsBroadcastService(
        IHubContext<AdminDashboardHub, IAdminDashboardClient> hubContext,
        IServiceProvider serviceProvider,
        ILogger<DashboardMetricsBroadcastService> logger)
    {
        _hubContext = hubContext;
        _serviceProvider = serviceProvider;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Dashboard metrics broadcast service started");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await BroadcastMetricsAsync();
                await Task.Delay(_broadcastInterval, stoppingToken);
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
            {
                break;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error broadcasting dashboard metrics");
                await Task.Delay(TimeSpan.FromSeconds(10), stoppingToken);
            }
        }
    }

    private async Task BroadcastMetricsAsync()
    {
        using var scope = _serviceProvider.CreateScope();
        var metricsService = scope.ServiceProvider.GetRequiredService<IDashboardMetricsService>();

        var metrics = await metricsService.GetCurrentMetricsAsync();
        var userActivity = await metricsService.GetUserActivityAsync();

        await _hubContext.Clients.Group("admins").MetricsUpdated(metrics);
        await _hubContext.Clients.Group("admins").UserActivityUpdated(userActivity);
    }
}
```

### DashboardMetricsService

```csharp
// Services/DashboardMetricsService.cs
public class DashboardMetricsService : IDashboardMetricsService
{
    private readonly IOrderRepository _orderRepository;
    private readonly IProductRepository _productRepository;
    private readonly IConnectionTracker _connectionTracker;
    private readonly IMemoryCache _cache;

    public DashboardMetricsService(
        IOrderRepository orderRepository,
        IProductRepository productRepository,
        IConnectionTracker connectionTracker,
        IMemoryCache cache)
    {
        _orderRepository = orderRepository;
        _productRepository = productRepository;
        _connectionTracker = connectionTracker;
        _cache = cache;
    }

    public async Task<DashboardMetrics> GetCurrentMetricsAsync()
    {
        var cacheKey = "dashboard_metrics";
        if (_cache.TryGetValue(cacheKey, out DashboardMetrics? cached) && cached != null)
            return cached;

        var today = DateTime.UtcNow.Date;
        var todayOrders = await _orderRepository.GetOrdersByDateRangeAsync(today, today.AddDays(1));
        var lowStockItems = await _productRepository.GetLowStockItemsAsync(threshold: 10);

        var metrics = new DashboardMetrics
        {
            TotalOrdersToday = todayOrders.Count,
            TotalRevenueToday = todayOrders.Sum(o => o.TotalAmount),
            ActiveUsers = await _connectionTracker.GetActiveUserCountAsync(),
            PendingOrders = todayOrders.Count(o => o.Status == OrderStatus.Pending),
            LowStockItems = lowStockItems.Count,
            AverageOrderValue = todayOrders.Any() 
                ? todayOrders.Average(o => o.TotalAmount) 
                : 0,
            LastUpdated = DateTime.UtcNow
        };

        _cache.Set(cacheKey, metrics, TimeSpan.FromSeconds(5));
        return metrics;
    }

    public async Task<UserActivityMetrics> GetUserActivityAsync()
    {
        var activeConnections = await _connectionTracker.GetActiveConnectionsAsync();
        
        return new UserActivityMetrics
        {
            OnlineUsers = activeConnections.DistinctBy(c => c.UserId).Count(),
            UsersViewingProducts = activeConnections.Count(c => c.CurrentPage?.StartsWith("/products") == true),
            UsersInCheckout = activeConnections.Count(c => c.CurrentPage?.StartsWith("/checkout") == true)
        };
    }
}
```

---

## Step 833: Live Inventory Updates

### อธิบาย

การอัพเดทสต็อกสินค้าแบบสดช่วยให้ลูกค้าที่กำลังดูสินค้าสามารถเห็นการเปลี่ยนแปลงสต็อกแบบ real-time ป้องกันการสั่งซื้อสินค้าที่หมดแล้ว เราจะใช้ System.Threading.Channels เพื่อจัดการ inventory events

### Inventory Channel และ Events

```csharp
// Events/InventoryEvents.cs
public abstract class InventoryEvent
{
    public Guid ProductId { get; set; }
    public string ProductName { get; set; } = string.Empty;
    public DateTime OccurredAt { get; set; } = DateTime.UtcNow;
}

public class StockReducedEvent : InventoryEvent
{
    public int PreviousStock { get; set; }
    public int CurrentStock { get; set; }
    public int QuantityReduced { get; set; }
    public bool IsLowStock => CurrentStock <= 10;
    public bool IsOutOfStock => CurrentStock == 0;
}

public class StockReplenishedEvent : InventoryEvent
{
    public int PreviousStock { get; set; }
    public int NewStock { get; set; }
    public int QuantityAdded { get; set; }
}

public class ProductUnavailableEvent : InventoryEvent
{
    public string Reason { get; set; } = string.Empty;
}
```

### Inventory Channel Service

```csharp
// Services/InventoryChannelService.cs
using System.Threading.Channels;

public class InventoryChannelService
{
    private readonly Channel<InventoryEvent> _channel;
    
    public InventoryChannelService()
    {
        var options = new BoundedChannelOptions(1000)
        {
            FullMode = BoundedChannelFullMode.DropOldest,
            SingleReader = false,
            SingleWriter = false
        };
        _channel = Channel.CreateBounded<InventoryEvent>(options);
    }

    public ChannelWriter<InventoryEvent> Writer => _channel.Writer;
    public ChannelReader<InventoryEvent> Reader => _channel.Reader;
}
```

### Inventory Broadcast Service

```csharp
// Services/InventoryBroadcastService.cs
public class InventoryBroadcastService : BackgroundService
{
    private readonly InventoryChannelService _channelService;
    private readonly IHubContext<InventoryHub> _hubContext;
    private readonly ILogger<InventoryBroadcastService> _logger;

    public InventoryBroadcastService(
        InventoryChannelService channelService,
        IHubContext<InventoryHub> hubContext,
        ILogger<InventoryBroadcastService> logger)
    {
        _channelService = channelService;
        _hubContext = hubContext;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var inventoryEvent in _channelService.Reader.ReadAllAsync(stoppingToken))
        {
            try
            {
                await ProcessInventoryEventAsync(inventoryEvent);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing inventory event for product {ProductId}", 
                    inventoryEvent.ProductId);
            }
        }
    }

    private async Task ProcessInventoryEventAsync(InventoryEvent inventoryEvent)
    {
        var productGroup = $"product-{inventoryEvent.ProductId}";

        switch (inventoryEvent)
        {
            case StockReducedEvent reduced:
                await _hubContext.Clients.Group(productGroup).SendAsync("StockUpdated", new
                {
                    productId = reduced.ProductId,
                    currentStock = reduced.CurrentStock,
                    isLowStock = reduced.IsLowStock,
                    isOutOfStock = reduced.IsOutOfStock
                });

                if (reduced.IsOutOfStock)
                {
                    await _hubContext.Clients.All.SendAsync("ProductOutOfStock", new
                    {
                        productId = reduced.ProductId,
                        productName = reduced.ProductName
                    });
                }
                break;

            case StockReplenishedEvent replenished:
                await _hubContext.Clients.Group(productGroup).SendAsync("StockReplenished", new
                {
                    productId = replenished.ProductId,
                    newStock = replenished.NewStock,
                    quantityAdded = replenished.QuantityAdded
                });
                break;
        }
    }
}
```

### InventoryHub

```csharp
// Hubs/InventoryHub.cs
public class InventoryHub : Hub
{
    private readonly IProductRepository _productRepository;

    public InventoryHub(IProductRepository productRepository)
    {
        _productRepository = productRepository;
    }

    // ลูกค้าสมัครรับการแจ้งเตือนสต็อกสินค้าเฉพาะ
    public async Task WatchProduct(Guid productId)
    {
        var groupName = $"product-{productId}";
        await Groups.AddToGroupAsync(Context.ConnectionId, groupName);

        var product = await _productRepository.GetByIdAsync(productId);
        if (product != null)
        {
            await Clients.Caller.SendAsync("CurrentStock", new
            {
                productId = product.Id,
                currentStock = product.StockQuantity,
                isLowStock = product.StockQuantity <= 10,
                isOutOfStock = product.StockQuantity == 0
            });
        }
    }

    public async Task UnwatchProduct(Guid productId)
    {
        var groupName = $"product-{productId}";
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, groupName);
    }
}
```

---

## Step 834: Chat Support System

### อธิบาย

ระบบแชทสนับสนุนช่วยให้ลูกค้าสามารถส่งข้อความถึงเจ้าหน้าที่ได้แบบ real-time เราจะสร้างระบบที่รองรับการ routing ข้อความไปยัง agent ที่ว่าง และแสดง typing indicators

### Chat Hub Interface

```csharp
// Hubs/IChatClient.cs
public interface IChatClient
{
    Task MessageReceived(ChatMessage message);
    Task AgentTyping(TypingIndicator indicator);
    Task AgentConnected(AgentInfo agent);
    Task AgentDisconnected(AgentInfo agent);
    Task ChatSessionStarted(ChatSession session);
    Task ChatSessionEnded(ChatSession session);
    Task QueuePositionUpdated(int position);
}

public class ChatMessage
{
    public Guid MessageId { get; set; } = Guid.NewGuid();
    public Guid SessionId { get; set; }
    public string SenderId { get; set; } = string.Empty;
    public string SenderName { get; set; } = string.Empty;
    public SenderRole SenderRole { get; set; }
    public string Content { get; set; } = string.Empty;
    public MessageType Type { get; set; }
    public DateTime SentAt { get; set; } = DateTime.UtcNow;
    public bool IsRead { get; set; }
}

public enum SenderRole { Customer, Agent, System }
public enum MessageType { Text, Image, File, System }

public class TypingIndicator
{
    public Guid SessionId { get; set; }
    public string UserId { get; set; } = string.Empty;
    public bool IsTyping { get; set; }
}
```

### ChatHub

```csharp
// Hubs/ChatHub.cs
[Authorize]
public class ChatHub : Hub<IChatClient>
{
    private readonly IChatService _chatService;
    private readonly IAgentQueueService _agentQueueService;
    private readonly ILogger<ChatHub> _logger;

    public ChatHub(
        IChatService chatService,
        IAgentQueueService agentQueueService,
        ILogger<ChatHub> logger)
    {
        _chatService = chatService;
        _agentQueueService = agentQueueService;
        _logger = logger;
    }

    // ลูกค้าเริ่มต้น chat session
    public async Task<ChatSession> StartChatSession(string initialMessage)
    {
        var userId = Context.UserIdentifier!;
        var session = await _chatService.CreateSessionAsync(userId, initialMessage);
        
        await Groups.AddToGroupAsync(Context.ConnectionId, $"chat-{session.Id}");
        
        // หา agent ที่ว่าง
        var agent = await _agentQueueService.AssignAgentAsync(session.Id);
        
        if (agent != null)
        {
            await Clients.Caller.AgentConnected(agent);
        }
        else
        {
            var queuePosition = await _agentQueueService.GetQueuePositionAsync(session.Id);
            await Clients.Caller.QueuePositionUpdated(queuePosition);
        }

        return session;
    }

    // ส่งข้อความ
    public async Task SendMessage(Guid sessionId, string content, MessageType type = MessageType.Text)
    {
        var userId = Context.UserIdentifier!;
        
        var message = await _chatService.SaveMessageAsync(new SaveMessageRequest
        {
            SessionId = sessionId,
            SenderId = userId,
            Content = content,
            Type = type
        });

        // ส่งข้อความไปยังทุกคนใน session group
        await Clients.Group($"chat-{sessionId}").MessageReceived(message);
    }

    // แสดง typing indicator
    public async Task SendTypingIndicator(Guid sessionId, bool isTyping)
    {
        var userId = Context.UserIdentifier!;
        
        await Clients.OthersInGroup($"chat-{sessionId}").AgentTyping(new TypingIndicator
        {
            SessionId = sessionId,
            UserId = userId,
            IsTyping = isTyping
        });
    }

    // Agent เข้าร่วม chat session
    [Authorize(Roles = "Agent,Admin")]
    public async Task JoinChatSession(Guid sessionId)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, $"chat-{sessionId}");
        
        var agentInfo = new AgentInfo
        {
            AgentId = Context.UserIdentifier!,
            Name = Context.User?.FindFirst("name")?.Value ?? "Agent"
        };
        
        await Clients.Group($"chat-{sessionId}").AgentConnected(agentInfo);
        
        // ส่งประวัติการสนทนา
        var history = await _chatService.GetMessageHistoryAsync(sessionId);
        foreach (var msg in history)
        {
            await Clients.Caller.MessageReceived(msg);
        }
    }

    // จบ chat session
    public async Task EndChatSession(Guid sessionId)
    {
        var session = await _chatService.EndSessionAsync(sessionId);
        await Clients.Group($"chat-{sessionId}").ChatSessionEnded(session);
    }
}
```

---

## Step 835: Push Notifications with WebPush

### อธิบาย

Browser Push Notifications ช่วยให้เราสามารถส่งการแจ้งเตือนไปยังผู้ใช้แม้ว่าพวกเขาจะไม่ได้เปิดเว็บไซต์อยู่ เราจะใช้ Lib.Net.Http.WebPush library

### การติดตั้ง Package

```bash
dotnet add package Lib.Net.Http.WebPush
```

### VAPID Configuration

```csharp
// appsettings.json
{
  "WebPush": {
    "PublicKey": "BNbxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "PrivateKey": "xxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "Subject": "mailto:admin@shopthai.com"
  }
}

// Configuration/WebPushOptions.cs
public class WebPushOptions
{
    public const string Section = "WebPush";
    public string PublicKey { get; set; } = string.Empty;
    public string PrivateKey { get; set; } = string.Empty;
    public string Subject { get; set; } = string.Empty;
}
```

### Push Notification Service

```csharp
// Services/PushNotificationService.cs
using Lib.Net.Http.WebPush;
using Lib.Net.Http.WebPush.Authentication;

public class PushNotificationService : IPushNotificationService
{
    private readonly PushServiceClient _pushClient;
    private readonly ISubscriptionRepository _subscriptionRepository;
    private readonly ILogger<PushNotificationService> _logger;
    private readonly WebPushOptions _options;

    public PushNotificationService(
        PushServiceClient pushClient,
        ISubscriptionRepository subscriptionRepository,
        ILogger<PushNotificationService> logger,
        IOptions<WebPushOptions> options)
    {
        _pushClient = pushClient;
        _subscriptionRepository = subscriptionRepository;
        _logger = logger;
        _options = options.Value;
        
        _pushClient.DefaultAuthentication = new VapidAuthentication(
            _options.PublicKey,
            _options.PrivateKey)
        {
            Subject = _options.Subject
        };
    }

    public async Task SendNotificationAsync(string userId, PushNotificationPayload payload)
    {
        var subscriptions = await _subscriptionRepository.GetUserSubscriptionsAsync(userId);
        
        var tasks = subscriptions.Select(sub => SendToSubscriptionAsync(sub, payload));
        await Task.WhenAll(tasks);
    }

    public async Task SendBroadcastNotificationAsync(PushNotificationPayload payload)
    {
        var allSubscriptions = await _subscriptionRepository.GetAllActiveSubscriptionsAsync();
        
        // ส่งแบบ batch เพื่อไม่ให้ overload
        var batches = allSubscriptions.Chunk(100);
        foreach (var batch in batches)
        {
            var tasks = batch.Select(sub => SendToSubscriptionAsync(sub, payload));
            await Task.WhenAll(tasks);
        }
    }

    private async Task SendToSubscriptionAsync(PushSubscription subscription, PushNotificationPayload payload)
    {
        try
        {
            var message = new PushMessage(System.Text.Json.JsonSerializer.Serialize(payload))
            {
                Topic = payload.Tag,
                TimeToLive = 86400 // 24 hours
            };

            var pushSubscription = new Lib.Net.Http.WebPush.PushSubscription
            {
                Endpoint = subscription.Endpoint,
                Keys = new Dictionary<string, string>
                {
                    { "p256dh", subscription.P256dh },
                    { "auth", subscription.Auth }
                }
            };

            await _pushClient.RequestPushMessageDeliveryAsync(pushSubscription, message);
        }
        catch (PushServiceClientException ex) when (ex.StatusCode == System.Net.HttpStatusCode.Gone)
        {
            // Subscription หมดอายุแล้ว ลบออกจาก database
            await _subscriptionRepository.RemoveAsync(subscription.Id);
            _logger.LogInformation("Removed expired subscription {SubId}", subscription.Id);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error sending push notification to subscription {SubId}", subscription.Id);
        }
    }
}

public class PushNotificationPayload
{
    public string Title { get; set; } = string.Empty;
    public string Body { get; set; } = string.Empty;
    public string? Icon { get; set; }
    public string? Badge { get; set; }
    public string? Url { get; set; }
    public string? Tag { get; set; }
    public Dictionary<string, string> Data { get; set; } = new();
}
```

### Subscription Controller

```csharp
// Controllers/PushSubscriptionController.cs
[ApiController]
[Route("api/push-subscription")]
[Authorize]
public class PushSubscriptionController : ControllerBase
{
    private readonly ISubscriptionRepository _subscriptionRepository;
    private readonly IPushNotificationService _pushService;

    [HttpPost("subscribe")]
    public async Task<IActionResult> Subscribe([FromBody] SubscribeRequest request)
    {
        var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value!;
        
        var subscription = new PushSubscription
        {
            UserId = userId,
            Endpoint = request.Endpoint,
            P256dh = request.Keys.P256dh,
            Auth = request.Keys.Auth,
            CreatedAt = DateTime.UtcNow
        };
        
        await _subscriptionRepository.UpsertAsync(subscription);
        
        // ส่ง welcome notification
        await _pushService.SendNotificationAsync(userId, new PushNotificationPayload
        {
            Title = "ShopThai",
            Body = "เปิดการแจ้งเตือนสำเร็จแล้ว!",
            Icon = "/icons/icon-192x192.png",
            Tag = "welcome"
        });
        
        return Ok();
    }

    [HttpDelete("unsubscribe")]
    public async Task<IActionResult> Unsubscribe([FromBody] UnsubscribeRequest request)
    {
        await _subscriptionRepository.RemoveByEndpointAsync(request.Endpoint);
        return Ok();
    }

    [HttpGet("vapid-public-key")]
    [AllowAnonymous]
    public IActionResult GetVapidPublicKey([FromServices] IOptions<WebPushOptions> options)
    {
        return Ok(new { publicKey = options.Value.PublicKey });
    }
}
```

---

## Step 836: Real-time Search Suggestions

### อธิบาย

การค้นหาแบบเรียลไทม์ช่วยให้ผู้ใช้ได้รับ suggestions ขณะพิมพ์ โดยใช้ debouncing เพื่อลดจำนวน request และใช้ SignalR เพื่อส่งผลลัพธ์กลับอย่างรวดเร็ว

### Search Hub

```csharp
// Hubs/SearchHub.cs
public class SearchHub : Hub
{
    private readonly ISearchService _searchService;
    private readonly ILogger<SearchHub> _logger;
    
    // เก็บ timer สำหรับ debounce ของแต่ละ connection
    private static readonly ConcurrentDictionary<string, CancellationTokenSource> _debounceTokens = new();

    public SearchHub(ISearchService searchService, ILogger<SearchHub> logger)
    {
        _searchService = searchService;
        _logger = logger;
    }

    public override Task OnDisconnectedAsync(Exception? exception)
    {
        if (_debounceTokens.TryRemove(Context.ConnectionId, out var cts))
        {
            cts.Cancel();
            cts.Dispose();
        }
        return base.OnDisconnectedAsync(exception);
    }

    // รับ query จาก client พร้อม debounce 300ms
    public async Task Search(string query, SearchOptions options)
    {
        if (string.IsNullOrWhiteSpace(query) || query.Length < 2)
        {
            await Clients.Caller.SendAsync("SearchCleared");
            return;
        }

        // ยกเลิก request เก่าถ้ามี
        if (_debounceTokens.TryRemove(Context.ConnectionId, out var oldCts))
        {
            oldCts.Cancel();
            oldCts.Dispose();
        }

        var cts = new CancellationTokenSource();
        _debounceTokens[Context.ConnectionId] = cts;

        try
        {
            // รอ 300ms (debounce)
            await Task.Delay(300, cts.Token);
            
            var results = await _searchService.SearchSuggestionsAsync(query, options);
            
            await Clients.Caller.SendAsync("SearchResults", new
            {
                query,
                suggestions = results.Suggestions,
                categories = results.Categories,
                topProducts = results.TopProducts,
                totalCount = results.TotalCount
            });
        }
        catch (OperationCanceledException)
        {
            // คำขอถูกยกเลิกจาก debounce ใหม่ ไม่ต้องทำอะไร
        }
        finally
        {
            _debounceTokens.TryRemove(Context.ConnectionId, out _);
        }
    }
}

public class SearchOptions
{
    public int MaxSuggestions { get; set; } = 8;
    public string? CategoryFilter { get; set; }
    public decimal? MinPrice { get; set; }
    public decimal? MaxPrice { get; set; }
}
```

---

## Step 837: Auction/Flash Sale Countdown Timer

### อธิบาย

ระบบประมูลและฟลากเซลต้องการ countdown timer ที่แม่นยำและ synchronized ทุก client เราจะใช้ SignalR ส่ง tick events ทุกๆ วินาทีพร้อมข้อมูลการประมูลแบบ real-time

### Auction Models

```csharp
// Models/Auction.cs
public class Auction
{
    public Guid Id { get; set; }
    public Guid ProductId { get; set; }
    public string ProductName { get; set; } = string.Empty;
    public decimal StartingPrice { get; set; }
    public decimal CurrentPrice { get; set; }
    public decimal BidIncrement { get; set; }
    public DateTime StartTime { get; set; }
    public DateTime EndTime { get; set; }
    public string? CurrentHighestBidderId { get; set; }
    public string? CurrentHighestBidderName { get; set; }
    public int TotalBids { get; set; }
    public AuctionStatus Status { get; set; }
    
    public bool IsActive => Status == AuctionStatus.Active && DateTime.UtcNow < EndTime;
    public TimeSpan TimeRemaining => EndTime - DateTime.UtcNow;
}

public enum AuctionStatus { Upcoming, Active, Ended, Cancelled }
```

### Auction Hub

```csharp
// Hubs/AuctionHub.cs
public class AuctionHub : Hub
{
    private readonly IAuctionService _auctionService;
    private readonly ILogger<AuctionHub> _logger;

    public AuctionHub(IAuctionService auctionService, ILogger<AuctionHub> logger)
    {
        _auctionService = auctionService;
        _logger = logger;
    }

    public async Task JoinAuction(Guid auctionId)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, $"auction-{auctionId}");
        
        var auction = await _auctionService.GetAuctionAsync(auctionId);
        if (auction != null)
        {
            await Clients.Caller.SendAsync("AuctionState", new
            {
                auctionId = auction.Id,
                currentPrice = auction.CurrentPrice,
                currentHighestBidder = auction.CurrentHighestBidderName,
                totalBids = auction.TotalBids,
                timeRemaining = auction.TimeRemaining.TotalSeconds,
                status = auction.Status.ToString()
            });
        }
    }

    [Authorize]
    public async Task PlaceBid(Guid auctionId, decimal bidAmount)
    {
        var userId = Context.UserIdentifier!;
        
        try
        {
            var result = await _auctionService.PlaceBidAsync(auctionId, userId, bidAmount);
            
            // แจ้งทุกคนที่ดูการประมูลนี้
            await Clients.Group($"auction-{auctionId}").SendAsync("NewBid", new
            {
                auctionId,
                newPrice = result.NewPrice,
                bidderId = userId,
                bidderName = result.BidderName,
                totalBids = result.TotalBids,
                timeRemaining = result.TimeRemaining.TotalSeconds
            });
        }
        catch (InvalidOperationException ex)
        {
            await Clients.Caller.SendAsync("BidError", new { message = ex.Message });
        }
    }
}
```

### Auction Timer Service

```csharp
// Services/AuctionTimerService.cs
public class AuctionTimerService : BackgroundService
{
    private readonly IHubContext<AuctionHub> _hubContext;
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<AuctionTimerService> _logger;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await BroadcastAuctionTimersAsync();
                await Task.Delay(1000, stoppingToken); // broadcast ทุก 1 วินาที
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
            {
                break;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error in auction timer service");
            }
        }
    }

    private async Task BroadcastAuctionTimersAsync()
    {
        using var scope = _serviceProvider.CreateScope();
        var auctionRepo = scope.ServiceProvider.GetRequiredService<IAuctionRepository>();
        
        var activeAuctions = await auctionRepo.GetActiveAuctionsAsync();
        
        foreach (var auction in activeAuctions)
        {
            var timeRemaining = auction.EndTime - DateTime.UtcNow;
            
            if (timeRemaining <= TimeSpan.Zero)
            {
                // ประมูลสิ้นสุดแล้ว
                await _hubContext.Clients.Group($"auction-{auction.Id}")
                    .SendAsync("AuctionEnded", new
                    {
                        auctionId = auction.Id,
                        winner = auction.CurrentHighestBidderName,
                        finalPrice = auction.CurrentPrice
                    });
                    
                await auctionRepo.EndAuctionAsync(auction.Id);
            }
            else
            {
                // ส่ง tick
                await _hubContext.Clients.Group($"auction-{auction.Id}")
                    .SendAsync("TimerTick", new
                    {
                        auctionId = auction.Id,
                        timeRemaining = timeRemaining.TotalSeconds,
                        currentPrice = auction.CurrentPrice
                    });
            }
        }
    }
}
```

---

## Step 838: Live Price Updates for Products

### อธิบาย

ราคาสินค้าอาจเปลี่ยนแปลงตลอดเวลาเนื่องจาก promotions, dynamic pricing หรือการเปลี่ยนแปลงต้นทุน เราจะ broadcast การเปลี่ยนแปลงราคาไปยัง client ที่กำลังดูสินค้านั้น

### Price Update Service

```csharp
// Services/PriceUpdateService.cs
public class PriceUpdateService : IPriceUpdateService
{
    private readonly IHubContext<InventoryHub> _hubContext;
    private readonly IProductRepository _productRepository;
    private readonly ILogger<PriceUpdateService> _logger;

    public PriceUpdateService(
        IHubContext<InventoryHub> hubContext,
        IProductRepository productRepository,
        ILogger<PriceUpdateService> logger)
    {
        _hubContext = hubContext;
        _productRepository = productRepository;
        _logger = logger;
    }

    public async Task UpdatePriceAsync(Guid productId, decimal newPrice, string reason)
    {
        var product = await _productRepository.GetByIdAsync(productId);
        if (product == null) throw new NotFoundException($"Product {productId} not found");

        var oldPrice = product.Price;
        product.Price = newPrice;
        product.PriceUpdatedAt = DateTime.UtcNow;
        
        await _productRepository.UpdateAsync(product);

        // Broadcast ราคาใหม่ไปยังทุกคนที่กำลังดูสินค้านี้
        var groupName = $"product-{productId}";
        await _hubContext.Clients.Group(groupName).SendAsync("PriceUpdated", new
        {
            productId,
            oldPrice,
            newPrice,
            changePercent = ((newPrice - oldPrice) / oldPrice * 100).ToString("F2"),
            isIncrease = newPrice > oldPrice,
            reason,
            updatedAt = DateTime.UtcNow
        });

        _logger.LogInformation(
            "Product {ProductId} price updated from {OldPrice} to {NewPrice}",
            productId, oldPrice, newPrice);
    }

    public async Task ApplyBulkDiscountAsync(string categoryId, decimal discountPercent)
    {
        var products = await _productRepository.GetByCategoryAsync(categoryId);
        
        foreach (var product in products)
        {
            var discountAmount = product.Price * (discountPercent / 100);
            var newPrice = product.Price - discountAmount;
            
            await UpdatePriceAsync(product.Id, newPrice, $"Bulk discount {discountPercent}%");
        }
    }
}
```

---

## Step 839: Multi-user Shopping Cart Collaboration

### อธิบาย

ฟีเจอร์นี้อนุญาตให้หลายคนสามารถแชร์และร่วมกันแก้ไขตะกร้าสินค้าได้แบบ real-time เช่น ครอบครัวที่ช้อปด้วยกัน หรือการซื้อของขวัญกลุ่ม

### Shared Cart Hub

```csharp
// Hubs/SharedCartHub.cs
[Authorize]
public class SharedCartHub : Hub
{
    private readonly ISharedCartService _cartService;
    private readonly ILogger<SharedCartHub> _logger;

    public SharedCartHub(ISharedCartService cartService, ILogger<SharedCartHub> logger)
    {
        _cartService = cartService;
        _logger = logger;
    }

    // สร้างหรือเข้าร่วมตะกร้าแชร์
    public async Task<SharedCart> JoinSharedCart(string inviteCode)
    {
        var userId = Context.UserIdentifier!;
        var userName = Context.User?.FindFirst("name")?.Value ?? "ผู้ใช้";

        var cart = await _cartService.JoinCartAsync(inviteCode, userId, userName);
        await Groups.AddToGroupAsync(Context.ConnectionId, $"cart-{cart.Id}");

        // แจ้งคนอื่นว่ามีคนเข้าร่วม
        await Clients.OthersInGroup($"cart-{cart.Id}").SendAsync("UserJoined", new
        {
            userId,
            userName,
            joinedAt = DateTime.UtcNow
        });

        return cart;
    }

    // เพิ่มสินค้าในตะกร้าแชร์
    public async Task AddToSharedCart(Guid cartId, Guid productId, int quantity)
    {
        var userId = Context.UserIdentifier!;
        
        var result = await _cartService.AddItemAsync(cartId, productId, quantity, userId);

        await Clients.Group($"cart-{cartId}").SendAsync("CartItemAdded", new
        {
            cartId,
            productId,
            productName = result.ProductName,
            quantity,
            price = result.Price,
            addedBy = userId,
            addedByName = result.AddedByName,
            totalItems = result.TotalItems,
            totalPrice = result.TotalPrice
        });
    }

    // ลบสินค้าออกจากตะกร้าแชร์
    public async Task RemoveFromSharedCart(Guid cartId, Guid productId)
    {
        var userId = Context.UserIdentifier!;
        
        var result = await _cartService.RemoveItemAsync(cartId, productId, userId);

        await Clients.Group($"cart-{cartId}").SendAsync("CartItemRemoved", new
        {
            cartId,
            productId,
            removedBy = userId,
            totalItems = result.TotalItems,
            totalPrice = result.TotalPrice
        });
    }

    // Indicate ว่ากำลังพิมพ์/ดูสินค้า
    public async Task SetPresence(Guid cartId, string currentAction)
    {
        var userId = Context.UserIdentifier!;

        await Clients.OthersInGroup($"cart-{cartId}").SendAsync("UserPresenceUpdated", new
        {
            userId,
            currentAction, // เช่น "Viewing Electronics", "Adding to cart"
            updatedAt = DateTime.UtcNow
        });
    }
}
```

---

## Step 840: Performance Optimization for SignalR at Scale

### อธิบาย

เมื่อจำนวนผู้ใช้เพิ่มขึ้น การ scale SignalR ต้องใช้ Redis Backplane เพื่อให้ servers หลายตัวสามารถ communicate กันได้ และต้องมีการ optimize การใช้ทรัพยากรต่างๆ

### Redis Backplane Configuration

```csharp
// Program.cs หรือ Startup.cs
builder.Services.AddSignalR(options =>
{
    options.EnableDetailedErrors = builder.Environment.IsDevelopment();
    options.KeepAliveInterval = TimeSpan.FromSeconds(15);
    options.ClientTimeoutInterval = TimeSpan.FromSeconds(30);
    options.HandshakeTimeout = TimeSpan.FromSeconds(10);
    options.MaximumReceiveMessageSize = 32 * 1024; // 32KB
    options.StreamBufferCapacity = 10;
})
.AddStackExchangeRedis(builder.Configuration.GetConnectionString("Redis")!, options =>
{
    options.Configuration.ChannelPrefix = RedisChannel.Literal("ShopThai");
    options.Configuration.AbortOnConnectFail = false;
    options.Configuration.ConnectRetry = 3;
    options.Configuration.ReconnectRetryPolicy = new LinearRetry(1000);
});
```

### Redis Configuration Options

```csharp
// Configuration/RedisBackplaneOptions.cs
public class RedisBackplaneOptions
{
    public string ConnectionString { get; set; } = string.Empty;
    public string ChannelPrefix { get; set; } = "ShopThai";
    public int ConnectRetry { get; set; } = 3;
    public bool AbortOnConnectFail { get; set; } = false;
}

// Extensions/SignalRExtensions.cs
public static class SignalRExtensions
{
    public static ISignalRServerBuilder AddShopThaiRedisBackplane(
        this ISignalRServerBuilder builder,
        IConfiguration configuration)
    {
        var redisOptions = configuration.GetSection("Redis").Get<RedisBackplaneOptions>()
            ?? throw new InvalidOperationException("Redis configuration is missing");

        return builder.AddStackExchangeRedis(redisOptions.ConnectionString, options =>
        {
            options.Configuration.ChannelPrefix = RedisChannel.Literal(redisOptions.ChannelPrefix);
            options.Configuration.AbortOnConnectFail = redisOptions.AbortOnConnectFail;
            options.Configuration.ConnectRetry = redisOptions.ConnectRetry;
        });
    }
}
```

### Connection Tracker with Redis

```csharp
// Services/RedisConnectionTracker.cs
public class RedisConnectionTracker : IConnectionTracker
{
    private readonly IConnectionMultiplexer _redis;
    private readonly IDatabase _db;
    private readonly string _keyPrefix = "connections:";
    private readonly TimeSpan _connectionExpiry = TimeSpan.FromHours(1);

    public RedisConnectionTracker(IConnectionMultiplexer redis)
    {
        _redis = redis;
        _db = redis.GetDatabase();
    }

    public async Task AddConnectionAsync(string userId, string connectionId)
    {
        var key = $"{_keyPrefix}{userId}";
        await _db.SetAddAsync(key, connectionId);
        await _db.KeyExpireAsync(key, _connectionExpiry);
        
        // เพิ่มใน active users set
        await _db.SortedSetAddAsync("active_users", userId, DateTimeOffset.UtcNow.ToUnixTimeSeconds());
    }

    public async Task RemoveConnectionAsync(string userId, string connectionId)
    {
        var key = $"{_keyPrefix}{userId}";
        await _db.SetRemoveAsync(key, connectionId);
        
        // ตรวจสอบว่า user ยังมี connection อื่นอยู่ไหม
        var remainingConnections = await _db.SetLengthAsync(key);
        if (remainingConnections == 0)
        {
            await _db.SortedSetRemoveAsync("active_users", userId);
        }
    }

    public async Task<int> GetActiveUserCountAsync()
    {
        // นับ users ที่ active ในช่วง 5 นาทีที่ผ่านมา
        var cutoff = DateTimeOffset.UtcNow.AddMinutes(-5).ToUnixTimeSeconds();
        return (int)await _db.SortedSetLengthAsync("active_users", cutoff, double.PositiveInfinity);
    }

    public async Task<List<ConnectionInfo>> GetActiveConnectionsAsync()
    {
        var activeUserIds = await _db.SortedSetRangeByScoreAsync("active_users");
        var connections = new List<ConnectionInfo>();

        foreach (var userId in activeUserIds)
        {
            var userConnections = await _db.SetMembersAsync($"{_keyPrefix}{userId}");
            connections.AddRange(userConnections.Select(c => new ConnectionInfo
            {
                UserId = userId.ToString(),
                ConnectionId = c.ToString()
            }));
        }

        return connections;
    }
}
```

### Hub Filters สำหรับ Rate Limiting

```csharp
// Filters/RateLimitHubFilter.cs
public class RateLimitHubFilter : IHubFilter
{
    private readonly IRateLimiter _rateLimiter;
    private readonly ILogger<RateLimitHubFilter> _logger;

    public RateLimitHubFilter(IRateLimiter rateLimiter, ILogger<RateLimitHubFilter> logger)
    {
        _rateLimiter = rateLimiter;
        _logger = logger;
    }

    public async ValueTask<object?> InvokeMethodAsync(
        HubInvocationContext invocationContext,
        Func<HubInvocationContext, ValueTask<object?>> next)
    {
        var userId = invocationContext.Context.UserIdentifier ?? invocationContext.Context.ConnectionId;
        var methodName = invocationContext.HubMethodName;
        var key = $"{userId}:{methodName}";

        using var lease = await _rateLimiter.AcquireAsync(key);

        if (!lease.IsAcquired)
        {
            throw new HubException("Too many requests. Please slow down.");
        }

        return await next(invocationContext);
    }
}
```

### Message Compression

```csharp
// Program.cs - เปิดใช้ Compression
builder.Services.AddResponseCompression(opts =>
{
    opts.MimeTypes = ResponseCompressionDefaults.MimeTypes.Concat(
        new[] { "application/octet-stream" });
});

// ใช้ MessagePack แทน JSON สำหรับ binary protocol ที่เร็วกว่า
builder.Services.AddSignalR()
    .AddMessagePackProtocol(options =>
    {
        options.SerializerOptions = MessagePackSerializerOptions.Standard
            .WithSecurity(MessagePackSecurity.UntrustedData)
            .WithCompression(MessagePackCompression.Lz4BlockArray);
    });
```

### Horizontal Scaling Configuration

```yaml
# docker-compose.yml
version: '3.8'

services:
  api-1:
    image: shopthai-api:latest
    environment:
      - ConnectionStrings__Redis=redis:6379
      - ASPNETCORE_URLS=http://+:80
    depends_on:
      - redis
      
  api-2:
    image: shopthai-api:latest
    environment:
      - ConnectionStrings__Redis=redis:6379
      - ASPNETCORE_URLS=http://+:80
    depends_on:
      - redis
      
  nginx:
    image: nginx:latest
    ports:
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - api-1
      - api-2

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --maxmemory 512mb --maxmemory-policy allkeys-lru
```

```nginx
# nginx.conf - Sticky Sessions สำหรับ WebSocket
upstream shopthai_api {
    ip_hash;  # Sticky sessions
    server api-1:80;
    server api-2:80;
}

server {
    listen 443 ssl;
    server_name shopthai.com;

    location /hubs/ {
        proxy_pass http://shopthai_api;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;
    }
}
```

### Health Check สำหรับ SignalR

```csharp
// HealthChecks/SignalRHealthCheck.cs
public class SignalRHealthCheck : IHealthCheck
{
    private readonly IHubContext<OrderTrackingHub> _hubContext;
    private readonly IConnectionMultiplexer _redis;

    public SignalRHealthCheck(
        IHubContext<OrderTrackingHub> hubContext,
        IConnectionMultiplexer redis)
    {
        _hubContext = hubContext;
        _redis = redis;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            // ทดสอบ Redis connection
            var redisDb = _redis.GetDatabase();
            await redisDb.PingAsync();

            return HealthCheckResult.Healthy("SignalR and Redis are healthy");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("SignalR/Redis health check failed", ex);
        }
    }
}

// Program.cs
builder.Services.AddHealthChecks()
    .AddCheck<SignalRHealthCheck>("signalr")
    .AddRedis(builder.Configuration.GetConnectionString("Redis")!)
    .AddSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")!);
```

### SignalR Metrics และ Monitoring

```csharp
// Services/SignalRMetricsService.cs
public class SignalRMetricsService
{
    private readonly IConnectionTracker _connectionTracker;
    private readonly ILogger<SignalRMetricsService> _logger;
    
    // Prometheus metrics
    private static readonly Counter ConnectionsTotal = Metrics
        .CreateCounter("signalr_connections_total", "Total number of SignalR connections");
    
    private static readonly Gauge ActiveConnections = Metrics
        .CreateGauge("signalr_active_connections", "Current active SignalR connections");
    
    private static readonly Counter MessagesTotal = Metrics
        .CreateCounter("signalr_messages_total", "Total messages sent through SignalR",
            new CounterConfiguration { LabelNames = new[] { "hub", "method" } });

    public void RecordConnection()
    {
        ConnectionsTotal.Inc();
        ActiveConnections.Inc();
    }

    public void RecordDisconnection()
    {
        ActiveConnections.Dec();
    }

    public void RecordMessage(string hubName, string methodName)
    {
        MessagesTotal.WithLabels(hubName, methodName).Inc();
    }
}
```

### การ Register Services ทั้งหมด

```csharp
// Extensions/RealtimeServicesExtensions.cs
public static class RealtimeServicesExtensions
{
    public static IServiceCollection AddRealtimeServices(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // SignalR
        services.AddSignalR(options =>
        {
            options.EnableDetailedErrors = false;
            options.KeepAliveInterval = TimeSpan.FromSeconds(15);
            options.ClientTimeoutInterval = TimeSpan.FromSeconds(30);
        })
        .AddShopThaiRedisBackplane(configuration)
        .AddMessagePackProtocol();

        // Connection Tracking
        services.AddSingleton<IConnectionTracker, RedisConnectionTracker>();

        // Inventory Channel
        services.AddSingleton<InventoryChannelService>();
        services.AddHostedService<InventoryBroadcastService>();

        // Dashboard Metrics Broadcasting
        services.AddHostedService<DashboardMetricsBroadcastService>();

        // Auction Timer
        services.AddHostedService<AuctionTimerService>();

        // Push Notifications
        services.Configure<WebPushOptions>(configuration.GetSection(WebPushOptions.Section));
        services.AddSingleton<PushServiceClient>();
        services.AddScoped<IPushNotificationService, PushNotificationService>();

        // Application Services
        services.AddScoped<IOrderStatusService, OrderStatusService>();
        services.AddScoped<IPriceUpdateService, PriceUpdateService>();
        services.AddScoped<IDashboardMetricsService, DashboardMetricsService>();
        services.AddScoped<ISearchService, SearchService>();
        services.AddScoped<IAuctionService, AuctionService>();
        services.AddScoped<IChatService, ChatService>();
        services.AddScoped<ISharedCartService, SharedCartService>();

        // Rate Limiting สำหรับ Hub
        services.AddSingleton<IRateLimiter, SlidingWindowRateLimiter>();

        // Metrics
        services.AddSingleton<SignalRMetricsService>();

        return services;
    }

    public static WebApplication MapRealtimeHubs(this WebApplication app)
    {
        app.MapHub<OrderTrackingHub>("/hubs/orders");
        app.MapHub<AdminDashboardHub>("/hubs/admin");
        app.MapHub<InventoryHub>("/hubs/inventory");
        app.MapHub<ChatHub>("/hubs/chat");
        app.MapHub<SearchHub>("/hubs/search");
        app.MapHub<AuctionHub>("/hubs/auction");
        app.MapHub<SharedCartHub>("/hubs/shared-cart");

        return app;
    }
}
```

---

## สรุปภาพรวม Real-time Features ทั้งหมด

ในส่วนนี้เราได้พัฒนาฟีเจอร์ real-time ครบครันสำหรับ ShopThai:

| ฟีเจอร์ | Hub | Protocol | ลักษณะการทำงาน |
|---------|-----|----------|----------------|
| Order Tracking | OrderTrackingHub | Typed | ลูกค้า subscribe ตาม orderId |
| Admin Dashboard | AdminDashboardHub | Typed | BackgroundService broadcast ทุก 5 วิ |
| Live Inventory | InventoryHub | Channel | Channel-based broadcasting |
| Chat Support | ChatHub | Typed | P2P messaging ผ่าน groups |
| Push Notification | - | WebPush API | Browser notification |
| Search | SearchHub | Dynamic | Debounce 300ms |
| Auction Timer | AuctionHub | Dynamic | Timer broadcast ทุก 1 วิ |
| Price Updates | InventoryHub | Dynamic | Event-driven |
| Shared Cart | SharedCartHub | Dynamic | Multi-user collaboration |

### Key Performance Considerations

1. **Redis Backplane**: จำเป็นสำหรับ horizontal scaling โดย Redis ทำหน้าที่เป็น message broker ระหว่าง server instances
2. **MessagePack**: ใช้ binary protocol แทน JSON ลด payload size ได้ 30-40%
3. **Connection Groups**: จัดกลุ่ม connections เพื่อส่งข้อความไปยังเฉพาะคนที่ต้องการ ไม่ broadcast ทั้งหมด
4. **Debouncing**: ลด server load จาก search requests
5. **Rate Limiting**: ป้องกัน abuse ผ่าน HubFilter
6. **Sticky Sessions**: Nginx ip_hash ช่วยให้ WebSocket connection ไปถึง server เดิม

---

**ก่อนหน้า → [Part 83: Capstone DevOps](part83-capstone-devops.md)**

**ต่อไป → [Part 85: Capstone - Advanced Search](part85-capstone-search.md)**
