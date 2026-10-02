# Part 57: Blazor Web Development
## ขั้นตอนที่ 561-570: สร้าง Web Apps ด้วย C# และ Blazor

---

## 🎯 เป้าหมายของ Part นี้
- Blazor Server vs Blazor WebAssembly
- Components และ Razor syntax
- Data Binding และ EventCallback
- Lifecycle methods
- Dependency Injection ใน Blazor
- Forms และ Validation
- HTTP Client ใน Blazor WASM

---

## ขั้นตอนที่ 561: Blazor Overview

```
Blazor Hosting Models:
┌─────────────────────────────────────────────────────────────┐
│ Blazor Server                   │ Blazor WebAssembly (WASM) │
├─────────────────────────────────┼───────────────────────────┤
│ Runs on server                  │ Runs in browser           │
│ SignalR for UI updates          │ .NET runtime in WASM      │
│ Smaller initial download        │ Larger download (~2MB)    │
│ Server has full .NET access     │ Offline capable           │
│ Latency on every interaction    │ Fast interactions         │
│ Use: Enterprise internal apps   │ Use: Public web apps      │
└─────────────────────────────────┴───────────────────────────┘

# Create projects:
dotnet new blazorserver -n MyBlazorServer
dotnet new blazorwasm -n MyBlazorWasm
```

---

## ขั้นตอนที่ 562: Components & Razor Syntax

```razor
@* Pages/Products.razor - A Blazor page component *@
@page "/products"
@inject IProductService ProductService
@inject NavigationManager Nav

<PageTitle>สินค้าทั้งหมด</PageTitle>

<h1>สินค้า (@totalCount รายการ)</h1>

@if (isLoading)
{
    <div class="spinner-border" role="status">
        <span class="visually-hidden">กำลังโหลด...</span>
    </div>
}
else if (products == null)
{
    <p>ไม่พบข้อมูล</p>
}
else
{
    <div class="row">
        @foreach (var product in products)
        {
            <div class="col-md-4 mb-4">
                <ProductCard Product="product" OnAddToCart="AddToCart" />
            </div>
        }
    </div>
    
    <Pagination CurrentPage="currentPage" TotalPages="totalPages"
                OnPageChange="LoadPage" />
}

@code {
    private List<ProductDto>? products;
    private bool isLoading = true;
    private int currentPage = 1;
    private int totalPages = 1;
    private int totalCount = 0;
    
    protected override async Task OnInitializedAsync()
    {
        await LoadPage(1);
    }
    
    private async Task LoadPage(int page)
    {
        isLoading = true;
        StateHasChanged(); // re-render loading state
        
        var result = await ProductService.GetPagedAsync(page, pageSize: 12);
        products = result.Items;
        currentPage = result.Page;
        totalPages = result.TotalPages;
        totalCount = result.TotalCount;
        isLoading = false;
    }
    
    private async Task AddToCart(ProductDto product)
    {
        await CartService.AddAsync(product.Id);
        // Show notification
    }
}
```

---

## ขั้นตอนที่ 563: Reusable Components

```razor
@* Shared/ProductCard.razor *@
@using MyApp.Models

<div class="card h-100">
    <img src="@Product.ImageUrl" class="card-img-top" alt="@Product.Name" />
    <div class="card-body">
        <h5 class="card-title">@Product.Name</h5>
        <p class="card-text text-muted">@Product.Description</p>
        <p class="card-text">
            <strong class="text-success fs-5">฿@Product.Price.ToString("N0")</strong>
        </p>
        @if (Product.Stock > 0)
        {
            <button class="btn btn-primary w-100" @onclick="HandleAddToCart">
                เพิ่มในตะกร้า
            </button>
        }
        else
        {
            <button class="btn btn-secondary w-100" disabled>
                สินค้าหมด
            </button>
        }
    </div>
</div>

@code {
    [Parameter] public ProductDto Product { get; set; } = null!;
    [Parameter] public EventCallback<ProductDto> OnAddToCart { get; set; }
    
    private async Task HandleAddToCart()
    {
        await OnAddToCart.InvokeAsync(Product);
    }
}

@* Shared/Pagination.razor *@
<nav aria-label="Page navigation">
    <ul class="pagination justify-content-center">
        <li class="page-item @(CurrentPage <= 1 ? "disabled" : "")">
            <button class="page-link" @onclick="() => OnPageChange.InvokeAsync(CurrentPage - 1)">
                ก่อนหน้า
            </button>
        </li>
        
        @for (int i = Math.Max(1, CurrentPage - 2); i <= Math.Min(TotalPages, CurrentPage + 2); i++)
        {
            var pageNum = i;
            <li class="page-item @(pageNum == CurrentPage ? "active" : "")">
                <button class="page-link" @onclick="() => OnPageChange.InvokeAsync(pageNum)">
                    @pageNum
                </button>
            </li>
        }
        
        <li class="page-item @(CurrentPage >= TotalPages ? "disabled" : "")">
            <button class="page-link" @onclick="() => OnPageChange.InvokeAsync(CurrentPage + 1)">
                ถัดไป
            </button>
        </li>
    </ul>
</nav>

@code {
    [Parameter] public int CurrentPage { get; set; }
    [Parameter] public int TotalPages { get; set; }
    [Parameter] public EventCallback<int> OnPageChange { get; set; }
}
```

---

## ขั้นตอนที่ 564: Data Binding

```razor
@* Two-way binding with @bind *@
<input @bind="searchTerm" @bind:event="oninput" placeholder="ค้นหา..." />
<input @bind="price" type="number" />
<select @bind="selectedCategory">
    @foreach (var cat in categories)
    {
        <option value="@cat.Id">@cat.Name</option>
    }
</select>

@* Component parameter binding with @bind-Value *@
<MyInput @bind-Value="userName" />

@* Custom binding format *@
<input @bind="dateValue" @bind:format="dd/MM/yyyy" type="date" />

@code {
    private string searchTerm = "";
    private decimal price = 0;
    private int selectedCategory = 0;
    private string userName = "";
    private DateTime dateValue = DateTime.Today;
    
    // React to changes
    private async Task OnSearchChanged()
    {
        await LoadProducts(); // called every keypress due to oninput
    }
}

@* Custom two-way binding component *@
@* Shared/MyInput.razor *@
<input value="@Value" @oninput="HandleInput" class="form-control" />

@code {
    [Parameter] public string Value { get; set; } = "";
    [Parameter] public EventCallback<string> ValueChanged { get; set; }
    
    private async Task HandleInput(ChangeEventArgs e)
    {
        var newValue = e.Value?.ToString() ?? "";
        await ValueChanged.InvokeAsync(newValue);
    }
}
```

---

## ขั้นตอนที่ 565: Forms & Validation

```razor
@page "/checkout"
@using System.ComponentModel.DataAnnotations

<EditForm Model="@model" OnValidSubmit="HandleSubmit">
    <DataAnnotationsValidator />
    <ValidationSummary />
    
    <div class="mb-3">
        <label>ชื่อ-นามสกุล</label>
        <InputText @bind-Value="model.FullName" class="form-control" />
        <ValidationMessage For="@(() => model.FullName)" />
    </div>
    
    <div class="mb-3">
        <label>อีเมล</label>
        <InputText @bind-Value="model.Email" class="form-control" type="email" />
        <ValidationMessage For="@(() => model.Email)" />
    </div>
    
    <div class="mb-3">
        <label>เบอร์โทร</label>
        <InputText @bind-Value="model.Phone" class="form-control" />
        <ValidationMessage For="@(() => model.Phone)" />
    </div>
    
    <div class="mb-3">
        <label>ที่อยู่จัดส่ง</label>
        <InputTextArea @bind-Value="model.Address" class="form-control" rows="3" />
        <ValidationMessage For="@(() => model.Address)" />
    </div>
    
    <button type="submit" class="btn btn-primary" disabled="@isSubmitting">
        @if (isSubmitting) { <span class="spinner-border spinner-border-sm me-2"></span> }
        ยืนยันคำสั่งซื้อ
    </button>
</EditForm>

@code {
    private CheckoutModel model = new();
    private bool isSubmitting;
    
    private async Task HandleSubmit()
    {
        isSubmitting = true;
        try
        {
            await OrderService.CheckoutAsync(model);
            Nav.NavigateTo("/order-success");
        }
        finally { isSubmitting = false; }
    }
    
    public class CheckoutModel
    {
        [Required(ErrorMessage = "กรุณาระบุชื่อ-นามสกุล")]
        [MaxLength(100)]
        public string FullName { get; set; } = "";
        
        [Required(ErrorMessage = "กรุณาระบุอีเมล")]
        [EmailAddress(ErrorMessage = "รูปแบบอีเมลไม่ถูกต้อง")]
        public string Email { get; set; } = "";
        
        [Required(ErrorMessage = "กรุณาระบุเบอร์โทร")]
        [RegularExpression(@"^0[0-9]{9}$", ErrorMessage = "เบอร์โทรต้องมี 10 หลัก")]
        public string Phone { get; set; } = "";
        
        [Required(ErrorMessage = "กรุณาระบุที่อยู่")]
        public string Address { get; set; } = "";
    }
}
```

---

## ขั้นตอนที่ 566: HTTP Client in Blazor WASM

```csharp
// Program.cs (Blazor WASM)
var builder = WebAssemblyHostBuilder.CreateDefault(args);
builder.RootComponents.Add<App>("#app");

// Configure HttpClient for API calls
builder.Services.AddScoped(sp => new HttpClient
{
    BaseAddress = new Uri("https://api.myapp.com/")
});

// Or use typed clients
builder.Services.AddHttpClient<IProductService, ProductApiService>(client =>
{
    client.BaseAddress = new Uri("https://api.myapp.com/");
    client.DefaultRequestHeaders.Add("Accept", "application/json");
});

await builder.Build().RunAsync();

// Services/ProductApiService.cs
public class ProductApiService : IProductService
{
    private readonly HttpClient _http;
    
    public ProductApiService(HttpClient http) => _http = http;
    
    public async Task<PagedResult<ProductDto>> GetPagedAsync(int page, int pageSize)
    {
        var result = await _http.GetFromJsonAsync<PagedResult<ProductDto>>(
            $"products?page={page}&pageSize={pageSize}");
        return result ?? new PagedResult<ProductDto>();
    }
    
    public async Task<ProductDto?> GetByIdAsync(int id)
        => await _http.GetFromJsonAsync<ProductDto>($"products/{id}");
    
    public async Task<bool> CreateAsync(CreateProductDto dto)
    {
        var response = await _http.PostAsJsonAsync("products", dto);
        return response.IsSuccessStatusCode;
    }
    
    public async Task<bool> UpdateAsync(int id, UpdateProductDto dto)
    {
        var response = await _http.PutAsJsonAsync($"products/{id}", dto);
        return response.IsSuccessStatusCode;
    }
}
```

---

## ขั้นตอนที่ 567: State Management

```csharp
// AppState.cs - Simple state container
public class AppState
{
    private readonly List<CartItem> _cart = new();
    
    public IReadOnlyList<CartItem> Cart => _cart.AsReadOnly();
    public int CartCount => _cart.Sum(i => i.Quantity);
    public decimal CartTotal => _cart.Sum(i => i.Price * i.Quantity);
    
    public event Action? OnCartChanged;
    
    public void AddToCart(ProductDto product, int qty = 1)
    {
        var existing = _cart.FirstOrDefault(i => i.ProductId == product.Id);
        if (existing != null)
            existing.Quantity += qty;
        else
            _cart.Add(new CartItem(product.Id, product.Name, product.Price, qty));
        
        OnCartChanged?.Invoke();
    }
    
    public void RemoveFromCart(int productId)
    {
        _cart.RemoveAll(i => i.ProductId == productId);
        OnCartChanged?.Invoke();
    }
    
    public void ClearCart()
    {
        _cart.Clear();
        OnCartChanged?.Invoke();
    }
}

// Register as Scoped (one per user session in Server, one in WASM)
builder.Services.AddScoped<AppState>();

// Use in component
@inject AppState State
@implements IDisposable

<span class="badge bg-primary">@State.CartCount</span>

@code {
    protected override void OnInitialized()
    {
        State.OnCartChanged += StateHasChanged; // re-render on cart change
    }
    
    public void Dispose()
    {
        State.OnCartChanged -= StateHasChanged; // prevent memory leak
    }
}
```

---

## ขั้นตอนที่ 568-570: Complete Product Management Page

```razor
@page "/admin/products"
@attribute [Authorize(Roles = "Admin")]
@inject IProductService ProductService
@inject IToastService Toast

<PageTitle>จัดการสินค้า</PageTitle>

<div class="d-flex justify-content-between align-items-center mb-4">
    <h1>สินค้า (@totalCount รายการ)</h1>
    <button class="btn btn-primary" @onclick="ShowCreateForm">+ เพิ่มสินค้า</button>
</div>

<div class="row mb-4">
    <div class="col-md-6">
        <input @bind="search" @bind:event="oninput" @oninput="OnSearchChanged"
               class="form-control" placeholder="ค้นหาสินค้า..." />
    </div>
    <div class="col-md-3">
        <select @bind="categoryFilter" @bind:after="LoadProducts" class="form-select">
            <option value="0">ทุกหมวดหมู่</option>
            @foreach (var cat in categories)
            {
                <option value="@cat.Id">@cat.Name</option>
            }
        </select>
    </div>
</div>

@if (isLoading)
{
    <div class="text-center py-5">
        <div class="spinner-border text-primary"></div>
    </div>
}
else
{
    <table class="table table-hover">
        <thead>
            <tr>
                <th>รหัส</th><th>ชื่อสินค้า</th><th>หมวดหมู่</th>
                <th>ราคา</th><th>สต็อก</th><th></th>
            </tr>
        </thead>
        <tbody>
            @foreach (var p in products)
            {
                <tr>
                    <td>@p.Code</td>
                    <td>@p.Name</td>
                    <td><span class="badge bg-secondary">@p.CategoryName</span></td>
                    <td class="text-end">฿@p.Price.ToString("N0")</td>
                    <td class="text-center">
                        <span class="badge @(p.Stock > 10 ? "bg-success" : p.Stock > 0 ? "bg-warning" : "bg-danger")">
                            @p.Stock
                        </span>
                    </td>
                    <td>
                        <button class="btn btn-sm btn-outline-primary me-1" @onclick="() => EditProduct(p)">แก้ไข</button>
                        <button class="btn btn-sm btn-outline-danger" @onclick="() => DeleteProduct(p)">ลบ</button>
                    </td>
                </tr>
            }
        </tbody>
    </table>
    
    <Pagination CurrentPage="currentPage" TotalPages="totalPages" OnPageChange="LoadPage" />
}

@if (showForm)
{
    <ProductFormModal Product="editingProduct" OnSave="SaveProduct" OnClose="() => showForm = false" />
}

@code {
    private List<ProductDto> products = new();
    private List<CategoryDto> categories = new();
    private bool isLoading = true;
    private bool showForm = false;
    private ProductDto? editingProduct;
    private string search = "";
    private int categoryFilter = 0;
    private int currentPage = 1, totalPages = 1, totalCount = 0;
    private Timer? searchDebounce;
    
    protected override async Task OnInitializedAsync()
    {
        categories = await ProductService.GetCategoriesAsync();
        await LoadProducts();
    }
    
    private void OnSearchChanged()
    {
        searchDebounce?.Dispose();
        searchDebounce = new Timer(async _ =>
        {
            await InvokeAsync(async () =>
            {
                currentPage = 1;
                await LoadProducts();
            });
        }, null, 400, Timeout.Infinite); // 400ms debounce
    }
    
    private async Task LoadProducts()
    {
        isLoading = true;
        var result = await ProductService.SearchAsync(search, categoryFilter, currentPage);
        products = result.Items;
        totalPages = result.TotalPages;
        totalCount = result.TotalCount;
        isLoading = false;
    }
    
    private Task LoadPage(int page) { currentPage = page; return LoadProducts(); }
    
    private void ShowCreateForm() { editingProduct = null; showForm = true; }
    private void EditProduct(ProductDto p) { editingProduct = p; showForm = true; }
    
    private async Task SaveProduct(ProductFormModel form)
    {
        var success = editingProduct == null 
            ? await ProductService.CreateAsync(form)
            : await ProductService.UpdateAsync(editingProduct.Id, form);
        
        if (success)
        {
            Toast.ShowSuccess(editingProduct == null ? "เพิ่มสินค้าสำเร็จ" : "แก้ไขสินค้าสำเร็จ");
            showForm = false;
            await LoadProducts();
        }
    }
    
    private async Task DeleteProduct(ProductDto p)
    {
        if (!await JS.InvokeAsync<bool>("confirm", $"ต้องการลบสินค้า '{p.Name}'?")) return;
        await ProductService.DeleteAsync(p.Id);
        Toast.ShowSuccess("ลบสินค้าสำเร็จ");
        await LoadProducts();
    }
}
```

---

## 📝 สรุป Part 57

| Concept | ใช้ใน |
|---------|-------|
| @page | Routing |
| @inject | DI |
| @bind | Two-way data binding |
| EventCallback | Child→Parent communication |
| [Parameter] | Parent→Child communication |
| EditForm | Form handling + validation |
| StateHasChanged() | Force re-render |
| IDisposable | Clean up event subscriptions |

---

**ก่อนหน้า → [Part 56: Advanced C#](part56-advanced-csharp.md)**  
**ต่อไป → [Part 58: ASP.NET Core REST API](part58-aspnet-core-api.md)**
