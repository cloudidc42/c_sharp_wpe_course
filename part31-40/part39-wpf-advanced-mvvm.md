# Part 39: WPF Advanced MVVM
## ขั้นตอนที่ 381-390: MVVM ขั้นสูงและ Patterns

---

## 🎯 เป้าหมายของ Part นี้
- Messenger/EventAggregator
- DialogService
- Async Commands
- Validation (INotifyDataErrorInfo)
- ViewModelLocator
- Dependency Injection ใน WPF
- Unit Testing ViewModels
- โปรแกรม Contact Management System (MVVM สมบูรณ์)

---

## ขั้นตอนที่ 381: Messenger Pattern

```csharp
// Services/Messenger.cs - loosely-coupled communication
public class Messenger
{
    private static readonly Messenger _instance = new();
    public static Messenger Default => _instance;
    
    private readonly Dictionary<Type, List<WeakReference<Delegate>>> _subscribers = new();
    
    public void Subscribe<TMessage>(Action<TMessage> action)
    {
        var type = typeof(TMessage);
        if (!_subscribers.ContainsKey(type))
            _subscribers[type] = new List<WeakReference<Delegate>>();
        
        _subscribers[type].Add(new WeakReference<Delegate>(action));
    }
    
    public void Unsubscribe<TMessage>(Action<TMessage> action)
    {
        var type = typeof(TMessage);
        if (!_subscribers.TryGetValue(type, out var list)) return;
        list.RemoveAll(wr => !wr.TryGetTarget(out var d) || d == (Delegate)action);
    }
    
    public void Send<TMessage>(TMessage message)
    {
        var type = typeof(TMessage);
        if (!_subscribers.TryGetValue(type, out var list)) return;
        
        var toRemove = new List<WeakReference<Delegate>>();
        foreach (var wr in list.ToList())
        {
            if (wr.TryGetTarget(out var d))
                ((Action<TMessage>)d)(message);
            else
                toRemove.Add(wr);
        }
        foreach (var r in toRemove) list.Remove(r);
    }
}

// Message types
public record ContactSelectedMessage(Contact Contact);
public record ContactDeletedMessage(int ContactId);
public record NavigateMessage(string ViewName, object? Parameter = null);
public record ShowNotificationMessage(string Text, NotificationType Type = NotificationType.Info);
public enum NotificationType { Info, Success, Warning, Error }
```

```csharp
// Sending a message
Messenger.Default.Send(new ContactSelectedMessage(selectedContact));
Messenger.Default.Send(new ShowNotificationMessage("บันทึกสำเร็จ!", NotificationType.Success));

// Receiving in another ViewModel
public class ContactDetailViewModel : ViewModelBase, IDisposable
{
    public ContactDetailViewModel()
    {
        Messenger.Default.Subscribe<ContactSelectedMessage>(OnContactSelected);
    }
    
    private void OnContactSelected(ContactSelectedMessage msg)
    {
        CurrentContact = msg.Contact;
    }
    
    public void Dispose()
    {
        Messenger.Default.Unsubscribe<ContactSelectedMessage>(OnContactSelected);
    }
}
```

---

## ขั้นตอนที่ 382: Async Command

```csharp
// AsyncRelayCommand - supports async operations with busy state
public class AsyncRelayCommand : ICommand
{
    private readonly Func<Task> _execute;
    private readonly Func<bool>? _canExecute;
    private bool _isExecuting;
    
    public AsyncRelayCommand(Func<Task> execute, Func<bool>? canExecute = null)
    {
        _execute = execute;
        _canExecute = canExecute;
    }
    
    public bool IsExecuting
    {
        get => _isExecuting;
        private set
        {
            _isExecuting = value;
            CommandManager.InvalidateRequerySuggested();
        }
    }
    
    public event EventHandler? CanExecuteChanged
    {
        add => CommandManager.RequerySuggested += value;
        remove => CommandManager.RequerySuggested -= value;
    }
    
    public bool CanExecute(object? parameter) => !_isExecuting && (_canExecute?.Invoke() ?? true);
    
    public async void Execute(object? parameter)
    {
        if (!CanExecute(parameter)) return;
        IsExecuting = true;
        try
        {
            await _execute();
        }
        finally
        {
            IsExecuting = false;
        }
    }
}

// Generic version
public class AsyncRelayCommand<T> : ICommand
{
    private readonly Func<T?, Task> _execute;
    private readonly Func<T?, bool>? _canExecute;
    private bool _isExecuting;
    
    public AsyncRelayCommand(Func<T?, Task> execute, Func<T?, bool>? canExecute = null)
    {
        _execute = execute;
        _canExecute = canExecute;
    }
    
    public event EventHandler? CanExecuteChanged
    {
        add => CommandManager.RequerySuggested += value;
        remove => CommandManager.RequerySuggested -= value;
    }
    
    public bool CanExecute(object? parameter) => !_isExecuting && 
        (_canExecute?.Invoke(parameter is T t ? t : default) ?? true);
    
    public async void Execute(object? parameter)
    {
        if (!CanExecute(parameter)) return;
        _isExecuting = true;
        CommandManager.InvalidateRequerySuggested();
        try
        {
            await _execute(parameter is T t ? t : default);
        }
        finally
        {
            _isExecuting = false;
            CommandManager.InvalidateRequerySuggested();
        }
    }
}
```

---

## ขั้นตอนที่ 383: Validation (INotifyDataErrorInfo)

```csharp
// ValidatingViewModelBase.cs
public abstract class ValidatingViewModelBase : ViewModelBase, INotifyDataErrorInfo
{
    private readonly Dictionary<string, List<string>> _errors = new();
    
    public bool HasErrors => _errors.Any(e => e.Value.Count > 0);
    
    public event EventHandler<DataErrorsChangedEventArgs>? ErrorsChanged;
    
    public IEnumerable GetErrors(string? propertyName)
    {
        if (string.IsNullOrEmpty(propertyName))
            return _errors.Values.SelectMany(e => e);
        
        return _errors.TryGetValue(propertyName, out var errors) ? errors : Enumerable.Empty<string>();
    }
    
    protected void SetError(string propertyName, string? error)
    {
        if (error == null)
            ClearError(propertyName);
        else
        {
            if (!_errors.ContainsKey(propertyName)) _errors[propertyName] = new();
            _errors[propertyName] = new List<string> { error };
        }
        ErrorsChanged?.Invoke(this, new DataErrorsChangedEventArgs(propertyName));
        OnPropertyChanged(nameof(HasErrors));
    }
    
    protected void ClearError(string propertyName)
    {
        _errors.Remove(propertyName);
        ErrorsChanged?.Invoke(this, new DataErrorsChangedEventArgs(propertyName));
        OnPropertyChanged(nameof(HasErrors));
    }
    
    protected void ClearAllErrors()
    {
        var keys = _errors.Keys.ToList();
        _errors.Clear();
        foreach (var key in keys)
            ErrorsChanged?.Invoke(this, new DataErrorsChangedEventArgs(key));
        OnPropertyChanged(nameof(HasErrors));
    }
    
    protected void Validate(string propertyName, object? value, params (Func<bool> rule, string message)[] rules)
    {
        foreach (var (rule, message) in rules)
        {
            if (!rule())
            {
                SetError(propertyName, message);
                return;
            }
        }
        ClearError(propertyName);
    }
}

// Contact ViewModel with validation
public class ContactEditViewModel : ValidatingViewModelBase
{
    private string _firstName = "";
    private string _lastName = "";
    private string _email = "";
    private string _phone = "";
    
    public string FirstName
    {
        get => _firstName;
        set
        {
            SetProperty(ref _firstName, value);
            Validate(nameof(FirstName), value,
                (() => !string.IsNullOrWhiteSpace(value), "กรุณากรอกชื่อ"),
                (() => value.Length >= 2, "ชื่อต้องมีอย่างน้อย 2 ตัวอักษร"));
        }
    }
    
    public string LastName
    {
        get => _lastName;
        set
        {
            SetProperty(ref _lastName, value);
            Validate(nameof(LastName), value,
                (() => !string.IsNullOrWhiteSpace(value), "กรุณากรอกนามสกุล"));
        }
    }
    
    public string Email
    {
        get => _email;
        set
        {
            SetProperty(ref _email, value);
            Validate(nameof(Email), value,
                (() => !string.IsNullOrWhiteSpace(value), "กรุณากรอกอีเมล"),
                (() => value.Contains('@') && value.Contains('.'), "รูปแบบอีเมลไม่ถูกต้อง"));
        }
    }
    
    public string Phone
    {
        get => _phone;
        set
        {
            SetProperty(ref _phone, value);
            Validate(nameof(Phone), value,
                (() => string.IsNullOrEmpty(value) || value.All(c => char.IsDigit(c) || c == '-'), 
                 "เบอร์โทรต้องเป็นตัวเลขเท่านั้น"));
        }
    }
    
    public bool IsValid => !HasErrors && !string.IsNullOrWhiteSpace(FirstName);
    
    public ICommand SaveCommand { get; }
    public ICommand CancelCommand { get; }
    
    public ContactEditViewModel()
    {
        SaveCommand = new RelayCommand(Save, () => IsValid);
        CancelCommand = new RelayCommand(Cancel);
    }
    
    private void Save()
    {
        // Validate all
        var contact = new Contact { FirstName = FirstName, LastName = LastName, 
                                    Email = Email, Phone = Phone };
        Messenger.Default.Send(new ContactSavedMessage(contact));
    }
    
    private void Cancel() => Messenger.Default.Send(new NavigateMessage("ContactList"));
}
```

```xml
<!-- XAML with validation error display -->
<TextBox Text="{Binding FirstName, UpdateSourceTrigger=PropertyChanged, ValidatesOnNotifyDataErrors=True}">
    <Validation.ErrorTemplate>
        <ControlTemplate>
            <StackPanel>
                <AdornedElementPlaceholder/>
                <TextBlock Text="{Binding [0].ErrorContent}" Foreground="Red" FontSize="11" Margin="0,2,0,0"/>
            </StackPanel>
        </ControlTemplate>
    </Validation.ErrorTemplate>
</TextBox>

<!-- Style when has error -->
<Style TargetType="TextBox">
    <Style.Triggers>
        <Trigger Property="Validation.HasError" Value="True">
            <Setter Property="BorderBrush" Value="Red"/>
            <Setter Property="ToolTip" Value="{Binding RelativeSource={RelativeSource Self}, Path=(Validation.Errors)[0].ErrorContent}"/>
        </Trigger>
    </Style.Triggers>
</Style>
```

---

## ขั้นตอนที่ 384: DialogService

```csharp
// Services/IDialogService.cs
public interface IDialogService
{
    bool? ShowDialog<TViewModel>(TViewModel viewModel) where TViewModel : ViewModelBase;
    void ShowMessage(string message, string title = "ข้อความ");
    bool Confirm(string message, string title = "ยืนยัน");
    string? OpenFile(string filter = "All files (*.*)|*.*");
    string? SaveFile(string filter = "All files (*.*)|*.*", string defaultName = "");
}

// Services/DialogService.cs
public class DialogService : IDialogService
{
    private readonly Dictionary<Type, Type> _viewModelToView = new();
    
    public void Register<TViewModel, TView>() 
        where TViewModel : ViewModelBase 
        where TView : Window, new()
    {
        _viewModelToView[typeof(TViewModel)] = typeof(TView);
    }
    
    public bool? ShowDialog<TViewModel>(TViewModel viewModel) where TViewModel : ViewModelBase
    {
        if (!_viewModelToView.TryGetValue(typeof(TViewModel), out var viewType))
            throw new InvalidOperationException($"No view registered for {typeof(TViewModel).Name}");
        
        var window = (Window)Activator.CreateInstance(viewType)!;
        window.DataContext = viewModel;
        window.Owner = Application.Current.MainWindow;
        return window.ShowDialog();
    }
    
    public void ShowMessage(string message, string title = "ข้อความ")
        => MessageBox.Show(message, title, MessageBoxButton.OK, MessageBoxImage.Information);
    
    public bool Confirm(string message, string title = "ยืนยัน")
        => MessageBox.Show(message, title, MessageBoxButton.YesNo, MessageBoxImage.Question) == MessageBoxResult.Yes;
    
    public string? OpenFile(string filter = "All files (*.*)|*.*")
    {
        var dlg = new Microsoft.Win32.OpenFileDialog { Filter = filter };
        return dlg.ShowDialog() == true ? dlg.FileName : null;
    }
    
    public string? SaveFile(string filter = "All files (*.*)|*.*", string defaultName = "")
    {
        var dlg = new Microsoft.Win32.SaveFileDialog { Filter = filter, FileName = defaultName };
        return dlg.ShowDialog() == true ? dlg.FileName : null;
    }
}
```

---

## ขั้นตอนที่ 385: Dependency Injection

```csharp
// Setup DI in App.xaml.cs
using Microsoft.Extensions.DependencyInjection;

public partial class App : Application
{
    private ServiceProvider _serviceProvider = null!;
    
    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);
        
        var services = new ServiceCollection();
        
        // Services
        services.AddSingleton<IDialogService, DialogService>();
        services.AddSingleton<Messenger>();
        services.AddSingleton<IContactRepository, ContactRepository>();
        
        // ViewModels
        services.AddTransient<ContactListViewModel>();
        services.AddTransient<ContactEditViewModel>();
        services.AddTransient<SettingsViewModel>();
        
        // Main window
        services.AddSingleton<MainViewModel>();
        services.AddSingleton<MainWindow>();
        
        _serviceProvider = services.BuildServiceProvider();
        
        // Register dialog views
        var dialogService = _serviceProvider.GetRequiredService<IDialogService>() as DialogService;
        dialogService?.Register<ContactEditViewModel, ContactEditDialog>();
        
        var mainWindow = _serviceProvider.GetRequiredService<MainWindow>();
        mainWindow.DataContext = _serviceProvider.GetRequiredService<MainViewModel>();
        mainWindow.Show();
    }
    
    protected override void OnExit(ExitEventArgs e)
    {
        _serviceProvider.Dispose();
        base.OnExit(e);
    }
}
```

---

## ขั้นตอนที่ 386-390: Unit Testing ViewModels

```csharp
// Tests/ContactListViewModelTests.cs
using Xunit;
using Moq;
using FluentAssertions;

public class ContactListViewModelTests
{
    private readonly Mock<IContactRepository> _repoMock;
    private readonly Mock<IDialogService> _dialogMock;
    private readonly Messenger _messenger;
    
    public ContactListViewModelTests()
    {
        _repoMock = new Mock<IContactRepository>();
        _dialogMock = new Mock<IDialogService>();
        _messenger = new Messenger();
        
        // Setup test data
        _repoMock.Setup(r => r.GetAllAsync()).ReturnsAsync(new List<Contact>
        {
            new() { Id=1, FirstName="สมชาย", LastName="ใจดี", Email="somchai@test.com" },
            new() { Id=2, FirstName="สมหญิง", LastName="รักเรียน", Email="somying@test.com" },
        });
    }
    
    [Fact]
    public async Task LoadContacts_ShouldPopulateContacts()
    {
        // Arrange
        var vm = new ContactListViewModel(_repoMock.Object, _dialogMock.Object, _messenger);
        
        // Act
        await vm.LoadContactsAsync();
        
        // Assert
        vm.Contacts.Should().HaveCount(2);
        vm.Contacts.First().FirstName.Should().Be("สมชาย");
    }
    
    [Fact]
    public async Task DeleteContact_ShouldRemoveFromList()
    {
        // Arrange
        var vm = new ContactListViewModel(_repoMock.Object, _dialogMock.Object, _messenger);
        await vm.LoadContactsAsync();
        _dialogMock.Setup(d => d.Confirm(It.IsAny<string>(), It.IsAny<string>())).Returns(true);
        _repoMock.Setup(r => r.DeleteAsync(1)).Returns(Task.CompletedTask);
        
        var toDelete = vm.Contacts.First(c => c.Id == 1);
        
        // Act
        vm.DeleteCommand.Execute(toDelete);
        
        // Assert
        vm.Contacts.Should().HaveCount(1);
        vm.Contacts.Should().NotContain(c => c.Id == 1);
    }
    
    [Fact]
    public void SearchContacts_ShouldFilterByName()
    {
        // Arrange
        var vm = new ContactListViewModel(_repoMock.Object, _dialogMock.Object, _messenger);
        vm.Contacts.Add(new Contact { Id=1, FirstName="สมชาย" });
        vm.Contacts.Add(new Contact { Id=2, FirstName="วิชัย" });
        
        // Act
        vm.SearchText = "สม";
        
        // Assert
        vm.FilteredContacts.Should().HaveCount(1);
        vm.FilteredContacts.First().FirstName.Should().Be("สมชาย");
    }
    
    [Fact]
    public void PropertyChanged_ShouldFireWhenSearchTextChanges()
    {
        // Arrange
        var vm = new ContactListViewModel(_repoMock.Object, _dialogMock.Object, _messenger);
        var raised = new List<string?>();
        vm.PropertyChanged += (_, e) => raised.Add(e.PropertyName);
        
        // Act
        vm.SearchText = "test";
        
        // Assert
        raised.Should().Contain(nameof(vm.SearchText));
        raised.Should().Contain(nameof(vm.FilteredContacts));
    }
}
```

---

## 📝 สรุป Part 39

| Pattern | ประโยชน์ |
|---------|---------|
| Messenger | Loosely-coupled ViewModel communication |
| AsyncRelayCommand | Async operations with busy state |
| INotifyDataErrorInfo | Validation ใน MVVM |
| DialogService | ViewModel เปิด dialog ได้โดยไม่รู้จัก View |
| DI Container | Manage dependencies, testability |
| Unit Tests | Test ViewModel logic ได้โดยไม่ต้องมี UI |

---

**ก่อนหน้า → [Part 38: WPF Resources](part38-wpf-resources.md)**  
**ต่อไป → [Part 40: WPF Final Project](part40-wpf-final.md)**
