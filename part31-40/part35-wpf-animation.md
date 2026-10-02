# Part 35: WPF Animation
## ขั้นตอนที่ 341-350: Animation และ Transitions

---

## 🎯 เป้าหมายของ Part นี้
- Storyboard และ Timeline
- DoubleAnimation, ColorAnimation
- BeginStoryboard ใน EventTrigger
- Easing functions
- KeyFrame Animations
- Trigger-based animations
- TransformAnimation (Scale, Rotate, Translate)
- โปรแกรม Animated Login Screen

---

## ขั้นตอนที่ 341: Storyboard พื้นฐาน

```xml
<!-- Simple fade-in animation -->
<Border x:Name="myBorder" Background="SteelBlue" Width="200" Height="100" Opacity="0">
    <Border.Triggers>
        <EventTrigger RoutedEvent="Border.Loaded">
            <BeginStoryboard>
                <Storyboard>
                    <DoubleAnimation
                        Storyboard.TargetName="myBorder"
                        Storyboard.TargetProperty="Opacity"
                        From="0" To="1"
                        Duration="0:0:1"
                        EasingFunction="CubicEase"/>
                </Storyboard>
            </BeginStoryboard>
        </EventTrigger>
    </Border.Triggers>
</Border>
```

---

## ขั้นตอนที่ 342: DoubleAnimation Properties

```xml
<Window.Resources>
    <!-- Reusable storyboard as resource -->
    <Storyboard x:Key="FadeIn">
        <DoubleAnimation Storyboard.TargetProperty="Opacity"
                         From="0" To="1" Duration="0:0:0.4">
            <DoubleAnimation.EasingFunction>
                <QuarticEase EasingMode="EaseOut"/>
            </DoubleAnimation.EasingFunction>
        </DoubleAnimation>
    </Storyboard>
    
    <Storyboard x:Key="SlideIn">
        <DoubleAnimation Storyboard.TargetProperty="(UIElement.RenderTransform).(TranslateTransform.X)"
                         From="-50" To="0" Duration="0:0:0.3">
            <DoubleAnimation.EasingFunction>
                <BackEase EasingMode="EaseOut" Amplitude="0.3"/>
            </DoubleAnimation.EasingFunction>
        </DoubleAnimation>
        <DoubleAnimation Storyboard.TargetProperty="Opacity"
                         From="0" To="1" Duration="0:0:0.3"/>
    </Storyboard>
    
    <!-- Pulse animation (AutoReverse + RepeatBehavior) -->
    <Storyboard x:Key="Pulse" RepeatBehavior="Forever" AutoReverse="True">
        <DoubleAnimation Storyboard.TargetProperty="Opacity"
                         From="1" To="0.3" Duration="0:0:0.8">
            <DoubleAnimation.EasingFunction>
                <SineEase EasingMode="EaseInOut"/>
            </DoubleAnimation.EasingFunction>
        </DoubleAnimation>
    </Storyboard>
</Window.Resources>

<!-- Element with transform ready for animation -->
<Border x:Name="panel" Opacity="0">
    <Border.RenderTransform>
        <TranslateTransform/>
    </Border.RenderTransform>
    <Border.Triggers>
        <EventTrigger RoutedEvent="Loaded">
            <BeginStoryboard Storyboard="{StaticResource SlideIn}"/>
        </EventTrigger>
    </Border.Triggers>
    <TextBlock Text="Hello WPF!" FontSize="20" Margin="20"/>
</Border>
```

---

## ขั้นตอนที่ 343: ColorAnimation และ Background

```xml
<Window.Resources>
    <!-- Color transition -->
    <Style x:Key="HoverButton" TargetType="Button">
        <Setter Property="Background" Value="#3498DB"/>
        <Setter Property="Foreground" Value="White"/>
        <Setter Property="BorderThickness" Value="0"/>
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="Button">
                    <Border x:Name="bd" Background="{TemplateBinding Background}" 
                            CornerRadius="6" Padding="{TemplateBinding Padding}">
                        <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center"/>
                    </Border>
                    <ControlTemplate.Triggers>
                        <EventTrigger RoutedEvent="MouseEnter">
                            <BeginStoryboard>
                                <Storyboard>
                                    <ColorAnimation
                                        Storyboard.TargetName="bd"
                                        Storyboard.TargetProperty="(Border.Background).(SolidColorBrush.Color)"
                                        To="#2980B9" Duration="0:0:0.2"/>
                                </Storyboard>
                            </BeginStoryboard>
                        </EventTrigger>
                        <EventTrigger RoutedEvent="MouseLeave">
                            <BeginStoryboard>
                                <Storyboard>
                                    <ColorAnimation
                                        Storyboard.TargetName="bd"
                                        Storyboard.TargetProperty="(Border.Background).(SolidColorBrush.Color)"
                                        To="#3498DB" Duration="0:0:0.2"/>
                                </Storyboard>
                            </BeginStoryboard>
                        </EventTrigger>
                    </ControlTemplate.Triggers>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>
</Window.Resources>

<Button Style="{StaticResource HoverButton}" Content="Hover me!" Padding="20,10" Width="150"/>
```

---

## ขั้นตอนที่ 344: Transform Animations

```xml
<!-- Rotate animation -->
<Border Width="60" Height="60" Background="#3498DB" CornerRadius="30">
    <Border.RenderTransformOrigin>0.5,0.5</Border.RenderTransformOrigin>
    <Border.RenderTransform>
        <RotateTransform/>
    </Border.RenderTransform>
    <Border.Triggers>
        <EventTrigger RoutedEvent="Loaded">
            <BeginStoryboard>
                <Storyboard RepeatBehavior="Forever">
                    <DoubleAnimation
                        Storyboard.TargetProperty="(UIElement.RenderTransform).(RotateTransform.Angle)"
                        From="0" To="360" Duration="0:0:2"/>
                </Storyboard>
            </BeginStoryboard>
        </EventTrigger>
    </Border.Triggers>
</Border>

<!-- Scale (zoom) on hover -->
<Border Width="100" Height="100" Background="#E74C3C" CornerRadius="8">
    <Border.RenderTransformOrigin>0.5,0.5</Border.RenderTransformOrigin>
    <Border.RenderTransform>
        <ScaleTransform ScaleX="1" ScaleY="1"/>
    </Border.RenderTransform>
    <Border.Triggers>
        <EventTrigger RoutedEvent="MouseEnter">
            <BeginStoryboard>
                <Storyboard>
                    <DoubleAnimation Storyboard.TargetProperty="RenderTransform.ScaleX" To="1.1" Duration="0:0:0.2"/>
                    <DoubleAnimation Storyboard.TargetProperty="RenderTransform.ScaleY" To="1.1" Duration="0:0:0.2"/>
                </Storyboard>
            </BeginStoryboard>
        </EventTrigger>
        <EventTrigger RoutedEvent="MouseLeave">
            <BeginStoryboard>
                <Storyboard>
                    <DoubleAnimation Storyboard.TargetProperty="RenderTransform.ScaleX" To="1" Duration="0:0:0.15"/>
                    <DoubleAnimation Storyboard.TargetProperty="RenderTransform.ScaleY" To="1" Duration="0:0:0.15"/>
                </Storyboard>
            </BeginStoryboard>
        </EventTrigger>
    </Border.Triggers>
</Border>
```

---

## ขั้นตอนที่ 345: KeyFrame Animation

```xml
<!-- DoubleAnimationUsingKeyFrames -->
<Border x:Name="ball" Width="40" Height="40" Background="#F1C40F" CornerRadius="20"
        Canvas.Left="0">
    <Border.Triggers>
        <EventTrigger RoutedEvent="Loaded">
            <BeginStoryboard>
                <Storyboard RepeatBehavior="Forever">
                    <!-- Bounce animation -->
                    <DoubleAnimationUsingKeyFrames
                        Storyboard.TargetName="ball"
                        Storyboard.TargetProperty="(Canvas.Top)">
                        <!-- SplineKeyFrame: control acceleration -->
                        <LinearDoubleKeyFrame KeyTime="0:0:0" Value="0"/>
                        <SplineDoubleKeyFrame KeyTime="0:0:0.5" Value="200"
                            KeySpline="0.2,0 0.8,1"/>
                        <SplineDoubleKeyFrame KeyTime="0:0:1" Value="0"
                            KeySpline="0.2,0 0.8,1"/>
                    </DoubleAnimationUsingKeyFrames>
                    
                    <!-- Horizontal movement -->
                    <DoubleAnimationUsingKeyFrames
                        Storyboard.TargetName="ball"
                        Storyboard.TargetProperty="(Canvas.Left)">
                        <LinearDoubleKeyFrame KeyTime="0:0:0" Value="0"/>
                        <LinearDoubleKeyFrame KeyTime="0:0:1" Value="200"/>
                    </DoubleAnimationUsingKeyFrames>
                </Storyboard>
            </BeginStoryboard>
        </EventTrigger>
    </Border.Triggers>
</Border>
```

---

## ขั้นตอนที่ 346: Storyboard ใน code-behind

```csharp
// Start animations from C# code
private void StartAnimation()
{
    var sb = new Storyboard();
    
    var fadeIn = new DoubleAnimation(0, 1, TimeSpan.FromMilliseconds(400));
    fadeIn.EasingFunction = new QuarticEase { EasingMode = EasingMode.EaseOut };
    Storyboard.SetTarget(fadeIn, myPanel);
    Storyboard.SetTargetProperty(fadeIn, new PropertyPath(OpacityProperty));
    
    var slideIn = new DoubleAnimation(-30, 0, TimeSpan.FromMilliseconds(400));
    slideIn.EasingFunction = new CubicEase { EasingMode = EasingMode.EaseOut };
    Storyboard.SetTarget(slideIn, myPanel);
    Storyboard.SetTargetProperty(slideIn, 
        new PropertyPath("(UIElement.RenderTransform).(TranslateTransform.Y)"));
    
    sb.Children.Add(fadeIn);
    sb.Children.Add(slideIn);
    sb.Begin();
}

// Await animation completion
private async Task AnimateOutAsync(UIElement element)
{
    var tcs = new TaskCompletionSource<bool>();
    var sb = new Storyboard();
    
    var fadeOut = new DoubleAnimation(1, 0, TimeSpan.FromMilliseconds(300));
    Storyboard.SetTarget(fadeOut, element);
    Storyboard.SetTargetProperty(fadeOut, new PropertyPath(OpacityProperty));
    sb.Children.Add(fadeOut);
    
    sb.Completed += (_, _) => tcs.SetResult(true);
    sb.Begin();
    
    await tcs.Task;
    element.Visibility = Visibility.Collapsed;
}
```

---

## ขั้นตอนที่ 347-350: Animated Login Screen

```xml
<!-- AnimatedLogin.xaml -->
<Window x:Class="AnimatedLogin.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Login" Height="500" Width="800"
        WindowStyle="None" AllowsTransparency="True"
        Background="Transparent" WindowStartupLocation="CenterScreen">
    
    <Window.Resources>
        <Storyboard x:Key="LoadIn">
            <DoubleAnimation Storyboard.TargetName="loginCard"
                             Storyboard.TargetProperty="Opacity"
                             From="0" To="1" Duration="0:0:0.5"/>
            <DoubleAnimation Storyboard.TargetName="loginCard"
                             Storyboard.TargetProperty="(UIElement.RenderTransform).(TranslateTransform.Y)"
                             From="30" To="0" Duration="0:0:0.5">
                <DoubleAnimation.EasingFunction>
                    <BackEase EasingMode="EaseOut" Amplitude="0.4"/>
                </DoubleAnimation.EasingFunction>
            </DoubleAnimation>
        </Storyboard>
        
        <Style x:Key="LoginInput" TargetType="TextBox">
            <Setter Property="FontSize" Value="14"/>
            <Setter Property="Padding" Value="12,10"/>
            <Setter Property="BorderBrush" Value="#E0E0E0"/>
            <Setter Property="BorderThickness" Value="0,0,0,2"/>
            <Setter Property="Background" Value="Transparent"/>
            <Setter Property="Margin" Value="0,0,0,16"/>
            <Style.Triggers>
                <Trigger Property="IsFocused" Value="True">
                    <Setter Property="BorderBrush" Value="#3498DB"/>
                </Trigger>
            </Style.Triggers>
        </Style>
        
        <Style x:Key="LoginBtn" TargetType="Button">
            <Setter Property="Background" Value="#3498DB"/>
            <Setter Property="Foreground" Value="White"/>
            <Setter Property="FontSize" Value="15"/>
            <Setter Property="FontWeight" Value="SemiBold"/>
            <Setter Property="Padding" Value="0,14"/>
            <Setter Property="BorderThickness" Value="0"/>
            <Setter Property="Cursor" Value="Hand"/>
            <Setter Property="Template">
                <Setter.Value>
                    <ControlTemplate TargetType="Button">
                        <Border x:Name="bd" Background="{TemplateBinding Background}" CornerRadius="6"
                                Padding="{TemplateBinding Padding}">
                            <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center"/>
                        </Border>
                        <ControlTemplate.Triggers>
                            <EventTrigger RoutedEvent="MouseEnter">
                                <BeginStoryboard>
                                    <Storyboard>
                                        <ColorAnimation Storyboard.TargetName="bd"
                                            Storyboard.TargetProperty="(Border.Background).(SolidColorBrush.Color)"
                                            To="#2980B9" Duration="0:0:0.2"/>
                                    </Storyboard>
                                </BeginStoryboard>
                            </EventTrigger>
                            <EventTrigger RoutedEvent="MouseLeave">
                                <BeginStoryboard>
                                    <Storyboard>
                                        <ColorAnimation Storyboard.TargetName="bd"
                                            Storyboard.TargetProperty="(Border.Background).(SolidColorBrush.Color)"
                                            To="#3498DB" Duration="0:0:0.2"/>
                                    </Storyboard>
                                </BeginStoryboard>
                            </EventTrigger>
                        </ControlTemplate.Triggers>
                    </ControlTemplate>
                </Setter.Value>
            </Setter>
        </Style>
    </Window.Resources>
    
    <!-- Animated background -->
    <Grid>
        <!-- Gradient background with animated dots (simulated) -->
        <Border>
            <Border.Background>
                <LinearGradientBrush StartPoint="0,0" EndPoint="1,1">
                    <GradientStop Color="#1A237E" Offset="0"/>
                    <GradientStop Color="#0D47A1" Offset="0.5"/>
                    <GradientStop Color="#1565C0" Offset="1"/>
                </LinearGradientBrush>
            </Border.Background>
        </Border>
        
        <!-- Animated circles in background -->
        <Canvas>
            <Ellipse x:Name="circle1" Canvas.Left="-50" Canvas.Top="-50" 
                     Width="300" Height="300" Opacity="0.1">
                <Ellipse.Fill>
                    <RadialGradientBrush>
                        <GradientStop Color="#64B5F6" Offset="0"/>
                        <GradientStop Color="Transparent" Offset="1"/>
                    </RadialGradientBrush>
                </Ellipse.Fill>
                <Ellipse.Triggers>
                    <EventTrigger RoutedEvent="Loaded">
                        <BeginStoryboard>
                            <Storyboard RepeatBehavior="Forever" AutoReverse="True">
                                <DoubleAnimation Storyboard.TargetProperty="(Canvas.Left)" 
                                                 From="-50" To="50" Duration="0:0:4">
                                    <DoubleAnimation.EasingFunction>
                                        <SineEase EasingMode="EaseInOut"/>
                                    </DoubleAnimation.EasingFunction>
                                </DoubleAnimation>
                            </Storyboard>
                        </BeginStoryboard>
                    </EventTrigger>
                </Ellipse.Triggers>
            </Ellipse>
        </Canvas>
        
        <!-- Login card -->
        <Border x:Name="loginCard" Opacity="0" Width="380" HorizontalAlignment="Center"
                VerticalAlignment="Center" Background="White" CornerRadius="16" Padding="40">
            <Border.RenderTransform>
                <TranslateTransform/>
            </Border.RenderTransform>
            <Border.Triggers>
                <EventTrigger RoutedEvent="Loaded">
                    <BeginStoryboard Storyboard="{StaticResource LoadIn}"/>
                </EventTrigger>
            </Border.Triggers>
            <Border.Effect>
                <DropShadowEffect BlurRadius="30" Opacity="0.3" ShadowDepth="0"/>
            </Border.Effect>
            
            <StackPanel>
                <!-- Logo -->
                <Border Width="60" Height="60" Background="#3498DB" CornerRadius="12"
                        HorizontalAlignment="Center" Margin="0,0,0,20">
                    <TextBlock Text="🔐" FontSize="28" HorizontalAlignment="Center" VerticalAlignment="Center"/>
                </Border>
                
                <TextBlock Text="ยินดีต้อนรับ" FontSize="22" FontWeight="Bold" HorizontalAlignment="Center"/>
                <TextBlock Text="เข้าสู่ระบบเพื่อดำเนินการต่อ" Foreground="Gray" 
                           HorizontalAlignment="Center" Margin="0,4,0,28"/>
                
                <!-- Error message -->
                <Border x:Name="errorPanel" Background="#FDECEA" CornerRadius="6" Padding="12,8"
                        Margin="0,0,0,16" Visibility="Collapsed" Opacity="0">
                    <TextBlock x:Name="txtError" Text="ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง"
                               Foreground="#C62828" FontSize="12"/>
                </Border>
                
                <TextBox x:Name="txtUsername" Style="{StaticResource LoginInput}"
                         Tag="อีเมลหรือชื่อผู้ใช้"/>
                <PasswordBox x:Name="pwdPassword" FontSize="14" Padding="12,10"
                             BorderBrush="#E0E0E0" BorderThickness="0,0,0,2"
                             Margin="0,0,0,24"/>
                
                <Button x:Name="btnLogin" Style="{StaticResource LoginBtn}" Content="เข้าสู่ระบบ"
                        Click="BtnLogin_Click"/>
                
                <TextBlock Text="ลืมรหัสผ่าน?" HorizontalAlignment="Center" 
                           Foreground="#3498DB" Margin="0,16,0,0" Cursor="Hand"/>
            </StackPanel>
        </Border>
        
        <!-- Close button -->
        <Button Content="✕" HorizontalAlignment="Right" VerticalAlignment="Top"
                Margin="12" Background="Transparent" BorderThickness="0"
                Foreground="White" FontSize="16" Click="BtnClose_Click"/>
    </Grid>
</Window>
```

```csharp
// AnimatedLogin.xaml.cs
using System.Windows;
using System.Windows.Media.Animation;

namespace AnimatedLogin;

public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();
        MouseLeftButtonDown += (_, _) => DragMove();
    }
    
    private async void BtnLogin_Click(object sender, RoutedEventArgs e)
    {
        var username = txtUsername.Text.Trim();
        var password = pwdPassword.Password;
        
        if (string.IsNullOrWhiteSpace(username) || string.IsNullOrWhiteSpace(password))
        {
            await ShowError("กรุณากรอกข้อมูลให้ครบถ้วน");
            return;
        }
        
        btnLogin.IsEnabled = false;
        btnLogin.Content = "กำลังตรวจสอบ...";
        
        await Task.Delay(1500); // simulate API call
        
        if (username == "admin" && password == "1234")
        {
            await HideCard();
            var home = new HomeWindow();
            home.Show();
            Close();
        }
        else
        {
            await ShowError("ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง");
            btnLogin.IsEnabled = true;
            btnLogin.Content = "เข้าสู่ระบบ";
            pwdPassword.Focus();
            pwdPassword.SelectAll();
        }
    }
    
    private async Task ShowError(string message)
    {
        txtError.Text = message;
        errorPanel.Visibility = Visibility.Visible;
        
        var tcs = new TaskCompletionSource<bool>();
        var sb = new Storyboard();
        var anim = new DoubleAnimation(0, 1, TimeSpan.FromMilliseconds(300));
        Storyboard.SetTarget(anim, errorPanel);
        Storyboard.SetTargetProperty(anim, new PropertyPath(OpacityProperty));
        sb.Children.Add(anim);
        sb.Completed += (_, _) => tcs.SetResult(true);
        sb.Begin();
        await tcs.Task;
    }
    
    private async Task HideCard()
    {
        var tcs = new TaskCompletionSource<bool>();
        var sb = new Storyboard();
        
        var fadeOut = new DoubleAnimation(1, 0, TimeSpan.FromMilliseconds(400));
        Storyboard.SetTarget(fadeOut, loginCard);
        Storyboard.SetTargetProperty(fadeOut, new PropertyPath(OpacityProperty));
        
        var scale = new DoubleAnimation(1, 0.9, TimeSpan.FromMilliseconds(400));
        scale.EasingFunction = new QuarticEase { EasingMode = EasingMode.EaseIn };
        Storyboard.SetTarget(scale, loginCard);
        Storyboard.SetTargetProperty(scale, new PropertyPath("RenderTransform.ScaleX"));
        
        sb.Children.Add(fadeOut);
        sb.Completed += (_, _) => tcs.SetResult(true);
        sb.Begin();
        await tcs.Task;
    }
    
    private void BtnClose_Click(object sender, RoutedEventArgs e) => Close();
}
```

---

## 📝 สรุป Part 35

| Concept | ใช้งาน |
|---------|--------|
| Storyboard | Container สำหรับ animations |
| DoubleAnimation | Animate numeric properties |
| ColorAnimation | Animate color properties |
| EventTrigger | Start animation on event |
| EasingFunction | ควบคุมการเร่ง/ชะลอ |
| KeyFrame | กำหนด state ณ จุดเวลาต่างๆ |
| RenderTransform | Scale, Rotate, Translate |

**Easing functions ที่นิยม:**
- `QuarticEase EaseOut` - เร็วแล้วชะลอ (UI เปิด)
- `BackEase EaseOut` - overshoot (bounce effect)
- `SineEase EaseInOut` - คลื่น sin (infinite loops)
- `CubicEase EaseIn` - ช้าแล้วเร็ว (UI ปิด)

---

**ก่อนหน้า → [Part 34: WPF Styles](part34-wpf-styles.md)**  
**ต่อไป → [Part 36: WPF Navigation](part36-wpf-navigation.md)**
