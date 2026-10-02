# Part 31: WPF Introduction
## ขั้นตอนที่ 301-310: เริ่มต้น Windows Presentation Foundation

---

## 🎯 เป้าหมายของ Part นี้
- WPF vs WinForms
- XAML syntax
- Window, Grid, StackPanel, DockPanel
- Controls พื้นฐาน
- Code-behind
- Property และ Dependency Property
- โปรแกรม Hello WPF

---

## ขั้นตอนที่ 301: WPF Project Setup

```xml
<!-- MyWpfApp.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net8.0-windows</TargetFramework>
    <Nullable>enable</Nullable>
    <UseWPF>true</UseWPF>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
</Project>
```

```xml
<!-- App.xaml -->
<Application x:Class="MyWpfApp.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             StartupUri="MainWindow.xaml">
    <Application.Resources>
        <!-- Global styles and resources here -->
    </Application.Resources>
</Application>
```

---

## ขั้นตอนที่ 302: XAML ขั้นพื้นฐาน

```xml
<!-- MainWindow.xaml -->
<Window x:Class="MyWpfApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="My First WPF App" 
        Height="450" Width="600"
        WindowStartupLocation="CenterScreen"
        MinHeight="300" MinWidth="400">
    
    <!-- Root layout: Grid -->
    <Grid>
        <!-- Define rows -->
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>     <!-- Header: takes needed space -->
            <RowDefinition Height="*"/>         <!-- Content: fills remaining -->
            <RowDefinition Height="Auto"/>     <!-- Footer -->
        </Grid.RowDefinitions>
        
        <!-- Row 0: Header -->
        <Border Grid.Row="0" Background="#2C3E50" Padding="15,10">
            <TextBlock Text="Welcome to WPF" 
                       FontSize="22" FontWeight="Bold"
                       Foreground="White" 
                       HorizontalAlignment="Center"/>
        </Border>
        
        <!-- Row 1: Content -->
        <StackPanel Grid.Row="1" Margin="20" Spacing="10">
            <TextBlock Text="กรอกชื่อของคุณ:" FontSize="14" Margin="0,0,0,5"/>
            
            <TextBox x:Name="txtName"
                     FontSize="14" Padding="8"
                     PlaceholderText="ชื่อของคุณ..."
                     BorderBrush="#3498DB" BorderThickness="2"/>
            
            <Button x:Name="btnGreet"
                    Content="👋 ทักทาย"
                    FontSize="14" Padding="15,8"
                    Background="#3498DB" Foreground="White"
                    BorderThickness="0"
                    HorizontalAlignment="Left"
                    Click="BtnGreet_Click"/>
            
            <TextBlock x:Name="lblResult"
                       FontSize="16" FontWeight="Bold"
                       Foreground="#27AE60"
                       TextWrapping="Wrap"/>
        </StackPanel>
        
        <!-- Row 2: Footer -->
        <Border Grid.Row="2" Background="#ECF0F1" Padding="10">
            <TextBlock Text="WPF Demo Application" 
                       Foreground="Gray" FontSize="11"
                       HorizontalAlignment="Center"/>
        </Border>
    </Grid>
</Window>
```

```csharp
// MainWindow.xaml.cs - Code-behind
using System.Windows;

namespace MyWpfApp;

public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();
    }
    
    private void BtnGreet_Click(object sender, RoutedEventArgs e)
    {
        string name = txtName.Text.Trim();
        
        if (string.IsNullOrEmpty(name))
        {
            MessageBox.Show("กรุณากรอกชื่อก่อน", "แจ้งเตือน",
                MessageBoxButton.OK, MessageBoxImage.Warning);
            txtName.Focus();
            return;
        }
        
        lblResult.Text = $"สวัสดี {name}! ยินดีต้อนรับสู่ WPF 🎉";
    }
}
```

---

## ขั้นตอนที่ 303: Layout Panels

```xml
<!-- Grid: แบ่งเป็น rows และ columns -->
<Grid>
    <Grid.RowDefinitions>
        <RowDefinition Height="50"/>    <!-- Fixed 50px -->
        <RowDefinition Height="2*"/>    <!-- 2/3 of remaining -->
        <RowDefinition Height="*"/>     <!-- 1/3 of remaining -->
    </Grid.RowDefinitions>
    
    <Grid.ColumnDefinitions>
        <ColumnDefinition Width="200"/>  <!-- Fixed 200px -->
        <ColumnDefinition Width="*"/>    <!-- Fill rest -->
        <ColumnDefinition Width="Auto"/> <!-- Auto size -->
    </Grid.ColumnDefinitions>
    
    <!-- Grid.Row and Grid.Column are attached properties -->
    <Button Grid.Row="0" Grid.Column="0" Content="Top-Left"/>
    <Button Grid.Row="1" Grid.Column="1" Grid.ColumnSpan="2" Content="Spanning 2 columns"/>
    <TextBox Grid.Row="2" Grid.Column="0" Grid.RowSpan="2" Text="Spanning 2 rows"/>
</Grid>

<!-- StackPanel: เรียงแนวตั้งหรือแนวนอน -->
<StackPanel Orientation="Vertical" Spacing="8">
    <Button Content="Button 1"/>
    <Button Content="Button 2"/>
    <Button Content="Button 3"/>
</StackPanel>

<StackPanel Orientation="Horizontal" Spacing="8">
    <Button Content="Left" Width="80"/>
    <Button Content="Center" Width="80"/>
    <Button Content="Right" Width="80"/>
</StackPanel>

<!-- WrapPanel: Wrap ไปบรรทัดถัดไป -->
<WrapPanel ItemWidth="100" ItemHeight="40">
    <Button Content="A"/>
    <Button Content="B"/>
    <Button Content="C"/>
    <!-- Will wrap when full -->
</WrapPanel>

<!-- DockPanel: dock ไปแต่ละด้าน -->
<DockPanel>
    <Menu DockPanel.Dock="Top">
        <MenuItem Header="_File"/>
    </Menu>
    <StatusBar DockPanel.Dock="Bottom">
        <StatusBarItem Content="Ready"/>
    </StatusBar>
    <TreeView DockPanel.Dock="Left" Width="200"/>
    <TextBox/>  <!-- Last child fills remaining -->
</DockPanel>

<!-- UniformGrid: ตารางขนาดเท่ากัน -->
<UniformGrid Rows="2" Columns="3">
    <Button Content="1"/>
    <Button Content="2"/>
    <Button Content="3"/>
    <Button Content="4"/>
    <Button Content="5"/>
    <Button Content="6"/>
</UniformGrid>
```

---

## ขั้นตอนที่ 304: XAML Controls

```xml
<!-- TextBlock vs TextBox -->
<TextBlock Text="Read-only text" FontSize="14" FontWeight="Bold" Foreground="Navy"/>
<TextBox Text="{Binding Name}" FontSize="14" Padding="5"/>
<RichTextBox>
    <FlowDocument>
        <Paragraph><Bold>Bold text</Bold> and normal text</Paragraph>
    </FlowDocument>
</RichTextBox>

<!-- Button variants -->
<Button Content="Normal Button" Click="OnClick"/>
<ToggleButton Content="Toggle Me" IsChecked="False"/>
<RepeatButton Content="Hold me" Delay="500" Interval="100" Click="OnRepeat"/>
<CheckBox Content="Check me" IsChecked="True"/>
<RadioButton GroupName="lang" Content="Thai" IsChecked="True"/>
<RadioButton GroupName="lang" Content="English"/>

<!-- ListBox, ComboBox -->
<ListBox SelectionMode="Multiple">
    <ListBoxItem Content="Item 1"/>
    <ListBoxItem Content="Item 2"/>
    <ListBoxItem Content="Item 3"/>
</ListBox>

<ComboBox SelectedIndex="0">
    <ComboBoxItem Content="Option A"/>
    <ComboBoxItem Content="Option B"/>
</ComboBox>

<!-- Slider, ProgressBar -->
<Slider Minimum="0" Maximum="100" Value="50" TickFrequency="10" IsSnapToTickEnabled="True"/>
<ProgressBar Minimum="0" Maximum="100" Value="65" Height="20"/>

<!-- Image -->
<Image Source="logo.png" Width="100" Height="100" Stretch="Uniform"/>

<!-- Border -->
<Border Background="LightBlue" CornerRadius="8" Padding="15" BorderBrush="SteelBlue" BorderThickness="2">
    <TextBlock Text="Rounded box"/>
</Border>
```

---

## ขั้นตอนที่ 305: Dependency Properties

```csharp
// Dependency Property เป็น property system ของ WPF
// ทำให้รองรับ: Binding, Animation, Styling, Inheritance

// Custom control with DependencyProperty
public class CircleControl : Control
{
    // Register dependency property
    public static readonly DependencyProperty RadiusProperty = 
        DependencyProperty.Register(
            nameof(Radius),
            typeof(double),
            typeof(CircleControl),
            new PropertyMetadata(50.0, OnRadiusChanged));
    
    public static readonly DependencyProperty FillColorProperty = 
        DependencyProperty.Register(
            nameof(FillColor),
            typeof(Brush),
            typeof(CircleControl),
            new PropertyMetadata(Brushes.Blue));
    
    // CLR wrapper (thin - no logic in getter/setter!)
    public double Radius
    {
        get => (double)GetValue(RadiusProperty);
        set => SetValue(RadiusProperty, value);
    }
    
    public Brush FillColor
    {
        get => (Brush)GetValue(FillColorProperty);
        set => SetValue(FillColorProperty, value);
    }
    
    private static void OnRadiusChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        if (d is CircleControl cc)
        {
            cc.Width = (double)e.NewValue * 2;
            cc.Height = (double)e.NewValue * 2;
        }
    }
    
    protected override void OnRender(DrawingContext dc)
    {
        dc.DrawEllipse(FillColor, null, new System.Windows.Point(Radius, Radius), Radius, Radius);
    }
}
```

---

## ขั้นตอนที่ 306-310: โปรแกรม Calculator WPF

```xml
<!-- Calculator.xaml -->
<Window x:Class="Calculator.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Calculator" 
        Height="480" Width="320"
        ResizeMode="CanMinimize"
        WindowStartupLocation="CenterScreen"
        Background="#1E1E2E">
    
    <Grid Margin="12">
        <Grid.RowDefinitions>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>
        
        <!-- Display -->
        <Border Grid.Row="0" Background="#2A2A3E" CornerRadius="8" Padding="15,10" Margin="0,0,0,12">
            <StackPanel>
                <TextBlock x:Name="lblExpression" 
                           Text="" Foreground="#888" FontSize="14"
                           HorizontalAlignment="Right" TextTrimming="CharacterEllipsis"/>
                <TextBlock x:Name="lblDisplay" 
                           Text="0" Foreground="White" FontSize="42" FontWeight="Light"
                           HorizontalAlignment="Right" TextTrimming="CharacterEllipsis"
                           Margin="0,5,0,0"/>
            </StackPanel>
        </Border>
        
        <!-- Buttons Grid -->
        <UniformGrid Grid.Row="1" Rows="5" Columns="4" >
            <UniformGrid.Resources>
                <Style TargetType="Button">
                    <Setter Property="Margin" Value="4"/>
                    <Setter Property="FontSize" Value="18"/>
                    <Setter Property="FontWeight" Value="Medium"/>
                    <Setter Property="BorderThickness" Value="0"/>
                    <Setter Property="Cursor" Value="Hand"/>
                    <Setter Property="Height" Value="60"/>
                </Style>
            </UniformGrid.Resources>
            
            <!-- Row 1: C, ±, %, ÷ -->
            <Button Content="C" Background="#3D3D5C" Foreground="#FF6B6B" Tag="C"/>
            <Button Content="±" Background="#3D3D5C" Foreground="#CCC" Tag="plusminus"/>
            <Button Content="%" Background="#3D3D5C" Foreground="#CCC" Tag="%"/>
            <Button Content="÷" Background="#FF9F43" Foreground="White" Tag="/"/>
            
            <!-- Row 2: 7,8,9,× -->
            <Button Content="7" Background="#2D2D42" Foreground="White" Tag="7"/>
            <Button Content="8" Background="#2D2D42" Foreground="White" Tag="8"/>
            <Button Content="9" Background="#2D2D42" Foreground="White" Tag="9"/>
            <Button Content="×" Background="#FF9F43" Foreground="White" Tag="*"/>
            
            <!-- Row 3: 4,5,6,- -->
            <Button Content="4" Background="#2D2D42" Foreground="White" Tag="4"/>
            <Button Content="5" Background="#2D2D42" Foreground="White" Tag="5"/>
            <Button Content="6" Background="#2D2D42" Foreground="White" Tag="6"/>
            <Button Content="-" Background="#FF9F43" Foreground="White" Tag="-"/>
            
            <!-- Row 4: 1,2,3,+ -->
            <Button Content="1" Background="#2D2D42" Foreground="White" Tag="1"/>
            <Button Content="2" Background="#2D2D42" Foreground="White" Tag="2"/>
            <Button Content="3" Background="#2D2D42" Foreground="White" Tag="3"/>
            <Button Content="+" Background="#FF9F43" Foreground="White" Tag="+"/>
            
            <!-- Row 5: 0(wide), ., = -->
            <Button Content="0" Background="#2D2D42" Foreground="White" Tag="0"/>
            <Button Content="00" Background="#2D2D42" Foreground="White" Tag="00"/>
            <Button Content="." Background="#2D2D42" Foreground="White" Tag="."/>
            <Button Content="=" Background="#EE5A24" Foreground="White" Tag="="/>
        </UniformGrid>
    </Grid>
</Window>
```

```csharp
// Calculator.xaml.cs
using System;
using System.Windows;
using System.Windows.Controls;

namespace Calculator;

public partial class MainWindow : Window
{
    private string _current = "0";
    private string _operator = "";
    private double _left = 0;
    private bool _newInput = true;
    private string _expression = "";
    
    public MainWindow()
    {
        InitializeComponent();
        
        // Wire all button clicks through one handler
        foreach (var btn in FindVisualChildren<Button>(this))
            btn.Click += Button_Click;
    }
    
    private void Button_Click(object sender, RoutedEventArgs e)
    {
        if (sender is not Button btn || btn.Tag is not string tag) return;
        
        switch (tag)
        {
            case "0": case "1": case "2": case "3": case "4":
            case "5": case "6": case "7": case "8": case "9":
                if (_newInput) { _current = tag; _newInput = false; }
                else _current = _current == "0" ? tag : _current + tag;
                break;
            
            case "00":
                if (!_newInput) _current = _current == "0" ? "0" : _current + "00";
                break;
            
            case ".":
                if (_newInput) { _current = "0."; _newInput = false; }
                else if (!_current.Contains('.')) _current += ".";
                break;
            
            case "+": case "-": case "*": case "/":
                _left = double.Parse(_current);
                _operator = tag;
                _expression = $"{_current} {OperatorDisplay(tag)}";
                _newInput = true;
                break;
            
            case "=":
                if (!string.IsNullOrEmpty(_operator))
                {
                    double right = double.Parse(_current);
                    lblExpression.Text = $"{_expression} {_current} =";
                    
                    double result = _operator switch
                    {
                        "+" => _left + right,
                        "-" => _left - right,
                        "*" => _left * right,
                        "/" => right != 0 ? _left / right : double.NaN,
                        _ => right
                    };
                    
                    _current = double.IsNaN(result) ? "Error" : 
                               result == Math.Floor(result) ? result.ToString("0") : 
                               result.ToString("G10");
                    _operator = "";
                    _newInput = true;
                }
                break;
            
            case "C":
                _current = "0"; _operator = ""; _left = 0;
                _expression = ""; _newInput = true;
                lblExpression.Text = "";
                break;
            
            case "plusminus":
                if (_current != "0")
                    _current = _current.StartsWith("-") ? _current[1..] : "-" + _current;
                break;
            
            case "%":
                if (double.TryParse(_current, out double pct))
                    _current = (pct / 100).ToString("G10");
                break;
        }
        
        lblDisplay.Text = _current.Length > 12 ? double.Parse(_current).ToString("G8") : _current;
        if (tag is not "=" and not "C") lblExpression.Text = _expression;
    }
    
    private static string OperatorDisplay(string op) => op switch { "*" => "×", "/" => "÷", _ => op };
    
    private static System.Collections.Generic.IEnumerable<T> FindVisualChildren<T>(DependencyObject depObj) where T : DependencyObject
    {
        if (depObj == null) yield break;
        for (int i = 0; i < System.Windows.Media.VisualTreeHelper.GetChildrenCount(depObj); i++)
        {
            var child = System.Windows.Media.VisualTreeHelper.GetChild(depObj, i);
            if (child is T t) yield return t;
            foreach (var childOfChild in FindVisualChildren<T>(child))
                yield return childOfChild;
        }
    }
}
```

---

## 📝 สรุป Part 31

| XAML Concept | C# Equivalent |
|-------------|--------------|
| `<Button Content="OK"/>` | `new Button { Content = "OK" }` |
| `Grid.Row="1"` | Attached property |
| `x:Name="btn"` | Field name for code-behind |
| `Click="Handler"` | Event subscription |
| `{Binding Name}` | Two-way data binding |
| DependencyProperty | Enhanced CLR property |

## WPF vs WinForms

| Feature | WinForms | WPF |
|---------|----------|-----|
| Layout | Pixel-based | Relative/Flexible |
| Styling | Code | XAML styles/templates |
| Data Binding | BindingSource | {Binding} expressions |
| Animation | Timer | Built-in Storyboard |
| Graphics | GDI+ | DirectX-accelerated |
| Resolution | DPI issues | DPI-aware by default |

---

**ก่อนหน้า → [Part 30: WinForms Final](../part21-30/part30-winforms-final.md)**  
**ต่อไป → [Part 32: WPF Data Binding](part32-wpf-databinding.md)**
