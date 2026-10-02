# Part 34: WPF Styles & Templates
## ขั้นตอนที่ 331-340: การตกแต่ง UI ขั้นสูง

---

## 🎯 เป้าหมายของ Part นี้
- Style: Setter, Trigger, BasedOn
- ControlTemplate: เปลี่ยน visual ของ control ทั้งหมด
- DataTemplate: กำหนดหน้าตาของ data
- ItemsControl และ ListBox ItemTemplate
- ContentControl และ ContentPresenter
- Implicit Style (ใช้กับทุก element)
- ResourceDictionary และ MergedDictionaries
- โปรแกรม Dashboard with Custom Styles

---

## ขั้นตอนที่ 331: Style Basics

```xml
<!-- Style สำหรับ Button -->
<Window.Resources>
    <!-- Named style -->
    <Style x:Key="PrimaryButton" TargetType="Button">
        <Setter Property="Background" Value="#3498DB"/>
        <Setter Property="Foreground" Value="White"/>
        <Setter Property="FontSize" Value="13"/>
        <Setter Property="Padding" Value="16,8"/>
        <Setter Property="BorderThickness" Value="0"/>
        <Setter Property="Cursor" Value="Hand"/>
        
        <Style.Triggers>
            <!-- Hover effect -->
            <Trigger Property="IsMouseOver" Value="True">
                <Setter Property="Background" Value="#2980B9"/>
            </Trigger>
            <!-- Pressed effect -->
            <Trigger Property="IsPressed" Value="True">
                <Setter Property="Background" Value="#1F618D"/>
                <Setter Property="RenderTransform">
                    <Setter.Value>
                        <TranslateTransform Y="1"/>
                    </Setter.Value>
                </Trigger>
            </Trigger>
            <!-- Disabled -->
            <Trigger Property="IsEnabled" Value="False">
                <Setter Property="Opacity" Value="0.4"/>
            </Trigger>
        </Style.Triggers>
    </Style>
    
    <!-- BasedOn - inherit from another style -->
    <Style x:Key="DangerButton" BasedOn="{StaticResource PrimaryButton}" TargetType="Button">
        <Setter Property="Background" Value="#E74C3C"/>
        <Style.Triggers>
            <Trigger Property="IsMouseOver" Value="True">
                <Setter Property="Background" Value="#C0392B"/>
            </Trigger>
        </Style.Triggers>
    </Style>
    
    <!-- Implicit style - applies to ALL TextBox in scope -->
    <Style TargetType="TextBox">
        <Setter Property="Padding" Value="8,5"/>
        <Setter Property="BorderBrush" Value="#BDC3C7"/>
        <Setter Property="BorderThickness" Value="1"/>
        <Style.Triggers>
            <Trigger Property="IsFocused" Value="True">
                <Setter Property="BorderBrush" Value="#3498DB"/>
                <Setter Property="BorderThickness" Value="2"/>
            </Trigger>
        </Style.Triggers>
    </Style>
</Window.Resources>

<!-- Using named style -->
<Button Style="{StaticResource PrimaryButton}" Content="บันทึก"/>
<Button Style="{StaticResource DangerButton}" Content="ลบ"/>
<!-- TextBox uses implicit style automatically -->
<TextBox Text="ชื่อ"/>
```

---

## ขั้นตอนที่ 332: DataTrigger (Binding-based trigger)

```xml
<Window.Resources>
    <!-- Style with DataTrigger -->
    <Style x:Key="StatusBadge" TargetType="Border">
        <Setter Property="CornerRadius" Value="10"/>
        <Setter Property="Padding" Value="8,3"/>
        <Setter Property="Background" Value="Gray"/>
    </Style>
    
    <Style x:Key="StockCell" TargetType="TextBlock">
        <Setter Property="FontWeight" Value="Bold"/>
        <Style.Triggers>
            <!-- Multiple triggers -->
            <DataTrigger Binding="{Binding Stock}" Value="0">
                <Setter Property="Foreground" Value="Red"/>
                <Setter Property="Text" Value="หมด"/>
            </DataTrigger>
            <!-- MultiDataTrigger -->
        </Style.Triggers>
    </Style>
</Window.Resources>

<!-- MultiDataTrigger: all conditions must be true -->
<TextBlock>
    <TextBlock.Style>
        <Style TargetType="TextBlock">
            <Style.Triggers>
                <MultiDataTrigger>
                    <MultiDataTrigger.Conditions>
                        <Condition Binding="{Binding IsActive}" Value="True"/>
                        <Condition Binding="{Binding IsAdmin}" Value="True"/>
                    </MultiDataTrigger.Conditions>
                    <Setter Property="Text" Value="Admin ที่ใช้งานอยู่"/>
                    <Setter Property="Foreground" Value="Gold"/>
                </MultiDataTrigger>
            </Style.Triggers>
        </Style>
    </TextBlock.Style>
</TextBlock>
```

---

## ขั้นตอนที่ 333: DataTemplate

```xml
<!-- DataTemplate สำหรับ Product -->
<Window.Resources>
    <!-- Named DataTemplate -->
    <DataTemplate x:Key="ProductCardTemplate" DataType="{x:Type local:Product}">
        <Border BorderThickness="1" BorderBrush="#E0E0E0" CornerRadius="8" Margin="4" Padding="12">
            <Border.Style>
                <Style TargetType="Border">
                    <Setter Property="Background" Value="White"/>
                    <Style.Triggers>
                        <Trigger Property="IsMouseOver" Value="True">
                            <Setter Property="Background" Value="#F8F9FA"/>
                            <Setter Property="Effect">
                                <Setter.Value>
                                    <DropShadowEffect BlurRadius="10" Opacity="0.2" ShadowDepth="3"/>
                                </Setter.Value>
                            </Setter>
                        </Trigger>
                    </Style.Triggers>
                </Style>
            </Border.Style>
            <Grid>
                <Grid.RowDefinitions>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="Auto"/>
                </Grid.RowDefinitions>
                <TextBlock Grid.Row="0" Text="{Binding Name}" FontSize="14" FontWeight="Bold"/>
                <TextBlock Grid.Row="1" Text="{Binding Category}" Foreground="Gray" FontSize="11"/>
                <TextBlock Grid.Row="2" FontSize="16" FontWeight="Bold" Foreground="#E74C3C" Margin="0,5,0,0"
                           Text="{Binding Price, StringFormat='฿{0:N0}'}"/>
            </Grid>
        </Border>
    </DataTemplate>
    
    <!-- Implicit DataTemplate (applies by type automatically) -->
    <DataTemplate DataType="{x:Type local:Customer}">
        <StackPanel Orientation="Horizontal" Margin="3">
            <Ellipse Width="30" Height="30" Fill="#3498DB" Margin="0,0,8,0"/>
            <StackPanel>
                <TextBlock Text="{Binding FullName}" FontWeight="SemiBold"/>
                <TextBlock Text="{Binding Email}" FontSize="11" Foreground="Gray"/>
            </StackPanel>
        </StackPanel>
    </DataTemplate>
</Window.Resources>

<!-- Using DataTemplate in ItemsControl -->
<ItemsControl ItemsSource="{Binding Products}" ItemTemplate="{StaticResource ProductCardTemplate}">
    <ItemsControl.ItemsPanel>
        <ItemsPanelTemplate>
            <WrapPanel/>
        </ItemsPanelTemplate>
    </ItemsControl.ItemsPanel>
</ItemsControl>

<!-- ListBox with implicit Customer DataTemplate -->
<ListBox ItemsSource="{Binding Customers}"/>
```

---

## ขั้นตอนที่ 334: ControlTemplate

```xml
<!-- Custom ControlTemplate สำหรับ Button -->
<Window.Resources>
    <Style x:Key="RoundedButton" TargetType="Button">
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="Button">
                    <Border x:Name="border" 
                            Background="{TemplateBinding Background}"
                            BorderBrush="{TemplateBinding BorderBrush}"
                            BorderThickness="{TemplateBinding BorderThickness}"
                            CornerRadius="20" Padding="{TemplateBinding Padding}">
                        <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center"/>
                    </Border>
                    <ControlTemplate.Triggers>
                        <Trigger Property="IsMouseOver" Value="True">
                            <Setter TargetName="border" Property="Opacity" Value="0.85"/>
                        </Trigger>
                        <Trigger Property="IsPressed" Value="True">
                            <Setter TargetName="border" Property="RenderTransform">
                                <Setter.Value>
                                    <ScaleTransform ScaleX="0.97" ScaleY="0.97" CenterX="50" CenterY="15"/>
                                </Setter.Value>
                            </Setter>
                        </Trigger>
                    </ControlTemplate.Triggers>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>
    
    <!-- Custom Slider template -->
    <Style x:Key="ModernSlider" TargetType="Slider">
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="Slider">
                    <Grid>
                        <Track x:Name="PART_Track">
                            <Track.DecreaseRepeatButton>
                                <RepeatButton>
                                    <RepeatButton.Template>
                                        <ControlTemplate TargetType="RepeatButton">
                                            <Border Height="4" Background="#3498DB" CornerRadius="2"/>
                                        </ControlTemplate>
                                    </RepeatButton.Template>
                                </RepeatButton>
                            </Track.DecreaseRepeatButton>
                            <Track.IncreaseRepeatButton>
                                <RepeatButton>
                                    <RepeatButton.Template>
                                        <ControlTemplate TargetType="RepeatButton">
                                            <Border Height="4" Background="#E0E0E0" CornerRadius="2"/>
                                        </ControlTemplate>
                                    </RepeatButton.Template>
                                </RepeatButton>
                            </Track.IncreaseRepeatButton>
                            <Track.Thumb>
                                <Thumb>
                                    <Thumb.Template>
                                        <ControlTemplate TargetType="Thumb">
                                            <Ellipse Width="18" Height="18" Fill="#3498DB"
                                                     Stroke="White" StrokeThickness="2"/>
                                        </ControlTemplate>
                                    </Thumb.Template>
                                </Thumb>
                            </Track.Thumb>
                        </Track>
                    </Grid>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>
</Window.Resources>
```

---

## ขั้นตอนที่ 335: ResourceDictionary

```xml
<!-- Themes/Colors.xaml -->
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    
    <!-- Color tokens -->
    <Color x:Key="PrimaryColor">#3498DB</Color>
    <Color x:Key="SecondaryColor">#2ECC71</Color>
    <Color x:Key="DangerColor">#E74C3C</Color>
    <Color x:Key="DarkBg">#2C3E50</Color>
    <Color x:Key="LightBg">#ECF0F1</Color>
    
    <!-- Brush tokens -->
    <SolidColorBrush x:Key="PrimaryBrush" Color="{StaticResource PrimaryColor}"/>
    <SolidColorBrush x:Key="SecondaryBrush" Color="{StaticResource SecondaryColor}"/>
    <SolidColorBrush x:Key="DangerBrush" Color="{StaticResource DangerColor}"/>
    
    <!-- Font sizes -->
    <sys:Double x:Key="TitleFontSize" xmlns:sys="clr-namespace:System;assembly=mscorlib">24</sys:Double>
    <sys:Double x:Key="BodyFontSize" xmlns:sys="clr-namespace:System;assembly=mscorlib">13</sys:Double>
</ResourceDictionary>

<!-- Themes/Buttons.xaml -->
<ResourceDictionary ...>
    <ResourceDictionary.MergedDictionaries>
        <ResourceDictionary Source="Colors.xaml"/>
    </ResourceDictionary.MergedDictionaries>
    
    <Style x:Key="PrimaryBtn" TargetType="Button">
        <Setter Property="Background" Value="{StaticResource PrimaryBrush}"/>
        <!-- ... -->
    </Style>
</ResourceDictionary>

<!-- App.xaml - merge all dictionaries -->
<Application.Resources>
    <ResourceDictionary>
        <ResourceDictionary.MergedDictionaries>
            <ResourceDictionary Source="Themes/Colors.xaml"/>
            <ResourceDictionary Source="Themes/Buttons.xaml"/>
            <ResourceDictionary Source="Themes/TextBoxes.xaml"/>
        </ResourceDictionary.MergedDictionaries>
    </ResourceDictionary>
</Application.Resources>
```

---

## ขั้นตอนที่ 336-340: Dashboard with Custom Styles

```xml
<!-- Dashboard.xaml -->
<Window x:Class="Dashboard.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:local="clr-namespace:Dashboard"
        Title="Sales Dashboard" Height="600" Width="900"
        Background="#F5F6FA">
    <Window.Resources>
        <!-- KPI Card style -->
        <Style x:Key="KpiCard" TargetType="Border">
            <Setter Property="Background" Value="White"/>
            <Setter Property="CornerRadius" Value="8"/>
            <Setter Property="Padding" Value="20"/>
            <Setter Property="Margin" Value="6"/>
            <Setter Property="Effect">
                <Setter.Value>
                    <DropShadowEffect BlurRadius="8" Opacity="0.1" ShadowDepth="2"/>
                </Setter.Value>
            </Setter>
        </Style>
        
        <!-- KPI value style -->
        <Style x:Key="KpiValue" TargetType="TextBlock">
            <Setter Property="FontSize" Value="28"/>
            <Setter Property="FontWeight" Value="Bold"/>
            <Setter Property="Margin" Value="0,8,0,4"/>
        </Style>
        
        <!-- Product row DataTemplate -->
        <DataTemplate x:Key="ProductRow" DataType="{x:Type local:SaleItem}">
            <Grid Margin="0,3">
                <Grid.ColumnDefinitions>
                    <ColumnDefinition Width="*"/>
                    <ColumnDefinition Width="80"/>
                    <ColumnDefinition Width="100"/>
                    <ColumnDefinition Width="80"/>
                </Grid.ColumnDefinitions>
                <TextBlock Grid.Column="0" Text="{Binding ProductName}" VerticalAlignment="Center"/>
                <TextBlock Grid.Column="1" Text="{Binding Quantity}" HorizontalAlignment="Center" VerticalAlignment="Center"/>
                <TextBlock Grid.Column="2" Text="{Binding Revenue, StringFormat='฿{0:N0}'}"
                           HorizontalAlignment="Right" VerticalAlignment="Center" FontWeight="Bold"/>
                <Border Grid.Column="3" CornerRadius="3" Padding="6,2" HorizontalAlignment="Right" VerticalAlignment="Center">
                    <Border.Style>
                        <Style TargetType="Border">
                            <Setter Property="Background" Value="#DAFCE7"/>
                            <Style.Triggers>
                                <DataTrigger Binding="{Binding IsDown}" Value="True">
                                    <Setter Property="Background" Value="#FFDEDE"/>
                                </DataTrigger>
                            </Style.Triggers>
                        </Style>
                    </Border.Style>
                    <TextBlock Text="{Binding Change, StringFormat='{}{0:+0.#;-0.#}%'}">
                        <TextBlock.Style>
                            <Style TargetType="TextBlock">
                                <Setter Property="Foreground" Value="#15803D"/>
                                <Setter Property="FontSize" Value="11"/>
                                <Style.Triggers>
                                    <DataTrigger Binding="{Binding IsDown}" Value="True">
                                        <Setter Property="Foreground" Value="#DC2626"/>
                                    </DataTrigger>
                                </Style.Triggers>
                            </Style>
                        </TextBlock.Style>
                    </TextBlock>
                </Border>
            </Grid>
        </DataTemplate>
    </Window.Resources>
    
    <Grid Margin="12">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
        </Grid.RowDefinitions>
        
        <!-- Page title -->
        <TextBlock Grid.Row="0" Text="Sales Dashboard" FontSize="22" FontWeight="Bold" Margin="6,0,0,12"/>
        
        <!-- KPI row -->
        <UniformGrid Grid.Row="1" Rows="1" Columns="4">
            <Border Style="{StaticResource KpiCard}">
                <StackPanel>
                    <TextBlock Text="💰 ยอดขายวันนี้" Foreground="Gray" FontSize="12"/>
                    <TextBlock Text="{Binding TodaySales, StringFormat='฿{0:N0}'}" Style="{StaticResource KpiValue}" Foreground="#3498DB"/>
                    <TextBlock Text="+12.5% จากเมื่อวาน" Foreground="Green" FontSize="11"/>
                </StackPanel>
            </Border>
            <Border Style="{StaticResource KpiCard}">
                <StackPanel>
                    <TextBlock Text="📦 คำสั่งซื้อ" Foreground="Gray" FontSize="12"/>
                    <TextBlock Text="{Binding OrderCount}" Style="{StaticResource KpiValue}" Foreground="#E67E22"/>
                    <TextBlock Text="+5 รายการใหม่" Foreground="Green" FontSize="11"/>
                </StackPanel>
            </Border>
            <Border Style="{StaticResource KpiCard}">
                <StackPanel>
                    <TextBlock Text="👥 ลูกค้าใหม่" Foreground="Gray" FontSize="12"/>
                    <TextBlock Text="{Binding NewCustomers}" Style="{StaticResource KpiValue}" Foreground="#9B59B6"/>
                    <TextBlock Text="-2 จากเมื่อวาน" Foreground="#E74C3C" FontSize="11"/>
                </StackPanel>
            </Border>
            <Border Style="{StaticResource KpiCard}">
                <StackPanel>
                    <TextBlock Text="⭐ Rating" Foreground="Gray" FontSize="12"/>
                    <TextBlock Text="{Binding AverageRating, StringFormat='{}{0:F1}'}" Style="{StaticResource KpiValue}" Foreground="#F1C40F"/>
                    <TextBlock Text="จากรีวิว 248 รายการ" Foreground="Gray" FontSize="11"/>
                </StackPanel>
            </Border>
        </UniformGrid>
        
        <!-- Product sales list -->
        <Border Grid.Row="2" Style="{StaticResource KpiCard}">
            <Grid>
                <Grid.RowDefinitions>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="*"/>
                </Grid.RowDefinitions>
                
                <TextBlock Grid.Row="0" Text="สินค้าขายดี" FontSize="14" FontWeight="Bold" Margin="0,0,0,10"/>
                
                <!-- Header -->
                <Grid Grid.Row="1" Margin="0,0,0,5">
                    <Grid.ColumnDefinitions>
                        <ColumnDefinition Width="*"/>
                        <ColumnDefinition Width="80"/>
                        <ColumnDefinition Width="100"/>
                        <ColumnDefinition Width="80"/>
                    </Grid.ColumnDefinitions>
                    <TextBlock Grid.Column="0" Text="ชื่อสินค้า" Foreground="Gray" FontSize="11"/>
                    <TextBlock Grid.Column="1" Text="จำนวน" Foreground="Gray" FontSize="11" HorizontalAlignment="Center"/>
                    <TextBlock Grid.Column="2" Text="รายรับ" Foreground="Gray" FontSize="11" HorizontalAlignment="Right"/>
                    <TextBlock Grid.Column="3" Text="เปลี่ยนแปลง" Foreground="Gray" FontSize="11" HorizontalAlignment="Right"/>
                </Grid>
                
                <ItemsControl Grid.Row="2" ItemsSource="{Binding TopProducts}" ItemTemplate="{StaticResource ProductRow}"/>
            </Grid>
        </Border>
    </Grid>
</Window>
```

```csharp
// Dashboard.xaml.cs
public class SaleItem
{
    public string ProductName { get; set; } = "";
    public int Quantity { get; set; }
    public decimal Revenue { get; set; }
    public double Change { get; set; }
    public bool IsDown => Change < 0;
}

public class DashboardViewModel : ViewModelBase
{
    public decimal TodaySales { get; } = 125_800;
    public int OrderCount { get; } = 47;
    public int NewCustomers { get; } = 12;
    public double AverageRating { get; } = 4.7;
    
    public List<SaleItem> TopProducts { get; } = new()
    {
        new() { ProductName="iPhone 15 Pro", Quantity=23, Revenue=862_300, Change=15.2 },
        new() { ProductName="MacBook Air M3", Quantity=8, Revenue=400_000, Change=-3.1 },
        new() { ProductName="iPad Pro 12.9", Quantity=15, Revenue=225_000, Change=8.7 },
        new() { ProductName="AirPods Pro", Quantity=42, Revenue=210_000, Change=22.4 },
        new() { ProductName="Apple Watch S9", Quantity=19, Revenue=133_000, Change=-1.5 },
    };
}

public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();
        DataContext = new DashboardViewModel();
    }
}
```

---

## 📝 สรุป Part 34

| Concept | ใช้งาน |
|---------|--------|
| Style | กำหนด visual properties |
| Trigger | เปลี่ยน style ตาม conditions |
| DataTemplate | กำหนดหน้าตาของ data objects |
| ControlTemplate | เปลี่ยน visual ของ control ทั้งหมด |
| ResourceDictionary | แยก resources เป็นไฟล์ |
| MergedDictionaries | รวม multiple resource files |
| DropShadowEffect | เพิ่ม shadow effect |

---

**ก่อนหน้า → [Part 33: WPF MVVM](part33-wpf-mvvm.md)**  
**ต่อไป → [Part 35: WPF Animation](part35-wpf-animation.md)**
