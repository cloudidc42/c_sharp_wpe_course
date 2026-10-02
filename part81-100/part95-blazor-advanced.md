# Part 95: Blazor Advanced Features
## ขั้นตอนที่ 941-950: Blazor ขั้นสูง

บทเรียนนี้ครอบคลุมฟีเจอร์ขั้นสูงของ Blazor ตั้งแต่ Render Modes ใน .NET 8, State Management, JavaScript Interop, Authentication, Real-time SignalR, Custom Validation, PWA, Data Virtualization, การสร้าง Component Library และการทดสอบด้วย bUnit

---

## ขั้นตอนที่ 941: Blazor Render Modes ใน .NET 8

### ความเข้าใจเกี่ยวกับ Render Modes

.NET 8 แนะนำ Unified Blazor Model ที่รวม Blazor Server และ Blazor WebAssembly เข้าด้วยกัน ในโปรเจกต์เดียว พร้อมเพิ่ม Render Modes ใหม่

**ประเภทของ Render Modes:**
- **Static SSR** - Render บน server ครั้งเดียว ไม่มี interactivity
- **Interactive Server** - Render และ interact บน server ผ่าน SignalR
- **Interactive WebAssembly** - Render และ interact บน client (WASM)
- **Interactive Auto** - เริ่มต้นด้วย Server แล้วเปลี่ยนเป็น WASM เมื่อโหลดเสร็จ

### การตั้งค่า Program.cs สำหรับ .NET 8 Unified Blazor

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// เพิ่ม Razor Components พร้อม Interactive Modes
builder.Services.AddRazorComponents()
    .AddInteractiveServerComponents()
    .AddInteractiveWebAssemblyComponents();

var app = builder.Build();

app.UseAntiforgery();

app.MapRazorComponents<App>()
    .AddInteractiveServerRenderMode()
    .AddInteractiveWebAssemblyRenderMode()
    .AddAdditionalAssemblies(typeof(Client.Pages._Imports).Assembly);

app.Run();
```

### App.razor - จุดเริ่มต้นของ Application

```razor
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Blazor Advanced App</title>
    <base href="/" />
    <link rel="stylesheet" href="app.css" />
    <HeadOutlet />
</head>
<body>
    <Routes />
    <script src="_framework/blazor.web.js"></script>
</body>
</html>
```

### ตัวอย่างการใช้ Render Modes ในแต่ละ Component

```razor
@* Static SSR - ไม่มี @rendermode *@
@page "/static"

<h1>หน้านี้เป็น Static SSR</h1>
<p>ไม่มี interactivity - render ครั้งเดียวบน server</p>
```

```razor
@* Interactive Server Component *@
@page "/counter-server"
@rendermode InteractiveServer

<h1>Counter (Server)</h1>
<p>จำนวนคลิก: @currentCount</p>
<button @onclick="IncrementCount">คลิกที่นี่</button>

@code {
    private int currentCount = 0;

    private void IncrementCount()
    {
        currentCount++;
    }
}
```

```razor
@* Interactive WebAssembly Component *@
@page "/counter-wasm"
@rendermode InteractiveWebAssembly

<h1>Counter (WebAssembly)</h1>
<p>จำนวนคลิก: @currentCount</p>
<button @onclick="IncrementCount">คลิกที่นี่</button>

@code {
    private int currentCount = 0;

    private void IncrementCount()
    {
        currentCount++;
    }
}
```

```razor
@* Interactive Auto - เริ่มต้น Server ก่อน แล้ว fallback เป็น WASM *@
@page "/counter-auto"
@rendermode InteractiveAuto

<h1>Counter (Auto)</h1>
<p>จำนวนคลิก: @currentCount</p>
<button @onclick="IncrementCount">คลิกที่นี่</button>

@code {
    private int currentCount = 0;

    private void IncrementCount()
    {
        currentCount++;
    }
}
```

### การกำหนด Render Mode บน Parent Component

```razor
@page "/dashboard"

<h1>Dashboard</h1>

@* กำหนด render mode เฉพาะส่วน *@
<WeatherWidget @rendermode="InteractiveServer" />
<ChartWidget @rendermode="InteractiveWebAssembly" />
<StaticContent />  @* ไม่มี rendermode = Static SSR *@
```

```csharp
// WeatherWidget.razor.cs
using Microsoft.AspNetCore.Components;

public partial class WeatherWidget : ComponentBase
{
    [Parameter]
    public string City { get; set; } = "Bangkok";

    private WeatherData? weather;

    protected override async Task OnInitializedAsync()
    {
        weather = await WeatherService.GetWeatherAsync(City);
    }
}
```

---

## ขั้นตอนที่ 942: State Management

### Cascading Parameters สำหรับ State Sharing

```razor
@* AppState.cs - Global State Model *@
public class AppState
{
    public string UserName { get; set; } = string.Empty;
    public int CartItemCount { get; set; }
    public bool IsDarkMode { get; set; }

    public event Action? OnChange;

    public void NotifyStateChanged() => OnChange?.Invoke();

    public void UpdateUserName(string name)
    {
        UserName = name;
        NotifyStateChanged();
    }

    public void AddToCart()
    {
        CartItemCount++;
        NotifyStateChanged();
    }
}
```

```csharp
// Program.cs - Register AppState as Singleton
builder.Services.AddSingleton<AppState>();
```

```razor
@* MainLayout.razor - Provide AppState via CascadingValue *@
@inherits LayoutComponentBase
@inject AppState AppState
@implements IDisposable

<CascadingValue Value="AppState">
    <div class="@(AppState.IsDarkMode ? "dark-theme" : "light-theme")">
        <NavMenu />
        <main>
            @Body
        </main>
    </div>
</CascadingValue>

@code {
    protected override void OnInitialized()
    {
        AppState.OnChange += StateHasChanged;
    }

    public void Dispose()
    {
        AppState.OnChange -= StateHasChanged;
    }
}
```

```razor
@* ProductPage.razor - ใช้ CascadingParameter *@
@page "/products"

<h1>สินค้า</h1>
<p>สวัสดี @AppState.UserName | ตะกร้า: @AppState.CartItemCount รายการ</p>

@foreach (var product in products)
{
    <div class="product-card">
        <h3>@product.Name</h3>
        <p>ราคา: @product.Price.ToString("C")</p>
        <button @onclick="() => AddToCart(product)">เพิ่มในตะกร้า</button>
    </div>
}

@code {
    [CascadingParameter]
    public AppState AppState { get; set; } = default!;

    private List<Product> products = new();

    protected override async Task OnInitializedAsync()
    {
        products = await ProductService.GetProductsAsync();
    }

    private void AddToCart(Product product)
    {
        AppState.AddToCart();
        // บันทึกสินค้าลง cart
    }
}
```

### Fluxor State Management (Redux Pattern)

```csharp
// ติดตั้ง NuGet: Fluxor.Blazor.Web

// Store/Counter/CounterState.cs
public record CounterState
{
    public int Count { get; init; }
    public bool IsLoading { get; init; }
}
```

```csharp
// Store/Counter/CounterFeature.cs
using Fluxor;

public class CounterFeature : Feature<CounterState>
{
    public override string GetName() => "Counter";

    protected override CounterState GetInitialState() =>
        new CounterState { Count = 0, IsLoading = false };
}
```

```csharp
// Store/Counter/CounterActions.cs
public class IncrementCounterAction { }
public class DecrementCounterAction { }
public class ResetCounterAction { }

public class LoadCounterAction { }
public class LoadCounterSuccessAction
{
    public int Count { get; }
    public LoadCounterSuccessAction(int count) => Count = count;
}
```

```csharp
// Store/Counter/CounterReducers.cs
using Fluxor;

public static class CounterReducers
{
    [ReducerMethod]
    public static CounterState ReduceIncrementCounterAction(
        CounterState state,
        IncrementCounterAction action) =>
        state with { Count = state.Count + 1 };

    [ReducerMethod]
    public static CounterState ReduceDecrementCounterAction(
        CounterState state,
        DecrementCounterAction action) =>
        state with { Count = state.Count - 1 };

    [ReducerMethod]
    public static CounterState ReduceResetCounterAction(
        CounterState state,
        ResetCounterAction action) =>
        state with { Count = 0 };

    [ReducerMethod]
    public static CounterState ReduceLoadCounterAction(
        CounterState state,
        LoadCounterAction action) =>
        state with { IsLoading = true };

    [ReducerMethod]
    public static CounterState ReduceLoadCounterSuccessAction(
        CounterState state,
        LoadCounterSuccessAction action) =>
        state with { Count = action.Count, IsLoading = false };
}
```

```csharp
// Store/Counter/CounterEffects.cs
using Fluxor;

public class CounterEffects
{
    private readonly ICounterService _counterService;

    public CounterEffects(ICounterService counterService)
    {
        _counterService = counterService;
    }

    [EffectMethod]
    public async Task HandleLoadCounterAction(
        LoadCounterAction action,
        IDispatcher dispatcher)
    {
        var count = await _counterService.GetCountAsync();
        dispatcher.Dispatch(new LoadCounterSuccessAction(count));
    }
}
```

```razor
@* Pages/FluxorCounter.razor *@
@page "/fluxor-counter"
@using Fluxor
@using Fluxor.Blazor.Web.Components
@inherits FluxorComponent

@inject IState<CounterState> CounterState
@inject IDispatcher Dispatcher

<h1>Fluxor Counter</h1>

@if (CounterState.Value.IsLoading)
{
    <p>กำลังโหลด...</p>
}
else
{
    <p>จำนวน: @CounterState.Value.Count</p>
    <button @onclick="Increment">เพิ่ม</button>
    <button @onclick="Decrement">ลด</button>
    <button @onclick="Reset">รีเซ็ต</button>
    <button @onclick="LoadFromServer">โหลดจาก Server</button>
}

@code {
    private void Increment() =>
        Dispatcher.Dispatch(new IncrementCounterAction());

    private void Decrement() =>
        Dispatcher.Dispatch(new DecrementCounterAction());

    private void Reset() =>
        Dispatcher.Dispatch(new ResetCounterAction());

    private void LoadFromServer() =>
        Dispatcher.Dispatch(new LoadCounterAction());
}
```

---

## ขั้นตอนที่ 943: JavaScript Interop

### การใช้ IJSRuntime เบื้องต้น

```razor
@page "/js-interop"
@inject IJSRuntime JS

<h1>JavaScript Interop</h1>

<button @onclick="ShowAlert">แสดง Alert</button>
<button @onclick="GetWindowWidth">รับขนาดหน้าต่าง</button>
<button @onclick="CopyToClipboard">คัดลอก</button>

<p>ความกว้างหน้าต่าง: @windowWidth px</p>

<input @ref="inputRef" type="text" value="ข้อความที่จะคัดลอก" />

@code {
    private int windowWidth;
    private ElementReference inputRef;

    private async Task ShowAlert()
    {
        await JS.InvokeVoidAsync("alert", "สวัสดีจาก Blazor!");
    }

    private async Task GetWindowWidth()
    {
        windowWidth = await JS.InvokeAsync<int>("eval", "window.innerWidth");
    }

    private async Task CopyToClipboard()
    {
        await JS.InvokeVoidAsync("navigator.clipboard.writeText",
            "ข้อความที่คัดลอก");
    }
}
```

### การสร้าง JavaScript Module

```javascript
// wwwroot/js/interop.js
export function showToast(message, type = 'info') {
    const toast = document.createElement('div');
    toast.className = `toast toast-${type}`;
    toast.textContent = message;
    document.body.appendChild(toast);
    
    setTimeout(() => {
        toast.classList.add('show');
    }, 10);
    
    setTimeout(() => {
        toast.classList.remove('show');
        setTimeout(() => document.body.removeChild(toast), 300);
    }, 3000);
}

export function initializeChart(elementId, data) {
    const ctx = document.getElementById(elementId).getContext('2d');
    return new Chart(ctx, {
        type: 'bar',
        data: {
            labels: data.labels,
            datasets: [{
                label: data.title,
                data: data.values,
                backgroundColor: 'rgba(54, 162, 235, 0.6)'
            }]
        }
    });
}

export function downloadFile(content, fileName, contentType) {
    const blob = new Blob([content], { type: contentType });
    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url;
    link.download = fileName;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
    URL.revokeObjectURL(url);
}
```

```razor
@page "/js-module"
@inject IJSRuntime JS
@implements IAsyncDisposable

<h1>JavaScript Module Interop</h1>

<button @onclick="ShowSuccessToast">แสดง Toast</button>
<button @onclick="DownloadReport">ดาวน์โหลด Report</button>

<canvas id="myChart" width="400" height="200"></canvas>

@code {
    private IJSObjectReference? module;

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            // Import JavaScript Module
            module = await JS.InvokeAsync<IJSObjectReference>(
                "import", "./js/interop.js");

            // Initialize chart หลัง render
            var chartData = new
            {
                labels = new[] { "ม.ค.", "ก.พ.", "มี.ค.", "เม.ย.", "พ.ค." },
                values = new[] { 65, 59, 80, 81, 56 },
                title = "ยอดขายรายเดือน"
            };
            await module.InvokeVoidAsync("initializeChart", "myChart", chartData);
        }
    }

    private async Task ShowSuccessToast()
    {
        if (module is not null)
        {
            await module.InvokeVoidAsync("showToast",
                "ดำเนินการสำเร็จ!", "success");
        }
    }

    private async Task DownloadReport()
    {
        if (module is not null)
        {
            var csvContent = "ชื่อ,อายุ,เมือง\nสมชาย,30,กรุงเทพ\nสมหญิง,25,เชียงใหม่";
            await module.InvokeVoidAsync("downloadFile",
                csvContent, "report.csv", "text/csv");
        }
    }

    public async ValueTask DisposeAsync()
    {
        if (module is not null)
        {
            await module.DisposeAsync();
        }
    }
}
```

### JSInvokable - เรียก C# จาก JavaScript

```csharp
// Services/BlazorBridgeService.cs
using Microsoft.JSInterop;

public class BlazorBridgeService
{
    private DotNetObjectReference<BlazorBridgeService>? _dotNetRef;

    public event Action<string>? OnMessageReceived;
    public event Action<int>? OnProgressUpdated;

    public DotNetObjectReference<BlazorBridgeService> GetReference()
    {
        _dotNetRef ??= DotNetObjectReference.Create(this);
        return _dotNetRef;
    }

    [JSInvokable]
    public void ReceiveMessage(string message)
    {
        OnMessageReceived?.Invoke(message);
    }

    [JSInvokable]
    public void UpdateProgress(int progress)
    {
        OnProgressUpdated?.Invoke(progress);
    }

    [JSInvokable]
    public async Task<string> ProcessDataAsync(string data)
    {
        // ประมวลผลข้อมูลบน C# side
        await Task.Delay(100); // simulate processing
        return $"ประมวลผลแล้ว: {data.ToUpper()}";
    }

    public void Dispose()
    {
        _dotNetRef?.Dispose();
    }
}
```

```javascript
// wwwroot/js/bridge.js
let dotNetRef = null;

export function initialize(dotNetObject) {
    dotNetRef = dotNetObject;
    
    // ส่ง message กลับ Blazor
    setInterval(async () => {
        if (dotNetRef) {
            await dotNetRef.invokeMethodAsync('ReceiveMessage', 
                `Heartbeat: ${new Date().toLocaleTimeString()}`);
        }
    }, 5000);
}

export async function processWithBlazor(data) {
    if (dotNetRef) {
        return await dotNetRef.invokeMethodAsync('ProcessDataAsync', data);
    }
    return null;
}
```

---

## ขั้นตอนที่ 944: Authentication และ Authorization ใน Blazor

### การตั้งค่า Authentication

```csharp
// Program.cs
builder.Services.AddAuthentication(options =>
{
    options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = CookieAuthenticationDefaults.AuthenticationScheme;
})
.AddCookie(options =>
{
    options.LoginPath = "/login";
    options.LogoutPath = "/logout";
    options.ExpireTimeSpan = TimeSpan.FromHours(8);
});

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"));
    options.AddPolicy("MinimumAge", policy =>
        policy.Requirements.Add(new MinimumAgeRequirement(18)));
    options.AddPolicy("PremiumUser", policy =>
        policy.RequireClaim("Subscription", "Premium", "Enterprise"));
});

builder.Services.AddCascadingAuthenticationState();
```

### Custom AuthenticationStateProvider

```csharp
// Services/CustomAuthStateProvider.cs
using Microsoft.AspNetCore.Components.Authorization;
using System.Security.Claims;

public class CustomAuthStateProvider : AuthenticationStateProvider
{
    private readonly IUserService _userService;
    private ClaimsPrincipal _currentUser = new(new ClaimsIdentity());

    public CustomAuthStateProvider(IUserService userService)
    {
        _userService = userService;
    }

    public override Task<AuthenticationState> GetAuthenticationStateAsync()
    {
        return Task.FromResult(new AuthenticationState(_currentUser));
    }

    public async Task LoginAsync(string username, string password)
    {
        var user = await _userService.ValidateUserAsync(username, password);

        if (user is not null)
        {
            var claims = new[]
            {
                new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
                new Claim(ClaimTypes.Name, user.Username),
                new Claim(ClaimTypes.Email, user.Email),
                new Claim(ClaimTypes.Role, user.Role),
                new Claim("FullName", user.FullName),
                new Claim("Department", user.Department ?? ""),
            };

            var identity = new ClaimsIdentity(claims, "Custom");
            _currentUser = new ClaimsPrincipal(identity);
        }
        else
        {
            _currentUser = new ClaimsPrincipal(new ClaimsIdentity());
        }

        NotifyAuthenticationStateChanged(
            Task.FromResult(new AuthenticationState(_currentUser)));
    }

    public void Logout()
    {
        _currentUser = new ClaimsPrincipal(new ClaimsIdentity());
        NotifyAuthenticationStateChanged(
            Task.FromResult(new AuthenticationState(_currentUser)));
    }
}
```

### การใช้ AuthorizeView Component

```razor
@page "/admin/dashboard"
@attribute [Authorize(Roles = "Admin,Manager")]

<h1>Admin Dashboard</h1>

<AuthorizeView>
    <Authorized>
        <p>ยินดีต้อนรับ, @context.User.Identity?.Name!</p>
    </Authorized>
    <NotAuthorized>
        <p>กรุณาเข้าสู่ระบบ</p>
        <a href="/login">เข้าสู่ระบบ</a>
    </NotAuthorized>
    <Authorizing>
        <p>กำลังตรวจสอบสิทธิ์...</p>
    </Authorizing>
</AuthorizeView>

<AuthorizeView Roles="Admin">
    <Authorized>
        <div class="admin-panel">
            <h2>แผงควบคุม Admin</h2>
            <button @onclick="DeleteAllData">ลบข้อมูลทั้งหมด</button>
        </div>
    </Authorized>
</AuthorizeView>

<AuthorizeView Policy="PremiumUser">
    <Authorized>
        <div class="premium-features">
            <h2>ฟีเจอร์ Premium</h2>
            <!-- ฟีเจอร์พิเศษ -->
        </div>
    </Authorized>
    <NotAuthorized>
        <div class="upgrade-prompt">
            <p>อัปเกรดเป็น Premium เพื่อใช้งานฟีเจอร์นี้</p>
        </div>
    </NotAuthorized>
</AuthorizeView>

@code {
    [CascadingParameter]
    private Task<AuthenticationState>? AuthStateTask { get; set; }

    private async Task DeleteAllData()
    {
        var authState = await AuthStateTask!;
        if (authState.User.IsInRole("Admin"))
        {
            // ดำเนินการลบ
        }
    }
}
```

### Login Page

```razor
@page "/login"
@inject CustomAuthStateProvider AuthProvider
@inject NavigationManager Nav

<div class="login-container">
    <h1>เข้าสู่ระบบ</h1>

    <EditForm Model="loginModel" OnValidSubmit="HandleLogin">
        <DataAnnotationsValidator />

        <div class="form-group">
            <label>ชื่อผู้ใช้</label>
            <InputText @bind-Value="loginModel.Username" class="form-control" />
            <ValidationMessage For="@(() => loginModel.Username)" />
        </div>

        <div class="form-group">
            <label>รหัสผ่าน</label>
            <InputText @bind-Value="loginModel.Password"
                       type="password" class="form-control" />
            <ValidationMessage For="@(() => loginModel.Password)" />
        </div>

        @if (!string.IsNullOrEmpty(errorMessage))
        {
            <div class="alert alert-danger">@errorMessage</div>
        }

        <button type="submit" class="btn btn-primary" disabled="@isLoading">
            @(isLoading ? "กำลังเข้าสู่ระบบ..." : "เข้าสู่ระบบ")
        </button>
    </EditForm>
</div>

@code {
    private LoginModel loginModel = new();
    private string errorMessage = string.Empty;
    private bool isLoading;

    private async Task HandleLogin()
    {
        isLoading = true;
        errorMessage = string.Empty;

        try
        {
            await AuthProvider.LoginAsync(loginModel.Username, loginModel.Password);
            Nav.NavigateTo("/dashboard");
        }
        catch (Exception ex)
        {
            errorMessage = "ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง";
        }
        finally
        {
            isLoading = false;
        }
    }

    public class LoginModel
    {
        [Required(ErrorMessage = "กรุณากรอกชื่อผู้ใช้")]
        public string Username { get; set; } = string.Empty;

        [Required(ErrorMessage = "กรุณากรอกรหัสผ่าน")]
        [MinLength(6, ErrorMessage = "รหัสผ่านต้องมีอย่างน้อย 6 ตัวอักษร")]
        public string Password { get; set; } = string.Empty;
    }
}
```

---

## ขั้นตอนที่ 945: Real-time Updates ด้วย SignalR ใน Blazor

### SignalR Hub

```csharp
// Hubs/ChatHub.cs
using Microsoft.AspNetCore.SignalR;

public class ChatHub : Hub
{
    private static readonly List<ChatMessage> MessageHistory = new();
    private static readonly Dictionary<string, string> ConnectedUsers = new();

    public override async Task OnConnectedAsync()
    {
        await Clients.Caller.SendAsync("Connected", Context.ConnectionId);
        await base.OnConnectedAsync();
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        if (ConnectedUsers.TryGetValue(Context.ConnectionId, out var userName))
        {
            ConnectedUsers.Remove(Context.ConnectionId);
            await Clients.All.SendAsync("UserLeft", userName);
        }
        await base.OnDisconnectedAsync(exception);
    }

    public async Task JoinChat(string userName)
    {
        ConnectedUsers[Context.ConnectionId] = userName;
        await Clients.Others.SendAsync("UserJoined", userName);

        // ส่งประวัติ message ให้ผู้ใช้ใหม่
        await Clients.Caller.SendAsync("MessageHistory",
            MessageHistory.TakeLast(50).ToList());

        await Clients.All.SendAsync("UpdateUserList",
            ConnectedUsers.Values.ToList());
    }

    public async Task SendMessage(string message)
    {
        if (!ConnectedUsers.TryGetValue(Context.ConnectionId, out var userName))
            return;

        var chatMessage = new ChatMessage
        {
            Id = Guid.NewGuid(),
            UserName = userName,
            Message = message,
            Timestamp = DateTime.UtcNow
        };

        MessageHistory.Add(chatMessage);

        // เก็บแค่ 1000 messages ล่าสุด
        if (MessageHistory.Count > 1000)
            MessageHistory.RemoveAt(0);

        await Clients.All.SendAsync("ReceiveMessage", chatMessage);
    }

    public async Task SendTypingIndicator(bool isTyping)
    {
        if (ConnectedUsers.TryGetValue(Context.ConnectionId, out var userName))
        {
            await Clients.Others.SendAsync("UserTyping", userName, isTyping);
        }
    }
}

public record ChatMessage
{
    public Guid Id { get; init; }
    public string UserName { get; init; } = string.Empty;
    public string Message { get; init; } = string.Empty;
    public DateTime Timestamp { get; init; }
}
```

### Blazor Chat Component

```razor
@page "/chat"
@rendermode InteractiveServer
@inject NavigationManager Nav
@implements IAsyncDisposable

<div class="chat-container">
    <h1>Chat Room</h1>

    @if (!isJoined)
    {
        <div class="join-form">
            <input @bind="userName" placeholder="ใส่ชื่อของคุณ" class="form-control" />
            <button @onclick="JoinChat" class="btn btn-primary">เข้าร่วม</button>
        </div>
    }
    else
    {
        <div class="chat-layout">
            <div class="user-list">
                <h3>ผู้ใช้ออนไลน์ (@userList.Count)</h3>
                @foreach (var user in userList)
                {
                    <div class="user-item @(user == userName ? "current-user" : "")">
                        @user
                    </div>
                }
            </div>

            <div class="chat-area">
                <div class="messages" id="messageContainer">
                    @foreach (var msg in messages)
                    {
                        <div class="message @(msg.UserName == userName ? "own-message" : "")">
                            <strong>@msg.UserName</strong>
                            <span class="time">@msg.Timestamp.ToLocalTime().ToString("HH:mm")</span>
                            <p>@msg.Message</p>
                        </div>
                    }
                </div>

                @if (typingUsers.Any())
                {
                    <div class="typing-indicator">
                        @string.Join(", ", typingUsers) กำลังพิมพ์...
                    </div>
                }

                <div class="message-input">
                    <input @bind="currentMessage"
                           @bind:event="oninput"
                           @onkeyup="HandleKeyUp"
                           placeholder="พิมพ์ข้อความ..."
                           class="form-control" />
                    <button @onclick="SendMessage" class="btn btn-primary">ส่ง</button>
                </div>
            </div>
        </div>
    }
</div>

@code {
    private HubConnection? hubConnection;
    private List<ChatMessage> messages = new();
    private List<string> userList = new();
    private HashSet<string> typingUsers = new();
    private string userName = string.Empty;
    private string currentMessage = string.Empty;
    private bool isJoined;
    private System.Timers.Timer? typingTimer;

    private async Task JoinChat()
    {
        if (string.IsNullOrWhiteSpace(userName)) return;

        hubConnection = new HubConnectionBuilder()
            .WithUrl(Nav.ToAbsoluteUri("/chathub"))
            .WithAutomaticReconnect()
            .Build();

        // ลงทะเบียน event handlers
        hubConnection.On<ChatMessage>("ReceiveMessage", (msg) =>
        {
            messages.Add(msg);
            InvokeAsync(StateHasChanged);
        });

        hubConnection.On<List<ChatMessage>>("MessageHistory", (history) =>
        {
            messages.AddRange(history);
            InvokeAsync(StateHasChanged);
        });

        hubConnection.On<List<string>>("UpdateUserList", (users) =>
        {
            userList = users;
            InvokeAsync(StateHasChanged);
        });

        hubConnection.On<string, bool>("UserTyping", (user, isTyping) =>
        {
            if (isTyping)
                typingUsers.Add(user);
            else
                typingUsers.Remove(user);
            InvokeAsync(StateHasChanged);
        });

        hubConnection.On<string>("UserJoined", (user) =>
        {
            messages.Add(new ChatMessage
            {
                UserName = "System",
                Message = $"{user} เข้าร่วมห้องแชท",
                Timestamp = DateTime.UtcNow
            });
            InvokeAsync(StateHasChanged);
        });

        await hubConnection.StartAsync();
        await hubConnection.SendAsync("JoinChat", userName);
        isJoined = true;
    }

    private async Task SendMessage()
    {
        if (string.IsNullOrWhiteSpace(currentMessage) || hubConnection is null) return;

        await hubConnection.SendAsync("SendMessage", currentMessage);
        currentMessage = string.Empty;
    }

    private async Task HandleKeyUp(KeyboardEventArgs e)
    {
        if (e.Key == "Enter")
        {
            await SendMessage();
            return;
        }

        // ส่ง typing indicator
        if (hubConnection is not null)
        {
            await hubConnection.SendAsync("SendTypingIndicator", true);

            typingTimer?.Dispose();
            typingTimer = new System.Timers.Timer(2000);
            typingTimer.Elapsed += async (_, _) =>
            {
                await hubConnection.SendAsync("SendTypingIndicator", false);
                typingTimer?.Dispose();
            };
            typingTimer.Start();
        }
    }

    public async ValueTask DisposeAsync()
    {
        typingTimer?.Dispose();
        if (hubConnection is not null)
        {
            await hubConnection.DisposeAsync();
        }
    }
}
```

---

## ขั้นตอนที่ 946: Custom Form Validation

### ValidationMessageStore และ Custom Validator

```csharp
// Models/OrderModel.cs
public class OrderModel
{
    [Required(ErrorMessage = "กรุณากรอกชื่อลูกค้า")]
    public string CustomerName { get; set; } = string.Empty;

    [Required(ErrorMessage = "กรุณากรอกอีเมล")]
    [EmailAddress(ErrorMessage = "รูปแบบอีเมลไม่ถูกต้อง")]
    public string Email { get; set; } = string.Empty;

    [Required(ErrorMessage = "กรุณาระบุวันส่งสินค้า")]
    public DateTime? DeliveryDate { get; set; }

    [Range(1, 1000, ErrorMessage = "จำนวนต้องอยู่ระหว่าง 1-1000")]
    public int Quantity { get; set; } = 1;

    public string PromoCode { get; set; } = string.Empty;

    public List<OrderItem> Items { get; set; } = new();
}

public class OrderItem
{
    [Required(ErrorMessage = "กรุณาเลือกสินค้า")]
    public string ProductId { get; set; } = string.Empty;

    [Range(1, 100, ErrorMessage = "จำนวนต้องอยู่ระหว่าง 1-100")]
    public int Quantity { get; set; } = 1;
}
```

```csharp
// Validators/OrderValidator.cs - Custom Validator Component
using Microsoft.AspNetCore.Components;
using Microsoft.AspNetCore.Components.Forms;

public class OrderValidator : ComponentBase
{
    private ValidationMessageStore? _messageStore;

    [CascadingParameter]
    private EditContext? CurrentEditContext { get; set; }

    [Inject]
    private IPromoCodeService PromoCodeService { get; set; } = default!;

    protected override void OnInitialized()
    {
        if (CurrentEditContext is null)
            throw new InvalidOperationException(
                "OrderValidator ต้องใช้ภายใน EditForm");

        _messageStore = new ValidationMessageStore(CurrentEditContext);

        // Subscribe to validation events
        CurrentEditContext.OnValidationRequested += ValidateModel;
        CurrentEditContext.OnFieldChanged += ValidateField;
    }

    private async void ValidateModel(object? sender, ValidationRequestedEventArgs e)
    {
        _messageStore?.Clear();

        var model = CurrentEditContext!.Model as OrderModel;
        if (model is null) return;

        // ตรวจสอบวันส่งสินค้า
        if (model.DeliveryDate.HasValue)
        {
            if (model.DeliveryDate.Value < DateTime.Today.AddDays(1))
            {
                _messageStore?.Add(
                    CurrentEditContext.Field(nameof(OrderModel.DeliveryDate)),
                    "วันส่งสินค้าต้องเป็นวันพรุ่งนี้เป็นต้นไป");
            }

            if (model.DeliveryDate.Value.DayOfWeek == DayOfWeek.Sunday)
            {
                _messageStore?.Add(
                    CurrentEditContext.Field(nameof(OrderModel.DeliveryDate)),
                    "ไม่สามารถส่งสินค้าในวันอาทิตย์ได้");
            }
        }

        // ตรวจสอบ PromoCode
        if (!string.IsNullOrEmpty(model.PromoCode))
        {
            var isValid = await PromoCodeService.ValidateAsync(model.PromoCode);
            if (!isValid)
            {
                _messageStore?.Add(
                    CurrentEditContext.Field(nameof(OrderModel.PromoCode)),
                    $"รหัสโปรโมชัน '{model.PromoCode}' ไม่ถูกต้องหรือหมดอายุแล้ว");
            }
        }

        // ตรวจสอบว่ามีสินค้าในออเดอร์
        if (!model.Items.Any())
        {
            _messageStore?.Add(
                CurrentEditContext.Field(nameof(OrderModel.Items)),
                "กรุณาเพิ่มสินค้าอย่างน้อย 1 รายการ");
        }

        CurrentEditContext.NotifyValidationStateChanged();
    }

    private void ValidateField(object? sender, FieldChangedEventArgs e)
    {
        _messageStore?.Clear(e.FieldIdentifier);
        CurrentEditContext?.NotifyValidationStateChanged();
    }
}
```

### FluentValidation Integration

```csharp
// ติดตั้ง NuGet: FluentValidation, Blazored.FluentValidation

// Validators/ProductValidator.cs
using FluentValidation;

public class ProductValidator : AbstractValidator<ProductModel>
{
    private readonly IProductService _productService;

    public ProductValidator(IProductService productService)
    {
        _productService = productService;

        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("กรุณากรอกชื่อสินค้า")
            .MinimumLength(3).WithMessage("ชื่อสินค้าต้องมีอย่างน้อย 3 ตัวอักษร")
            .MaximumLength(100).WithMessage("ชื่อสินค้าต้องไม่เกิน 100 ตัวอักษร")
            .MustAsync(BeUniqueName).WithMessage("ชื่อสินค้านี้มีอยู่แล้ว");

        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("ราคาต้องมากกว่า 0")
            .LessThanOrEqualTo(1000000).WithMessage("ราคาต้องไม่เกิน 1,000,000 บาท");

        RuleFor(x => x.Stock)
            .GreaterThanOrEqualTo(0).WithMessage("จำนวนสต็อกต้องไม่ติดลบ");

        RuleFor(x => x.CategoryId)
            .NotEmpty().WithMessage("กรุณาเลือกหมวดหมู่");

        RuleFor(x => x.Description)
            .MaximumLength(500).WithMessage("คำอธิบายต้องไม่เกิน 500 ตัวอักษร");

        When(x => x.HasDiscount, () =>
        {
            RuleFor(x => x.DiscountPercent)
                .InclusiveBetween(1, 90)
                .WithMessage("เปอร์เซ็นต์ส่วนลดต้องอยู่ระหว่าง 1-90%");
        });
    }

    private async Task<bool> BeUniqueName(
        ProductModel model, string name, CancellationToken token)
    {
        return await _productService.IsNameUniqueAsync(name, model.Id);
    }
}
```

```razor
@page "/products/create"
@using Blazored.FluentValidation

<h1>เพิ่มสินค้าใหม่</h1>

<EditForm Model="product" OnValidSubmit="HandleSubmit">
    <FluentValidationValidator />
    <ValidationSummary />

    <div class="form-group">
        <label>ชื่อสินค้า</label>
        <InputText @bind-Value="product.Name" class="form-control" />
        <ValidationMessage For="@(() => product.Name)" />
    </div>

    <div class="form-group">
        <label>ราคา (บาท)</label>
        <InputNumber @bind-Value="product.Price" class="form-control" />
        <ValidationMessage For="@(() => product.Price)" />
    </div>

    <div class="form-group">
        <label>จำนวนสต็อก</label>
        <InputNumber @bind-Value="product.Stock" class="form-control" />
        <ValidationMessage For="@(() => product.Stock)" />
    </div>

    <div class="form-check">
        <InputCheckbox @bind-Value="product.HasDiscount" class="form-check-input" />
        <label class="form-check-label">มีส่วนลด</label>
    </div>

    @if (product.HasDiscount)
    {
        <div class="form-group">
            <label>ส่วนลด (%)</label>
            <InputNumber @bind-Value="product.DiscountPercent" class="form-control" />
            <ValidationMessage For="@(() => product.DiscountPercent)" />
        </div>
    }

    <button type="submit" class="btn btn-primary">บันทึก</button>
</EditForm>

@code {
    private ProductModel product = new();

    private async Task HandleSubmit()
    {
        await ProductService.CreateAsync(product);
        Nav.NavigateTo("/products");
    }
}
```

---

## ขั้นตอนที่ 947: Blazor PWA (Progressive Web App)

### การตั้งค่า PWA

```json
// wwwroot/manifest.json
{
  "name": "แอปพลิเคชันของฉัน",
  "short_name": "MyApp",
  "description": "แอปพลิเคชัน Blazor PWA",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#0078d7",
  "prefer_related_applications": false,
  "icons": [
    {
      "src": "icons/icon-192.png",
      "type": "image/png",
      "sizes": "192x192"
    },
    {
      "src": "icons/icon-512.png",
      "type": "image/png",
      "sizes": "512x512"
    },
    {
      "src": "icons/icon-512.png",
      "type": "image/png",
      "sizes": "512x512",
      "purpose": "maskable"
    }
  ],
  "shortcuts": [
    {
      "name": "หน้าหลัก",
      "url": "/",
      "icons": [{ "src": "icons/home.png", "sizes": "96x96" }]
    }
  ]
}
```

```javascript
// wwwroot/service-worker.js
const CACHE_NAME = 'myapp-v1';
const OFFLINE_URL = '/offline.html';

const PRECACHE_ASSETS = [
    '/',
    '/index.html',
    '/css/app.css',
    '/manifest.json',
    OFFLINE_URL
];

// Install - precache assets
self.addEventListener('install', event => {
    event.waitUntil(
        caches.open(CACHE_NAME).then(cache => {
            return cache.addAll(PRECACHE_ASSETS);
        })
    );
    self.skipWaiting();
});

// Activate - clean old caches
self.addEventListener('activate', event => {
    event.waitUntil(
        caches.keys().then(cacheNames => {
            return Promise.all(
                cacheNames
                    .filter(name => name !== CACHE_NAME)
                    .map(name => caches.delete(name))
            );
        })
    );
    self.clients.claim();
});

// Fetch - network first, fallback to cache
self.addEventListener('fetch', event => {
    if (event.request.mode === 'navigate') {
        event.respondWith(
            fetch(event.request).catch(() => {
                return caches.match(OFFLINE_URL);
            })
        );
        return;
    }

    // API calls - network only
    if (event.request.url.includes('/api/')) {
        event.respondWith(
            fetch(event.request).catch(() => {
                return new Response(JSON.stringify({ error: 'Offline' }), {
                    headers: { 'Content-Type': 'application/json' }
                });
            })
        );
        return;
    }

    // Static assets - cache first
    event.respondWith(
        caches.match(event.request).then(cached => {
            if (cached) return cached;

            return fetch(event.request).then(response => {
                if (response.ok) {
                    const responseClone = response.clone();
                    caches.open(CACHE_NAME).then(cache => {
                        cache.put(event.request, responseClone);
                    });
                }
                return response;
            });
        })
    );
});

// Push Notifications
self.addEventListener('push', event => {
    const data = event.data?.json() ?? { title: 'การแจ้งเตือนใหม่', body: '' };

    event.waitUntil(
        self.registration.showNotification(data.title, {
            body: data.body,
            icon: '/icons/icon-192.png',
            badge: '/icons/badge-72.png',
            data: data.url,
            actions: [
                { action: 'open', title: 'เปิด' },
                { action: 'close', title: 'ปิด' }
            ]
        })
    );
});

self.addEventListener('notificationclick', event => {
    event.notification.close();
    if (event.action === 'open') {
        event.waitUntil(clients.openWindow(event.notification.data || '/'));
    }
});
```

```csharp
// Services/PwaService.cs
using Microsoft.JSInterop;

public class PwaService
{
    private readonly IJSRuntime _js;

    public PwaService(IJSRuntime js)
    {
        _js = js;
    }

    public async Task<bool> IsInstalledAsync()
    {
        try
        {
            return await _js.InvokeAsync<bool>(
                "eval",
                "window.matchMedia('(display-mode: standalone)').matches");
        }
        catch
        {
            return false;
        }
    }

    public async Task RegisterServiceWorkerAsync()
    {
        await _js.InvokeVoidAsync("eval", @"
            if ('serviceWorker' in navigator) {
                navigator.serviceWorker.register('/service-worker.js')
                    .then(reg => console.log('SW registered'))
                    .catch(err => console.error('SW registration failed:', err));
            }
        ");
    }

    public async Task<bool> RequestNotificationPermissionAsync()
    {
        var permission = await _js.InvokeAsync<string>(
            "eval",
            "Notification.requestPermission()");
        return permission == "granted";
    }

    public async Task ShowInstallPromptAsync()
    {
        await _js.InvokeVoidAsync("showInstallPrompt");
    }
}
```

```razor
@* Components/PwaInstallBanner.razor *@
@inject PwaService PwaService
@implements IAsyncDisposable

@if (showBanner)
{
    <div class="pwa-install-banner">
        <div class="banner-content">
            <img src="/icons/icon-192.png" alt="App Icon" width="48" />
            <div>
                <strong>ติดตั้งแอปของเรา</strong>
                <p>เพิ่มลงหน้าจอหลักเพื่อใช้งานได้ง่ายขึ้น</p>
            </div>
        </div>
        <div class="banner-actions">
            <button @onclick="InstallApp" class="btn btn-primary">ติดตั้ง</button>
            <button @onclick="DismissBanner" class="btn btn-link">ไม่ใช้ตอนนี้</button>
        </div>
    </div>
}

@code {
    private bool showBanner;

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            var isInstalled = await PwaService.IsInstalledAsync();
            if (!isInstalled)
            {
                showBanner = true;
                StateHasChanged();
            }
        }
    }

    private async Task InstallApp()
    {
        await PwaService.ShowInstallPromptAsync();
        showBanner = false;
    }

    private void DismissBanner() => showBanner = false;

    public ValueTask DisposeAsync() => ValueTask.CompletedTask;
}
```

---

## ขั้นตอนที่ 948: Data Virtualization ด้วย Virtualize Component

### Virtualize Component พื้นฐาน

```razor
@page "/virtual-list"
@rendermode InteractiveServer

<h1>Virtual List - @totalItems รายการ</h1>

<div style="height: 600px; overflow-y: auto; border: 1px solid #ddd;">
    <Virtualize Items="allProducts" Context="product" OverscanCount="5">
        <ItemContent>
            <div class="product-row" style="height: 60px; padding: 10px; border-bottom: 1px solid #eee;">
                <img src="@product.ImageUrl" width="40" height="40" style="float:left; margin-right:10px;" />
                <div>
                    <strong>@product.Name</strong>
                    <br/>
                    <span class="text-muted">@product.Price.ToString("C0")</span>
                    <span class="badge @(product.InStock ? "badge-success" : "badge-danger")">
                        @(product.InStock ? "มีสต็อก" : "หมดสต็อก")
                    </span>
                </div>
            </div>
        </ItemContent>
        <Placeholder>
            <div class="product-row skeleton" style="height: 60px; padding: 10px;">
                <div class="skeleton-box" style="width: 40px; height: 40px;"></div>
                <div class="skeleton-text"></div>
            </div>
        </Placeholder>
    </Virtualize>
</div>

@code {
    private List<Product> allProducts = new();
    private int totalItems;

    protected override async Task OnInitializedAsync()
    {
        allProducts = await ProductService.GetAllAsync();
        totalItems = allProducts.Count;
    }
}
```

### Virtualize กับ ItemsProvider (Server-side Paging)

```csharp
// Services/IProductService.cs
public interface IProductService
{
    Task<ProductQueryResult> GetPageAsync(
        int startIndex,
        int count,
        string? searchTerm = null,
        string? sortBy = null);
}

public record ProductQueryResult
{
    public List<Product> Items { get; init; } = new();
    public int TotalCount { get; init; }
}
```

```razor
@page "/virtual-server"
@rendermode InteractiveServer
@inject IProductService ProductService

<h1>Virtual List (Server Paging)</h1>

<div class="search-bar">
    <input @bind="searchTerm"
           @bind:event="oninput"
           @oninput="RefreshData"
           placeholder="ค้นหาสินค้า..."
           class="form-control" />
</div>

@if (totalItems > 0)
{
    <p class="text-muted">แสดง @totalItems รายการ</p>
}

<div style="height: 600px; overflow-y: auto;">
    <Virtualize ItemsProvider="LoadProducts"
                @ref="virtualize"
                Context="product"
                ItemSize="70"
                OverscanCount="3">
        <ItemContent>
            <div class="product-row" style="height: 70px; padding: 12px; border-bottom: 1px solid #eee; display: flex; align-items: center; gap: 12px;">
                <div class="product-rank">#@(product.Rank)</div>
                <img src="@product.ImageUrl" width="45" height="45" style="border-radius: 8px;" />
                <div class="product-info" style="flex: 1;">
                    <div class="product-name">@product.Name</div>
                    <div class="product-meta">
                        <span class="category">@product.Category</span>
                        <span class="price">@product.Price.ToString("C0")</span>
                    </div>
                </div>
                <div class="product-actions">
                    <button @onclick="() => ViewProduct(product.Id)"
                            class="btn btn-sm btn-outline-primary">ดู</button>
                </div>
            </div>
        </ItemContent>
        <Placeholder>
            <div style="height: 70px; padding: 12px; border-bottom: 1px solid #eee;">
                <div class="loading-skeleton"></div>
            </div>
        </Placeholder>
        <EmptyContent>
            <div class="empty-state">
                <p>ไม่พบสินค้าที่ตรงกับ "@searchTerm"</p>
            </div>
        </EmptyContent>
    </Virtualize>
</div>

@code {
    private Virtualize<Product>? virtualize;
    private string searchTerm = string.Empty;
    private int totalItems;

    private async ValueTask<ItemsProviderResult<Product>> LoadProducts(
        ItemsProviderRequest request)
    {
        var result = await ProductService.GetPageAsync(
            request.StartIndex,
            request.Count,
            searchTerm,
            "Name");

        totalItems = result.TotalCount;
        StateHasChanged(); // อัปเดต totalItems display

        return new ItemsProviderResult<Product>(result.Items, result.TotalCount);
    }

    private async Task RefreshData()
    {
        if (virtualize is not null)
        {
            await virtualize.RefreshDataAsync();
        }
    }

    private void ViewProduct(int productId)
    {
        Nav.NavigateTo($"/products/{productId}");
    }
}
```

### Table Virtualization

```razor
@page "/virtual-table"

<h1>ตาราง Virtual (100,000 แถว)</h1>

<div style="height: 500px; overflow-y: auto; border: 1px solid #ddd;">
    <table class="table table-striped" style="width: 100%;">
        <thead style="position: sticky; top: 0; background: white; z-index: 1;">
            <tr>
                <th style="width: 60px;">ลำดับ</th>
                <th>ชื่อพนักงาน</th>
                <th>แผนก</th>
                <th>ตำแหน่ง</th>
                <th style="text-align: right;">เงินเดือน</th>
                <th>สถานะ</th>
            </tr>
        </thead>
        <tbody>
            <Virtualize Items="employees" Context="emp">
                <tr>
                    <td>@emp.EmployeeNumber</td>
                    <td>@emp.FullName</td>
                    <td>@emp.Department</td>
                    <td>@emp.Position</td>
                    <td style="text-align: right;">@emp.Salary.ToString("N0")</td>
                    <td>
                        <span class="badge @(emp.IsActive ? "bg-success" : "bg-secondary")">
                            @(emp.IsActive ? "ทำงาน" : "ลาออก")
                        </span>
                    </td>
                </tr>
            </Virtualize>
        </tbody>
    </table>
</div>

@code {
    private List<Employee> employees = new();

    protected override void OnInitialized()
    {
        // สร้างข้อมูลจำลอง 100,000 รายการ
        employees = Enumerable.Range(1, 100_000)
            .Select(i => new Employee
            {
                EmployeeNumber = i,
                FullName = $"พนักงาน {i:D6}",
                Department = i % 5 switch
                {
                    0 => "IT",
                    1 => "การตลาด",
                    2 => "บัญชี",
                    3 => "ฝ่ายขาย",
                    _ => "บุคคล"
                },
                Position = i % 3 == 0 ? "ผู้จัดการ" : "พนักงาน",
                Salary = Random.Shared.Next(20000, 150000),
                IsActive = Random.Shared.Next(10) > 1
            })
            .ToList();
    }
}
```

---

## ขั้นตอนที่ 949: การสร้าง Blazor Component Library

### โครงสร้าง Component Library Project

```xml
<!-- MyCompany.BlazorComponents.csproj -->
<Project Sdk="Microsoft.NET.Sdk.Razor">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <RootNamespace>MyCompany.BlazorComponents</RootNamespace>
    
    <!-- NuGet Package Settings -->
    <PackageId>MyCompany.BlazorComponents</PackageId>
    <Version>1.0.0</Version>
    <Authors>MyCompany</Authors>
    <Description>Blazor component library สำหรับ MyCompany</Description>
    <PackageTags>blazor;components;ui</PackageTags>
    <RepositoryUrl>https://github.com/mycompany/blazor-components</RepositoryUrl>
    <GeneratePackageOnBuild>true</GeneratePackageOnBuild>
    
    <!-- Static Assets Bundling -->
    <StaticWebAssetBasePath>mycompany</StaticWebAssetBasePath>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.Components.Web" Version="8.0.*" />
  </ItemGroup>
</Project>
```

### Button Component

```razor
@* Components/McButton.razor *@
@namespace MyCompany.BlazorComponents

<button type="@Type"
        class="mc-btn mc-btn-@Variant mc-btn-@Size @(IsFullWidth ? "mc-btn-full" : "") @CssClass"
        disabled="@(IsDisabled || IsLoading)"
        @onclick="HandleClick"
        @attributes="AdditionalAttributes">
    @if (IsLoading)
    {
        <span class="mc-spinner"></span>
    }
    @if (Icon is not null && IconPosition == "left")
    {
        <span class="mc-btn-icon">@Icon</span>
    }
    <span class="mc-btn-text">@ChildContent</span>
    @if (Icon is not null && IconPosition == "right")
    {
        <span class="mc-btn-icon">@Icon</span>
    }
</button>

@code {
    [Parameter] public RenderFragment? ChildContent { get; set; }
    [Parameter] public RenderFragment? Icon { get; set; }
    [Parameter] public string Type { get; set; } = "button";
    [Parameter] public string Variant { get; set; } = "primary"; // primary, secondary, danger, ghost
    [Parameter] public string Size { get; set; } = "md"; // sm, md, lg
    [Parameter] public string IconPosition { get; set; } = "left"; // left, right
    [Parameter] public bool IsDisabled { get; set; }
    [Parameter] public bool IsLoading { get; set; }
    [Parameter] public bool IsFullWidth { get; set; }
    [Parameter] public string CssClass { get; set; } = string.Empty;
    [Parameter] public EventCallback<MouseEventArgs> OnClick { get; set; }
    [Parameter(CaptureUnmatchedValues = true)]
    public Dictionary<string, object>? AdditionalAttributes { get; set; }

    private async Task HandleClick(MouseEventArgs e)
    {
        if (!IsDisabled && !IsLoading)
        {
            await OnClick.InvokeAsync(e);
        }
    }
}
```

### Modal Component

```razor
@* Components/McModal.razor *@
@namespace MyCompany.BlazorComponents
@implements IDisposable

@if (IsVisible)
{
    <div class="mc-modal-overlay @(IsVisible ? "mc-modal-open" : "")"
         @onclick="HandleOverlayClick">
        <div class="mc-modal mc-modal-@Size"
             @onclick:stopPropagation="true"
             role="dialog"
             aria-modal="true"
             aria-labelledby="modal-title">

            @if (ShowHeader)
            {
                <div class="mc-modal-header">
                    <h5 id="modal-title" class="mc-modal-title">@Title</h5>
                    @if (ShowCloseButton)
                    {
                        <button class="mc-modal-close" @onclick="Close"
                                aria-label="ปิด">&times;</button>
                    }
                </div>
            }

            <div class="mc-modal-body">
                @ChildContent
            </div>

            @if (Footer is not null)
            {
                <div class="mc-modal-footer">
                    @Footer
                </div>
            }
        </div>
    </div>
}

@code {
    [Parameter] public bool IsVisible { get; set; }
    [Parameter] public EventCallback<bool> IsVisibleChanged { get; set; }
    [Parameter] public string Title { get; set; } = string.Empty;
    [Parameter] public string Size { get; set; } = "md"; // sm, md, lg, xl
    [Parameter] public bool ShowHeader { get; set; } = true;
    [Parameter] public bool ShowCloseButton { get; set; } = true;
    [Parameter] public bool CloseOnOverlayClick { get; set; } = true;
    [Parameter] public RenderFragment? ChildContent { get; set; }
    [Parameter] public RenderFragment? Footer { get; set; }
    [Parameter] public EventCallback OnClosed { get; set; }

    private async Task Close()
    {
        await IsVisibleChanged.InvokeAsync(false);
        await OnClosed.InvokeAsync();
    }

    private async Task HandleOverlayClick()
    {
        if (CloseOnOverlayClick)
        {
            await Close();
        }
    }

    public void Dispose() { }
}
```

### DataTable Component

```razor
@* Components/McDataTable.razor *@
@namespace MyCompany.BlazorComponents
@typeparam TItem

<div class="mc-datatable-wrapper">
    @if (ShowSearch)
    {
        <div class="mc-datatable-toolbar">
            <input @bind="searchTerm"
                   @bind:event="oninput"
                   placeholder="ค้นหา..."
                   class="mc-search-input" />
        </div>
    }

    <table class="mc-table @(IsStriped ? "mc-table-striped" : "") @(IsHoverable ? "mc-table-hover" : "")">
        <thead>
            <tr>
                @foreach (var col in Columns)
                {
                    <th @onclick="() => SortBy(col)"
                        style="@(col.Width is not null ? $"width:{col.Width}" : "") cursor:pointer">
                        @col.Title
                        @if (sortColumn == col.Field)
                        {
                            <span>@(sortAscending ? "▲" : "▼")</span>
                        }
                    </th>
                }
            </tr>
        </thead>
        <tbody>
            @foreach (var item in FilteredAndSortedItems)
            {
                <tr @onclick="() => RowClicked(item)"
                    class="@(SelectedItem?.Equals(item) == true ? "mc-row-selected" : "")">
                    @RowTemplate(item)
                </tr>
            }
        </tbody>
    </table>

    @if (ShowPagination)
    {
        <div class="mc-pagination">
            <span>รายการ @((currentPage - 1) * PageSize + 1)-@Math.Min(currentPage * PageSize, TotalCount) จากทั้งหมด @TotalCount</span>
            <button @onclick="PreviousPage" disabled="@(currentPage == 1)">‹ ก่อนหน้า</button>
            @for (int i = 1; i <= TotalPages; i++)
            {
                var page = i;
                <button @onclick="() => GoToPage(page)"
                        class="@(currentPage == i ? "mc-page-active" : "")">@i</button>
            }
            <button @onclick="NextPage" disabled="@(currentPage == TotalPages)">ถัดไป ›</button>
        </div>
    }
</div>

@code {
    [Parameter, EditorRequired] public List<TItem> Items { get; set; } = new();
    [Parameter, EditorRequired] public List<ColumnDefinition> Columns { get; set; } = new();
    [Parameter, EditorRequired] public RenderFragment<TItem> RowTemplate { get; set; } = default!;
    [Parameter] public bool ShowSearch { get; set; } = true;
    [Parameter] public bool ShowPagination { get; set; } = true;
    [Parameter] public int PageSize { get; set; } = 20;
    [Parameter] public bool IsStriped { get; set; } = true;
    [Parameter] public bool IsHoverable { get; set; } = true;
    [Parameter] public TItem? SelectedItem { get; set; }
    [Parameter] public EventCallback<TItem> OnRowClick { get; set; }

    private string searchTerm = string.Empty;
    private string? sortColumn;
    private bool sortAscending = true;
    private int currentPage = 1;

    private int TotalCount => FilteredItems.Count();
    private int TotalPages => (int)Math.Ceiling((double)TotalCount / PageSize);

    private IEnumerable<TItem> FilteredItems =>
        string.IsNullOrEmpty(searchTerm) ? Items : Items.Where(item =>
            Columns.Any(col =>
                col.SearchSelector?.Invoke(item)?.Contains(searchTerm,
                    StringComparison.OrdinalIgnoreCase) == true));

    private IEnumerable<TItem> FilteredAndSortedItems
    {
        get
        {
            var items = FilteredItems;
            if (sortColumn is not null)
            {
                var col = Columns.FirstOrDefault(c => c.Field == sortColumn);
                if (col?.SortSelector is not null)
                {
                    items = sortAscending
                        ? items.OrderBy(col.SortSelector)
                        : items.OrderByDescending(col.SortSelector);
                }
            }
            return items.Skip((currentPage - 1) * PageSize).Take(PageSize);
        }
    }

    private void SortBy(ColumnDefinition col)
    {
        if (sortColumn == col.Field)
            sortAscending = !sortAscending;
        else
        {
            sortColumn = col.Field;
            sortAscending = true;
        }
    }

    private async Task RowClicked(TItem item) =>
        await OnRowClick.InvokeAsync(item);

    private void PreviousPage() { if (currentPage > 1) currentPage--; }
    private void NextPage() { if (currentPage < TotalPages) currentPage++; }
    private void GoToPage(int page) => currentPage = page;

    public class ColumnDefinition
    {
        public string Title { get; set; } = string.Empty;
        public string Field { get; set; } = string.Empty;
        public string? Width { get; set; }
        public Func<TItem, string?>? SearchSelector { get; set; }
        public Func<TItem, object?>? SortSelector { get; set; }
    }
}
```

### Service Extension สำหรับ Library

```csharp
// Extensions/ServiceCollectionExtensions.cs
using Microsoft.Extensions.DependencyInjection;

namespace MyCompany.BlazorComponents;

public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddMyCompanyComponents(
        this IServiceCollection services,
        Action<ComponentOptions>? configure = null)
    {
        var options = new ComponentOptions();
        configure?.Invoke(options);

        services.AddSingleton(options);
        services.AddScoped<ToastService>();
        services.AddScoped<ModalService>();
        services.AddScoped<ThemeService>();

        return services;
    }
}

public class ComponentOptions
{
    public string Theme { get; set; } = "light";
    public string PrimaryColor { get; set; } = "#0078d7";
    public string FontFamily { get; set; } = "Sarabun, sans-serif";
    public bool EnableAnimations { get; set; } = true;
}
```

---

## ขั้นตอนที่ 950: Blazor Testing ด้วย bUnit

### การตั้งค่า bUnit

```xml
<!-- MyApp.Tests.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <IsPackable>false</IsPackable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="bunit" Version="1.*" />
    <PackageReference Include="xunit" Version="2.*" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.*" />
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.*" />
    <PackageReference Include="Moq" Version="4.*" />
    <PackageReference Include="FluentAssertions" Version="6.*" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\MyApp\MyApp.csproj" />
  </ItemGroup>
</Project>
```

### Unit Tests สำหรับ Component

```csharp
// Tests/CounterTests.cs
using Bunit;
using Xunit;
using FluentAssertions;

public class CounterTests : TestContext
{
    [Fact]
    public void Counter_RendersInitialCount_AsZero()
    {
        // Arrange & Act
        var cut = RenderComponent<Counter>();

        // Assert
        cut.Find("p").TextContent.Should().Contain("0");
    }

    [Fact]
    public void Counter_WhenButtonClicked_IncrementsCount()
    {
        // Arrange
        var cut = RenderComponent<Counter>();

        // Act
        cut.Find("button").Click();

        // Assert
        cut.Find("p").TextContent.Should().Contain("1");
    }

    [Fact]
    public void Counter_WhenButtonClickedThreeTimes_CountIsThree()
    {
        // Arrange
        var cut = RenderComponent<Counter>();

        // Act
        cut.Find("button").Click();
        cut.Find("button").Click();
        cut.Find("button").Click();

        // Assert
        cut.Find("p").TextContent.Should().Contain("3");
    }

    [Fact]
    public void Counter_WithInitialCount_StartsAtCorrectValue()
    {
        // Arrange & Act
        var cut = RenderComponent<Counter>(parameters => parameters
            .Add(p => p.InitialCount, 10));

        // Assert
        cut.Find("p").TextContent.Should().Contain("10");
    }
}
```

### Testing Components ที่ใช้ Services

```csharp
// Tests/ProductListTests.cs
using Bunit;
using Xunit;
using Moq;
using FluentAssertions;
using Microsoft.Extensions.DependencyInjection;

public class ProductListTests : TestContext
{
    private readonly Mock<IProductService> _productServiceMock;

    public ProductListTests()
    {
        _productServiceMock = new Mock<IProductService>();
        Services.AddSingleton(_productServiceMock.Object);
    }

    [Fact]
    public async Task ProductList_ShowsLoadingState_Initially()
    {
        // Arrange
        var tcs = new TaskCompletionSource<List<Product>>();
        _productServiceMock
            .Setup(s => s.GetAllAsync())
            .Returns(tcs.Task);

        // Act
        var cut = RenderComponent<ProductList>();

        // Assert - ก่อน task complete
        cut.Find(".loading-spinner").Should().NotBeNull();

        // Complete the task
        tcs.SetResult(new List<Product>());
        await cut.WaitForStateAsync(() => !cut.FindAll(".loading-spinner").Any());
    }

    [Fact]
    public async Task ProductList_DisplaysProducts_AfterLoad()
    {
        // Arrange
        var products = new List<Product>
        {
            new() { Id = 1, Name = "สินค้า A", Price = 100 },
            new() { Id = 2, Name = "สินค้า B", Price = 200 },
            new() { Id = 3, Name = "สินค้า C", Price = 300 }
        };

        _productServiceMock
            .Setup(s => s.GetAllAsync())
            .ReturnsAsync(products);

        // Act
        var cut = RenderComponent<ProductList>();

        await cut.WaitForStateAsync(() => cut.FindAll(".product-card").Count == 3);

        // Assert
        var cards = cut.FindAll(".product-card");
        cards.Should().HaveCount(3);
        cards[0].TextContent.Should().Contain("สินค้า A");
        cards[1].TextContent.Should().Contain("สินค้า B");
    }

    [Fact]
    public async Task ProductList_ShowsEmptyMessage_WhenNoProducts()
    {
        // Arrange
        _productServiceMock
            .Setup(s => s.GetAllAsync())
            .ReturnsAsync(new List<Product>());

        // Act
        var cut = RenderComponent<ProductList>();

        await cut.WaitForStateAsync(() =>
            cut.FindAll(".empty-message").Any());

        // Assert
        cut.Find(".empty-message").TextContent.Should().Contain("ไม่พบสินค้า");
    }
}
```

### Testing กับ Authentication

```csharp
// Tests/AuthorizedComponentTests.cs
using Bunit;
using Bunit.TestDoubles;
using Xunit;
using FluentAssertions;

public class AuthorizedComponentTests : TestContext
{
    [Fact]
    public void AdminPanel_ShowsContent_WhenUserIsAdmin()
    {
        // Arrange
        var authContext = this.AddTestAuthorization();
        authContext.SetAuthorized("testuser");
        authContext.SetRoles("Admin");
        authContext.SetClaims(
            new System.Security.Claims.Claim("Department", "IT"));

        // Act
        var cut = RenderComponent<AdminPanel>();

        // Assert
        cut.Find(".admin-content").Should().NotBeNull();
        cut.FindAll(".unauthorized-message").Should().BeEmpty();
    }

    [Fact]
    public void AdminPanel_ShowsUnauthorized_WhenUserIsNotAdmin()
    {
        // Arrange
        var authContext = this.AddTestAuthorization();
        authContext.SetAuthorized("testuser");
        authContext.SetRoles("User"); // ไม่ใช่ Admin

        // Act
        var cut = RenderComponent<AdminPanel>();

        // Assert
        cut.Find(".unauthorized-message").Should().NotBeNull();
        cut.FindAll(".admin-content").Should().BeEmpty();
    }

    [Fact]
    public void AdminPanel_ShowsLogin_WhenUserIsNotAuthenticated()
    {
        // Arrange
        var authContext = this.AddTestAuthorization();
        authContext.SetNotAuthorized();

        // Act
        var cut = RenderComponent<AdminPanel>();

        // Assert
        cut.Find(".login-prompt").Should().NotBeNull();
    }
}
```

### Testing Event Callbacks และ Parameters

```csharp
// Tests/ModalTests.cs
using Bunit;
using Xunit;
using FluentAssertions;

public class ModalTests : TestContext
{
    [Fact]
    public void Modal_IsVisible_WhenIsVisibleIsTrue()
    {
        // Arrange & Act
        var cut = RenderComponent<McModal>(parameters => parameters
            .Add(p => p.IsVisible, true)
            .Add(p => p.Title, "ทดสอบ Modal"));

        // Assert
        cut.Find(".mc-modal-overlay").Should().NotBeNull();
        cut.Find(".mc-modal-title").TextContent.Should().Be("ทดสอบ Modal");
    }

    [Fact]
    public void Modal_IsNotVisible_WhenIsVisibleIsFalse()
    {
        // Arrange & Act
        var cut = RenderComponent<McModal>(parameters => parameters
            .Add(p => p.IsVisible, false));

        // Assert
        cut.FindAll(".mc-modal-overlay").Should().BeEmpty();
    }

    [Fact]
    public void Modal_FiresOnClosed_WhenCloseButtonClicked()
    {
        // Arrange
        var onClosedFired = false;
        var cut = RenderComponent<McModal>(parameters => parameters
            .Add(p => p.IsVisible, true)
            .Add(p => p.OnClosed, EventCallback.Factory.Create(this,
                () => onClosedFired = true)));

        // Act
        cut.Find(".mc-modal-close").Click();

        // Assert
        onClosedFired.Should().BeTrue();
    }

    [Fact]
    public void Modal_UpdatesIsVisible_WhenClosed()
    {
        // Arrange
        var isVisible = true;
        var cut = RenderComponent<McModal>(parameters => parameters
            .Add(p => p.IsVisible, isVisible)
            .Add(p => p.IsVisibleChanged, EventCallback.Factory.Create<bool>(
                this, val => isVisible = val)));

        // Act
        cut.Find(".mc-modal-close").Click();

        // Assert
        isVisible.Should().BeFalse();
    }
}
```

### Snapshot Testing

```csharp
// Tests/SnapshotTests.cs
using Bunit;
using Xunit;

public class SnapshotTests : TestContext
{
    [Fact]
    public void Button_MatchesSnapshot_PrimaryVariant()
    {
        // Arrange & Act
        var cut = RenderComponent<McButton>(parameters => parameters
            .Add(p => p.Variant, "primary")
            .Add(p => p.ChildContent, builder =>
                builder.AddContent(0, "คลิกที่นี่")));

        // Assert - บันทึกหรือเปรียบเทียบ snapshot
        cut.MarkupMatches(
            @"<button type=""button"" class=""mc-btn mc-btn-primary mc-btn-md "">
                <span class=""mc-btn-text"">คลิกที่นี่</span>
              </button>");
    }

    [Fact]
    public void ProductCard_RendersCorrectly_WithAllProps()
    {
        // Arrange
        var product = new Product
        {
            Id = 1,
            Name = "สินค้าทดสอบ",
            Price = 299.99m,
            InStock = true,
            ImageUrl = "/images/test.jpg"
        };

        // Act
        var cut = RenderComponent<ProductCard>(parameters => parameters
            .Add(p => p.Product, product));

        // Assert
        cut.Find(".product-name").TextContent.Should().Be("สินค้าทดสอบ");
        cut.Find(".product-price").TextContent.Should().Contain("299.99");
        cut.Find(".stock-badge").TextContent.Should().Contain("มีสต็อก");
    }
}
```

### Integration Test กับ TestServer

```csharp
// Tests/IntegrationTests.cs
using Microsoft.AspNetCore.Mvc.Testing;
using Xunit;
using System.Net.Http.Json;
using FluentAssertions;

public class BlazorIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    private readonly HttpClient _client;

    public BlazorIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // แทนที่ services ด้วย test doubles
                services.AddScoped<IProductService, TestProductService>();
                services.AddScoped<IEmailService, MockEmailService>();
            });
        });
        _client = _factory.CreateClient();
    }

    [Fact]
    public async Task HomePage_Returns200()
    {
        var response = await _client.GetAsync("/");
        response.StatusCode.Should().Be(System.Net.HttpStatusCode.OK);
    }

    [Fact]
    public async Task ProductsApi_ReturnsProducts()
    {
        var products = await _client.GetFromJsonAsync<List<Product>>("/api/products");
        products.Should().NotBeNull();
        products.Should().HaveCountGreaterThan(0);
    }

    [Fact]
    public async Task AdminPage_RedirectsToLogin_WhenNotAuthenticated()
    {
        var response = await _client.GetAsync("/admin");
        response.RequestMessage?.RequestUri?.PathAndQuery
            .Should().StartWith("/login");
    }
}
```

---

## สรุปบทเรียน Part 95

| ขั้นตอน | หัวข้อ | สิ่งที่เรียนรู้ |
|---------|--------|----------------|
| 941 | Render Modes | Static SSR, Interactive Server/WASM/Auto |
| 942 | State Management | CascadingParameter, Fluxor (Redux pattern) |
| 943 | JavaScript Interop | IJSRuntime, JS Modules, JSInvokable |
| 944 | Authentication | AuthorizeView, CascadingAuthState, Custom Provider |
| 945 | SignalR | Real-time Hub, Chat component, Reconnection |
| 946 | Form Validation | ValidationMessageStore, FluentValidation |
| 947 | PWA | Manifest, Service Worker, Push Notifications |
| 948 | Data Virtualization | Virtualize component, ItemsProvider |
| 949 | Component Library | NuGet package, Reusable components |
| 950 | bUnit Testing | Unit tests, Auth tests, Snapshot tests |

### เทคนิคสำคัญที่ควรจำ

1. **เลือก Render Mode ที่เหมาะสม** - Static SSR สำหรับหน้าที่ไม่ต้องการ interactivity, Interactive Server สำหรับ real-time, Interactive WASM สำหรับ offline-capable
2. **State Management** - ใช้ CascadingParameter สำหรับ app-wide state, Fluxor สำหรับ complex state
3. **JS Interop** - ใช้ JavaScript Modules แทนการเขียน global functions
4. **Authentication** - ใช้ `[Authorize]` attribute ร่วมกับ `<AuthorizeView>` สำหรับ fine-grained control
5. **Performance** - ใช้ `<Virtualize>` เมื่อมีข้อมูลมากกว่า 1,000 รายการ
6. **Testing** - เขียน tests สำหรับ component behavior, ไม่ใช่แค่ markup

---

## แหล่งข้อมูลเพิ่มเติม

- [Blazor Documentation](https://docs.microsoft.com/aspnet/core/blazor)
- [bUnit Documentation](https://bunit.dev)
- [Fluxor Documentation](https://github.com/mrpmorris/Fluxor)
- [FluentValidation for Blazor](https://github.com/Blazored/FluentValidation)
- [Blazor PWA Guide](https://docs.microsoft.com/aspnet/core/blazor/progressive-web-app)

---

## การนำทาง

[← Part 94: Blazor Fundamentals](./part94-blazor-fundamentals.md) | [Part 96: Microservices Architecture →](./part96-microservices.md)
