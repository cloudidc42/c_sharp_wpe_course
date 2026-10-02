# Part 37: WPF Custom Controls
## ขั้นตอนที่ 361-370: สร้าง Custom Controls ขั้นสูง

---

## 🎯 เป้าหมายของ Part นี้
- UserControl vs Custom Control
- DependencyProperty: Register, Metadata, Callbacks
- RoutedEvent: Register, EventArgs
- PART_ naming convention (lookless controls)
- TemplatePart attribute
- Generic.xaml (default style)
- Attached Behavior
- โปรแกรม Custom UI Component Library

---

## ขั้นตอนที่ 361: UserControl vs Custom Control

```
UserControl:
  - สร้างเร็ว, มี XAML ของตัวเอง
  - ยาก override Template
  - เหมาะสำหรับ reusable UI sections

Custom Control (TemplatedControl):
  - เป็น lookless - style/template ได้อิสระ
  - DependencyProperty + RoutedEvent
  - Generic.xaml เก็บ default template
  - เหมาะสำหรับ control library
```

---

## ขั้นตอนที่ 362: DependencyProperty

```csharp
// Custom Control: CircularProgressBar
[TemplatePart(Name = "PART_Indicator", Type = typeof(Arc))]
[TemplatePart(Name = "PART_ValueText", Type = typeof(TextBlock))]
public class CircularProgressBar : Control
{
    // DependencyProperty registration pattern
    public static readonly DependencyProperty ValueProperty =
        DependencyProperty.Register(
            nameof(Value),          // property name
            typeof(double),          // property type
            typeof(CircularProgressBar), // owner type
            new FrameworkPropertyMetadata(
                0.0,                  // default value
                FrameworkPropertyMetadataOptions.AffectsRender,
                OnValueChanged));     // callback
    
    public static readonly DependencyProperty MaximumProperty =
        DependencyProperty.Register(
            nameof(Maximum), typeof(double), typeof(CircularProgressBar),
            new PropertyMetadata(100.0, OnValueChanged));
    
    public static readonly DependencyProperty StrokeThicknessProperty =
        DependencyProperty.Register(
            nameof(StrokeThickness), typeof(double), typeof(CircularProgressBar),
            new PropertyMetadata(8.0, OnValueChanged));
    
    public static readonly DependencyProperty ProgressColorProperty =
        DependencyProperty.Register(
            nameof(ProgressColor), typeof(Brush), typeof(CircularProgressBar),
            new PropertyMetadata(Brushes.DodgerBlue));
    
    public static readonly DependencyProperty TrackColorProperty =
        DependencyProperty.Register(
            nameof(TrackColor), typeof(Brush), typeof(CircularProgressBar),
            new PropertyMetadata(Brushes.LightGray));
    
    public static readonly DependencyProperty ShowTextProperty =
        DependencyProperty.Register(
            nameof(ShowText), typeof(bool), typeof(CircularProgressBar),
            new PropertyMetadata(true));
    
    // CLR wrappers - NO logic here, only GetValue/SetValue
    public double Value
    {
        get => (double)GetValue(ValueProperty);
        set => SetValue(ValueProperty, value);
    }
    
    public double Maximum
    {
        get => (double)GetValue(MaximumProperty);
        set => SetValue(MaximumProperty, value);
    }
    
    public double StrokeThickness
    {
        get => (double)GetValue(StrokeThicknessProperty);
        set => SetValue(StrokeThicknessProperty, value);
    }
    
    public Brush ProgressColor
    {
        get => (Brush)GetValue(ProgressColorProperty);
        set => SetValue(ProgressColorProperty, value);
    }
    
    public Brush TrackColor
    {
        get => (Brush)GetValue(TrackColorProperty);
        set => SetValue(TrackColorProperty, value);
    }
    
    public bool ShowText
    {
        get => (bool)GetValue(ShowTextProperty);
        set => SetValue(ShowTextProperty, value);
    }
    
    // Computed
    public double Percentage => Maximum > 0 ? (Value / Maximum) * 100 : 0;
    
    static CircularProgressBar()
    {
        // Override default style key to look in Generic.xaml
        DefaultStyleKeyProperty.OverrideMetadata(
            typeof(CircularProgressBar),
            new FrameworkPropertyMetadata(typeof(CircularProgressBar)));
    }
    
    private static void OnValueChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        if (d is CircularProgressBar bar)
        {
            bar.UpdateVisual();
            bar.CoerceValue(ValueProperty);
        }
    }
    
    public override void OnApplyTemplate()
    {
        base.OnApplyTemplate();
        UpdateVisual();
    }
    
    private void UpdateVisual()
    {
        // Find template parts
        var indicator = GetTemplateChild("PART_Indicator") as System.Windows.Shapes.Path;
        var text = GetTemplateChild("PART_ValueText") as TextBlock;
        
        if (indicator != null)
        {
            double pct = Maximum > 0 ? Value / Maximum : 0;
            double angle = pct * 360;
            double radius = (ActualWidth / 2) - StrokeThickness;
            
            if (radius > 0)
            {
                indicator.Data = CreateArcGeometry(radius, angle, StrokeThickness);
            }
        }
        
        if (text != null)
        {
            text.Text = $"{Percentage:F0}%";
        }
    }
    
    private Geometry CreateArcGeometry(double radius, double angleDegrees, double thickness)
    {
        double cx = ActualWidth / 2;
        double cy = ActualHeight / 2;
        double startAngle = -90; // Start at top
        double endAngle = startAngle + Math.Min(angleDegrees, 359.99);
        
        double startRad = startAngle * Math.PI / 180;
        double endRad = endAngle * Math.PI / 180;
        
        var startPoint = new System.Windows.Point(cx + radius * Math.Cos(startRad), cy + radius * Math.Sin(startRad));
        var endPoint = new System.Windows.Point(cx + radius * Math.Cos(endRad), cy + radius * Math.Sin(endRad));
        
        bool isLarge = angleDegrees >= 180;
        
        var geo = new StreamGeometry();
        using var ctx = geo.Open();
        ctx.BeginFigure(startPoint, false, false);
        ctx.ArcTo(endPoint, new System.Windows.Size(radius, radius), 0, isLarge,
            SweepDirection.Clockwise, true, false);
        geo.Freeze();
        return geo;
    }
}
```

---

## ขั้นตอนที่ 363: Generic.xaml (Default Style)

```xml
<!-- Themes/Generic.xaml -->
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
                    xmlns:local="clr-namespace:MyControls">
    
    <!-- Default style for CircularProgressBar -->
    <Style TargetType="{x:Type local:CircularProgressBar}">
        <Setter Property="Width" Value="100"/>
        <Setter Property="Height" Value="100"/>
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="{x:Type local:CircularProgressBar}">
                    <Grid>
                        <!-- Track circle -->
                        <Ellipse Stroke="{TemplateBinding TrackColor}"
                                 StrokeThickness="{TemplateBinding StrokeThickness}"
                                 Fill="Transparent"/>
                        
                        <!-- Progress arc (PART_) -->
                        <Path x:Name="PART_Indicator"
                              Stroke="{TemplateBinding ProgressColor}"
                              StrokeThickness="{TemplateBinding StrokeThickness}"
                              StrokeLineCap="Round"
                              Fill="Transparent"/>
                        
                        <!-- Center text -->
                        <TextBlock x:Name="PART_ValueText"
                                   HorizontalAlignment="Center" VerticalAlignment="Center"
                                   FontSize="18" FontWeight="Bold"
                                   Foreground="{TemplateBinding Foreground}"
                                   Visibility="{TemplateBinding ShowText, Converter={...}}"/>
                    </Grid>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>
</ResourceDictionary>
```

---

## ขั้นตอนที่ 364: RoutedEvent

```csharp
// Custom RoutedEvent
public class RatingControl : Control
{
    // RoutedEvent declaration
    public static readonly RoutedEvent RatingChangedEvent =
        EventManager.RegisterRoutedEvent(
            nameof(RatingChanged),
            RoutingStrategy.Bubble,        // Bubble up the tree
            typeof(RoutedPropertyChangedEventHandler<double>),
            typeof(RatingControl));
    
    // CLR event wrapper
    public event RoutedPropertyChangedEventHandler<double> RatingChanged
    {
        add => AddHandler(RatingChangedEvent, value);
        remove => RemoveHandler(RatingChangedEvent, value);
    }
    
    public static readonly DependencyProperty RatingProperty =
        DependencyProperty.Register(
            nameof(Rating), typeof(double), typeof(RatingControl),
            new PropertyMetadata(0.0, OnRatingChanged));
    
    public double Rating
    {
        get => (double)GetValue(RatingProperty);
        set => SetValue(RatingProperty, value);
    }
    
    private static void OnRatingChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        var ctrl = (RatingControl)d;
        var args = new RoutedPropertyChangedEventArgs<double>(
            (double)e.OldValue, (double)e.NewValue, RatingChangedEvent);
        ctrl.RaiseEvent(args);
        ctrl.UpdateStars();
    }
    
    static RatingControl()
    {
        DefaultStyleKeyProperty.OverrideMetadata(typeof(RatingControl),
            new FrameworkPropertyMetadata(typeof(RatingControl)));
    }
    
    private List<System.Windows.Shapes.Path> _stars = new();
    
    public override void OnApplyTemplate()
    {
        base.OnApplyTemplate();
        for (int i = 1; i <= 5; i++)
        {
            var star = GetTemplateChild($"PART_Star{i}") as System.Windows.Shapes.Path;
            if (star != null)
            {
                int rating = i;
                star.MouseLeftButtonUp += (_, _) => Rating = rating;
                _stars.Add(star);
            }
        }
        UpdateStars();
    }
    
    private void UpdateStars()
    {
        for (int i = 0; i < _stars.Count; i++)
        {
            _stars[i].Fill = i < Rating ? Brushes.Gold : Brushes.LightGray;
        }
    }
}
```

---

## ขั้นตอนที่ 365: Attached Behavior

```csharp
// WatermarkBehavior.cs - เพิ่ม placeholder text ให้ TextBox
public static class WatermarkBehavior
{
    public static readonly DependencyProperty WatermarkProperty =
        DependencyProperty.RegisterAttached(
            "Watermark",
            typeof(string),
            typeof(WatermarkBehavior),
            new PropertyMetadata(null, OnWatermarkChanged));
    
    public static string GetWatermark(DependencyObject obj)
        => (string)obj.GetValue(WatermarkProperty);
    
    public static void SetWatermark(DependencyObject obj, string value)
        => obj.SetValue(WatermarkProperty, value);
    
    private static void OnWatermarkChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        if (d is not TextBox tb) return;
        
        tb.GotFocus -= UpdateWatermark;
        tb.LostFocus -= UpdateWatermark;
        
        if (e.NewValue != null)
        {
            tb.GotFocus += UpdateWatermark;
            tb.LostFocus += UpdateWatermark;
            UpdateWatermark(tb, null!);
        }
    }
    
    private static void UpdateWatermark(object sender, RoutedEventArgs e)
    {
        // This is simplified; real impl uses AdornerLayer
        var tb = (TextBox)sender;
        string? watermark = GetWatermark(tb);
        if (watermark == null) return;
        
        if (!tb.IsFocused && string.IsNullOrEmpty(tb.Text))
            tb.Text = watermark; // Would use adorner in production
    }
}
```

```xml
<!-- Using attached behavior -->
<TextBox local:WatermarkBehavior.Watermark="ค้นหาสินค้า..."/>
<TextBox local:WatermarkBehavior.Watermark="กรอกอีเมล"/>
```

---

## ขั้นตอนที่ 366: FlatButton UserControl

```xml
<!-- Controls/FlatButton.xaml -->
<UserControl x:Class="MyApp.Controls.FlatButton"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <UserControl.Resources>
        <Style x:Key="InnerBtn" TargetType="Button">
            <Setter Property="Background" Value="{Binding Background, ElementName=root}"/>
            <Setter Property="Foreground" Value="{Binding Foreground, ElementName=root}"/>
            <Setter Property="Padding" Value="{Binding Padding, ElementName=root}"/>
            <Setter Property="BorderThickness" Value="0"/>
            <Setter Property="FontSize" Value="{Binding FontSize, ElementName=root}"/>
            <Setter Property="Template">
                <Setter.Value>
                    <ControlTemplate TargetType="Button">
                        <Border x:Name="bd" Background="{TemplateBinding Background}"
                                CornerRadius="{Binding CornerRadius, ElementName=root}"
                                Padding="{TemplateBinding Padding}">
                            <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center"/>
                        </Border>
                        <ControlTemplate.Triggers>
                            <Trigger Property="IsMouseOver" Value="True">
                                <Setter TargetName="bd" Property="Opacity" Value="0.85"/>
                            </Trigger>
                            <Trigger Property="IsPressed" Value="True">
                                <Setter TargetName="bd" Property="Opacity" Value="0.7"/>
                            </Trigger>
                        </ControlTemplate.Triggers>
                    </ControlTemplate>
                </Setter.Value>
            </Setter>
        </Style>
    </UserControl.Resources>
    
    <Button x:Name="innerButton" Style="{StaticResource InnerBtn}"
            Content="{Binding Text, ElementName=root}"
            Command="{Binding Command, ElementName=root}"
            CommandParameter="{Binding CommandParameter, ElementName=root}"
            Click="InnerButton_Click"/>
</UserControl>
```

```csharp
// Controls/FlatButton.xaml.cs
public partial class FlatButton : UserControl
{
    public static readonly DependencyProperty TextProperty =
        DependencyProperty.Register(nameof(Text), typeof(string), typeof(FlatButton));
    
    public static readonly DependencyProperty CommandProperty =
        DependencyProperty.Register(nameof(Command), typeof(ICommand), typeof(FlatButton));
    
    public static readonly DependencyProperty CommandParameterProperty =
        DependencyProperty.Register(nameof(CommandParameter), typeof(object), typeof(FlatButton));
    
    public static readonly DependencyProperty CornerRadiusProperty =
        DependencyProperty.Register(nameof(CornerRadius), typeof(CornerRadius), typeof(FlatButton),
            new PropertyMetadata(new CornerRadius(6)));
    
    public static readonly RoutedEvent ClickEvent =
        EventManager.RegisterRoutedEvent(nameof(Click), RoutingStrategy.Bubble,
            typeof(RoutedEventHandler), typeof(FlatButton));
    
    public string Text
    {
        get => (string)GetValue(TextProperty);
        set => SetValue(TextProperty, value);
    }
    
    public ICommand Command
    {
        get => (ICommand)GetValue(CommandProperty);
        set => SetValue(CommandProperty, value);
    }
    
    public object CommandParameter
    {
        get => GetValue(CommandParameterProperty);
        set => SetValue(CommandParameterProperty, value);
    }
    
    public CornerRadius CornerRadius
    {
        get => (CornerRadius)GetValue(CornerRadiusProperty);
        set => SetValue(CornerRadiusProperty, value);
    }
    
    public event RoutedEventHandler Click
    {
        add => AddHandler(ClickEvent, value);
        remove => RemoveHandler(ClickEvent, value);
    }
    
    public FlatButton() => InitializeComponent();
    
    private void InnerButton_Click(object sender, RoutedEventArgs e)
    {
        RaiseEvent(new RoutedEventArgs(ClickEvent, this));
    }
}
```

---

## ขั้นตอนที่ 367-370: Component Library Demo

```xml
<!-- MainWindow.xaml - ใช้ controls จาก library -->
<Window ...>
    <StackPanel Margin="20" Spacing="20">
        <TextBlock Text="Custom Control Library Demo" FontSize="20" FontWeight="Bold"/>
        
        <!-- FlatButton -->
        <StackPanel Orientation="Horizontal" Spacing="10">
            <ctrl:FlatButton Text="Primary" Background="#3498DB" Foreground="White" Padding="16,8"/>
            <ctrl:FlatButton Text="Success" Background="#27AE60" Foreground="White" Padding="16,8"/>
            <ctrl:FlatButton Text="Danger" Background="#E74C3C" Foreground="White" Padding="16,8"/>
        </StackPanel>
        
        <!-- Circular progress bars -->
        <StackPanel Orientation="Horizontal" Spacing="20">
            <ctrl:CircularProgressBar Value="75" Maximum="100" ProgressColor="#3498DB" Width="100" Height="100"/>
            <ctrl:CircularProgressBar Value="45" Maximum="100" ProgressColor="#27AE60" Width="80" Height="80"/>
            <ctrl:CircularProgressBar Value="90" Maximum="100" ProgressColor="#E74C3C" Width="60" Height="60"/>
        </StackPanel>
        
        <!-- Rating control -->
        <ctrl:RatingControl Rating="3.5" RatingChanged="OnRatingChanged"/>
        
        <!-- TextBox with watermark -->
        <TextBox ctrl:WatermarkBehavior.Watermark="พิมพ์เพื่อค้นหา..." Width="250" HorizontalAlignment="Left"/>
    </StackPanel>
</Window>
```

---

## 📝 สรุป Part 37

| Concept | สิ่งสำคัญ |
|---------|----------|
| DependencyProperty | Register, FrameworkPropertyMetadata |
| CLR wrapper | ห้ามมี logic ใน get/set |
| RoutedEvent | Bubble/Tunnel routing |
| Generic.xaml | Default style/template |
| TemplatePart | PART_ naming convention |
| Attached Property | เพิ่ม property ให้ element อื่น |
| UserControl | ง่าย มี XAML |
| Custom Control | Lookless, styleable |

---

**ก่อนหน้า → [Part 36: WPF Navigation](part36-wpf-navigation.md)**  
**ต่อไป → [Part 38: WPF Resources & Localization](part38-wpf-resources.md)**
