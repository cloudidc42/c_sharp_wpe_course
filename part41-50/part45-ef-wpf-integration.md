# Part 45: EF Core + WPF Integration
## ขั้นตอนที่ 441-450: Full-Stack Desktop App

---

## 🎯 เป้าหมายของ Part นี้
- IDbContextFactory ใน WPF
- Async data loading ใน ViewModel
- Search and filtering with IQueryable
- Pagination
- Background data operations
- Progress reporting
- Complete CRM Application

---

## ขั้นตอนที่ 441: WPF + EF Core Setup

```csharp
// App.xaml.cs
public partial class App : Application
{
    private ServiceProvider _sp = null!;
    
    protected override async void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);
        
        _sp = BuildServices();
        
        // Migrate database
        using var scope = _sp.CreateScope();
        var factory = scope.ServiceProvider.GetRequiredService<IDbContextFactory<CrmDbContext>>();
        await using var ctx = await factory.CreateDbContextAsync();
        await ctx.Database.MigrateAsync();
        
        // Show main window
        var mainWindow = _sp.GetRequiredService<MainWindow>();
        mainWindow.Show();
    }
    
    private static ServiceProvider BuildServices()
    {
        var s = new ServiceCollection();
        
        string dbPath = Path.Combine(
            Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
            "CRM", "crm.db");
        Directory.CreateDirectory(Path.GetDirectoryName(dbPath)!);
        
        // Use Factory for desktop (not AddDbContext which is scoped)
        s.AddDbContextFactory<CrmDbContext>(o => o.UseSqlite($"Data Source={dbPath}"));
        
        // ViewModels
        s.AddTransient<MainViewModel>();
        s.AddTransient<CustomerListViewModel>();
        s.AddTransient<CustomerEditViewModel>();
        s.AddTransient<DealListViewModel>();
        
        // Services
        s.AddSingleton<IDialogService, DialogService>();
        s.AddSingleton<Messenger>();
        
        // Windows
        s.AddSingleton<MainWindow>(sp =>
        {
            var w = new MainWindow();
            w.DataContext = sp.GetRequiredService<MainViewModel>();
            return w;
        });
        
        return s.BuildServiceProvider();
    }
    
    protected override void OnExit(ExitEventArgs e)
    {
        _sp.Dispose();
        base.OnExit(e);
    }
}
```

---

## ขั้นตอนที่ 442: Entities

```csharp
// Entities/Customer.cs
public class Customer
{
    public int Id { get; set; }
    public string FirstName { get; set; } = "";
    public string LastName { get; set; } = "";
    public string Email { get; set; } = "";
    public string? Phone { get; set; }
    public string? Company { get; set; }
    public CustomerStatus Status { get; set; } = CustomerStatus.Active;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? LastContactAt { get; set; }
    
    public ICollection<Deal> Deals { get; set; } = new List<Deal>();
    public ICollection<Activity> Activities { get; set; } = new List<Activity>();
    
    public string FullName => $"{FirstName} {LastName}";
}

public enum CustomerStatus { Lead, Active, Inactive, Lost }

// Entities/Deal.cs
public class Deal
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public decimal Value { get; set; }
    public DealStage Stage { get; set; } = DealStage.Prospect;
    public int CustomerId { get; set; }
    public Customer Customer { get; set; } = null!;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? ExpectedCloseDate { get; set; }
}

public enum DealStage { Prospect, Qualified, Proposal, Negotiation, Won, Lost }

// Data/CrmDbContext.cs
public class CrmDbContext : DbContext
{
    public DbSet<Customer> Customers { get; set; }
    public DbSet<Deal> Deals { get; set; }
    public DbSet<Activity> Activities { get; set; }
    
    public CrmDbContext(DbContextOptions<CrmDbContext> options) : base(options) { }
    
    protected override void OnModelCreating(ModelBuilder mb)
    {
        mb.Entity<Customer>(e =>
        {
            e.HasIndex(c => c.Email).IsUnique();
            e.Property(c => c.Status).HasConversion<string>();
        });
        mb.Entity<Deal>(e => e.Property(d => d.Stage).HasConversion<string>());
    }
}
```

---

## ขั้นตอนที่ 443: ViewModel with Async Loading

```csharp
// ViewModels/CustomerListViewModel.cs
public class CustomerListViewModel : ViewModelBase
{
    private readonly IDbContextFactory<CrmDbContext> _ctxFactory;
    private readonly IDialogService _dialog;
    
    private ObservableCollection<CustomerRowVm> _customers = new();
    private CustomerRowVm? _selected;
    private string _search = "";
    private CustomerStatus? _filterStatus;
    private bool _isLoading;
    private int _currentPage = 1;
    private int _totalPages;
    private const int PageSize = 20;
    
    public ObservableCollection<CustomerRowVm> Customers
    {
        get => _customers;
        private set => SetProperty(ref _customers, value);
    }
    
    public CustomerRowVm? SelectedCustomer
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
            {
                _currentPage = 1;
                _ = LoadAsync(); // fire-and-forget (debounce would be better)
            }
        }
    }
    
    public bool IsLoading
    {
        get => _isLoading;
        set => SetProperty(ref _isLoading, value);
    }
    
    public int CurrentPage
    {
        get => _currentPage;
        set
        {
            if (SetProperty(ref _currentPage, value))
                _ = LoadAsync();
        }
    }
    
    public int TotalPages
    {
        get => _totalPages;
        private set
        {
            SetProperty(ref _totalPages, value);
            OnPropertyChanged(nameof(HasNextPage));
            OnPropertyChanged(nameof(HasPrevPage));
        }
    }
    
    public bool HasNextPage => CurrentPage < TotalPages;
    public bool HasPrevPage => CurrentPage > 1;
    
    // Summary counts
    public int TotalCount { get; private set; }
    public int ActiveCount { get; private set; }
    public int LeadCount { get; private set; }
    
    // Commands
    public AsyncRelayCommand LoadCommand { get; }
    public AsyncRelayCommand AddCommand { get; }
    public AsyncRelayCommand<CustomerRowVm?> EditCommand { get; }
    public AsyncRelayCommand<CustomerRowVm?> DeleteCommand { get; }
    public RelayCommand NextPageCommand { get; }
    public RelayCommand PrevPageCommand { get; }
    public AsyncRelayCommand ExportCommand { get; }
    
    public CustomerListViewModel(IDbContextFactory<CrmDbContext> ctxFactory, IDialogService dialog)
    {
        _ctxFactory = ctxFactory;
        _dialog = dialog;
        
        LoadCommand = new AsyncRelayCommand(LoadAsync);
        AddCommand = new AsyncRelayCommand(AddAsync);
        EditCommand = new AsyncRelayCommand<CustomerRowVm?>(EditAsync);
        DeleteCommand = new AsyncRelayCommand<CustomerRowVm?>(DeleteAsync, c => c != null);
        NextPageCommand = new RelayCommand(() => CurrentPage++, () => HasNextPage);
        PrevPageCommand = new RelayCommand(() => CurrentPage--, () => HasPrevPage);
        ExportCommand = new AsyncRelayCommand(ExportCsvAsync);
    }
    
    public async Task LoadAsync()
    {
        IsLoading = true;
        try
        {
            await using var ctx = await _ctxFactory.CreateDbContextAsync();
            
            var query = ctx.Customers.AsNoTracking().AsQueryable();
            
            if (!string.IsNullOrWhiteSpace(_search))
                query = query.Where(c => 
                    c.FirstName.Contains(_search) || 
                    c.LastName.Contains(_search) ||
                    c.Email.Contains(_search) ||
                    (c.Company != null && c.Company.Contains(_search)));
            
            if (_filterStatus.HasValue)
                query = query.Where(c => c.Status == _filterStatus.Value);
            
            // Count for pagination
            TotalCount = await query.CountAsync();
            TotalPages = (int)Math.Ceiling(TotalCount / (double)PageSize);
            
            var rows = await query
                .OrderBy(c => c.FirstName)
                .Skip((_currentPage - 1) * PageSize)
                .Take(PageSize)
                .Select(c => new CustomerRowVm
                {
                    Id = c.Id,
                    FullName = c.FirstName + " " + c.LastName,
                    Email = c.Email,
                    Phone = c.Phone ?? "",
                    Company = c.Company ?? "",
                    Status = c.Status.ToString(),
                    DealCount = c.Deals.Count,
                    TotalDealValue = c.Deals.Sum(d => d.Value)
                })
                .ToListAsync();
            
            Customers = new ObservableCollection<CustomerRowVm>(rows);
            
            // Summary
            ActiveCount = await ctx.Customers.CountAsync(c => c.Status == CustomerStatus.Active);
            LeadCount = await ctx.Customers.CountAsync(c => c.Status == CustomerStatus.Lead);
            OnPropertyChanged(nameof(TotalCount));
            OnPropertyChanged(nameof(ActiveCount));
            OnPropertyChanged(nameof(LeadCount));
        }
        finally { IsLoading = false; }
    }
    
    private async Task AddAsync()
    {
        var vm = new CustomerEditViewModel();
        if (_dialog.ShowDialog(vm) != true) return;
        
        await using var ctx = await _ctxFactory.CreateDbContextAsync();
        ctx.Customers.Add(vm.ToCustomer());
        await ctx.SaveChangesAsync();
        await LoadAsync();
    }
    
    private async Task EditAsync(CustomerRowVm? row)
    {
        if (row == null) return;
        
        await using var ctx = await _ctxFactory.CreateDbContextAsync();
        var customer = await ctx.Customers.FindAsync(row.Id);
        if (customer == null) return;
        
        var vm = new CustomerEditViewModel(customer);
        if (_dialog.ShowDialog(vm) != true) return;
        
        vm.ApplyTo(customer);
        await ctx.SaveChangesAsync();
        await LoadAsync();
    }
    
    private async Task DeleteAsync(CustomerRowVm? row)
    {
        if (row == null) return;
        if (!_dialog.Confirm($"ลบลูกค้า '{row.FullName}'?")) return;
        
        await using var ctx = await _ctxFactory.CreateDbContextAsync();
        var customer = await ctx.Customers.FindAsync(row.Id);
        if (customer != null)
        {
            ctx.Customers.Remove(customer);
            await ctx.SaveChangesAsync();
        }
        await LoadAsync();
    }
    
    private async Task ExportCsvAsync()
    {
        var path = _dialog.SaveFile("CSV files (*.csv)|*.csv", "customers.csv");
        if (path == null) return;
        
        await using var ctx = await _ctxFactory.CreateDbContextAsync();
        var all = await ctx.Customers.AsNoTracking()
            .Select(c => $"{c.Id},{c.FirstName},{c.LastName},{c.Email},{c.Company},{c.Status}")
            .ToListAsync();
        
        await File.WriteAllLinesAsync(path, all.Prepend("Id,FirstName,LastName,Email,Company,Status"));
        _dialog.ShowMessage($"Export สำเร็จ: {all.Count} รายการ");
    }
}

public class CustomerRowVm
{
    public int Id { get; set; }
    public string FullName { get; set; } = "";
    public string Email { get; set; } = "";
    public string Phone { get; set; } = "";
    public string Company { get; set; } = "";
    public string Status { get; set; } = "";
    public int DealCount { get; set; }
    public decimal TotalDealValue { get; set; }
}
```

---

## ขั้นตอนที่ 444-450: CustomerListView XAML

```xml
<!-- Views/CustomerListView.xaml -->
<UserControl x:Class="CRM.Views.CustomerListView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             Loaded="OnLoaded">
    
    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>
        
        <!-- Summary bar -->
        <WrapPanel Grid.Row="0" Margin="12,8">
            <Border Background="#EBF5FB" CornerRadius="4" Padding="12,6" Margin="0,0,8,0">
                <TextBlock Text="{Binding TotalCount, StringFormat='ทั้งหมด: {0}'}"/>
            </Border>
            <Border Background="#EAFAF1" CornerRadius="4" Padding="12,6" Margin="0,0,8,0">
                <TextBlock Text="{Binding ActiveCount, StringFormat='Active: {0}'}" Foreground="Green"/>
            </Border>
            <Border Background="#FEF9E7" CornerRadius="4" Padding="12,6">
                <TextBlock Text="{Binding LeadCount, StringFormat='Lead: {0}'}" Foreground="Orange"/>
            </Border>
        </WrapPanel>
        
        <!-- Toolbar -->
        <DockPanel Grid.Row="1" Margin="12,0,12,8" LastChildFill="True">
            <Button DockPanel.Dock="Right" Content="📤 Export" Padding="10,5" Margin="5,0,0,0"
                    Command="{Binding ExportCommand}"/>
            <Button DockPanel.Dock="Right" Content="➕ เพิ่ม" Padding="12,5" Margin="5,0,0,0"
                    Background="#3498DB" Foreground="White" Command="{Binding AddCommand}"/>
            <TextBox Text="{Binding SearchText, UpdateSourceTrigger=PropertyChanged}"
                     FontSize="13"/>
        </DockPanel>
        
        <!-- DataGrid with loading overlay -->
        <Grid Grid.Row="2" Margin="12,0">
            <DataGrid ItemsSource="{Binding Customers}"
                      SelectedItem="{Binding SelectedCustomer}"
                      AutoGenerateColumns="False" CanUserAddRows="False"
                      IsReadOnly="True" SelectionMode="Single">
                <DataGrid.Columns>
                    <DataGridTextColumn Header="ชื่อ-สกุล" Binding="{Binding FullName}" Width="160"/>
                    <DataGridTextColumn Header="อีเมล" Binding="{Binding Email}" Width="200"/>
                    <DataGridTextColumn Header="เบอร์โทร" Binding="{Binding Phone}" Width="120"/>
                    <DataGridTextColumn Header="บริษัท" Binding="{Binding Company}" Width="150"/>
                    <DataGridTextColumn Header="สถานะ" Binding="{Binding Status}" Width="90"/>
                    <DataGridTextColumn Header="ดีล" Binding="{Binding DealCount}" Width="60"/>
                    <DataGridTextColumn Header="มูลค่าดีล" 
                                        Binding="{Binding TotalDealValue, StringFormat='฿{0:N0}'}" Width="100"/>
                    <DataGridTemplateColumn Width="120">
                        <DataGridTemplateColumn.CellTemplate>
                            <DataTemplate>
                                <StackPanel Orientation="Horizontal">
                                    <Button Content="✏️" Padding="6,3" Margin="2"
                                            Command="{Binding DataContext.EditCommand, RelativeSource={RelativeSource AncestorType=DataGrid}}"
                                            CommandParameter="{Binding}"/>
                                    <Button Content="🗑️" Padding="6,3" Margin="2"
                                            Command="{Binding DataContext.DeleteCommand, RelativeSource={RelativeSource AncestorType=DataGrid}}"
                                            CommandParameter="{Binding}" Background="#E74C3C" Foreground="White"/>
                                </StackPanel>
                            </DataTemplate>
                        </DataGridTemplateColumn.CellTemplate>
                    </DataGridTemplateColumn>
                </DataGrid.Columns>
            </DataGrid>
            
            <!-- Loading overlay -->
            <Border Background="#80FFFFFF" Visibility="{Binding IsLoading, Converter={StaticResource BoolToVis}}">
                <StackPanel HorizontalAlignment="Center" VerticalAlignment="Center">
                    <TextBlock Text="กำลังโหลด..." FontSize="16"/>
                </StackPanel>
            </Border>
        </Grid>
        
        <!-- Pagination -->
        <StackPanel Grid.Row="3" Orientation="Horizontal" HorizontalAlignment="Center" Margin="0,8">
            <Button Content="◀" Padding="10,5" Command="{Binding PrevPageCommand}"/>
            <TextBlock VerticalAlignment="Center" Margin="12,0"
                       Text="{Binding CurrentPage, StringFormat='หน้า {0}'}"/>
            <TextBlock VerticalAlignment="Center" Margin="0,0,12,0"
                       Text="{Binding TotalPages, StringFormat='/ {0}'}"/>
            <Button Content="▶" Padding="10,5" Command="{Binding NextPageCommand}"/>
        </StackPanel>
    </Grid>
</UserControl>
```

```csharp
// Views/CustomerListView.xaml.cs
public partial class CustomerListView : UserControl
{
    public CustomerListView() => InitializeComponent();
    
    private async void OnLoaded(object sender, RoutedEventArgs e)
    {
        if (DataContext is CustomerListViewModel vm)
            await vm.LoadCommand.ExecuteAsync();
    }
}
```

---

## 📝 สรุป Part 45

| Concept | สิ่งสำคัญ |
|---------|---------|
| IDbContextFactory | สร้าง DbContext ใหม่ทุกครั้ง |
| Async VM | async Task Load() |
| IQueryable + dynamic | Build query at runtime |
| Pagination | Skip + Take + Count |
| Server-side filter | WHERE ใน SQL ไม่ใช่ใน C# |
| Loading overlay | IsLoading binding |

---

**ก่อนหน้า → [Part 44: LINQ Advanced](part44-linq-advanced.md)**  
**ต่อไป → [Part 46: Design Patterns - SOLID](part46-solid-principles.md)**
