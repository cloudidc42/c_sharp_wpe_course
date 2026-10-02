# Part 40: WPF Final Project - Inventory Management System
## ขั้นตอนที่ 391-400: โปรแกรมจัดการคลังสินค้าสมบูรณ์

---

## 🎯 เป้าหมายของ Part นี้
- MVVM สมบูรณ์แบบพร้อม DI
- SQLite database integration
- Full CRUD operations
- Charts (bar/line) แบบ custom
- Print reports
- Export CSV/Excel
- Responsive layout
- โปรแกรม Inventory Management System

---

## ขั้นตอนที่ 391: โครงสร้างโปรเจค

```
InventoryApp/
├── Models/
│   ├── Product.cs
│   ├── Category.cs
│   ├── StockMovement.cs
│   └── Supplier.cs
├── Data/
│   ├── AppDbContext.cs
│   ├── IProductRepository.cs
│   └── ProductRepository.cs
├── Services/
│   ├── IDialogService.cs
│   ├── DialogService.cs
│   └── ReportService.cs
├── ViewModels/
│   ├── ViewModelBase.cs
│   ├── RelayCommand.cs
│   ├── MainViewModel.cs
│   ├── DashboardViewModel.cs
│   ├── ProductListViewModel.cs
│   ├── ProductEditViewModel.cs
│   └── ReportsViewModel.cs
├── Views/
│   ├── MainWindow.xaml
│   ├── DashboardView.xaml
│   ├── ProductListView.xaml
│   ├── ProductEditDialog.xaml
│   └── ReportsView.xaml
├── Converters/
│   └── Converters.cs
├── Themes/
│   ├── LightTheme.xaml
│   └── DarkTheme.xaml
└── App.xaml
```

---

## ขั้นตอนที่ 392: Models และ Database

```csharp
// Models/Product.cs
public class Product
{
    public int Id { get; set; }
    public string Code { get; set; } = "";
    public string Name { get; set; } = "";
    public string Description { get; set; } = "";
    public int CategoryId { get; set; }
    public int SupplierId { get; set; }
    public decimal CostPrice { get; set; }
    public decimal SellPrice { get; set; }
    public int Stock { get; set; }
    public int MinStock { get; set; }
    public int MaxStock { get; set; }
    public string Unit { get; set; } = "ชิ้น";
    public bool IsActive { get; set; } = true;
    public DateTime CreatedAt { get; set; } = DateTime.Now;
    
    // Navigation (not in DB)
    public string? CategoryName { get; set; }
    public string? SupplierName { get; set; }
    
    public bool IsLowStock => Stock <= MinStock;
    public bool IsOverStock => Stock >= MaxStock;
    public decimal ProfitMargin => SellPrice > 0 ? (SellPrice - CostPrice) / SellPrice * 100 : 0;
    public decimal StockValue => Stock * CostPrice;
}

// Data/ProductRepository.cs
using Microsoft.Data.Sqlite;

public class ProductRepository : IProductRepository
{
    private readonly string _connStr;
    
    public ProductRepository(string connectionString)
    {
        _connStr = connectionString;
        InitializeDb();
    }
    
    private void InitializeDb()
    {
        using var conn = new SqliteConnection(_connStr);
        conn.Open();
        using var cmd = conn.CreateCommand();
        cmd.CommandText = @"
            CREATE TABLE IF NOT EXISTS Categories (
                Id INTEGER PRIMARY KEY AUTOINCREMENT,
                Name TEXT NOT NULL UNIQUE
            );
            
            CREATE TABLE IF NOT EXISTS Suppliers (
                Id INTEGER PRIMARY KEY AUTOINCREMENT,
                Name TEXT NOT NULL,
                Contact TEXT,
                Phone TEXT
            );
            
            CREATE TABLE IF NOT EXISTS Products (
                Id INTEGER PRIMARY KEY AUTOINCREMENT,
                Code TEXT NOT NULL UNIQUE,
                Name TEXT NOT NULL,
                Description TEXT DEFAULT '',
                CategoryId INTEGER,
                SupplierId INTEGER,
                CostPrice REAL NOT NULL DEFAULT 0,
                SellPrice REAL NOT NULL DEFAULT 0,
                Stock INTEGER NOT NULL DEFAULT 0,
                MinStock INTEGER NOT NULL DEFAULT 5,
                MaxStock INTEGER NOT NULL DEFAULT 100,
                Unit TEXT NOT NULL DEFAULT 'ชิ้น',
                IsActive INTEGER NOT NULL DEFAULT 1,
                CreatedAt TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (CategoryId) REFERENCES Categories(Id),
                FOREIGN KEY (SupplierId) REFERENCES Suppliers(Id)
            );
            
            CREATE TABLE IF NOT EXISTS StockMovements (
                Id INTEGER PRIMARY KEY AUTOINCREMENT,
                ProductId INTEGER NOT NULL,
                MovementType TEXT NOT NULL,
                Quantity INTEGER NOT NULL,
                Notes TEXT,
                MovedAt TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (ProductId) REFERENCES Products(Id)
            );
            
            INSERT OR IGNORE INTO Categories (Name) VALUES 
                ('อิเล็กทรอนิกส์'), ('เสื้อผ้า'), ('อาหาร'), ('เครื่องใช้ไฟฟ้า'), ('กีฬา');
        ";
        cmd.ExecuteNonQuery();
        SeedData(conn);
    }
    
    private void SeedData(SqliteConnection conn)
    {
        using var countCmd = conn.CreateCommand();
        countCmd.CommandText = "SELECT COUNT(*) FROM Products";
        var count = (long)countCmd.ExecuteScalar()!;
        if (count > 0) return;
        
        using var cmd = conn.CreateCommand();
        cmd.CommandText = @"
            INSERT INTO Products (Code, Name, CategoryId, CostPrice, SellPrice, Stock, MinStock, MaxStock) VALUES
            ('PRD001', 'iPhone 15 Pro', 1, 25000, 35000, 15, 5, 50),
            ('PRD002', 'Samsung Galaxy S24', 1, 20000, 28000, 22, 5, 60),
            ('PRD003', 'เสื้อยืด Basic', 2, 150, 350, 200, 20, 500),
            ('PRD004', 'กางเกงยีนส์', 2, 500, 1200, 85, 10, 200),
            ('PRD005', 'AirPods Pro', 1, 6000, 9500, 30, 10, 100),
            ('PRD006', 'MacBook Air M3', 1, 38000, 52000, 8, 3, 30),
            ('PRD007', 'หูฟัง Sony WH-1000XM5', 4, 8000, 12000, 12, 5, 40);
        ";
        cmd.ExecuteNonQuery();
    }
    
    public async Task<List<Product>> GetAllAsync(bool includeInactive = false)
    {
        using var conn = new SqliteConnection(_connStr);
        await conn.OpenAsync();
        using var cmd = conn.CreateCommand();
        cmd.CommandText = @"
            SELECT p.*, c.Name AS CategoryName
            FROM Products p
            LEFT JOIN Categories c ON p.CategoryId = c.Id
            WHERE (@includeInactive = 1 OR p.IsActive = 1)
            ORDER BY p.Name";
        cmd.Parameters.AddWithValue("@includeInactive", includeInactive ? 1 : 0);
        
        var products = new List<Product>();
        using var reader = await cmd.ExecuteReaderAsync();
        while (await reader.ReadAsync())
            products.Add(MapProduct(reader));
        return products;
    }
    
    public async Task<Product?> GetByIdAsync(int id)
    {
        using var conn = new SqliteConnection(_connStr);
        await conn.OpenAsync();
        using var cmd = conn.CreateCommand();
        cmd.CommandText = "SELECT p.*, c.Name AS CategoryName FROM Products p LEFT JOIN Categories c ON p.CategoryId = c.Id WHERE p.Id = @id";
        cmd.Parameters.AddWithValue("@id", id);
        
        using var reader = await cmd.ExecuteReaderAsync();
        return await reader.ReadAsync() ? MapProduct(reader) : null;
    }
    
    public async Task<int> CreateAsync(Product product)
    {
        using var conn = new SqliteConnection(_connStr);
        await conn.OpenAsync();
        using var cmd = conn.CreateCommand();
        cmd.CommandText = @"
            INSERT INTO Products (Code, Name, Description, CategoryId, SupplierId, 
                CostPrice, SellPrice, Stock, MinStock, MaxStock, Unit)
            VALUES (@code, @name, @desc, @catId, @supId, @cost, @sell, @stock, @min, @max, @unit);
            SELECT last_insert_rowid();";
        cmd.Parameters.AddWithValue("@code", product.Code);
        cmd.Parameters.AddWithValue("@name", product.Name);
        cmd.Parameters.AddWithValue("@desc", product.Description ?? "");
        cmd.Parameters.AddWithValue("@catId", product.CategoryId > 0 ? product.CategoryId : DBNull.Value);
        cmd.Parameters.AddWithValue("@supId", product.SupplierId > 0 ? product.SupplierId : DBNull.Value);
        cmd.Parameters.AddWithValue("@cost", product.CostPrice);
        cmd.Parameters.AddWithValue("@sell", product.SellPrice);
        cmd.Parameters.AddWithValue("@stock", product.Stock);
        cmd.Parameters.AddWithValue("@min", product.MinStock);
        cmd.Parameters.AddWithValue("@max", product.MaxStock);
        cmd.Parameters.AddWithValue("@unit", product.Unit);
        
        return (int)(long)await cmd.ExecuteScalarAsync()!;
    }
    
    public async Task UpdateAsync(Product product)
    {
        using var conn = new SqliteConnection(_connStr);
        await conn.OpenAsync();
        using var cmd = conn.CreateCommand();
        cmd.CommandText = @"
            UPDATE Products SET 
                Code=@code, Name=@name, Description=@desc, CategoryId=@catId,
                CostPrice=@cost, SellPrice=@sell, Stock=@stock, 
                MinStock=@min, MaxStock=@max, Unit=@unit
            WHERE Id=@id";
        cmd.Parameters.AddWithValue("@id", product.Id);
        cmd.Parameters.AddWithValue("@code", product.Code);
        cmd.Parameters.AddWithValue("@name", product.Name);
        cmd.Parameters.AddWithValue("@desc", product.Description ?? "");
        cmd.Parameters.AddWithValue("@catId", product.CategoryId > 0 ? product.CategoryId : DBNull.Value);
        cmd.Parameters.AddWithValue("@cost", product.CostPrice);
        cmd.Parameters.AddWithValue("@sell", product.SellPrice);
        cmd.Parameters.AddWithValue("@stock", product.Stock);
        cmd.Parameters.AddWithValue("@min", product.MinStock);
        cmd.Parameters.AddWithValue("@max", product.MaxStock);
        cmd.Parameters.AddWithValue("@unit", product.Unit);
        await cmd.ExecuteNonQueryAsync();
    }
    
    public async Task DeleteAsync(int id)
    {
        using var conn = new SqliteConnection(_connStr);
        await conn.OpenAsync();
        using var cmd = conn.CreateCommand();
        cmd.CommandText = "UPDATE Products SET IsActive=0 WHERE Id=@id";
        cmd.Parameters.AddWithValue("@id", id);
        await cmd.ExecuteNonQueryAsync();
    }
    
    public async Task<Dictionary<string, int>> GetStockByCategoryAsync()
    {
        using var conn = new SqliteConnection(_connStr);
        await conn.OpenAsync();
        using var cmd = conn.CreateCommand();
        cmd.CommandText = @"
            SELECT c.Name, SUM(p.Stock) AS Total
            FROM Products p
            LEFT JOIN Categories c ON p.CategoryId = c.Id
            WHERE p.IsActive = 1
            GROUP BY c.Name ORDER BY Total DESC";
        
        var result = new Dictionary<string, int>();
        using var reader = await cmd.ExecuteReaderAsync();
        while (await reader.ReadAsync())
            result[reader.GetString(0)] = reader.GetInt32(1);
        return result;
    }
    
    private static Product MapProduct(SqliteDataReader r) => new()
    {
        Id = r.GetInt32(r.GetOrdinal("Id")),
        Code = r.GetString(r.GetOrdinal("Code")),
        Name = r.GetString(r.GetOrdinal("Name")),
        Description = r.IsDBNull(r.GetOrdinal("Description")) ? "" : r.GetString(r.GetOrdinal("Description")),
        CategoryId = r.IsDBNull(r.GetOrdinal("CategoryId")) ? 0 : r.GetInt32(r.GetOrdinal("CategoryId")),
        CostPrice = (decimal)r.GetDouble(r.GetOrdinal("CostPrice")),
        SellPrice = (decimal)r.GetDouble(r.GetOrdinal("SellPrice")),
        Stock = r.GetInt32(r.GetOrdinal("Stock")),
        MinStock = r.GetInt32(r.GetOrdinal("MinStock")),
        MaxStock = r.GetInt32(r.GetOrdinal("MaxStock")),
        Unit = r.GetString(r.GetOrdinal("Unit")),
        IsActive = r.GetInt32(r.GetOrdinal("IsActive")) == 1,
        CategoryName = r.GetOrdinal("CategoryName") >= 0 && !r.IsDBNull(r.GetOrdinal("CategoryName"))
            ? r.GetString(r.GetOrdinal("CategoryName")) : null,
    };
}
```

---

## ขั้นตอนที่ 393-396: ViewModels

```csharp
// ViewModels/ProductListViewModel.cs
public class ProductListViewModel : ViewModelBase
{
    private readonly IProductRepository _repo;
    private readonly IDialogService _dialog;
    private List<Product> _allProducts = new();
    private ObservableCollection<Product> _products = new();
    private Product? _selected;
    private string _search = "";
    private bool _isLoading;
    
    public ObservableCollection<Product> Products
    {
        get => _products;
        private set => SetProperty(ref _products, value);
    }
    
    public Product? SelectedProduct
    {
        get => _selected;
        set => SetProperty(ref _selected, value);
    }
    
    public string SearchText
    {
        get => _search;
        set
        {
            if (SetProperty(ref _search, value))
                ApplyFilter();
        }
    }
    
    public bool IsLoading
    {
        get => _isLoading;
        set => SetProperty(ref _isLoading, value);
    }
    
    // Alerts
    public int LowStockCount => _allProducts.Count(p => p.IsLowStock && p.IsActive);
    public decimal TotalStockValue => _allProducts.Where(p => p.IsActive).Sum(p => p.StockValue);
    
    public AsyncRelayCommand LoadCommand { get; }
    public AsyncRelayCommand AddCommand { get; }
    public AsyncRelayCommand<Product?> EditCommand { get; }
    public AsyncRelayCommand<Product?> DeleteCommand { get; }
    public RelayCommand<Product?> AdjustStockCommand { get; }
    public RelayCommand ExportCsvCommand { get; }
    
    public ProductListViewModel(IProductRepository repo, IDialogService dialog)
    {
        _repo = repo;
        _dialog = dialog;
        
        LoadCommand = new AsyncRelayCommand(LoadAsync);
        AddCommand = new AsyncRelayCommand(AddAsync);
        EditCommand = new AsyncRelayCommand<Product?>(EditAsync);
        DeleteCommand = new AsyncRelayCommand<Product?>(DeleteAsync, p => p != null);
        AdjustStockCommand = new RelayCommand<Product?>(AdjustStock, p => p != null);
        ExportCsvCommand = new RelayCommand(ExportCsv, () => _allProducts.Count > 0);
    }
    
    private async Task LoadAsync()
    {
        IsLoading = true;
        try
        {
            _allProducts = await _repo.GetAllAsync();
            ApplyFilter();
            OnPropertyChanged(nameof(LowStockCount));
            OnPropertyChanged(nameof(TotalStockValue));
        }
        finally { IsLoading = false; }
    }
    
    private async Task AddAsync()
    {
        var vm = new ProductEditViewModel();
        if (_dialog.ShowDialog(vm) == true)
        {
            await _repo.CreateAsync(vm.ToProduct());
            await LoadAsync();
        }
    }
    
    private async Task EditAsync(Product? product)
    {
        if (product == null) return;
        var vm = new ProductEditViewModel(product);
        if (_dialog.ShowDialog(vm) == true)
        {
            vm.ApplyTo(product);
            await _repo.UpdateAsync(product);
            await LoadAsync();
        }
    }
    
    private async Task DeleteAsync(Product? product)
    {
        if (product == null) return;
        if (!_dialog.Confirm($"ลบสินค้า '{product.Name}'?")) return;
        await _repo.DeleteAsync(product.Id);
        await LoadAsync();
    }
    
    private void AdjustStock(Product? product)
    {
        if (product == null) return;
        var vm = new StockAdjustViewModel(product);
        _dialog.ShowDialog(vm);
    }
    
    private void ApplyFilter()
    {
        var q = _allProducts.AsEnumerable();
        if (!string.IsNullOrWhiteSpace(_search))
            q = q.Where(p => p.Name.Contains(_search, StringComparison.OrdinalIgnoreCase) ||
                              p.Code.Contains(_search, StringComparison.OrdinalIgnoreCase));
        Products = new ObservableCollection<Product>(q);
    }
    
    private void ExportCsv()
    {
        var path = _dialog.SaveFile("CSV files (*.csv)|*.csv", "inventory_export.csv");
        if (path == null) return;
        
        var lines = new List<string> { "Code,Name,Category,Stock,CostPrice,SellPrice,StockValue" };
        foreach (var p in _allProducts)
            lines.Add($"{p.Code},{p.Name},{p.CategoryName},{p.Stock},{p.CostPrice},{p.SellPrice},{p.StockValue}");
        
        File.WriteAllLines(path, lines);
        _dialog.ShowMessage($"Export สำเร็จ: {path}");
    }
}
```

---

## ขั้นตอนที่ 397-400: Main XAML

```xml
<!-- Views/ProductListView.xaml -->
<UserControl x:Class="InventoryApp.Views.ProductListView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:local="clr-namespace:InventoryApp"
             Loaded="OnLoaded">
    
    <UserControl.Resources>
        <local:BoolToColorConverter x:Key="LowStockColor"/>
    </UserControl.Resources>
    
    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>
        
        <!-- Alert bar -->
        <Border Grid.Row="0" Background="#FFF3CD" Padding="12,8"
                Visibility="{Binding LowStockCount, Converter={StaticResource ...}}">
            <TextBlock Foreground="#856404"
                       Text="{Binding LowStockCount, StringFormat='⚠️ มีสินค้า {0} รายการที่ใกล้หมดสต็อก'}"/>
        </Border>
        
        <!-- Toolbar -->
        <DockPanel Grid.Row="1" Margin="12,8" LastChildFill="True">
            <Button DockPanel.Dock="Right" Content="📤 Export CSV" Padding="12,5" Margin="5,0,0,0"
                    Command="{Binding ExportCsvCommand}"/>
            <Button DockPanel.Dock="Right" Content="➕ เพิ่มสินค้า" Padding="12,5" Margin="5,0,0,0"
                    Command="{Binding AddCommand}" Background="#27AE60" Foreground="White"/>
            <TextBox Text="{Binding SearchText, UpdateSourceTrigger=PropertyChanged}"
                     FontSize="13"/>
        </DockPanel>
        
        <!-- KPI row -->
        <StackPanel Grid.Row="1" HorizontalAlignment="Right" Orientation="Horizontal" Margin="12">
            <Border Background="#EBF5FB" CornerRadius="4" Padding="12,6" Margin="5,0">
                <TextBlock Text="{Binding Products.Count, StringFormat='สินค้า: {0} รายการ'}" FontSize="12"/>
            </Border>
            <Border Background="#EAFAF1" CornerRadius="4" Padding="12,6" Margin="5,0">
                <TextBlock Text="{Binding TotalStockValue, StringFormat='มูลค่า: ฿{0:N0}'}" FontSize="12"/>
            </Border>
        </StackPanel>
        
        <!-- Product grid -->
        <DataGrid Grid.Row="2" Margin="12,0"
                  ItemsSource="{Binding Products}"
                  SelectedItem="{Binding SelectedProduct}"
                  AutoGenerateColumns="False"
                  CanUserAddRows="False"
                  IsReadOnly="True"
                  GridLinesVisibility="Horizontal"
                  HeadersVisibility="Column"
                  SelectionMode="Single"
                  FontSize="13">
            <DataGrid.Columns>
                <DataGridTextColumn Header="รหัส" Binding="{Binding Code}" Width="90"/>
                <DataGridTextColumn Header="ชื่อสินค้า" Binding="{Binding Name}" Width="*"/>
                <DataGridTextColumn Header="หมวดหมู่" Binding="{Binding CategoryName}" Width="120"/>
                <DataGridTextColumn Header="ราคาทุน" Binding="{Binding CostPrice, StringFormat='฿{0:N0}'}" Width="100"/>
                <DataGridTextColumn Header="ราคาขาย" Binding="{Binding SellPrice, StringFormat='฿{0:N0}'}" Width="100"/>
                <DataGridTemplateColumn Header="สต็อก" Width="100">
                    <DataGridTemplateColumn.CellTemplate>
                        <DataTemplate>
                            <StackPanel Orientation="Horizontal">
                                <TextBlock Text="{Binding Stock}" FontWeight="Bold">
                                    <TextBlock.Foreground>
                                        <Binding Path="IsLowStock" Converter="{StaticResource LowStockColor}"/>
                                    </TextBlock.Foreground>
                                </TextBlock>
                                <TextBlock Text="{Binding Unit, StringFormat=' {0}'}" Foreground="Gray" FontSize="11"/>
                            </StackPanel>
                        </DataTemplate>
                    </DataGridTemplateColumn.CellTemplate>
                </DataGridTemplateColumn>
                <DataGridTemplateColumn Header="การดำเนินการ" Width="160">
                    <DataGridTemplateColumn.CellTemplate>
                        <DataTemplate>
                            <StackPanel Orientation="Horizontal">
                                <Button Content="✏️ แก้ไข" Padding="8,3" Margin="2"
                                        Command="{Binding DataContext.EditCommand, RelativeSource={RelativeSource AncestorType=DataGrid}}"
                                        CommandParameter="{Binding}"/>
                                <Button Content="📦" Padding="8,3" Margin="2"
                                        Command="{Binding DataContext.AdjustStockCommand, RelativeSource={RelativeSource AncestorType=DataGrid}}"
                                        CommandParameter="{Binding}"
                                        ToolTip="ปรับสต็อก"/>
                                <Button Content="🗑️" Padding="8,3" Margin="2"
                                        Command="{Binding DataContext.DeleteCommand, RelativeSource={RelativeSource AncestorType=DataGrid}}"
                                        CommandParameter="{Binding}"
                                        Background="#E74C3C" Foreground="White"/>
                            </StackPanel>
                        </DataTemplate>
                    </DataGridTemplateColumn.CellTemplate>
                </DataGridTemplateColumn>
            </DataGrid.Columns>
        </DataGrid>
        
        <!-- Status bar -->
        <StatusBar Grid.Row="3">
            <StatusBarItem>
                <TextBlock Text="{Binding IsLoading, Converter={StaticResource ...}, ConverterParameter='กำลังโหลด...|พร้อมใช้งาน'}"/>
            </StatusBarItem>
        </StatusBar>
    </Grid>
</UserControl>
```

```csharp
// Views/ProductListView.xaml.cs
public partial class ProductListView : UserControl
{
    public ProductListView() => InitializeComponent();
    
    private async void OnLoaded(object sender, RoutedEventArgs e)
    {
        if (DataContext is ProductListViewModel vm)
            await vm.LoadCommand.ExecuteAsync();
    }
}
```

---

## 📋 Project Summary

| Layer | Component | Technology |
|-------|-----------|-----------|
| UI | MainWindow + Views | WPF XAML |
| Logic | ViewModels | MVVM + Commands |
| Data | Repository | SQLite (Microsoft.Data.Sqlite) |
| DI | App.xaml.cs | Microsoft.Extensions.DI |
| Services | Dialog, Report | Interface-based |
| Theme | Light/Dark | ResourceDictionary |

**Tech Stack:**
```xml
<PackageReference Include="Microsoft.Data.Sqlite" Version="8.0.0"/>
<PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="8.0.0"/>
```

---

## 🏆 WPF Section Complete (Parts 31-40)

| Part | หัวข้อ | Steps |
|------|--------|-------|
| 31 | WPF Introduction | 301-310 |
| 32 | Data Binding | 311-320 |
| 33 | MVVM Pattern | 321-330 |
| 34 | Styles & Templates | 331-340 |
| 35 | Animation | 341-350 |
| 36 | Navigation | 351-360 |
| 37 | Custom Controls | 361-370 |
| 38 | Resources & Theming | 371-380 |
| 39 | Advanced MVVM | 381-390 |
| 40 | Final Project | 391-400 |

---

**ก่อนหน้า → [Part 39: Advanced MVVM](part39-wpf-advanced-mvvm.md)**  
**ต่อไป → [Part 41: Entity Framework Core](../part41-50/part41-ef-core-basics.md)**
