# Part 33: WPF MVVM Pattern
## ขั้นตอนที่ 321-330: Model-View-ViewModel

---

## 🎯 เป้าหมายของ Part นี้
- MVVM Architecture overview
- ICommand และ RelayCommand
- Commands ใน ViewModel
- EventToCommand (รับ event ใน VM)
- Messenger/EventAggregator pattern
- Dependency Injection พื้นฐาน
- Navigation ระหว่าง View
- โปรแกรม Task Manager (MVVM เต็มรูปแบบ)

---

## ขั้นตอนที่ 321: MVVM Architecture

```
┌─────────────────────────────────────────────────────┐
│                     MVVM Pattern                     │
│                                                     │
│  ┌──────────┐    Binding    ┌──────────────────┐   │
│  │   VIEW   │◄─────────────►│   VIEWMODEL      │   │
│  │  (XAML)  │    Commands   │  (INotifyProp..) │   │
│  └──────────┘               └────────┬─────────┘   │
│                                      │              │
│                                      │ calls        │
│                              ┌───────▼────────┐    │
│                              │     MODEL      │    │
│                              │  (Data/Logic)  │    │
│                              └────────────────┘    │
└─────────────────────────────────────────────────────┘

View:       รับผิดชอบ UI เท่านั้น ไม่มี business logic
ViewModel:  state + commands ของ view, ไม่รู้จัก UI
Model:      data + business rules, ไม่รู้จัก UI
```

---

## ขั้นตอนที่ 322: ICommand และ RelayCommand

```csharp
// ICommand interface
// Execute(object? parameter)
// CanExecute(object? parameter) → bool
// CanExecuteChanged event

// RelayCommand - implementation ทั่วไป
public class RelayCommand : ICommand
{
    private readonly Action<object?> _execute;
    private readonly Func<object?, bool>? _canExecute;
    
    public RelayCommand(Action<object?> execute, Func<object?, bool>? canExecute = null)
    {
        _execute = execute ?? throw new ArgumentNullException(nameof(execute));
        _canExecute = canExecute;
    }
    
    // Shortcut constructor for no-parameter actions
    public RelayCommand(Action execute, Func<bool>? canExecute = null)
        : this(_ => execute(), canExecute == null ? null : _ => canExecute()) { }
    
    public event EventHandler? CanExecuteChanged
    {
        add => CommandManager.RequerySuggested += value;
        remove => CommandManager.RequerySuggested -= value;
    }
    
    public bool CanExecute(object? parameter) => _canExecute == null || _canExecute(parameter);
    public void Execute(object? parameter) => _execute(parameter);
    
    // Call this to manually refresh CanExecute
    public static void RaiseCanExecuteChanged() => CommandManager.InvalidateRequerySuggested();
}

// Generic version for typed parameters
public class RelayCommand<T> : ICommand
{
    private readonly Action<T?> _execute;
    private readonly Func<T?, bool>? _canExecute;
    
    public RelayCommand(Action<T?> execute, Func<T?, bool>? canExecute = null)
    {
        _execute = execute;
        _canExecute = canExecute;
    }
    
    public event EventHandler? CanExecuteChanged
    {
        add => CommandManager.RequerySuggested += value;
        remove => CommandManager.RequerySuggested -= value;
    }
    
    public bool CanExecute(object? parameter)
    {
        if (_canExecute == null) return true;
        return parameter is T t ? _canExecute(t) : _canExecute(default);
    }
    
    public void Execute(object? parameter)
    {
        if (parameter is T t) _execute(t);
        else _execute(default);
    }
}
```

---

## ขั้นตอนที่ 323: Task Model

```csharp
// Models/TaskItem.cs
public enum TaskPriority { Low, Medium, High, Critical }
public enum TaskStatus { Todo, InProgress, Done, Cancelled }

public class TaskItem : ViewModelBase
{
    private string _title = "";
    private string _description = "";
    private TaskPriority _priority = TaskPriority.Medium;
    private TaskStatus _status = TaskStatus.Todo;
    private DateTime _createdAt = DateTime.Now;
    private DateTime? _dueDate;
    private bool _isSelected;
    
    public int Id { get; set; }
    
    public string Title
    {
        get => _title;
        set => SetProperty(ref _title, value);
    }
    
    public string Description
    {
        get => _description;
        set => SetProperty(ref _description, value);
    }
    
    public TaskPriority Priority
    {
        get => _priority;
        set => SetProperty(ref _priority, value);
    }
    
    public TaskStatus Status
    {
        get => _status;
        set
        {
            if (SetProperty(ref _status, value))
                OnPropertyChanged(nameof(IsCompleted));
        }
    }
    
    public DateTime CreatedAt
    {
        get => _createdAt;
        set => SetProperty(ref _createdAt, value);
    }
    
    public DateTime? DueDate
    {
        get => _dueDate;
        set
        {
            if (SetProperty(ref _dueDate, value))
                OnPropertyChanged(nameof(IsOverdue));
        }
    }
    
    public bool IsSelected
    {
        get => _isSelected;
        set => SetProperty(ref _isSelected, value);
    }
    
    public bool IsCompleted => Status == TaskStatus.Done;
    public bool IsOverdue => DueDate.HasValue && DueDate.Value < DateTime.Now && Status != TaskStatus.Done;
    
    public string PriorityLabel => Priority switch
    {
        TaskPriority.Critical => "🔴 วิกฤต",
        TaskPriority.High => "🟠 สูง",
        TaskPriority.Medium => "🟡 กลาง",
        TaskPriority.Low => "🟢 ต่ำ",
        _ => ""
    };
    
    public string StatusLabel => Status switch
    {
        TaskStatus.Todo => "📋 รอดำเนินการ",
        TaskStatus.InProgress => "⚡ กำลังทำ",
        TaskStatus.Done => "✅ เสร็จแล้ว",
        TaskStatus.Cancelled => "❌ ยกเลิก",
        _ => ""
    };
}
```

---

## ขั้นตอนที่ 324: TaskListViewModel

```csharp
// ViewModels/TaskListViewModel.cs
using System.Collections.ObjectModel;
using System.Windows.Input;

public class TaskListViewModel : ViewModelBase
{
    private ObservableCollection<TaskItem> _tasks = new();
    private ObservableCollection<TaskItem> _filteredTasks = new();
    private TaskItem? _selectedTask;
    private string _searchText = "";
    private TaskStatus? _filterStatus;
    private string _statusMessage = "พร้อมใช้งาน";
    private int _nextId = 1;
    
    public TaskListViewModel()
    {
        LoadSampleData();
        
        // Commands
        AddTaskCommand = new RelayCommand(AddTask);
        DeleteTaskCommand = new RelayCommand<TaskItem>(DeleteTask);
        ToggleStatusCommand = new RelayCommand<TaskItem>(ToggleStatus);
        ClearCompletedCommand = new RelayCommand(ClearCompleted, () => Tasks.Any(t => t.IsCompleted));
        FilterCommand = new RelayCommand<string?>(ApplyFilter);
    }
    
    public ObservableCollection<TaskItem> Tasks
    {
        get => _tasks;
        private set => SetProperty(ref _tasks, value);
    }
    
    public ObservableCollection<TaskItem> FilteredTasks
    {
        get => _filteredTasks;
        private set => SetProperty(ref _filteredTasks, value);
    }
    
    public TaskItem? SelectedTask
    {
        get => _selectedTask;
        set => SetProperty(ref _selectedTask, value);
    }
    
    public string SearchText
    {
        get => _searchText;
        set
        {
            if (SetProperty(ref _searchText, value))
                RefreshFilter();
        }
    }
    
    public string StatusMessage
    {
        get => _statusMessage;
        set => SetProperty(ref _statusMessage, value);
    }
    
    // Summary
    public int TotalCount => Tasks.Count;
    public int DoneCount => Tasks.Count(t => t.Status == TaskStatus.Done);
    public int InProgressCount => Tasks.Count(t => t.Status == TaskStatus.InProgress);
    public int TodoCount => Tasks.Count(t => t.Status == TaskStatus.Todo);
    
    // Commands
    public ICommand AddTaskCommand { get; }
    public ICommand DeleteTaskCommand { get; }
    public ICommand ToggleStatusCommand { get; }
    public ICommand ClearCompletedCommand { get; }
    public ICommand FilterCommand { get; }
    
    private void AddTask()
    {
        var task = new TaskItem
        {
            Id = _nextId++,
            Title = $"Task ใหม่ #{_nextId - 1}",
            Priority = TaskPriority.Medium,
            Status = TaskStatus.Todo
        };
        Tasks.Add(task);
        SelectedTask = task;
        RefreshFilter();
        UpdateCounts();
        StatusMessage = $"เพิ่ม '{task.Title}' แล้ว";
    }
    
    private void DeleteTask(TaskItem? task)
    {
        if (task == null) return;
        Tasks.Remove(task);
        if (SelectedTask == task) SelectedTask = null;
        RefreshFilter();
        UpdateCounts();
        StatusMessage = $"ลบ '{task.Title}' แล้ว";
    }
    
    private void ToggleStatus(TaskItem? task)
    {
        if (task == null) return;
        task.Status = task.Status == TaskStatus.Done ? TaskStatus.Todo : TaskStatus.Done;
        UpdateCounts();
    }
    
    private void ClearCompleted()
    {
        var completed = Tasks.Where(t => t.IsCompleted).ToList();
        foreach (var t in completed) Tasks.Remove(t);
        RefreshFilter();
        UpdateCounts();
        StatusMessage = $"ลบ {completed.Count} งานที่เสร็จแล้ว";
    }
    
    private void ApplyFilter(string? status)
    {
        _filterStatus = status == null ? null : Enum.Parse<TaskStatus>(status);
        RefreshFilter();
    }
    
    private void RefreshFilter()
    {
        var query = Tasks.AsEnumerable();
        
        if (!string.IsNullOrWhiteSpace(_searchText))
            query = query.Where(t => t.Title.Contains(_searchText, StringComparison.OrdinalIgnoreCase));
        
        if (_filterStatus.HasValue)
            query = query.Where(t => t.Status == _filterStatus.Value);
        
        FilteredTasks = new ObservableCollection<TaskItem>(query.OrderByDescending(t => t.Priority));
    }
    
    private void UpdateCounts()
    {
        OnPropertyChanged(nameof(TotalCount));
        OnPropertyChanged(nameof(DoneCount));
        OnPropertyChanged(nameof(InProgressCount));
        OnPropertyChanged(nameof(TodoCount));
    }
    
    private void LoadSampleData()
    {
        var samples = new[]
        {
            new TaskItem { Id=_nextId++, Title="ทำรายงานประจำเดือน", Priority=TaskPriority.High, Status=TaskStatus.InProgress },
            new TaskItem { Id=_nextId++, Title="ประชุมทีม", Priority=TaskPriority.Critical, Status=TaskStatus.Todo, DueDate=DateTime.Now.AddDays(1) },
            new TaskItem { Id=_nextId++, Title="อัพเดท documentation", Priority=TaskPriority.Medium, Status=TaskStatus.Done },
            new TaskItem { Id=_nextId++, Title="Code review PR #42", Priority=TaskPriority.High, Status=TaskStatus.Todo },
            new TaskItem { Id=_nextId++, Title="ตอบอีเมล client", Priority=TaskPriority.Medium, Status=TaskStatus.Todo },
        };
        foreach (var t in samples) Tasks.Add(t);
        RefreshFilter();
        UpdateCounts();
    }
}
```

---

## ขั้นตอนที่ 325: XAML สำหรับ Task Manager

```xml
<!-- TaskManager.xaml -->
<Window x:Class="TaskManager.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:local="clr-namespace:TaskManager"
        Title="Task Manager (MVVM)" Height="600" Width="850"
        WindowStartupLocation="CenterScreen">
    <Window.Resources>
        <local:BoolToVisibilityConverter x:Key="BoolToVis"/>
        <local:TaskPriorityColorConverter x:Key="PriorityColor"/>
        
        <!-- Task row style -->
        <Style x:Key="TaskRowStyle" TargetType="ListViewItem">
            <Setter Property="Padding" Value="5"/>
            <Style.Triggers>
                <DataTrigger Binding="{Binding IsCompleted}" Value="True">
                    <Setter Property="Opacity" Value="0.5"/>
                </DataTrigger>
                <DataTrigger Binding="{Binding IsOverdue}" Value="True">
                    <Setter Property="Background" Value="#FFEAEA"/>
                </DataTrigger>
            </Style.Triggers>
        </Style>
    </Window.Resources>
    
    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>
        
        <!-- Header -->
        <Border Grid.Row="0" Background="#34495E" Padding="15,10">
            <Grid>
                <TextBlock Text="📋 Task Manager" FontSize="18" FontWeight="Bold" Foreground="White"/>
                <StackPanel HorizontalAlignment="Right" Orientation="Horizontal">
                    <Border Background="#3498DB" CornerRadius="10" Padding="8,3" Margin="5,0">
                        <TextBlock Foreground="White" FontSize="11"
                                   Text="{Binding InProgressCount, StringFormat='⚡ {0}'}"/>
                    </Border>
                    <Border Background="#E74C3C" CornerRadius="10" Padding="8,3" Margin="5,0">
                        <TextBlock Foreground="White" FontSize="11"
                                   Text="{Binding TodoCount, StringFormat='📋 {0}'}"/>
                    </Border>
                    <Border Background="#27AE60" CornerRadius="10" Padding="8,3" Margin="5,0">
                        <TextBlock Foreground="White" FontSize="11"
                                   Text="{Binding DoneCount, StringFormat='✅ {0}'}"/>
                    </Border>
                </StackPanel>
            </Grid>
        </Border>
        
        <!-- Search + Add -->
        <DockPanel Grid.Row="1" Margin="10,8" LastChildFill="True">
            <Button DockPanel.Dock="Right" Content="➕ เพิ่มงาน" Padding="12,5" Margin="5,0,0,0"
                    Command="{Binding AddTaskCommand}" Background="#27AE60" Foreground="White"/>
            <Button DockPanel.Dock="Right" Content="🗑️ ลบที่เสร็จ" Padding="12,5" Margin="5,0,0,0"
                    Command="{Binding ClearCompletedCommand}" Background="#E74C3C" Foreground="White"/>
            <TextBox Text="{Binding SearchText, UpdateSourceTrigger=PropertyChanged}"
                     FontSize="13"/>
        </DockPanel>
        
        <!-- Filter buttons -->
        <StackPanel Grid.Row="2" Orientation="Horizontal" Margin="10,0,10,5">
            <Button Content="ทั้งหมด" Padding="10,4" Margin="0,0,5,0"
                    Command="{Binding FilterCommand}" CommandParameter="{x:Null}"/>
            <Button Content="📋 รอดำเนินการ" Padding="10,4" Margin="0,0,5,0"
                    Command="{Binding FilterCommand}" CommandParameter="Todo"/>
            <Button Content="⚡ กำลังทำ" Padding="10,4" Margin="0,0,5,0"
                    Command="{Binding FilterCommand}" CommandParameter="InProgress"/>
            <Button Content="✅ เสร็จแล้ว" Padding="10,4"
                    Command="{Binding FilterCommand}" CommandParameter="Done"/>
        </StackPanel>
        
        <!-- Task List -->
        <ListView Grid.Row="3" Margin="10,0" 
                  ItemsSource="{Binding FilteredTasks}"
                  SelectedItem="{Binding SelectedTask}"
                  ItemContainerStyle="{StaticResource TaskRowStyle}">
            <ListView.View>
                <GridView>
                    <GridViewColumn Width="30">
                        <GridViewColumn.CellTemplate>
                            <DataTemplate>
                                <CheckBox IsChecked="{Binding IsCompleted}"
                                          Command="{Binding DataContext.ToggleStatusCommand, RelativeSource={RelativeSource AncestorType=ListView}}"
                                          CommandParameter="{Binding}"/>
                            </DataTemplate>
                        </GridViewColumn.CellTemplate>
                    </GridViewColumn>
                    <GridViewColumn Header="ชื่องาน" Width="250">
                        <GridViewColumn.CellTemplate>
                            <DataTemplate>
                                <TextBlock Text="{Binding Title}">
                                    <TextBlock.Style>
                                        <Style TargetType="TextBlock">
                                            <Style.Triggers>
                                                <DataTrigger Binding="{Binding IsCompleted}" Value="True">
                                                    <Setter Property="TextDecorations" Value="Strikethrough"/>
                                                </DataTrigger>
                                            </Style.Triggers>
                                        </Style>
                                    </TextBlock.Style>
                                </TextBlock>
                            </DataTemplate>
                        </GridViewColumn.CellTemplate>
                    </GridViewColumn>
                    <GridViewColumn Header="ความสำคัญ" Width="100" DisplayMemberBinding="{Binding PriorityLabel}"/>
                    <GridViewColumn Header="สถานะ" Width="130" DisplayMemberBinding="{Binding StatusLabel}"/>
                    <GridViewColumn Header="ครบกำหนด" Width="110"
                                    DisplayMemberBinding="{Binding DueDate, StringFormat=dd/MM/yyyy}"/>
                    <GridViewColumn Header="" Width="60">
                        <GridViewColumn.CellTemplate>
                            <DataTemplate>
                                <Button Content="🗑️" Padding="4,2"
                                        Command="{Binding DataContext.DeleteTaskCommand, RelativeSource={RelativeSource AncestorType=ListView}}"
                                        CommandParameter="{Binding}"
                                        Background="Transparent" BorderThickness="0"/>
                            </DataTemplate>
                        </GridViewColumn.CellTemplate>
                    </GridViewColumn>
                </GridView>
            </ListView.View>
        </ListView>
        
        <!-- Status bar -->
        <StatusBar Grid.Row="4">
            <StatusBarItem Content="{Binding StatusMessage}"/>
            <Separator/>
            <StatusBarItem Content="{Binding TotalCount, StringFormat='รวม {0} งาน'}"/>
        </StatusBar>
    </Grid>
</Window>
```

```csharp
// TaskManager.xaml.cs - minimal code-behind!
using System.Windows;

namespace TaskManager;

public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();
        DataContext = new TaskListViewModel();
    }
}
```

---

## ขั้นตอนที่ 326-330: Task Priority Color Converter

```csharp
// TaskPriorityColorConverter.cs
public class TaskPriorityColorConverter : IValueConverter
{
    public object Convert(object value, Type targetType, object parameter, CultureInfo culture)
    {
        if (value is TaskPriority p)
        {
            return p switch
            {
                TaskPriority.Critical => new SolidColorBrush(Color.FromRgb(192, 57, 43)),
                TaskPriority.High => new SolidColorBrush(Color.FromRgb(211, 84, 0)),
                TaskPriority.Medium => new SolidColorBrush(Color.FromRgb(39, 174, 96)),
                TaskPriority.Low => new SolidColorBrush(Color.FromRgb(52, 152, 219)),
                _ => Brushes.Gray
            };
        }
        return Brushes.Gray;
    }
    
    public object ConvertBack(object value, Type targetType, object parameter, CultureInfo culture)
        => throw new NotImplementedException();
}
```

---

## 📝 สรุป Part 33

| Concept | ใช้งาน |
|---------|--------|
| MVVM | Model-View-ViewModel architecture |
| RelayCommand | ICommand implementation พร้อมใช้ |
| Commands in XAML | `Command="{Binding AddCommand}"` |
| CommandParameter | ส่ง parameter ไปกับ Command |
| DataTrigger | เปลี่ยน style ตาม data |
| RelativeSource | Bind กับ parent element |

---

**ก่อนหน้า → [Part 32: WPF Data Binding](part32-wpf-databinding.md)**  
**ต่อไป → [Part 34: WPF Styles & Templates](part34-wpf-styles.md)**
