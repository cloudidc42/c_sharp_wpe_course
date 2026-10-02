# Part 36: WPF Navigation
## ขั้นตอนที่ 351-360: Single Page Application Navigation

---

## 🎯 เป้าหมาของ Part นี้
- ContentControl + DataTemplate navigation
- Frame และ Page navigation
- View switching ใน MVVM
- NavigationService
- Tab-based navigation
- ViewLocator pattern
- Transition animations ระหว่าง pages
- โปรแกรม Multi-page App with Sidebar

---

## ขั้นตอนที่ 351: ContentControl Navigation

```csharp
// ViewModel-based navigation: แสดง ViewModel → DataTemplate → View
public class NavigationViewModel : ViewModelBase
{
    private object? _currentView;
    
    public object? CurrentView
    {
        get => _currentView;
        set => SetProperty(ref _currentView, value);
    }
    
    // Commands
    public ICommand GoToHomeCommand { get; }
    public ICommand GoToUsersCommand { get; }
    public ICommand GoToSettingsCommand { get; }
    
    public NavigationViewModel()
    {
        GoToHomeCommand = new RelayCommand(() => CurrentView = new HomeViewModel());
        GoToUsersCommand = new RelayCommand(() => CurrentView = new UsersViewModel());
        GoToSettingsCommand = new RelayCommand(() => CurrentView = new SettingsViewModel());
        
        // Start on Home
        CurrentView = new HomeViewModel();
    }
}
```

```xml
<!-- MainWindow.xaml - ContentControl as router -->
<Window.Resources>
    <!-- Map each ViewModel to a View (DataTemplate) -->
    <DataTemplate DataType="{x:Type vm:HomeViewModel}">
        <views:HomeView/>
    </DataTemplate>
    <DataTemplate DataType="{x:Type vm:UsersViewModel}">
        <views:UsersView/>
    </DataTemplate>
    <DataTemplate DataType="{x:Type vm:SettingsViewModel}">
        <views:SettingsView/>
    </DataTemplate>
</Window.Resources>

<Grid>
    <Grid.ColumnDefinitions>
        <ColumnDefinition Width="200"/>
        <ColumnDefinition Width="*"/>
    </Grid.ColumnDefinitions>
    
    <!-- Sidebar navigation -->
    <StackPanel Grid.Column="0" Background="#2C3E50">
        <Button Content="🏠 หน้าแรก" Command="{Binding GoToHomeCommand}"/>
        <Button Content="👥 ผู้ใช้งาน" Command="{Binding GoToUsersCommand}"/>
        <Button Content="⚙️ ตั้งค่า" Command="{Binding GoToSettingsCommand}"/>
    </StackPanel>
    
    <!-- Content area - auto-resolves DataTemplate -->
    <ContentControl Grid.Column="1" Content="{Binding CurrentView}"/>
</Grid>
```

---

## ขั้นตอนที่ 352: NavigationService

```csharp
// Services/NavigationService.cs
public interface INavigationService
{
    object? CurrentView { get; }
    void NavigateTo<TViewModel>() where TViewModel : ViewModelBase;
    void NavigateTo(ViewModelBase viewModel);
    void GoBack();
    bool CanGoBack { get; }
    event EventHandler? Navigated;
}

public class NavigationService : INavigationService, INotifyPropertyChanged
{
    private readonly Func<Type, ViewModelBase> _viewModelFactory;
    private readonly Stack<ViewModelBase> _history = new();
    private object? _currentView;
    
    public event PropertyChangedEventHandler? PropertyChanged;
    public event EventHandler? Navigated;
    
    public NavigationService(Func<Type, ViewModelBase> viewModelFactory)
    {
        _viewModelFactory = viewModelFactory;
    }
    
    public object? CurrentView
    {
        get => _currentView;
        private set
        {
            _currentView = value;
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(CurrentView)));
        }
    }
    
    public bool CanGoBack => _history.Count > 1;
    
    public void NavigateTo<TViewModel>() where TViewModel : ViewModelBase
    {
        var vm = _viewModelFactory(typeof(TViewModel));
        NavigateTo(vm);
    }
    
    public void NavigateTo(ViewModelBase viewModel)
    {
        _history.Push(viewModel);
        CurrentView = viewModel;
        Navigated?.Invoke(this, EventArgs.Empty);
    }
    
    public void GoBack()
    {
        if (!CanGoBack) return;
        _history.Pop(); // Remove current
        CurrentView = _history.Peek();
        Navigated?.Invoke(this, EventArgs.Empty);
    }
}
```

---

## ขั้นตอนที่ 353: ActiveTab Navigation

```csharp
// Tab-based navigation ViewModel
public class AppViewModel : ViewModelBase
{
    private readonly NavigationService _nav;
    private NavItem? _selectedNavItem;
    
    public ObservableCollection<NavItem> NavItems { get; } = new();
    
    public NavItem? SelectedNavItem
    {
        get => _selectedNavItem;
        set
        {
            if (SetProperty(ref _selectedNavItem, value) && value != null)
                _nav.NavigateTo(value.CreateViewModel());
        }
    }
    
    public object? CurrentView => _nav.CurrentView;
    
    public AppViewModel()
    {
        _nav = new NavigationService(type => (ViewModelBase)Activator.CreateInstance(type)!);
        _nav.Navigated += (_, _) => OnPropertyChanged(nameof(CurrentView));
        
        NavItems.Add(new NavItem("🏠", "Dashboard", typeof(DashboardViewModel)));
        NavItems.Add(new NavItem("📦", "สินค้า", typeof(ProductsViewModel)));
        NavItems.Add(new NavItem("👥", "ลูกค้า", typeof(CustomersViewModel)));
        NavItems.Add(new NavItem("📊", "รายงาน", typeof(ReportsViewModel)));
        NavItems.Add(new NavItem("⚙️", "ตั้งค่า", typeof(SettingsViewModel)));
        
        SelectedNavItem = NavItems.First();
    }
}

public class NavItem
{
    public string Icon { get; }
    public string Title { get; }
    private readonly Type _viewModelType;
    
    public NavItem(string icon, string title, Type viewModelType)
    {
        Icon = icon;
        Title = title;
        _viewModelType = viewModelType;
    }
    
    public ViewModelBase CreateViewModel() 
        => (ViewModelBase)Activator.CreateInstance(_viewModelType)!;
}
```

---

## ขั้นตอนที่ 354: Animated Page Transitions

```csharp
// AnimatedContentControl.cs
using System.Windows;
using System.Windows.Controls;
using System.Windows.Media.Animation;

public class AnimatedContentControl : ContentControl
{
    protected override void OnContentChanged(object oldContent, object newContent)
    {
        base.OnContentChanged(oldContent, newContent);
        PlayTransitionAnimation();
    }
    
    private void PlayTransitionAnimation()
    {
        if (Template.FindName("PART_Content", this) is not FrameworkElement content) return;
        
        var sb = new Storyboard();
        
        // Fade in
        var fade = new DoubleAnimation(0, 1, TimeSpan.FromMilliseconds(250));
        fade.EasingFunction = new QuarticEase { EasingMode = EasingMode.EaseOut };
        Storyboard.SetTarget(fade, content);
        Storyboard.SetTargetProperty(fade, new PropertyPath(OpacityProperty));
        
        // Slide from right
        var slide = new DoubleAnimation(20, 0, TimeSpan.FromMilliseconds(300));
        slide.EasingFunction = new CubicEase { EasingMode = EasingMode.EaseOut };
        Storyboard.SetTarget(slide, content);
        Storyboard.SetTargetProperty(slide, 
            new PropertyPath("(UIElement.RenderTransform).(TranslateTransform.X)"));
        
        sb.Children.Add(fade);
        sb.Children.Add(slide);
        sb.Begin();
    }
}
```

```xml
<!-- AnimatedContentControl template -->
<Style TargetType="local:AnimatedContentControl">
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="local:AnimatedContentControl">
                <ContentPresenter x:Name="PART_Content">
                    <ContentPresenter.RenderTransform>
                        <TranslateTransform/>
                    </ContentPresenter.RenderTransform>
                </ContentPresenter>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
</Style>
```

---

## ขั้นตอนที่ 355-360: Multi-page App with Sidebar

```xml
<!-- App.xaml.cs -->
```csharp
// App.xaml.cs
using System.Windows;
namespace MultiPageApp;

public partial class App : Application
{
    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);
        var mainWindow = new MainWindow();
        mainWindow.DataContext = new AppViewModel();
        mainWindow.Show();
    }
}
```

```xml
<!-- MainWindow.xaml -->
<Window x:Class="MultiPageApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:vm="clr-namespace:MultiPageApp.ViewModels"
        xmlns:views="clr-namespace:MultiPageApp.Views"
        xmlns:local="clr-namespace:MultiPageApp"
        Title="Multi Page App" Height="600" Width="950"
        WindowStartupLocation="CenterScreen">
    
    <Window.Resources>
        <!-- ViewModel → View mapping -->
        <DataTemplate DataType="{x:Type vm:DashboardViewModel}">
            <views:DashboardView/>
        </DataTemplate>
        <DataTemplate DataType="{x:Type vm:ProductsViewModel}">
            <views:ProductsView/>
        </DataTemplate>
        <DataTemplate DataType="{x:Type vm:CustomersViewModel}">
            <views:CustomersView/>
        </DataTemplate>
        <DataTemplate DataType="{x:Type vm:ReportsViewModel}">
            <views:ReportsView/>
        </DataTemplate>
        <DataTemplate DataType="{x:Type vm:SettingsViewModel}">
            <views:SettingsView/>
        </DataTemplate>
        
        <!-- Sidebar nav button style -->
        <Style x:Key="NavBtn" TargetType="RadioButton">
            <Setter Property="GroupName" Value="NavGroup"/>
            <Setter Property="Height" Value="48"/>
            <Setter Property="Foreground" Value="#95A5A6"/>
            <Setter Property="FontSize" Value="13"/>
            <Setter Property="Cursor" Value="Hand"/>
            <Setter Property="Template">
                <Setter.Value>
                    <ControlTemplate TargetType="RadioButton">
                        <Border x:Name="bd" Background="Transparent" Padding="16,0">
                            <StackPanel Orientation="Horizontal" VerticalAlignment="Center">
                                <TextBlock Text="{Binding Icon}" FontSize="18" Width="28"/>
                                <TextBlock Text="{Binding Title}" Margin="8,0,0,0" VerticalAlignment="Center"/>
                            </StackPanel>
                        </Border>
                        <ControlTemplate.Triggers>
                            <Trigger Property="IsChecked" Value="True">
                                <Setter TargetName="bd" Property="Background" Value="#3D566E"/>
                                <Setter Property="Foreground" Value="White"/>
                            </Trigger>
                            <Trigger Property="IsMouseOver" Value="True">
                                <Setter TargetName="bd" Property="Background" Value="#354A5E"/>
                            </Trigger>
                        </ControlTemplate.Triggers>
                    </ControlTemplate>
                </Setter.Value>
            </Setter>
        </Style>
    </Window.Resources>
    
    <Grid>
        <Grid.ColumnDefinitions>
            <ColumnDefinition Width="200"/>
            <ColumnDefinition Width="*"/>
        </Grid.ColumnDefinitions>
        
        <!-- Sidebar -->
        <Grid Grid.Column="0" Background="#2C3E50">
            <Grid.RowDefinitions>
                <RowDefinition Height="Auto"/>
                <RowDefinition Height="*"/>
                <RowDefinition Height="Auto"/>
            </Grid.RowDefinitions>
            
            <!-- App logo -->
            <StackPanel Grid.Row="0" Margin="16,20">
                <TextBlock Text="⚡ MyApp" FontSize="18" FontWeight="Bold" Foreground="White"/>
                <TextBlock Text="Management System" FontSize="10" Foreground="#7F8C8D"/>
            </StackPanel>
            
            <!-- Nav items -->
            <ItemsControl Grid.Row="1" ItemsSource="{Binding NavItems}">
                <ItemsControl.ItemTemplate>
                    <DataTemplate>
                        <RadioButton Style="{StaticResource NavBtn}"
                                     IsChecked="{Binding IsSelected, Mode=TwoWay}"
                                     Command="{Binding DataContext.SelectNavCommand, RelativeSource={RelativeSource AncestorType=Window}}"
                                     CommandParameter="{Binding}"/>
                    </DataTemplate>
                </ItemsControl.ItemTemplate>
            </ItemsControl>
            
            <!-- User info at bottom -->
            <Border Grid.Row="2" BorderBrush="#3D566E" BorderThickness="0,1,0,0" Padding="12,10">
                <StackPanel Orientation="Horizontal">
                    <Ellipse Width="32" Height="32" Fill="#3498DB"/>
                    <StackPanel Margin="8,0,0,0">
                        <TextBlock Text="Admin User" Foreground="White" FontSize="12"/>
                        <TextBlock Text="admin@example.com" Foreground="#7F8C8D" FontSize="10"/>
                    </StackPanel>
                </StackPanel>
            </Border>
        </Grid>
        
        <!-- Content area -->
        <Grid Grid.Column="1">
            <Grid.RowDefinitions>
                <RowDefinition Height="Auto"/>
                <RowDefinition Height="*"/>
            </Grid.RowDefinitions>
            
            <!-- Top bar -->
            <Border Grid.Row="0" Background="White" BorderBrush="#E0E0E0" BorderThickness="0,0,0,1" Padding="20,12">
                <TextBlock Text="{Binding SelectedNavItem.Title}" FontSize="18" FontWeight="SemiBold"/>
            </Border>
            
            <!-- Page content with animation -->
            <ContentControl Grid.Row="1" Margin="20" Content="{Binding CurrentView}"/>
        </Grid>
    </Grid>
</Window>
```

```csharp
// ViewModels/AppViewModel.cs
public class AppViewModel : ViewModelBase
{
    private NavItem? _selectedNavItem;
    private object? _currentView;
    
    public ObservableCollection<NavItem> NavItems { get; } = new();
    
    public NavItem? SelectedNavItem
    {
        get => _selectedNavItem;
        set
        {
            if (SetProperty(ref _selectedNavItem, value) && value != null)
            {
                foreach (var item in NavItems) item.IsSelected = false;
                value.IsSelected = true;
                CurrentView = value.CreateViewModel();
            }
        }
    }
    
    public object? CurrentView
    {
        get => _currentView;
        set => SetProperty(ref _currentView, value);
    }
    
    public ICommand SelectNavCommand { get; }
    
    public AppViewModel()
    {
        SelectNavCommand = new RelayCommand<NavItem?>(item => { if (item != null) SelectedNavItem = item; });
        
        NavItems.Add(new NavItem("🏠", "Dashboard", () => new DashboardViewModel()));
        NavItems.Add(new NavItem("📦", "สินค้า", () => new ProductsViewModel()));
        NavItems.Add(new NavItem("👥", "ลูกค้า", () => new CustomersViewModel()));
        NavItems.Add(new NavItem("📊", "รายงาน", () => new ReportsViewModel()));
        NavItems.Add(new NavItem("⚙️", "ตั้งค่า", () => new SettingsViewModel()));
        
        SelectedNavItem = NavItems.First();
    }
}

public class NavItem : ViewModelBase
{
    private bool _isSelected;
    private readonly Func<ViewModelBase> _factory;
    
    public string Icon { get; }
    public string Title { get; }
    
    public bool IsSelected
    {
        get => _isSelected;
        set => SetProperty(ref _isSelected, value);
    }
    
    public NavItem(string icon, string title, Func<ViewModelBase> factory)
    {
        Icon = icon;
        Title = title;
        _factory = factory;
    }
    
    public ViewModelBase CreateViewModel() => _factory();
}

// ViewModels/DashboardViewModel.cs
public class DashboardViewModel : ViewModelBase
{
    public string WelcomeMessage => $"สวัสดี! วันนี้ {DateTime.Now:dddd, dd MMMM yyyy}";
    public decimal TodayRevenue { get; } = 85_400;
    public int ActiveOrders { get; } = 23;
}

// ViewModels/ProductsViewModel.cs
public class ProductsViewModel : ViewModelBase
{
    public ObservableCollection<string> Products { get; } = new()
    {
        "iPhone 15 Pro", "MacBook Air M3", "iPad Pro", "AirPods Pro"
    };
}

// Views/DashboardView.xaml.cs (simple UserControl)
```

---

## 📝 สรุป Part 36

| Pattern | คำอธิบาย |
|---------|----------|
| ContentControl | แสดง ViewModel โดยใช้ DataTemplate |
| DataTemplate per Type | WPF resolves View อัตโนมัติ |
| NavigationService | Stack-based history |
| AnimatedContentControl | Override OnContentChanged |
| RadioButton NavGroup | Active state ใน sidebar |

---

**ก่อนหน้า → [Part 35: WPF Animation](part35-wpf-animation.md)**  
**ต่อไป → [Part 37: WPF Custom Controls](part37-wpf-custom-controls.md)**
