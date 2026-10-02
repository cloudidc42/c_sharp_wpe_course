# Part 77: WPF Advanced Techniques
## ขั้นตอนที่ 761-770: WPF ระดับสูง

---

## 🎯 เป้าหมายของ Part นี้
- Custom Controls & Control Templates
- Styles, Triggers, Visual States
- Animations & Storyboards
- Data Virtualization (VirtualizingStackPanel)
- Drag & Drop
- Custom Attached Properties & Behaviors
- WPF Performance tuning
- Interop with Win32

---

## ขั้นตอนที่ 761: Custom Control Template

```xml
<!-- Custom Button with ControlTemplate -->
<Style TargetType="Button" x:Key="GradientButton">
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="Button">
                <Border x:Name="PART_Border"
                        Background="{TemplateBinding Background}"
                        BorderBrush="{TemplateBinding BorderBrush}"
                        BorderThickness="{TemplateBinding BorderThickness}"
                        CornerRadius="8"
                        Padding="{TemplateBinding Padding}">
                    <Border.Background>
                        <LinearGradientBrush StartPoint="0,0" EndPoint="0,1">
                            <GradientStop Color="#4A90E2" Offset="0"/>
                            <GradientStop Color="#357ABD" Offset="1"/>
                        </LinearGradientBrush>
                    </Border.Background>
                    
                    <ContentPresenter HorizontalAlignment="Center"
                                      VerticalAlignment="Center"/>
                    
                    <VisualStateManager.VisualStateGroups>
                        <VisualStateGroup Name="CommonStates">
                            <VisualState Name="Normal"/>
                            <VisualState Name="MouseOver">
                                <Storyboard>
                                    <ColorAnimation Storyboard.TargetName="PART_Border"
                                                    Storyboard.TargetProperty="(Border.Background).(SolidColorBrush.Color)"
                                                    To="#5BA0F2" Duration="0:0:0.15"/>
                                </Storyboard>
                            </VisualState>
                            <VisualState Name="Pressed">
                                <Storyboard>
                                    <DoubleAnimation Storyboard.TargetName="PART_Border"
                                                     Storyboard.TargetProperty="Opacity"
                                                     To="0.8" Duration="0:0:0.1"/>
                                </Storyboard>
                            </VisualState>
                            <VisualState Name="Disabled">
                                <Storyboard>
                                    <DoubleAnimation Storyboard.TargetName="PART_Border"
                                                     Storyboard.TargetProperty="Opacity"
                                                     To="0.4" Duration="0"/>
                                </Storyboard>
                            </VisualState>
                        </VisualStateGroup>
                    </VisualStateManager.VisualStateGroups>
                </Border>
                
                <ControlTemplate.Triggers>
                    <Trigger Property="IsMouseOver" Value="True">
                        <Setter TargetName="PART_Border" Property="Effect">
                            <Setter.Value>
                                <DropShadowEffect BlurRadius="8" Opacity="0.4"/>
                            </Setter.Value>
                        </Setter>
                    </Trigger>
                </ControlTemplate.Triggers>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
    <Setter Property="Foreground" Value="White"/>
    <Setter Property="Padding" Value="16,8"/>
    <Setter Property="FontWeight" Value="SemiBold"/>
</Style>
```

---

## ขั้นตอนที่ 762: Custom UserControl

```csharp
// CustomRatingControl: star rating input
public class RatingControl : UserControl
{
    // DependencyProperty for MVVM binding
    public static readonly DependencyProperty ValueProperty =
        DependencyProperty.Register(
            nameof(Value),
            typeof(int),
            typeof(RatingControl),
            new FrameworkPropertyMetadata(
                0,
                FrameworkPropertyMetadataOptions.BindsTwoWayByDefault,
                OnValueChanged));
    
    public static readonly DependencyProperty MaxRatingProperty =
        DependencyProperty.Register(
            nameof(MaxRating), typeof(int), typeof(RatingControl),
            new PropertyMetadata(5, OnMaxRatingChanged));
    
    public int Value
    {
        get => (int)GetValue(ValueProperty);
        set => SetValue(ValueProperty, value);
    }
    
    public int MaxRating
    {
        get => (int)GetValue(MaxRatingProperty);
        set => SetValue(MaxRatingProperty, value);
    }
    
    private StackPanel? _starsPanel;
    
    public override void OnApplyTemplate()
    {
        base.OnApplyTemplate();
        _starsPanel = GetTemplateChild("PART_StarsPanel") as StackPanel;
        RebuildStars();
    }
    
    private void RebuildStars()
    {
        if (_starsPanel == null) return;
        _starsPanel.Children.Clear();
        
        for (int i = 1; i <= MaxRating; i++)
        {
            var star = CreateStar(i);
            _starsPanel.Children.Add(star);
        }
        UpdateStarStates();
    }
    
    private Button CreateStar(int starIndex)
    {
        var button = new Button
        {
            Content = "★",
            FontSize = 24,
            Padding = new Thickness(2),
            Background = Brushes.Transparent,
            BorderThickness = new Thickness(0),
            Tag = starIndex,
            Cursor = Cursors.Hand
        };
        button.Click += (_, _) => Value = starIndex;
        button.MouseEnter += (s, _) => HighlightUpTo(((Button)s).Tag is int i ? i : 0);
        button.MouseLeave += (_, _) => UpdateStarStates();
        return button;
    }
    
    private void HighlightUpTo(int index)
    {
        foreach (Button star in _starsPanel!.Children)
        {
            var i = (int)(star.Tag ?? 0);
            star.Foreground = i <= index ? Brushes.Gold : Brushes.LightGray;
        }
    }
    
    private void UpdateStarStates() => HighlightUpTo(Value);
    
    private static void OnValueChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
        => ((RatingControl)d).UpdateStarStates();
    
    private static void OnMaxRatingChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
        => ((RatingControl)d).RebuildStars();
}
```

```xml
<!-- Usage in XAML -->
<local:RatingControl Value="{Binding ProductRating, Mode=TwoWay}" MaxRating="5"/>
```

---

## ขั้นตอนที่ 763: Animations & Storyboards

```xml
<!-- Page transition animation -->
<Page.Resources>
    <Storyboard x:Key="FadeIn">
        <DoubleAnimation Storyboard.TargetProperty="Opacity"
                         From="0" To="1" Duration="0:0:0.3"
                         EasingFunction="{StaticResource EaseOut}"/>
        <DoubleAnimation Storyboard.TargetProperty="(UIElement.RenderTransform).(TranslateTransform.Y)"
                         From="20" To="0" Duration="0:0:0.3"
                         EasingFunction="{StaticResource EaseOut}"/>
    </Storyboard>
    
    <Storyboard x:Key="Shake">
        <DoubleAnimationUsingKeyFrames Storyboard.TargetProperty="(UIElement.RenderTransform).(TranslateTransform.X)">
            <LinearDoubleKeyFrame KeyTime="0:0:0" Value="0"/>
            <LinearDoubleKeyFrame KeyTime="0:0:0.05" Value="-10"/>
            <LinearDoubleKeyFrame KeyTime="0:0:0.1" Value="10"/>
            <LinearDoubleKeyFrame KeyTime="0:0:0.15" Value="-8"/>
            <LinearDoubleKeyFrame KeyTime="0:0:0.2" Value="8"/>
            <LinearDoubleKeyFrame KeyTime="0:0:0.25" Value="-5"/>
            <LinearDoubleKeyFrame KeyTime="0:0:0.3" Value="5"/>
            <LinearDoubleKeyFrame KeyTime="0:0:0.35" Value="0"/>
        </DoubleAnimationUsingKeyFrames>
    </Storyboard>
</Page.Resources>

<Grid>
    <Grid.RenderTransform>
        <TranslateTransform/>
    </Grid.RenderTransform>
    <Grid.Triggers>
        <EventTrigger RoutedEvent="Loaded">
            <BeginStoryboard Storyboard="{StaticResource FadeIn}"/>
        </EventTrigger>
    </Grid.Triggers>
</Grid>
```

```csharp
// Code-behind animation
public static class AnimationHelper
{
    public static Task FadeInAsync(UIElement element, double durationSeconds = 0.3)
    {
        var tcs = new TaskCompletionSource();
        var animation = new DoubleAnimation(0, 1, TimeSpan.FromSeconds(durationSeconds))
        {
            EasingFunction = new CubicEase { EasingMode = EasingMode.EaseOut }
        };
        animation.Completed += (_, _) => tcs.SetResult();
        element.BeginAnimation(UIElement.OpacityProperty, animation);
        return tcs.Task;
    }
    
    public static void ShakeElement(FrameworkElement element)
    {
        var transform = new TranslateTransform();
        element.RenderTransform = transform;
        
        var animation = new DoubleAnimationUsingKeyFrames();
        foreach (var (time, value) in new (double, double)[]
            { (0, 0), (.05, -10), (.1, 10), (.15, -8), (.2, 8), (.25, -5), (.3, 5), (.35, 0) })
        {
            animation.KeyFrames.Add(new LinearDoubleKeyFrame(value, KeyTime.FromTimeSpan(TimeSpan.FromSeconds(time))));
        }
        
        transform.BeginAnimation(TranslateTransform.XProperty, animation);
    }
}
```

---

## ขั้นตอนที่ 764: Attached Properties & Behaviors

```csharp
// Attached Property: add behavior without subclassing
public static class FocusBehavior
{
    public static readonly DependencyProperty IsFocusedProperty =
        DependencyProperty.RegisterAttached(
            "IsFocused",
            typeof(bool),
            typeof(FocusBehavior),
            new PropertyMetadata(false, OnIsFocusedChanged));
    
    public static bool GetIsFocused(DependencyObject obj) => (bool)obj.GetValue(IsFocusedProperty);
    public static void SetIsFocused(DependencyObject obj, bool value) => obj.SetValue(IsFocusedProperty, value);
    
    private static void OnIsFocusedChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        if (d is UIElement element && (bool)e.NewValue)
            element.Focus();
    }
}

// TextBox PasswordHelper
public static class PasswordHelper
{
    public static readonly DependencyProperty PasswordProperty =
        DependencyProperty.RegisterAttached(
            "Password",
            typeof(string),
            typeof(PasswordHelper),
            new FrameworkPropertyMetadata(string.Empty,
                FrameworkPropertyMetadataOptions.BindsTwoWayByDefault,
                OnPasswordPropertyChanged));
    
    private static bool _updating;
    
    public static string GetPassword(DependencyObject obj) => (string)obj.GetValue(PasswordProperty);
    public static void SetPassword(DependencyObject obj, string value) => obj.SetValue(PasswordProperty, value);
    
    private static void OnPasswordPropertyChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        if (d is PasswordBox pb && !_updating)
            pb.Password = (string)(e.NewValue ?? "");
    }
    
    public static void Attach(PasswordBox pb)
    {
        pb.PasswordChanged += (s, _) =>
        {
            _updating = true;
            SetPassword((PasswordBox)s, ((PasswordBox)s).Password);
            _updating = false;
        };
    }
}
```

```xml
<!-- Usage -->
<TextBox local:FocusBehavior.IsFocused="{Binding ShouldFocusSearch}"/>
<PasswordBox local:PasswordHelper.Password="{Binding Password}"/>
```

---

## ขั้นตอนที่ 765-770: Data Virtualization & Performance

```csharp
// VirtualizingCollectionView: load items on demand
// For huge lists (100k+ items)
public class VirtualizingCollection<T> : IList<T>, IList, INotifyCollectionChanged, INotifyPropertyChanged
{
    private readonly Func<int, int, Task<IEnumerable<T>>> _fetchPage;
    private readonly int _pageSize;
    private readonly Dictionary<int, T[]> _pages = new();
    private int _count;
    
    public VirtualizingCollection(int totalCount, int pageSize, Func<int, int, Task<IEnumerable<T>>> fetchPage)
    {
        _count = totalCount;
        _pageSize = pageSize;
        _fetchPage = fetchPage;
    }
    
    public int Count => _count;
    
    public T this[int index]
    {
        get
        {
            var pageIndex = index / _pageSize;
            var offset = index % _pageSize;
            
            if (_pages.TryGetValue(pageIndex, out var page))
                return page[offset];
            
            // Start async load and return placeholder
            _ = LoadPageAsync(pageIndex);
            return default!;
        }
        set => throw new NotSupportedException();
    }
    
    private async Task LoadPageAsync(int pageIndex)
    {
        if (_pages.ContainsKey(pageIndex)) return;
        
        var items = (await _fetchPage(pageIndex * _pageSize, _pageSize)).ToArray();
        _pages[pageIndex] = items;
        
        // Notify collection changed for loaded page
        Application.Current.Dispatcher.Invoke(() =>
        {
            CollectionChanged?.Invoke(this, new NotifyCollectionChangedEventArgs(
                NotifyCollectionChangedAction.Replace, 
                items.Cast<object>().ToList(),
                Enumerable.Repeat<object?>(null, items.Length).ToList(),
                pageIndex * _pageSize));
        });
    }
    
    public event NotifyCollectionChangedEventHandler? CollectionChanged;
    public event PropertyChangedEventHandler? PropertyChanged;
    
    // IList<T> members...
    public bool IsReadOnly => true;
    public IEnumerator<T> GetEnumerator() => Enumerable.Range(0, _count).Select(i => this[i]).GetEnumerator();
    System.Collections.IEnumerator System.Collections.IEnumerable.GetEnumerator() => GetEnumerator();
    public void Add(T item) => throw new NotSupportedException();
    public void Clear() => throw new NotSupportedException();
    public bool Contains(T item) => false;
    public void CopyTo(T[] array, int arrayIndex) { }
    public bool Remove(T item) => throw new NotSupportedException();
    public int IndexOf(T item) => -1;
    public void Insert(int index, T item) => throw new NotSupportedException();
    public void RemoveAt(int index) => throw new NotSupportedException();
    
    // IList members
    public bool IsFixedSize => true;
    public bool IsSynchronized => false;
    public object SyncRoot => this;
    public int Add(object? value) => throw new NotSupportedException();
    public bool Contains(object? value) => false;
    public int IndexOf(object? value) => -1;
    public void Insert(int index, object? value) => throw new NotSupportedException();
    public void Remove(object? value) => throw new NotSupportedException();
    object? System.Collections.IList.this[int index] { get => this[index]; set => throw new NotSupportedException(); }
    public void CopyTo(Array array, int index) { }
}

// WPF performance tips
// 1. Enable virtualization
// <ListView VirtualizingPanel.IsVirtualizing="True"
//           VirtualizingPanel.VirtualizationMode="Recycling"
//           ScrollViewer.IsDeferredScrollingEnabled="True"/>

// 2. Freeze unchanging resources
var brush = new SolidColorBrush(Colors.Blue);
brush.Freeze(); // immutable = no property change tracking

// 3. Use DrawingContext for custom rendering
public class FastCanvasControl : FrameworkElement
{
    protected override void OnRender(DrawingContext dc)
    {
        var pen = new Pen(Brushes.Blue, 1);
        pen.Freeze();
        
        for (int i = 0; i < 10000; i++)
        {
            dc.DrawEllipse(Brushes.Red, pen, new Point(i % 100 * 10, i / 100 * 10), 3, 3);
        }
    }
}
```

---

## 📝 สรุป Part 77

| Technique | ใช้เมื่อ |
|-----------|---------|
| ControlTemplate | ปรับรูปร่าง control ทั้งหมด |
| Style + Triggers | ปรับรูปแบบตาม state |
| Attached Properties | เพิ่ม behavior โดยไม่ subclass |
| Animations | ทำให้ UI เคลื่อนไหว |
| VirtualizingStackPanel | รายการขนาดใหญ่ |
| DrawingContext | Render ด้วย code (performance) |

---

**ก่อนหน้า → [Part 76: Functional C#](part76-functional-csharp.md)**  
**ต่อไป → [Part 78: WinForms Advanced](part78-winforms-advanced.md)**
