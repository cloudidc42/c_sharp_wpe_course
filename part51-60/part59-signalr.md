# Part 59: SignalR - Real-time Communication
## ขั้นตอนที่ 581-590: Real-time Apps ด้วย SignalR

---

## 🎯 เป้าหมายของ Part นี้
- SignalR คืออะไร
- Hub และ Hub<T>
- Client methods และ Groups
- ส่ง message แบบ broadcast, group, user-specific
- Reconnection handling
- SignalR ใน Blazor
- Real-time Dashboard

---

## ขั้นตอนที่ 581: SignalR Overview & Setup

```csharp
// SignalR: real-time bidirectional communication
// Uses WebSockets (fallback: SSE, Long Polling)

// Server setup
// Install: dotnet add package Microsoft.AspNetCore.SignalR

// Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSignalR(options =>
{
    options.EnableDetailedErrors = true; // development only
    options.MaximumReceiveMessageSize = 102400; // 100KB
    options.ClientTimeoutInterval = TimeSpan.FromSeconds(60);
    options.KeepAliveInterval = TimeSpan.FromSeconds(15);
});
builder.Services.AddCors(opt =>
{
    opt.AddPolicy("AllowAll", p => p
        .WithOrigins("https://myapp.com")
        .AllowAnyMethod()
        .AllowAnyHeader()
        .AllowCredentials()); // required for SignalR
});

var app = builder.Build();
app.UseCors("AllowAll");

// Map hubs
app.MapHub<ChatHub>("/hubs/chat");
app.MapHub<DashboardHub>("/hubs/dashboard");
app.MapHub<NotificationHub>("/hubs/notifications");

app.Run();
```

---

## ขั้นตอนที่ 582: Chat Hub

```csharp
// Hubs/ChatHub.cs
public class ChatHub : Hub
{
    private readonly IChatService _chatService;
    private readonly ILogger<ChatHub> _logger;
    
    public ChatHub(IChatService chat, ILogger<ChatHub> logger)
    { _chatService = chat; _logger = logger; }
    
    // Called when client connects
    public override async Task OnConnectedAsync()
    {
        var userId = Context.UserIdentifier!;
        var user = await _chatService.GetUserAsync(userId);
        
        // Join user-specific group
        await Groups.AddToGroupAsync(Context.ConnectionId, $"user:{userId}");
        
        // Notify others in their rooms
        foreach (var roomId in user.JoinedRooms)
        {
            await Groups.AddToGroupAsync(Context.ConnectionId, $"room:{roomId}");
            await Clients.OthersInGroup($"room:{roomId}").SendAsync("UserOnline", userId);
        }
        
        _logger.LogInformation("User {UserId} connected: {ConnId}", userId, Context.ConnectionId);
        await base.OnConnectedAsync();
    }
    
    // Called when client disconnects
    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        var userId = Context.UserIdentifier!;
        await Clients.All.SendAsync("UserOffline", userId);
        _logger.LogInformation("User {UserId} disconnected", userId);
        await base.OnDisconnectedAsync(exception);
    }
    
    // Client calls this to send a message
    public async Task SendMessage(string roomId, string message)
    {
        var userId = Context.UserIdentifier!;
        if (string.IsNullOrWhiteSpace(message)) return;
        
        var savedMessage = await _chatService.SaveMessageAsync(roomId, userId, message);
        
        // Broadcast to everyone in the room (including sender)
        await Clients.Group($"room:{roomId}").SendAsync("ReceiveMessage", new
        {
            savedMessage.Id,
            savedMessage.Text,
            savedMessage.SenderName,
            savedMessage.SentAt
        });
    }
    
    public async Task JoinRoom(string roomId)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, $"room:{roomId}");
        await Clients.OthersInGroup($"room:{roomId}").SendAsync("UserJoined", Context.UserIdentifier);
    }
    
    public async Task LeaveRoom(string roomId)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, $"room:{roomId}");
        await Clients.OthersInGroup($"room:{roomId}").SendAsync("UserLeft", Context.UserIdentifier);
    }
    
    public async Task StartTyping(string roomId)
        => await Clients.OthersInGroup($"room:{roomId}").SendAsync("UserTyping", Context.UserIdentifier);
    
    public async Task StopTyping(string roomId)
        => await Clients.OthersInGroup($"room:{roomId}").SendAsync("UserStoppedTyping", Context.UserIdentifier);
}
```

---

## ขั้นตอนที่ 583: Strongly-Typed Hub

```csharp
// Strongly-typed hub: compiler checks client method names
public interface IChatClient
{
    Task ReceiveMessage(MessageDto message);
    Task UserOnline(string userId);
    Task UserOffline(string userId);
    Task UserTyping(string userId, string roomId);
    Task RoomUpdated(RoomDto room);
}

public class TypedChatHub : Hub<IChatClient>
{
    public async Task SendMessage(string roomId, string text)
    {
        var msg = new MessageDto
        {
            Id = Guid.NewGuid(),
            Text = text,
            SenderId = Context.UserIdentifier!,
            SentAt = DateTime.UtcNow
        };
        
        // Compiler error if 'ReceiveMessage' doesn't exist in IChatClient!
        await Clients.Group($"room:{roomId}").ReceiveMessage(msg);
    }
}

// Send from outside Hub (e.g., background service)
public class NotificationService
{
    private readonly IHubContext<TypedChatHub, IChatClient> _hubContext;
    
    public NotificationService(IHubContext<TypedChatHub, IChatClient> hub) => _hubContext = hub;
    
    // Notify specific user
    public async Task NotifyUserAsync(string userId, MessageDto message)
        => await _hubContext.Clients.User(userId).ReceiveMessage(message);
    
    // Broadcast to all
    public async Task BroadcastAsync(MessageDto message)
        => await _hubContext.Clients.All.ReceiveMessage(message);
    
    // Send to group
    public async Task NotifyRoomAsync(string roomId, MessageDto message)
        => await _hubContext.Clients.Group($"room:{roomId}").ReceiveMessage(message);
}
```

---

## ขั้นตอนที่ 584: Dashboard Hub - Real-time Metrics

```csharp
// Hubs/DashboardHub.cs - Push live metrics to dashboard
public interface IDashboardClient
{
    Task ReceiveMetrics(DashboardMetrics metrics);
    Task OrderCreated(OrderSummary order);
    Task StockAlert(StockAlertDto alert);
}

public class DashboardHub : Hub<IDashboardClient>
{
    public override async Task OnConnectedAsync()
    {
        // Add to role-based groups
        if (Context.User?.IsInRole("Admin") == true)
            await Groups.AddToGroupAsync(Context.ConnectionId, "admins");
        else if (Context.User?.IsInRole("Manager") == true)
            await Groups.AddToGroupAsync(Context.ConnectionId, "managers");
        
        await base.OnConnectedAsync();
    }
    
    public async Task SubscribeToMetrics(string[] metricTypes)
    {
        foreach (var type in metricTypes)
            await Groups.AddToGroupAsync(Context.ConnectionId, $"metrics:{type}");
    }
}

// Background service: push metrics every second
public class MetricsPushService : BackgroundService
{
    private readonly IHubContext<DashboardHub, IDashboardClient> _hub;
    private readonly IMetricsService _metrics;
    
    public MetricsPushService(IHubContext<DashboardHub, IDashboardClient> hub, IMetricsService metrics)
    { _hub = hub; _metrics = metrics; }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var metrics = await _metrics.GetCurrentAsync();
            
            // Send to admins group
            await _hub.Clients.Group("admins").ReceiveMetrics(metrics);
            
            await Task.Delay(TimeSpan.FromSeconds(1), stoppingToken);
        }
    }
}
```

---

## ขั้นตอนที่ 585: SignalR JavaScript Client

```javascript
// wwwroot/js/chat.js
import * as signalR from "@microsoft/signalr";

const connection = new signalR.HubConnectionBuilder()
    .withUrl("/hubs/chat", {
        accessTokenFactory: () => localStorage.getItem("jwt_token")
    })
    .withAutomaticReconnect([0, 2000, 5000, 10000, 30000]) // retry intervals
    .configureLogging(signalR.LogLevel.Information)
    .build();

// Handle reconnection
connection.onreconnecting(() => {
    document.getElementById("status").textContent = "กำลังเชื่อมต่อใหม่...";
});

connection.onreconnected(() => {
    document.getElementById("status").textContent = "เชื่อมต่อแล้ว";
});

connection.onclose(() => {
    document.getElementById("status").textContent = "ขาดการเชื่อมต่อ";
});

// Receive messages
connection.on("ReceiveMessage", (message) => {
    const div = document.createElement("div");
    div.innerHTML = `<strong>${message.senderName}</strong>: ${escapeHtml(message.text)}`;
    document.getElementById("messages").appendChild(div);
});

connection.on("UserOnline", (userId) => {
    console.log(`${userId} is online`);
});

// Start connection
await connection.start();
console.log("Connected to SignalR hub");

// Send message
document.getElementById("send").addEventListener("click", async () => {
    const text = document.getElementById("messageInput").value;
    await connection.invoke("SendMessage", currentRoomId, text);
    document.getElementById("messageInput").value = "";
});

// Typing indicator
let typingTimer;
document.getElementById("messageInput").addEventListener("input", () => {
    connection.invoke("StartTyping", currentRoomId);
    clearTimeout(typingTimer);
    typingTimer = setTimeout(() => {
        connection.invoke("StopTyping", currentRoomId);
    }, 1000);
});

function escapeHtml(text) {
    return text
        .replace(/&/g, "&amp;")
        .replace(/</g, "&lt;")
        .replace(/>/g, "&gt;");
}
```

---

## ขั้นตอนที่ 586: SignalR in Blazor

```razor
@* Pages/LiveDashboard.razor *@
@page "/dashboard"
@inject NavigationManager Nav
@inject IAccessTokenProvider TokenProvider
@implements IAsyncDisposable

<h1>Dashboard สด</h1>

<div class="row">
    <div class="col-md-3">
        <div class="card bg-primary text-white">
            <div class="card-body">
                <h5>ยอดขายวันนี้</h5>
                <h2>฿@metrics?.TodaySales.ToString("N0")</h2>
            </div>
        </div>
    </div>
    <div class="col-md-3">
        <div class="card bg-success text-white">
            <div class="card-body">
                <h5>คำสั่งซื้อใหม่</h5>
                <h2>@metrics?.NewOrders</h2>
            </div>
        </div>
    </div>
</div>

@if (latestOrder != null)
{
    <div class="alert alert-info mt-4">
        คำสั่งซื้อใหม่: #@latestOrder.Id - ฿@latestOrder.Amount.ToString("N0")
    </div>
}

@code {
    private HubConnection? hubConnection;
    private DashboardMetrics? metrics;
    private OrderSummary? latestOrder;
    
    protected override async Task OnInitializedAsync()
    {
        // Get JWT token for SignalR auth
        var tokenResult = await TokenProvider.RequestAccessToken();
        tokenResult.TryGetToken(out var token);
        
        hubConnection = new HubConnectionBuilder()
            .WithUrl(Nav.ToAbsoluteUri("/hubs/dashboard"), options =>
            {
                options.AccessTokenProvider = () => Task.FromResult<string?>(token?.Value);
            })
            .WithAutomaticReconnect()
            .Build();
        
        hubConnection.On<DashboardMetrics>("ReceiveMetrics", m =>
        {
            metrics = m;
            InvokeAsync(StateHasChanged); // update UI on background thread
        });
        
        hubConnection.On<OrderSummary>("OrderCreated", order =>
        {
            latestOrder = order;
            InvokeAsync(StateHasChanged);
        });
        
        await hubConnection.StartAsync();
        await hubConnection.SendAsync("SubscribeToMetrics", new[] { "sales", "orders" });
    }
    
    public async ValueTask DisposeAsync()
    {
        if (hubConnection != null)
            await hubConnection.DisposeAsync();
    }
}
```

---

## ขั้นตอนที่ 587-590: Real-time Order Tracking

```csharp
// Hubs/OrderTrackingHub.cs
public interface IOrderTrackingClient
{
    Task OrderStatusUpdated(OrderStatusUpdate update);
    Task TrackingLocationUpdated(LocationUpdate location);
}

public class OrderTrackingHub : Hub<IOrderTrackingClient>
{
    public async Task TrackOrder(string orderId)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, $"order:{orderId}");
        // Send current status immediately
    }
    
    public async Task StopTracking(string orderId)
        => await Groups.RemoveFromGroupAsync(Context.ConnectionId, $"order:{orderId}");
}

// Update status from OrderService
public class OrderStatusUpdater
{
    private readonly IHubContext<OrderTrackingHub, IOrderTrackingClient> _hub;
    
    public OrderStatusUpdater(IHubContext<OrderTrackingHub, IOrderTrackingClient> hub)
        => _hub = hub;
    
    public async Task UpdateStatusAsync(Order order)
    {
        var update = new OrderStatusUpdate
        {
            OrderId = order.Id,
            Status = order.Status.ToString(),
            UpdatedAt = DateTime.UtcNow,
            Message = GetStatusMessage(order.Status)
        };
        
        // Notify everyone tracking this order
        await _hub.Clients.Group($"order:{order.Id}").OrderStatusUpdated(update);
    }
    
    private string GetStatusMessage(OrderStatus status) => status switch
    {
        OrderStatus.Confirmed => "✅ ยืนยันคำสั่งซื้อแล้ว",
        OrderStatus.Processing => "📦 กำลังจัดเตรียมสินค้า",
        OrderStatus.Shipped => "🚚 จัดส่งแล้ว",
        OrderStatus.Delivered => "🎉 จัดส่งสำเร็จ",
        OrderStatus.Cancelled => "❌ ยกเลิกคำสั่งซื้อ",
        _ => status.ToString()
    };
}
```

---

## 📝 สรุป Part 59

| Concept | ใช้ทำ |
|---------|-------|
| Hub | Server endpoint for clients |
| Hub<T> | Strongly-typed client methods |
| Clients.All | Broadcast to everyone |
| Clients.Group() | Send to group members |
| Clients.User() | Send to specific user |
| IHubContext | Send from outside Hub |
| withAutomaticReconnect | Auto-reconnect logic |
| Groups | Logical client grouping |

---

**ก่อนหน้า → [Part 58: ASP.NET Core API](part58-aspnet-core-api.md)**  
**ต่อไป → [Part 60: Docker & Deployment](part60-docker-deployment.md)**
