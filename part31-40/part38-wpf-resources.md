# Part 38: WPF Resources & Theming
## ขั้นตอนที่ 371-380: Resources, Themes, Localization

---

## 🎯 เป้าหมายของ Part นี้
- ResourceDictionary hierarchy
- StaticResource vs DynamicResource
- Theme switching (Light/Dark)
- Localization ด้วย .resx
- StringFormat และ Binding
- Converter registry
- Application-level resources
- โปรแกรม Themed App with Language Switch

---

## ขั้นตอนที่ 371: Resource Hierarchy

```
Resources ถูก resolve จาก closest → furthest:
1. Element.Resources (ตัว element เอง)
2. Parent.Resources (parent ขึ้นไปเรื่อยๆ)
3. Window.Resources
4. Application.Resources (App.xaml)
5. System Resources
```

```xml
<!-- App.xaml - global resources -->
<Application.Resources>
    <ResourceDictionary>
        <ResourceDictionary.MergedDictionaries>
            <!-- Theme dictionary (switchable) -->
            <ResourceDictionary x:Name="ThemeDictionary" Source="Themes/LightTheme.xaml"/>
            <!-- Component styles -->
            <ResourceDictionary Source="Styles/Buttons.xaml"/>
            <ResourceDictionary Source="Styles/TextBoxes.xaml"/>
            <!-- Converters -->
            <ResourceDictionary Source="Converters/Converters.xaml"/>
        </ResourceDictionary.MergedDictionaries>
    </ResourceDictionary>
</Application.Resources>
```

---

## ขั้นตอนที่ 372: StaticResource vs DynamicResource

```xml
<!-- StaticResource: resolved ONCE at load time (faster) -->
<Button Background="{StaticResource PrimaryBrush}"/>

<!-- DynamicResource: re-evaluated when resource changes (required for theming) -->
<Button Background="{DynamicResource PrimaryBrush}"/>
<Window Background="{DynamicResource WindowBackground}"/>
<TextBlock Foreground="{DynamicResource TextPrimary}"/>

<!-- ต้องใช้ DynamicResource สำหรับ:
     - Resources ที่อาจเปลี่ยนแปลงที่ runtime
     - Theme switching
     - Resources ที่ forward reference (referenced before defined) -->
```

---

## ขั้นตอนที่ 373: Light/Dark Theme Files

```xml
<!-- Themes/LightTheme.xaml -->
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    
    <!-- Background colors -->
    <SolidColorBrush x:Key="WindowBackground" Color="#F5F6FA"/>
    <SolidColorBrush x:Key="SurfaceBackground" Color="White"/>
    <SolidColorBrush x:Key="SidebarBackground" Color="#2C3E50"/>
    <SolidColorBrush x:Key="CardBackground" Color="White"/>
    
    <!-- Text colors -->
    <SolidColorBrush x:Key="TextPrimary" Color="#2C3E50"/>
    <SolidColorBrush x:Key="TextSecondary" Color="#7F8C8D"/>
    <SolidColorBrush x:Key="TextMuted" Color="#BDC3C7"/>
    
    <!-- Accent colors -->
    <SolidColorBrush x:Key="AccentPrimary" Color="#3498DB"/>
    <SolidColorBrush x:Key="AccentSuccess" Color="#27AE60"/>
    <SolidColorBrush x:Key="AccentDanger" Color="#E74C3C"/>
    <SolidColorBrush x:Key="AccentWarning" Color="#F39C12"/>
    
    <!-- Border colors -->
    <SolidColorBrush x:Key="BorderLight" Color="#E0E0E0"/>
    <SolidColorBrush x:Key="BorderMedium" Color="#BDC3C7"/>
    
    <!-- Shadow -->
    <sys:Double x:Key="ShadowOpacity" xmlns:sys="clr-namespace:System;assembly=mscorlib">0.1</sys:Double>
</ResourceDictionary>

<!-- Themes/DarkTheme.xaml -->
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    
    <SolidColorBrush x:Key="WindowBackground" Color="#1A1A2E"/>
    <SolidColorBrush x:Key="SurfaceBackground" Color="#16213E"/>
    <SolidColorBrush x:Key="SidebarBackground" Color="#0F3460"/>
    <SolidColorBrush x:Key="CardBackground" Color="#16213E"/>
    
    <SolidColorBrush x:Key="TextPrimary" Color="#E0E0E0"/>
    <SolidColorBrush x:Key="TextSecondary" Color="#A0A0B0"/>
    <SolidColorBrush x:Key="TextMuted" Color="#606080"/>
    
    <SolidColorBrush x:Key="AccentPrimary" Color="#4ECDC4"/>
    <SolidColorBrush x:Key="AccentSuccess" Color="#2ECC71"/>
    <SolidColorBrush x:Key="AccentDanger" Color="#E74C3C"/>
    <SolidColorBrush x:Key="AccentWarning" Color="#F39C12"/>
    
    <SolidColorBrush x:Key="BorderLight" Color="#2A2A4A"/>
    <SolidColorBrush x:Key="BorderMedium" Color="#3A3A5A"/>
    
    <sys:Double x:Key="ShadowOpacity" xmlns:sys="clr-namespace:System;assembly=mscorlib">0.3</sys:Double>
</ResourceDictionary>
```

---

## ขั้นตอนที่ 374: Theme Switcher

```csharp
// Services/ThemeService.cs
public enum AppTheme { Light, Dark }

public static class ThemeService
{
    private static AppTheme _current = AppTheme.Light;
    
    public static AppTheme CurrentTheme => _current;
    
    public static void SetTheme(AppTheme theme)
    {
        _current = theme;
        
        var uri = theme == AppTheme.Light
            ? new Uri("Themes/LightTheme.xaml", UriKind.Relative)
            : new Uri("Themes/DarkTheme.xaml", UriKind.Relative);
        
        // Replace the theme dictionary
        var appDict = Application.Current.Resources.MergedDictionaries;
        var themeDict = appDict.FirstOrDefault(d => 
            d.Source?.OriginalString.Contains("Theme") == true);
        
        if (themeDict != null)
        {
            int idx = appDict.IndexOf(themeDict);
            appDict.Remove(themeDict);
            appDict.Insert(idx, new ResourceDictionary { Source = uri });
        }
        
        // Save preference
        Properties.Settings.Default.Theme = theme.ToString();
        Properties.Settings.Default.Save();
    }
    
    public static void LoadSavedTheme()
    {
        if (Enum.TryParse<AppTheme>(Properties.Settings.Default.Theme, out var saved))
            SetTheme(saved);
    }
}
```

---

## ขั้นตอนที่ 375: Localization (RESX)

```
โครงสร้างไฟล์ Resources:
Properties/
  Strings.resx          (default / English)
  Strings.th-TH.resx    (Thai)
  Strings.ja-JP.resx    (Japanese)
```

```xml
<!-- Strings.resx (English) -->
<data name="AppTitle" xml:space="preserve">
  <value>My Application</value>
</data>
<data name="BtnSave" xml:space="preserve">
  <value>Save</value>
</data>
<data name="BtnCancel" xml:space="preserve">
  <value>Cancel</value>
</data>
<data name="MsgConfirmDelete" xml:space="preserve">
  <value>Are you sure you want to delete this item?</value>
</data>

<!-- Strings.th-TH.resx (Thai) -->
<data name="AppTitle" xml:space="preserve">
  <value>แอปพลิเคชันของฉัน</value>
</data>
<data name="BtnSave" xml:space="preserve">
  <value>บันทึก</value>
</data>
<data name="BtnCancel" xml:space="preserve">
  <value>ยกเลิก</value>
</data>
<data name="MsgConfirmDelete" xml:space="preserve">
  <value>คุณต้องการลบรายการนี้หรือไม่?</value>
</data>
```

```csharp
// Using localized strings in code
using MyApp.Properties;

string title = Strings.AppTitle;
MessageBox.Show(Strings.MsgConfirmDelete, title, MessageBoxButton.YesNo);

// Switch language at runtime
public static void SetLanguage(string cultureName)
{
    var culture = new CultureInfo(cultureName);
    Thread.CurrentThread.CurrentCulture = culture;
    Thread.CurrentThread.CurrentUICulture = culture;
    
    // Reload window for immediate effect
    var newWindow = new MainWindow();
    newWindow.Show();
    Application.Current.MainWindow.Close();
    Application.Current.MainWindow = newWindow;
}
```

```xml
<!-- Using in XAML via x:Static -->
<Button Content="{x:Static prop:Strings.BtnSave}"/>
<TextBlock Text="{x:Static prop:Strings.AppTitle}"/>
```

---

## ขั้นตอนที่ 376-380: Themed App Full Example

```xml
<!-- MainWindow.xaml - uses DynamicResource throughout -->
<Window x:Class="ThemedApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:local="clr-namespace:ThemedApp"
        Background="{DynamicResource WindowBackground}"
        Foreground="{DynamicResource TextPrimary}"
        Title="Themed App" Height="560" Width="800"
        WindowStartupLocation="CenterScreen">
    
    <Grid>
        <Grid.ColumnDefinitions>
            <ColumnDefinition Width="200"/>
            <ColumnDefinition Width="*"/>
        </Grid.ColumnDefinitions>
        
        <!-- Sidebar -->
        <Border Grid.Column="0" Background="{DynamicResource SidebarBackground}">
            <StackPanel>
                <!-- Logo -->
                <Border Padding="16,20">
                    <TextBlock Text="⚡ ThemedApp" FontSize="16" FontWeight="Bold" Foreground="White"/>
                </Border>
                
                <!-- Nav items -->
                <Button Content="🏠 หน้าแรก" Foreground="White" Background="Transparent"
                        BorderThickness="0" HorizontalContentAlignment="Left" Padding="16,12"/>
                <Button Content="📦 สินค้า" Foreground="White" Background="Transparent"
                        BorderThickness="0" HorizontalContentAlignment="Left" Padding="16,12"/>
                <Button Content="📊 รายงาน" Foreground="White" Background="Transparent"
                        BorderThickness="0" HorizontalContentAlignment="Left" Padding="16,12"/>
                
                <Separator Margin="12" Background="#44556E"/>
                
                <!-- Theme toggle -->
                <StackPanel Margin="16,8" Orientation="Horizontal">
                    <TextBlock Text="🌙 Dark Mode" Foreground="White" VerticalAlignment="Center" Margin="0,0,12,0"/>
                    <ToggleButton x:Name="toggleTheme" Width="44" Height="24"
                                  Checked="ToggleTheme_Checked" Unchecked="ToggleTheme_Unchecked"/>
                </StackPanel>
                
                <!-- Language switch -->
                <StackPanel Margin="16,8">
                    <TextBlock Text="ภาษา" Foreground="#95A5A6" FontSize="11" Margin="0,0,0,6"/>
                    <ComboBox x:Name="cmbLanguage" SelectionChanged="CmbLanguage_Changed">
                        <ComboBoxItem Content="🇹🇭 ภาษาไทย" Tag="th-TH"/>
                        <ComboBoxItem Content="🇺🇸 English" Tag="en-US"/>
                    </ComboBox>
                </StackPanel>
            </StackPanel>
        </Border>
        
        <!-- Content -->
        <ScrollViewer Grid.Column="1" Background="{DynamicResource WindowBackground}">
            <StackPanel Margin="24">
                <TextBlock Text="ยินดีต้อนรับ" FontSize="24" FontWeight="Bold"
                           Foreground="{DynamicResource TextPrimary}" Margin="0,0,0,8"/>
                <TextBlock Text="นี่คือตัวอย่าง WPF App ที่รองรับ Light/Dark theme"
                           Foreground="{DynamicResource TextSecondary}" Margin="0,0,0,20"/>
                
                <!-- Cards -->
                <UniformGrid Rows="2" Columns="2" Margin="0,0,0,20">
                    <Border Background="{DynamicResource CardBackground}" CornerRadius="8" Padding="20" Margin="0,0,8,8">
                        <Border.Effect>
                            <DropShadowEffect BlurRadius="10" ShadowDepth="0" 
                                              Opacity="{DynamicResource ShadowOpacity}"/>
                        </Border.Effect>
                        <StackPanel>
                            <TextBlock Text="💰 รายรับ" Foreground="{DynamicResource TextSecondary}" FontSize="12"/>
                            <TextBlock Text="฿125,400" FontSize="28" FontWeight="Bold"
                                       Foreground="{DynamicResource AccentPrimary}"/>
                        </StackPanel>
                    </Border>
                    <Border Background="{DynamicResource CardBackground}" CornerRadius="8" Padding="20" Margin="8,0,0,8">
                        <StackPanel>
                            <TextBlock Text="📦 คำสั่งซื้อ" Foreground="{DynamicResource TextSecondary}" FontSize="12"/>
                            <TextBlock Text="243" FontSize="28" FontWeight="Bold"
                                       Foreground="{DynamicResource AccentSuccess}"/>
                        </StackPanel>
                    </Border>
                    <Border Background="{DynamicResource CardBackground}" CornerRadius="8" Padding="20" Margin="0,0,8,0">
                        <StackPanel>
                            <TextBlock Text="👥 ลูกค้า" Foreground="{DynamicResource TextSecondary}" FontSize="12"/>
                            <TextBlock Text="1,842" FontSize="28" FontWeight="Bold"
                                       Foreground="{DynamicResource AccentWarning}"/>
                        </StackPanel>
                    </Border>
                    <Border Background="{DynamicResource CardBackground}" CornerRadius="8" Padding="20" Margin="8,0,0,0">
                        <StackPanel>
                            <TextBlock Text="⭐ Rating" Foreground="{DynamicResource TextSecondary}" FontSize="12"/>
                            <TextBlock Text="4.8 / 5.0" FontSize="28" FontWeight="Bold"
                                       Foreground="{DynamicResource AccentDanger}"/>
                        </StackPanel>
                    </Border>
                </UniformGrid>
                
                <!-- List with themed borders -->
                <Border Background="{DynamicResource CardBackground}" CornerRadius="8" Padding="20">
                    <StackPanel>
                        <TextBlock Text="รายการล่าสุด" FontSize="16" FontWeight="SemiBold"
                                   Foreground="{DynamicResource TextPrimary}" Margin="0,0,0,12"/>
                        <ItemsControl ItemsSource="{Binding RecentItems}">
                            <ItemsControl.ItemTemplate>
                                <DataTemplate>
                                    <Border BorderBrush="{DynamicResource BorderLight}" BorderThickness="0,0,0,1"
                                            Padding="0,10">
                                        <Grid>
                                            <TextBlock Text="{Binding Name}"
                                                       Foreground="{DynamicResource TextPrimary}"/>
                                            <TextBlock Text="{Binding Date, StringFormat=dd/MM/yyyy}"
                                                       HorizontalAlignment="Right"
                                                       Foreground="{DynamicResource TextSecondary}"
                                                       FontSize="12"/>
                                        </Grid>
                                    </Border>
                                </DataTemplate>
                            </ItemsControl.ItemTemplate>
                        </ItemsControl>
                    </StackPanel>
                </Border>
            </StackPanel>
        </ScrollViewer>
    </Grid>
</Window>
```

```csharp
// MainWindow.xaml.cs
public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();
        DataContext = new MainViewModel();
        ThemeService.LoadSavedTheme();
        
        // Set toggle state
        toggleTheme.IsChecked = ThemeService.CurrentTheme == AppTheme.Dark;
    }
    
    private void ToggleTheme_Checked(object sender, RoutedEventArgs e)
        => ThemeService.SetTheme(AppTheme.Dark);
    
    private void ToggleTheme_Unchecked(object sender, RoutedEventArgs e)
        => ThemeService.SetTheme(AppTheme.Light);
    
    private void CmbLanguage_Changed(object sender, SelectionChangedEventArgs e)
    {
        if (cmbLanguage.SelectedItem is ComboBoxItem item)
        {
            string culture = item.Tag.ToString()!;
            LocalizationService.SetLanguage(culture);
        }
    }
}
```

---

## 📝 สรุป Part 38

| Concept | Key Points |
|---------|-----------|
| StaticResource | Resolved once, faster |
| DynamicResource | Re-evaluated at runtime |
| Theme switching | Replace MergedDictionary |
| .resx files | Localization strings |
| x:Static | Access static members in XAML |
| ResourceDictionary | Organize resources by concern |

---

**ก่อนหน้า → [Part 37: WPF Custom Controls](part37-wpf-custom-controls.md)**  
**ต่อไป → [Part 39: WPF Advanced MVVM](part39-wpf-advanced-mvvm.md)**
