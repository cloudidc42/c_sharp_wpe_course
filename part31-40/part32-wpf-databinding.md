# Part 32: WPF Data Binding
## ขั้นตอนที่ 311-320: Data Binding ขั้นสูง

---

## 🎯 เป้าหมายของ Part นี้
- {Binding} expression
- INotifyPropertyChanged
- ObservableCollection<T>
- DataContext
- Binding Modes: OneWay, TwoWay, OneTime
- Value Converters
- MultiBinding
- RelativeSource Binding
- โปรแกรม Student Grade Manager

---

## ขั้นตอนที่ 311: INotifyPropertyChanged

```csharp
// ViewModel base class
using System.ComponentModel;
using System.Runtime.CompilerServices;

public abstract class ViewModelBase : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler? PropertyChanged;
    
    protected virtual void OnPropertyChanged([CallerMemberName] string? propertyName = null)
        => PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    
    protected bool SetProperty<T>(ref T field, T value, [CallerMemberName] string? propertyName = null)
    {
        if (EqualityComparer<T>.Default.Equals(field, value)) return false;
        field = value;
        OnPropertyChanged(propertyName);
        return true;
    }
}

// Student model
public class Student : ViewModelBase
{
    private string _name = "";
    private int _age;
    private double _grade;
    private string _status = "";
    
    public int Id { get; set; }
    
    public string Name
    {
        get => _name;
        set
        {
            if (SetProperty(ref _name, value))
                OnPropertyChanged(nameof(DisplayName));
        }
    }
    
    public int Age
    {
        get => _age;
        set => SetProperty(ref _age, value);
    }
    
    public double Grade
    {
        get => _grade;
        set
        {
            if (SetProperty(ref _grade, value))
            {
                OnPropertyChanged(nameof(GradeDisplay));
                OnPropertyChanged(nameof(GradeLevel));
                Status = GetStatus(value);
            }
        }
    }
    
    public string Status
    {
        get => _status;
        set => SetProperty(ref _status, value);
    }
    
    // Computed properties - no setter
    public string DisplayName => string.IsNullOrEmpty(Name) ? "(ไม่ระบุชื่อ)" : Name;
    public string GradeDisplay => $"{Grade:F1}%";
    public string GradeLevel => Grade switch
    {
        >= 90 => "A",
        >= 80 => "B",
        >= 70 => "C",
        >= 60 => "D",
        _ => "F"
    };
    
    private static string GetStatus(double grade) => grade >= 60 ? "ผ่าน ✓" : "ไม่ผ่าน ✗";
}
```

---

## ขั้นตอนที่ 312: Binding Expressions

```xml
<!-- Binding syntax -->

<!-- OneWay: source → UI (read-only display) -->
<TextBlock Text="{Binding Name, Mode=OneWay}"/>

<!-- TwoWay: UI ↔ source (default for TextBox) -->
<TextBox Text="{Binding Name, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged}"/>

<!-- OneTime: set once at startup -->
<TextBlock Text="{Binding CreatedDate, Mode=OneTime}"/>

<!-- Default modes by control:
     TextBox.Text = TwoWay
     TextBlock.Text = OneWay
     CheckBox.IsChecked = TwoWay -->

<!-- Binding to nested property -->
<TextBlock Text="{Binding Address.City}"/>

<!-- StringFormat -->
<TextBlock Text="{Binding Price, StringFormat=ราคา: ฿{0:N0}}"/>
<TextBlock Text="{Binding Grade, StringFormat={}{0:F1}%}"/>
<TextBlock Text="{Binding BirthDate, StringFormat=dd/MM/yyyy}"/>

<!-- FallbackValue and TargetNullValue -->
<TextBlock Text="{Binding MiddleName, TargetNullValue='ไม่มีชื่อกลาง'}"/>
<TextBlock Text="{Binding Status, FallbackValue='กำลังโหลด...'}"/>

<!-- Element binding: bind to another control -->
<Slider x:Name="sldOpacity" Minimum="0" Maximum="1" Value="1"/>
<Rectangle Opacity="{Binding ElementName=sldOpacity, Path=Value}" Fill="Blue" Height="50"/>

<!-- Self binding (RelativeSource) -->
<TextBlock Text="{Binding RelativeSource={RelativeSource Self}, Path=ActualWidth, StringFormat='Width: {0:F0}'}"/>

<!-- Parent binding -->
<TextBlock Text="{Binding RelativeSource={RelativeSource AncestorType=Window}, Path=Title}"/>

<!-- DataContext from code-behind -->
<!-- this.DataContext = new StudentViewModel(); -->
```

---

## ขั้นตอนที่ 313: Value Converters

```csharp
// IValueConverter สำหรับแปลงค่า
using System.Globalization;
using System.Windows.Data;
using System.Windows.Media;

// Bool to Visibility
[ValueConversion(typeof(bool), typeof(Visibility))]
public class BoolToVisibilityConverter : IValueConverter
{
    public object Convert(object value, Type targetType, object parameter, CultureInfo culture)
    {
        bool boolVal = value is bool b && b;
        bool inverse = parameter as string == "inverse";
        return (boolVal ^ inverse) ? Visibility.Visible : Visibility.Collapsed;
    }
    
    public object ConvertBack(object value, Type targetType, object parameter, CultureInfo culture)
        => value is Visibility v && v == Visibility.Visible;
}

// Grade to color
public class GradeToColorConverter : IValueConverter
{
    public object Convert(object value, Type targetType, object parameter, CultureInfo culture)
    {
        if (value is double grade)
        {
            return grade switch
            {
                >= 90 => new SolidColorBrush(Color.FromRgb(39, 174, 96)),   // Green
                >= 70 => new SolidColorBrush(Color.FromRgb(52, 152, 219)),  // Blue
                >= 60 => new SolidColorBrush(Color.FromRgb(243, 156, 18)), // Orange
                _ => new SolidColorBrush(Color.FromRgb(231, 76, 60))        // Red
            };
        }
        return Brushes.Gray;
    }
    
    public object ConvertBack(object value, Type targetType, object parameter, CultureInfo culture)
        => throw new NotImplementedException();
}

// String to bool (IsNullOrEmpty check)
public class StringToBoolConverter : IValueConverter
{
    public object Convert(object value, Type targetType, object parameter, CultureInfo culture)
        => !string.IsNullOrWhiteSpace(value as string);
    
    public object ConvertBack(object value, Type targetType, object parameter, CultureInfo culture)
        => throw new NotImplementedException();
}
```

```xml
<!-- Using converters in XAML -->
<Window.Resources>
    <local:BoolToVisibilityConverter x:Key="BoolToVis"/>
    <local:GradeToColorConverter x:Key="GradeToColor"/>
    <local:StringToBoolConverter x:Key="StrToBool"/>
</Window.Resources>

<!-- Hide/Show -->
<StackPanel Visibility="{Binding IsAdmin, Converter={StaticResource BoolToVis}}">
    <Button Content="Admin Only Button"/>
</StackPanel>

<!-- Color by grade -->
<TextBlock Text="{Binding Grade, StringFormat={}{0:F1}}"
           Foreground="{Binding Grade, Converter={StaticResource GradeToColor}}"/>

<!-- Enable button only when text is filled -->
<Button Content="Submit"
        IsEnabled="{Binding Name, Converter={StaticResource StrToBool}}"/>

<!-- Inverse visibility -->
<TextBlock Text="No items" 
           Visibility="{Binding HasItems, Converter={StaticResource BoolToVis}, ConverterParameter=inverse}"/>
```

---

## ขั้นตอนที่ 314: ObservableCollection

```csharp
// ObservableCollection<T> แจ้ง UI เมื่อ Add/Remove/Clear
using System.Collections.ObjectModel;

public class ClassroomViewModel : ViewModelBase
{
    private ObservableCollection<Student> _students = new();
    private Student? _selectedStudent;
    private string _searchText = "";
    
    public ObservableCollection<Student> Students
    {
        get => _students;
        set => SetProperty(ref _students, value);
    }
    
    public Student? SelectedStudent
    {
        get => _selectedStudent;
        set => SetProperty(ref _selectedStudent, value);
    }
    
    public string SearchText
    {
        get => _searchText;
        set
        {
            if (SetProperty(ref _searchText, value))
                FilterStudents();
        }
    }
    
    // Filtered view
    private List<Student> _allStudents = new();
    
    public void LoadStudents()
    {
        _allStudents = new List<Student>
        {
            new() { Id=1, Name="สมชาย ใจดี", Age=20, Grade=85.5 },
            new() { Id=2, Name="สมหญิง รักเรียน", Age=19, Grade=92.0 },
            new() { Id=3, Name="วิชัย เก่งมาก", Age=21, Grade=55.0 },
            new() { Id=4, Name="มาลี ขยัน", Age=20, Grade=78.5 },
            new() { Id=5, Name="ประยุทธ์ ตั้งใจ", Age=22, Grade=68.0 },
        };
        
        FilterStudents();
    }
    
    private void FilterStudents()
    {
        var filtered = string.IsNullOrWhiteSpace(_searchText)
            ? _allStudents
            : _allStudents.Where(s => s.Name.Contains(_searchText, StringComparison.OrdinalIgnoreCase));
        
        Students = new ObservableCollection<Student>(filtered);
    }
    
    public void AddStudent(Student student)
    {
        student.Id = _allStudents.Count > 0 ? _allStudents.Max(s => s.Id) + 1 : 1;
        _allStudents.Add(student);
        FilterStudents();
    }
    
    public void RemoveStudent(Student student)
    {
        _allStudents.Remove(student);
        FilterStudents();
    }
    
    // Summary
    public double AverageGrade => _allStudents.Count > 0 ? _allStudents.Average(s => s.Grade) : 0;
    public int PassCount => _allStudents.Count(s => s.Grade >= 60);
    public int FailCount => _allStudents.Count(s => s.Grade < 60);
}
```

---

## ขั้นตอนที่ 315-320: Student Grade Manager

```xml
<!-- GradeManager.xaml -->
<Window x:Class="GradeManager.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:local="clr-namespace:GradeManager"
        Title="Student Grade Manager" 
        Height="650" Width="900"
        WindowStartupLocation="CenterScreen">
    
    <Window.Resources>
        <local:GradeToColorConverter x:Key="GradeToColor"/>
        <local:BoolToVisibilityConverter x:Key="BoolToVis"/>
        
        <!-- Grade level style -->
        <Style x:Key="GradeLevelStyle" TargetType="TextBlock">
            <Setter Property="FontWeight" Value="Bold"/>
            <Setter Property="FontSize" Value="14"/>
            <Setter Property="Foreground" Value="{Binding Grade, Converter={StaticResource GradeToColor}}"/>
        </Style>
    </Window.Resources>
    
    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>
        
        <!-- Header -->
        <Border Grid.Row="0" Background="#2C3E50" Padding="15,10">
            <StackPanel Orientation="Horizontal">
                <TextBlock Text="🎓 Student Grade Manager" FontSize="18" FontWeight="Bold" Foreground="White"/>
                <TextBlock Margin="20,0,0,0" Foreground="Silver" FontSize="13"
                           Text="{Binding AverageGrade, StringFormat='ค่าเฉลี่ยชั้น: {0:F1}%'}"/>
            </StackPanel>
        </Border>
        
        <!-- Toolbar -->
        <StackPanel Grid.Row="1" Orientation="Horizontal" Margin="12,8" Spacing="8">
            <TextBox Width="250" x:Name="txtSearch" 
                     Text="{Binding SearchText, UpdateSourceTrigger=PropertyChanged}"
                     FontSize="13"/>
            <Button Content="➕ เพิ่มนักเรียน" Padding="12,5" Click="BtnAdd_Click"
                    Background="#27AE60" Foreground="White"/>
            <Button Content="🗑️ ลบ" Padding="12,5" Click="BtnDelete_Click"
                    Background="#E74C3C" Foreground="White"
                    IsEnabled="{Binding SelectedStudent, Converter={StaticResource GradeToColor}}"/>
        </StackPanel>
        
        <!-- Stats -->
        <WrapPanel Grid.Row="1" HorizontalAlignment="Right" Margin="12,10" Orientation="Horizontal">
            <Border Background="#EBF5FB" CornerRadius="4" Padding="10,5" Margin="5,0">
                <TextBlock Text="{Binding Students.Count, StringFormat='รวม: {0} คน'}" FontSize="12"/>
            </Border>
            <Border Background="#EAFAF1" CornerRadius="4" Padding="10,5" Margin="5,0">
                <TextBlock Text="{Binding PassCount, StringFormat='ผ่าน: {0} คน'}" FontSize="12" Foreground="Green"/>
            </Border>
            <Border Background="#FDEDEC" CornerRadius="4" Padding="10,5" Margin="5,0">
                <TextBlock Text="{Binding FailCount, StringFormat='ไม่ผ่าน: {0} คน'}" FontSize="12" Foreground="Red"/>
            </Border>
        </WrapPanel>
        
        <!-- Student List -->
        <Grid Grid.Row="2" Margin="12,0,12,0">
            <Grid.ColumnDefinitions>
                <ColumnDefinition Width="*"/>
                <ColumnDefinition Width="280"/>
            </Grid.ColumnDefinitions>
            
            <!-- ListView -->
            <ListView ItemsSource="{Binding Students}" 
                      SelectedItem="{Binding SelectedStudent}"
                      BorderThickness="1" BorderBrush="#DDD">
                <ListView.View>
                    <GridView>
                        <GridViewColumn Header="ID" Width="50" DisplayMemberBinding="{Binding Id}"/>
                        <GridViewColumn Header="ชื่อ-สกุล" Width="180" DisplayMemberBinding="{Binding Name}"/>
                        <GridViewColumn Header="อายุ" Width="60" DisplayMemberBinding="{Binding Age}"/>
                        <GridViewColumn Header="คะแนน" Width="90">
                            <GridViewColumn.CellTemplate>
                                <DataTemplate>
                                    <TextBlock Text="{Binding GradeDisplay}"
                                               Foreground="{Binding Grade, Converter={StaticResource GradeToColor}}"
                                               FontWeight="Bold"/>
                                </DataTemplate>
                            </GridViewColumn.CellTemplate>
                        </GridViewColumn>
                        <GridViewColumn Header="เกรด" Width="60">
                            <GridViewColumn.CellTemplate>
                                <DataTemplate>
                                    <TextBlock Text="{Binding GradeLevel}" Style="{StaticResource GradeLevelStyle}"/>
                                </DataTemplate>
                            </GridViewColumn.CellTemplate>
                        </GridViewColumn>
                        <GridViewColumn Header="สถานะ" Width="90" DisplayMemberBinding="{Binding Status}"/>
                    </GridView>
                </ListView.View>
            </ListView>
            
            <!-- Detail Panel -->
            <Border Grid.Column="1" Margin="12,0,0,0" BorderThickness="1" BorderBrush="#DDD" CornerRadius="4" Padding="15">
                <StackPanel>
                    <TextBlock Text="รายละเอียด" FontSize="14" FontWeight="Bold" Margin="0,0,0,12"/>
                    
                    <Visibility Visibility="{Binding SelectedStudent, Converter={StaticResource BoolToVis}}">
                    </Visibility>
                    
                    <StackPanel DataContext="{Binding SelectedStudent}">
                        <TextBlock Text="ชื่อ:" FontSize="11" Foreground="Gray"/>
                        <TextBlock Text="{Binding Name}" FontSize="16" FontWeight="Bold" Margin="0,2,0,10"/>
                        
                        <TextBlock Text="คะแนน:" FontSize="11" Foreground="Gray"/>
                        <TextBlock Text="{Binding GradeDisplay}" FontSize="32" FontWeight="Bold"
                                   Foreground="{Binding Grade, Converter={StaticResource GradeToColor}}"
                                   Margin="0,2,0,5"/>
                        
                        <TextBlock Text="{Binding GradeLevel, StringFormat='เกรด: {0}'}" FontSize="18" FontWeight="Bold"
                                   Foreground="{Binding Grade, Converter={StaticResource GradeToColor}}"
                                   Margin="0,0,0,10"/>
                        
                        <Slider x:Name="sldGrade" Minimum="0" Maximum="100"
                                Value="{Binding Grade}" TickFrequency="10"/>
                        
                        <TextBlock Text="{Binding Status}" FontSize="14" Margin="0,8,0,0"/>
                    </StackPanel>
                </StackPanel>
            </Border>
        </Grid>
        
        <!-- Status bar -->
        <StatusBar Grid.Row="3">
            <StatusBarItem Content="{Binding AverageGrade, StringFormat='ค่าเฉลี่ย: {0:F2}%'}"/>
        </StatusBar>
    </Grid>
</Window>
```

```csharp
// GradeManager.xaml.cs
using System.Windows;

namespace GradeManager;

public partial class MainWindow : Window
{
    private readonly ClassroomViewModel _vm;
    
    public MainWindow()
    {
        InitializeComponent();
        _vm = new ClassroomViewModel();
        _vm.LoadStudents();
        DataContext = _vm;
    }
    
    private void BtnAdd_Click(object sender, RoutedEventArgs e)
    {
        var dlg = new AddStudentDialog { Owner = this };
        if (dlg.ShowDialog() == true && dlg.Result != null)
        {
            _vm.AddStudent(dlg.Result);
        }
    }
    
    private void BtnDelete_Click(object sender, RoutedEventArgs e)
    {
        if (_vm.SelectedStudent == null) return;
        var result = MessageBox.Show($"ลบ '{_vm.SelectedStudent.Name}'?", "ยืนยัน",
            MessageBoxButton.YesNo, MessageBoxImage.Question);
        if (result == MessageBoxResult.Yes)
            _vm.RemoveStudent(_vm.SelectedStudent);
    }
}
```

---

## 📝 สรุป Part 32

| Concept | ใช้งาน |
|---------|--------|
| INotifyPropertyChanged | แจ้ง UI เมื่อ property เปลี่ยน |
| ObservableCollection<T> | List ที่แจ้ง UI เมื่อ Add/Remove |
| {Binding} | เชื่อมต่อ UI กับ ViewModel |
| IValueConverter | แปลงค่าก่อนแสดงผล |
| DataContext | Root object สำหรับ Binding |
| UpdateSourceTrigger | เมื่อไหร่จะ update source |

---

**ก่อนหน้า → [Part 31: WPF Intro](part31-wpf-intro.md)**  
**ต่อไป → [Part 33: WPF MVVM Pattern](part33-wpf-mvvm.md)**
