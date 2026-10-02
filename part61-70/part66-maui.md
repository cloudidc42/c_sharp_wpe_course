# Part 66: .NET MAUI (Multi-platform App UI) — Steps 651–660

> **ภาษา:** ไทย (คำอธิบาย) + English (โค้ด)
> **ระดับ:** Intermediate → Advanced
> **เวลาเรียน:** ~6–8 ชั่วโมง

---

## ภาพรวม

.NET MAUI (Multi-platform App UI) คือ framework สำหรับสร้าง UI application ที่รัน
บนหลายแพลตฟอร์มจาก codebase เดียว ได้แก่ iOS, Android, Windows และ macOS
ใน Part นี้เราจะเรียนตั้งแต่พื้นฐานจนสร้าง Note-taking App ที่ใช้งานได้จริง

---

## Step 651: MAUI Overview

### .NET MAUI คืออะไร?

.NET MAUI เป็น evolution จาก Xamarin.Forms ที่ Microsoft สร้างขึ้นเพื่อให้นักพัฒนา
C# เขียน application ครั้งเดียวแล้วสามารถ deploy ไปยัง 4 แพลตฟอร์มหลักได้:

| แพลตฟอร์ม | Native Runtime | UI Engine |
|------------|----------------|-----------|
| **iOS**    | .NET iOS       | UIKit / SwiftUI |
| **Android**| .NET Android   | Android Views |
| **Windows**| WinUI 3        | Win32 / UWP |
| **macOS**  | .NET Mac Catalyst | AppKit |

ข้อดีหลักคือ **"Single codebase, native performance"** — โค้ดที่เราเขียนใน C#
จะถูก compile ลงไปเป็น native binary สำหรับแต่ละแพลตฟอร์ม ไม่ใช่ web wrapper
เหมือน Electron หรือ Cordova

### ความแตกต่างจาก Xamarin.Forms

```
Xamarin.Forms → .NET MAUI

- Separate assemblies  → Single project, multi-target
- Renderers            → Handlers (ง่ายกว่า, เร็วกว่า)
- Effects              → Platform behaviors
- DependencyService    → .NET DI built-in
- MessagingCenter      → CommunityToolkit.Mvvm Messenger
```

### เปรียบเทียบ WPF vs MAUI

```csharp
// WPF — Windows เท่านั้น
// ไม่มีทางรัน iOS หรือ Android ได้

<Window x:Class="WpfApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation">
    <Grid>
        <TextBlock Text="Hello WPF" />
    </Grid>
</Window>
```

```xml
<!-- MAUI — รัน iOS, Android, Windows, macOS ได้ทันที -->
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="MauiApp.MainPage">
    <Label Text="Hello MAUI" 
           HorizontalOptions="Center"
           VerticalOptions="Center"/>
</ContentPage>
```

**เมื่อไรควรใช้ WPF:**
- App ที่ต้องใช้ Windows-specific feature เช่น COM interop, WMI
- Legacy codebase ที่มี WPF อยู่แล้ว
- Desktop-only และต้องการ mature ecosystem

**เมื่อไรควรใช้ MAUI:**
- ต้องการ cross-platform (iOS + Android + Desktop)
- Project ใหม่ที่ต้องการ mobile support
- B2B enterprise apps ที่ต้องรันหลายระบบ

### โครงสร้าง Project

```
MyMauiApp/
├── MauiProgram.cs            ← Entry point, DI setup
├── App.xaml / App.xaml.cs    ← Application lifecycle
├── AppShell.xaml             ← Navigation structure
├── MainPage.xaml             ← Default page
├── Platforms/
│   ├── Android/
│   │   ├── AndroidManifest.xml
│   │   └── MainActivity.cs
│   ├── iOS/
│   │   ├── Info.plist
│   │   └── AppDelegate.cs
│   ├── Windows/
│   │   └── Package.appxmanifest
│   └── MacCatalyst/
│       └── Info.plist
├── Resources/
│   ├── AppIcon/              ← App icons (auto-resize)
│   ├── Fonts/                ← Custom fonts
│   ├── Images/               ← Images (SVG auto-convert)
│   ├── Raw/                  ← Raw files (JSON, HTML, etc.)
│   └── Styles/
│       └── Colors.xaml       ← Global color resources
└── ViewModels/
    └── MainViewModel.cs
```

### MauiProgram.cs — Entry Point

```csharp
using Microsoft.Extensions.Logging;

namespace MyMauiApp;

public static class MauiProgram
{
    public static MauiApp CreateMauiApp()
    {
        var builder = MauiApp.CreateBuilder();
        
        builder
            .UseMauiApp<App>()
            .ConfigureFonts(fonts =>
            {
                fonts.AddFont("OpenSans-Regular.ttf", "OpenSansRegular");
                fonts.AddFont("OpenSans-Semibold.ttf", "OpenSansSemibold");
            });

        // ลงทะเบียน Services
        builder.Services.AddSingleton<DatabaseService>();
        builder.Services.AddTransient<NotesViewModel>();
        builder.Services.AddTransient<NotesPage>();

#if DEBUG
        builder.Logging.AddDebug();
#endif

        return builder.Build();
    }
}
```

### Shell Navigation

Shell คือ system สำหรับจัดการ navigation ใน MAUI ทำให้ลด boilerplate code ลงมาก

```xml
<!-- AppShell.xaml -->
<Shell xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
       xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
       xmlns:views="clr-namespace:MyMauiApp.Views"
       x:Class="MyMauiApp.AppShell"
       Title="MyMauiApp">

    <!-- Tab bar navigation -->
    <TabBar>
        <ShellContent Title="Notes"
                      Icon="notes.png"
                      ContentTemplate="{DataTemplate views:NotesPage}"
                      Route="notes"/>
        <ShellContent Title="Settings"
                      Icon="settings.png"
                      ContentTemplate="{DataTemplate views:SettingsPage}"
                      Route="settings"/>
    </TabBar>

</Shell>
```

```csharp
// AppShell.xaml.cs — ลงทะเบียน routes สำหรับ detail pages
public partial class AppShell : Shell
{
    public AppShell()
    {
        InitializeComponent();
        
        // ลงทะเบียน route ที่ไม่ได้อยู่ใน Shell hierarchy
        Routing.RegisterRoute("notedetail", typeof(NoteDetailPage));
    }
}
```

```csharp
// Navigate ด้วย GoToAsync
await Shell.Current.GoToAsync("notedetail");

// ส่ง parameter ผ่าน query string
await Shell.Current.GoToAsync($"notedetail?id={note.Id}");

// Navigate แบบ absolute path
await Shell.Current.GoToAsync("//notes");

// Navigate กลับ
await Shell.Current.GoToAsync("..");
```

---

## Step 652: MAUI Pages and Controls

### Pages หลักใน MAUI

#### ContentPage — หน้าพื้นฐาน

```xml
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="MyMauiApp.Views.MainPage"
             Title="หน้าหลัก"
             BackgroundColor="{AppThemeBinding Light=White, Dark=#1C1C1E}">
    
    <ContentPage.Content>
        <StackLayout Padding="20">
            <Label Text="สวัสดี MAUI!" 
                   FontSize="24"
                   FontAttributes="Bold"/>
        </StackLayout>
    </ContentPage.Content>
    
</ContentPage>
```

#### FlyoutPage — Side menu navigation

```xml
<FlyoutPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
            xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
            x:Class="MyMauiApp.Views.MainFlyoutPage">
    
    <FlyoutPage.Flyout>
        <ContentPage Title="Menu">
            <StackLayout>
                <Button Text="หน้าหลัก" Clicked="OnHomeClicked"/>
                <Button Text="โปรไฟล์" Clicked="OnProfileClicked"/>
            </StackLayout>
        </ContentPage>
    </FlyoutPage.Flyout>
    
    <FlyoutPage.Detail>
        <NavigationPage>
            <x:Arguments>
                <views:HomePage/>
            </x:Arguments>
        </NavigationPage>
    </FlyoutPage.Detail>
    
</FlyoutPage>
```

### Controls พื้นฐาน

#### Label — แสดงข้อความ

```xml
<!-- Label พื้นฐาน -->
<Label Text="ข้อความธรรมดา" />

<!-- Label แบบ Formatted (หลาย style ในบรรทัดเดียว) -->
<Label>
    <Label.FormattedText>
        <FormattedString>
            <Span Text="ชื่อ: " FontAttributes="Bold" FontSize="16"/>
            <Span Text="สมชาย" TextColor="Blue"/>
            <Span Text=" (Admin)" FontAttributes="Italic" TextColor="Gray"/>
        </FormattedString>
    </Label.FormattedText>
</Label>

<!-- Label แบบ Multi-line -->
<Label Text="บรรทัดยาวมากที่ต้องการ&#10;ขึ้นบรรทัดใหม่"
       LineBreakMode="WordWrap"
       MaxLines="3"/>
```

#### Button และ Event Handling

```xml
<!-- Button พื้นฐาน -->
<Button Text="กดฉัน" 
        Clicked="OnButtonClicked"
        BackgroundColor="#007AFF"
        TextColor="White"
        CornerRadius="10"
        Padding="15,10"/>

<!-- ImageButton -->
<ImageButton Source="add_icon.png"
             BackgroundColor="Transparent"
             Clicked="OnAddClicked"
             WidthRequest="44"
             HeightRequest="44"/>
```

```csharp
// Code-behind
private void OnButtonClicked(object sender, EventArgs e)
{
    // ใน MVVM pattern เราจะใช้ Command แทน Clicked event
    DisplayAlert("แจ้งเตือน", "คุณกดปุ่มแล้ว!", "ตกลง");
}
```

#### Entry และ Editor — รับข้อมูล

```xml
<!-- Entry — input บรรทัดเดียว -->
<Entry Placeholder="ใส่ชื่อของคุณ"
       Text="{Binding Name}"
       Keyboard="Text"
       ReturnType="Next"
       MaxLength="100"/>

<!-- Entry สำหรับ password -->
<Entry Placeholder="รหัสผ่าน"
       IsPassword="True"
       Text="{Binding Password}"/>

<!-- Entry สำหรับ email -->
<Entry Placeholder="อีเมล"
       Keyboard="Email"
       Text="{Binding Email}"/>

<!-- Editor — input หลายบรรทัด -->
<Editor Placeholder="เขียนโน้ตที่นี่..."
        Text="{Binding Content}"
        AutoSize="TextChanges"
        MinimumHeightRequest="100"
        MaxLength="5000"/>
```

### CollectionView — แสดง List ข้อมูล

CollectionView เป็น control ที่ทันสมัยกว่า ListView รองรับ layout หลายแบบ
และ performance ดีกว่ามาก

```xml
<!-- CollectionView แบบ vertical list -->
<CollectionView ItemsSource="{Binding Notes}"
                SelectionMode="Single"
                SelectionChanged="OnNoteSelected"
                EmptyView="ยังไม่มีโน้ต กด + เพื่อเพิ่ม">
    
    <CollectionView.ItemTemplate>
        <DataTemplate>
            <SwipeView>
                <!-- Swipe action ลบ -->
                <SwipeView.RightItems>
                    <SwipeItems>
                        <SwipeItem Text="ลบ"
                                   BackgroundColor="Red"
                                   Command="{Binding Source={RelativeSource AncestorType={x:Type ContentPage}}, 
                                             Path=BindingContext.DeleteNoteCommand}"
                                   CommandParameter="{Binding .}"/>
                    </SwipeItems>
                </SwipeView.RightItems>
                
                <!-- Card item -->
                <Border BackgroundColor="{AppThemeBinding Light=White, Dark=#2C2C2E}"
                        StrokeShape="RoundRectangle 12"
                        Margin="16,8"
                        Padding="16">
                    <Grid RowDefinitions="Auto,Auto,Auto" 
                          ColumnDefinitions="*,Auto">
                        <Label Grid.Row="0" Grid.Column="0"
                               Text="{Binding Title}"
                               FontAttributes="Bold"
                               FontSize="16"/>
                        <Label Grid.Row="0" Grid.Column="1"
                               Text="{Binding CreatedAt, StringFormat='{0:dd/MM/yy}'}"
                               FontSize="12"
                               TextColor="Gray"/>
                        <Label Grid.Row="1" Grid.ColumnSpan="2"
                               Text="{Binding Preview}"
                               FontSize="14"
                               TextColor="Gray"
                               LineBreakMode="TailTruncation"
                               MaxLines="2"/>
                    </Grid>
                </Border>
            </SwipeView>
        </DataTemplate>
    </CollectionView.ItemTemplate>
    
    <!-- Pull to refresh -->
    <CollectionView.Header>
        <RefreshView IsRefreshing="{Binding IsRefreshing}"
                     Command="{Binding RefreshCommand}">
            <ContentView/>
        </RefreshView>
    </CollectionView.Header>
    
</CollectionView>
```

### Layouts

#### Grid — Layout แบบตาราง

```xml
<Grid RowDefinitions="Auto,*,60" 
      ColumnDefinitions="*,*"
      RowSpacing="8"
      ColumnSpacing="8"
      Padding="16">
    
    <!-- span 2 columns -->
    <Label Grid.Row="0" Grid.ColumnSpan="2" 
           Text="หัวข้อ" FontSize="24"/>
    
    <!-- content area -->
    <Editor Grid.Row="1" Grid.ColumnSpan="2" 
            Text="{Binding Content}"/>
    
    <!-- buttons row -->
    <Button Grid.Row="2" Grid.Column="0" Text="ยกเลิก"/>
    <Button Grid.Row="2" Grid.Column="1" Text="บันทึก"/>
    
</Grid>
```

#### StackLayout และ HorizontalStackLayout

```xml
<!-- VerticalStackLayout — stack แนวตั้ง -->
<VerticalStackLayout Spacing="12" Padding="16">
    <Label Text="ชื่อ" FontAttributes="Bold"/>
    <Entry Text="{Binding Name}"/>
    <Label Text="อีเมล" FontAttributes="Bold"/>
    <Entry Text="{Binding Email}" Keyboard="Email"/>
    <Button Text="บันทึก" Command="{Binding SaveCommand}"/>
</VerticalStackLayout>

<!-- HorizontalStackLayout — stack แนวนอน -->
<HorizontalStackLayout Spacing="8">
    <Image Source="avatar.png" WidthRequest="40" HeightRequest="40"/>
    <VerticalStackLayout>
        <Label Text="{Binding Name}" FontAttributes="Bold"/>
        <Label Text="{Binding Email}" TextColor="Gray"/>
    </VerticalStackLayout>
</HorizontalStackLayout>
```

#### FlexLayout — CSS Flexbox สำหรับ MAUI

```xml
<!-- FlexLayout wrap — เหมาะสำหรับ tags/chips -->
<FlexLayout Wrap="Wrap" 
            Direction="Row"
            AlignItems="Center"
            JustifyContent="Start">
    
    <Border StrokeShape="RoundRectangle 16" 
            BackgroundColor="#007AFF"
            Margin="4">
        <Label Text="C#" TextColor="White" Padding="8,4"/>
    </Border>
    <Border StrokeShape="RoundRectangle 16"
            BackgroundColor="#34C759"
            Margin="4">
        <Label Text="MAUI" TextColor="White" Padding="8,4"/>
    </Border>
    <Border StrokeShape="RoundRectangle 16"
            BackgroundColor="#FF9500"
            Margin="4">
        <Label Text=".NET 8" TextColor="White" Padding="8,4"/>
    </Border>
    
</FlexLayout>
```

### Platform-specific Rendering

```csharp
// Platform-specific code ใน shared code
public string GetPlatformInfo()
{
#if ANDROID
    return $"Android {Android.OS.Build.VERSION.Release}";
#elif IOS
    return $"iOS {UIKit.UIDevice.CurrentDevice.SystemVersion}";
#elif WINDOWS
    return "Windows";
#elif MACCATALYST
    return "macOS";
#else
    return "Unknown";
#endif
}
```

```xml
<!-- OnPlatform ใน XAML -->
<Label>
    <Label.FontSize>
        <OnPlatform x:TypeArguments="x:Double">
            <On Platform="iOS" Value="16"/>
            <On Platform="Android" Value="14"/>
            <On Platform="WinUI" Value="18"/>
        </OnPlatform>
    </Label.FontSize>
</Label>

<!-- OnIdiom — Phone vs Tablet vs Desktop -->
<Grid>
    <Grid.ColumnDefinitions>
        <ColumnDefinition>
            <ColumnDefinition.Width>
                <OnIdiom x:TypeArguments="GridLength">
                    <OnIdiom.Phone>*</OnIdiom.Phone>
                    <OnIdiom.Tablet>300</OnIdiom.Tablet>
                    <OnIdiom.Desktop>400</OnIdiom.Desktop>
                </OnIdiom>
            </ColumnDefinition.Width>
        </ColumnDefinition>
    </Grid.ColumnDefinitions>
</Grid>
```

---

## Step 653: MVVM ใน MAUI

### CommunityToolkit.Mvvm

CommunityToolkit.Mvvm เป็น package ที่ Microsoft recommend สำหรับ MVVM ใน MAUI
มี Source Generator ที่ช่วยลด boilerplate code ได้มาก

```xml
<!-- .csproj -->
<PackageReference Include="CommunityToolkit.Mvvm" Version="8.2.2" />
```

### ViewModel พื้นฐาน

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

namespace MyMauiApp.ViewModels;

// ObservableObject เป็น base class ที่ implement INotifyPropertyChanged
[INotifyPropertyChanged]
public partial class NotesViewModel : ObservableObject
{
    private readonly DatabaseService _database;
    
    // [ObservableProperty] จะ generate property Name พร้อม OnNameChanged/OnNameChanging
    [ObservableProperty]
    private string _title = string.Empty;
    
    [ObservableProperty]
    private string _content = string.Empty;
    
    [ObservableProperty]
    [NotifyPropertyChangedFor(nameof(IsNotBusy))]  // แจ้ง IsNotBusy เมื่อ IsBusy เปลี่ยน
    private bool _isBusy;
    
    [ObservableProperty]
    private bool _isRefreshing;
    
    // Computed property
    public bool IsNotBusy => !IsBusy;
    
    // ObservableCollection — UI อัปเดตอัตโนมัติเมื่อ item เพิ่ม/ลบ
    public ObservableCollection<Note> Notes { get; } = new();
    
    public NotesViewModel(DatabaseService database)
    {
        _database = database;
    }
    
    // [RelayCommand] จะ generate LoadNotesCommand
    [RelayCommand]
    private async Task LoadNotes()
    {
        if (IsBusy) return;
        
        try
        {
            IsBusy = true;
            var notes = await _database.GetAllNotesAsync();
            
            Notes.Clear();
            foreach (var note in notes.OrderByDescending(n => n.ModifiedAt))
            {
                Notes.Add(note);
            }
        }
        catch (Exception ex)
        {
            await Shell.Current.DisplayAlert("ข้อผิดพลาด", 
                $"ไม่สามารถโหลดโน้ตได้: {ex.Message}", "ตกลง");
        }
        finally
        {
            IsBusy = false;
            IsRefreshing = false;
        }
    }
    
    // Command ที่รับ parameter
    [RelayCommand]
    private async Task DeleteNote(Note note)
    {
        bool confirm = await Shell.Current.DisplayAlert(
            "ยืนยันการลบ",
            $"ต้องการลบโน้ต '{note.Title}' ใช่หรือไม่?",
            "ลบ", "ยกเลิก");
        
        if (!confirm) return;
        
        await _database.DeleteNoteAsync(note);
        Notes.Remove(note);
    }
    
    // Command ที่มีเงื่อนไข (CanExecute)
    [RelayCommand(CanExecute = nameof(CanSave))]
    private async Task SaveNote()
    {
        var note = new Note
        {
            Title = Title,
            Content = Content,
            CreatedAt = DateTime.Now,
            ModifiedAt = DateTime.Now
        };
        
        await _database.SaveNoteAsync(note);
        await Shell.Current.GoToAsync("..");
    }
    
    private bool CanSave() => !string.IsNullOrWhiteSpace(Title);
    
    // เรียก NotifySaveNoteCommandCanExecuteChanged เมื่อ Title เปลี่ยน
    partial void OnTitleChanged(string value)
    {
        SaveNoteCommand.NotifyCanExecuteChanged();
    }
    
    // Command ที่ Navigate
    [RelayCommand]
    private async Task GoToNoteDetail(Note note)
    {
        if (note is null) return;
        
        // ส่ง object ผ่าน Shell navigation
        await Shell.Current.GoToAsync(nameof(NoteDetailPage), 
            new Dictionary<string, object>
            {
                { "Note", note }
            });
    }
    
    [RelayCommand]
    private async Task Refresh()
    {
        IsRefreshing = true;
        await LoadNotes();
    }
}
```

### Receiving Navigation Parameters

```csharp
// NoteDetailViewModel.cs
[QueryProperty(nameof(Note), "Note")]
[QueryProperty(nameof(NoteId), "id")]
public partial class NoteDetailViewModel : ObservableObject
{
    [ObservableProperty]
    private Note? _note;
    
    [ObservableProperty]
    private int _noteId;
    
    // เมื่อ NoteId เปลี่ยน ให้โหลดข้อมูล
    partial void OnNoteIdChanged(int value)
    {
        _ = LoadNoteById(value);
    }
    
    private async Task LoadNoteById(int id)
    {
        // โหลดจาก database
    }
}
```

### Binding ใน XAML

```xml
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             xmlns:vm="clr-namespace:MyMauiApp.ViewModels"
             x:Class="MyMauiApp.Views.NotesPage">
    
    <ContentPage.BindingContext>
        <vm:NotesViewModel/>
    </ContentPage.BindingContext>
    
    <Grid RowDefinitions="*,Auto">
        
        <CollectionView Grid.Row="0"
                        ItemsSource="{Binding Notes}"
                        IsVisible="{Binding IsNotBusy}">
            <!-- ItemTemplate ... -->
        </CollectionView>
        
        <!-- Loading indicator -->
        <ActivityIndicator Grid.Row="0"
                           IsRunning="{Binding IsBusy}"
                           IsVisible="{Binding IsBusy}"
                           Color="#007AFF"/>
        
        <!-- FAB (Floating Action Button) -->
        <Button Grid.Row="1"
                Text="+ เพิ่มโน้ต"
                Command="{Binding GoToAddNoteCommand}"
                HorizontalOptions="End"
                Margin="16"/>
        
    </Grid>
    
</ContentPage>
```

```csharp
// Code-behind ที่เรียบง่ายเมื่อใช้ MVVM + DI
public partial class NotesPage : ContentPage
{
    public NotesPage(NotesViewModel viewModel)
    {
        InitializeComponent();
        BindingContext = viewModel;
    }
    
    protected override async void OnAppearing()
    {
        base.OnAppearing();
        // โหลดข้อมูลทุกครั้งที่หน้าแสดง
        await ((NotesViewModel)BindingContext).LoadNotesCommand.ExecuteAsync(null);
    }
}
```

### Value Converters

```csharp
// Converter แปลง bool เป็น string
public class BoolToTextConverter : IValueConverter
{
    public object Convert(object? value, Type targetType, object? parameter, CultureInfo culture)
    {
        if (value is bool boolValue && parameter is string param)
        {
            var parts = param.Split('|');
            return boolValue ? parts[0] : (parts.Length > 1 ? parts[1] : string.Empty);
        }
        return string.Empty;
    }
    
    public object ConvertBack(object? value, Type targetType, object? parameter, CultureInfo culture)
        => throw new NotImplementedException();
}

// Converter แปลง DateTime เป็น relative time
public class RelativeTimeConverter : IValueConverter
{
    public object Convert(object? value, Type targetType, object? parameter, CultureInfo culture)
    {
        if (value is DateTime dateTime)
        {
            var diff = DateTime.Now - dateTime;
            return diff.TotalMinutes < 1 ? "เมื่อสักครู่" :
                   diff.TotalHours < 1   ? $"{(int)diff.TotalMinutes} นาทีที่แล้ว" :
                   diff.TotalDays < 1    ? $"{(int)diff.TotalHours} ชั่วโมงที่แล้ว" :
                   diff.TotalDays < 7    ? $"{(int)diff.TotalDays} วันที่แล้ว" :
                   dateTime.ToString("dd/MM/yyyy");
        }
        return string.Empty;
    }
    
    public object ConvertBack(object? value, Type targetType, object? parameter, CultureInfo culture)
        => throw new NotImplementedException();
}
```

```xml
<!-- ลงทะเบียน converter ใน App.xaml -->
<Application.Resources>
    <ResourceDictionary>
        <converters:BoolToTextConverter x:Key="BoolToText"/>
        <converters:RelativeTimeConverter x:Key="RelativeTime"/>
    </ResourceDictionary>
</Application.Resources>

<!-- ใช้ใน XAML -->
<Label Text="{Binding IsActive, Converter={StaticResource BoolToText}, 
             ConverterParameter='ใช้งาน|ไม่ใช้งาน'}"/>
<Label Text="{Binding ModifiedAt, Converter={StaticResource RelativeTime}}"/>
```

---

## Step 654: Data and Storage

### SQLite ด้วย sqlite-net-pcl

```xml
<!-- .csproj -->
<PackageReference Include="sqlite-net-pcl" Version="1.9.172" />
<PackageReference Include="SQLitePCLRaw.bundle_green" Version="2.1.6" />
```

```csharp
// Models/Note.cs
using SQLite;

namespace MyMauiApp.Models;

[Table("Notes")]
public class Note
{
    [PrimaryKey, AutoIncrement]
    public int Id { get; set; }
    
    [MaxLength(200)]
    public string Title { get; set; } = string.Empty;
    
    public string Content { get; set; } = string.Empty;
    
    public DateTime CreatedAt { get; set; }
    public DateTime ModifiedAt { get; set; }
    
    [Ignore]  // ไม่บันทึกลง database
    public string Preview => Content.Length > 100 
        ? Content[..97] + "..." 
        : Content;
    
    [Ignore]
    public bool IsFavorite { get; set; }  // สำหรับ UI state เท่านั้น
}
```

```csharp
// Services/DatabaseService.cs
namespace MyMauiApp.Services;

public class DatabaseService
{
    private SQLiteAsyncConnection? _database;
    private static readonly string DbPath = Path.Combine(
        FileSystem.AppDataDirectory, "notes.db3");
    
    // Lazy initialization
    private async Task<SQLiteAsyncConnection> GetDatabaseAsync()
    {
        if (_database is null)
        {
            _database = new SQLiteAsyncConnection(DbPath, 
                SQLiteOpenFlags.ReadWrite | 
                SQLiteOpenFlags.Create | 
                SQLiteOpenFlags.SharedCache);
            
            // สร้าง table ถ้ายังไม่มี
            await _database.CreateTableAsync<Note>();
        }
        return _database;
    }
    
    public async Task<List<Note>> GetAllNotesAsync()
    {
        var db = await GetDatabaseAsync();
        return await db.Table<Note>()
                       .OrderByDescending(n => n.ModifiedAt)
                       .ToListAsync();
    }
    
    public async Task<Note?> GetNoteAsync(int id)
    {
        var db = await GetDatabaseAsync();
        return await db.Table<Note>()
                       .Where(n => n.Id == id)
                       .FirstOrDefaultAsync();
    }
    
    public async Task<List<Note>> SearchNotesAsync(string query)
    {
        var db = await GetDatabaseAsync();
        var keyword = $"%{query}%";
        return await db.Table<Note>()
                       .Where(n => n.Title.Contains(query) || 
                                   n.Content.Contains(query))
                       .ToListAsync();
    }
    
    public async Task<int> SaveNoteAsync(Note note)
    {
        var db = await GetDatabaseAsync();
        note.ModifiedAt = DateTime.Now;
        
        if (note.Id == 0)
        {
            note.CreatedAt = DateTime.Now;
            return await db.InsertAsync(note);
        }
        return await db.UpdateAsync(note);
    }
    
    public async Task<int> DeleteNoteAsync(Note note)
    {
        var db = await GetDatabaseAsync();
        return await db.DeleteAsync(note);
    }
    
    public async Task<int> GetNotesCountAsync()
    {
        var db = await GetDatabaseAsync();
        return await db.Table<Note>().CountAsync();
    }
}
```

### Preferences — Key-Value Storage

```csharp
// บันทึกค่า Settings ง่ายๆ
public class SettingsService
{
    private const string ThemeKey = "app_theme";
    private const string FontSizeKey = "font_size";
    private const string SortOrderKey = "sort_order";
    
    // Theme
    public AppTheme CurrentTheme
    {
        get => (AppTheme)Preferences.Default.Get(ThemeKey, (int)AppTheme.System);
        set => Preferences.Default.Set(ThemeKey, (int)value);
    }
    
    // Font size
    public double FontSize
    {
        get => Preferences.Default.Get(FontSizeKey, 14.0);
        set => Preferences.Default.Set(FontSizeKey, value);
    }
    
    // Sort order
    public string SortOrder
    {
        get => Preferences.Default.Get(SortOrderKey, "ModifiedAt");
        set => Preferences.Default.Set(SortOrderKey, value);
    }
    
    // ล้างค่าทั้งหมด
    public void Reset()
    {
        Preferences.Default.Clear();
    }
    
    // ลบค่าเดียว
    public void RemoveFontSize()
    {
        Preferences.Default.Remove(FontSizeKey);
    }
}
```

### SecureStorage — เก็บข้อมูลที่ sensitive

```csharp
// เก็บ token หรือ password อย่างปลอดภัย
public class SecureStorageService
{
    private const string AuthTokenKey = "auth_token";
    private const string RefreshTokenKey = "refresh_token";
    
    public async Task SaveAuthTokenAsync(string token)
    {
        try
        {
            await SecureStorage.Default.SetAsync(AuthTokenKey, token);
        }
        catch (Exception)
        {
            // บาง platform อาจไม่รองรับ Keychain
            // ให้ fallback ไปใช้ Preferences แทน (แต่ไม่ secure)
            Preferences.Default.Set(AuthTokenKey, token);
        }
    }
    
    public async Task<string?> GetAuthTokenAsync()
    {
        try
        {
            return await SecureStorage.Default.GetAsync(AuthTokenKey);
        }
        catch
        {
            return Preferences.Default.Get(AuthTokenKey, null as string);
        }
    }
    
    public void RemoveAuthToken()
    {
        SecureStorage.Default.Remove(AuthTokenKey);
    }
    
    public void ClearAll()
    {
        SecureStorage.Default.RemoveAll();
    }
}
```

### File System Access

```csharp
public class FileService
{
    // AppDataDirectory — ไฟล์ private ของ app (ไม่โชว์ใน Files app)
    public string GetPrivateFilePath(string filename) =>
        Path.Combine(FileSystem.AppDataDirectory, filename);
    
    // CacheDirectory — ไฟล์ชั่วคราว (OS อาจลบได้)
    public string GetCacheFilePath(string filename) =>
        Path.Combine(FileSystem.CacheDirectory, filename);
    
    // บันทึก text file
    public async Task SaveTextFileAsync(string filename, string content)
    {
        var path = GetPrivateFilePath(filename);
        await File.WriteAllTextAsync(path, content);
    }
    
    // อ่าน text file
    public async Task<string?> ReadTextFileAsync(string filename)
    {
        var path = GetPrivateFilePath(filename);
        if (!File.Exists(path)) return null;
        return await File.ReadAllTextAsync(path);
    }
    
    // export ไฟล์ให้ user
    public async Task ShareFileAsync(string filename, string mimeType = "text/plain")
    {
        var path = GetPrivateFilePath(filename);
        if (!File.Exists(path)) return;
        
        await Share.Default.RequestAsync(new ShareFileRequest
        {
            Title = "แชร์ไฟล์",
            File = new ShareFile(path, mimeType)
        });
    }
    
    // copy file จาก App Bundle (Resources/Raw/)
    public async Task<string> CopyBundleFileAsync(string filename)
    {
        using var stream = await FileSystem.OpenAppPackageFileAsync(filename);
        var targetPath = GetCacheFilePath(filename);
        
        using var fileStream = File.OpenWrite(targetPath);
        await stream.CopyToAsync(fileStream);
        
        return targetPath;
    }
}
```

### Connectivity Check

```csharp
public class ConnectivityService
{
    public bool IsConnected => 
        Connectivity.Current.NetworkAccess == NetworkAccess.Internet;
    
    public bool IsWifi => 
        Connectivity.Current.ConnectionProfiles.Contains(ConnectionProfile.WiFi);
    
    public void StartMonitoring()
    {
        Connectivity.Current.ConnectivityChanged += OnConnectivityChanged;
    }
    
    public void StopMonitoring()
    {
        Connectivity.Current.ConnectivityChanged -= OnConnectivityChanged;
    }
    
    private void OnConnectivityChanged(object? sender, ConnectivityChangedEventArgs e)
    {
        var status = e.NetworkAccess == NetworkAccess.Internet 
            ? "ออนไลน์" : "ออฟไลน์";
        
        // แจ้งเตือน UI (ต้อง dispatch ไปยัง main thread)
        MainThread.BeginInvokeOnMainThread(() =>
        {
            // อัปเดต UI
        });
    }
}
```

---

## Step 655: Platform Integration

### Geolocation

```xml
<!-- Android: AndroidManifest.xml -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

<!-- iOS: Info.plist -->
<key>NSLocationWhenInUseUsageDescription</key>
<string>แอปต้องการตำแหน่งเพื่อแสดงโน้ตใกล้เคียง</string>
```

```csharp
public class LocationService
{
    public async Task<Location?> GetCurrentLocationAsync()
    {
        // ขอ permission ก่อน
        var status = await Permissions.RequestAsync<Permissions.LocationWhenInUse>();
        if (status != PermissionStatus.Granted)
        {
            await Shell.Current.DisplayAlert("Permission", 
                "ต้องการ permission ตำแหน่ง", "ตกลง");
            return null;
        }
        
        try
        {
            var request = new GeolocationRequest(
                GeolocationAccuracy.Medium,
                TimeSpan.FromSeconds(10));
            
            return await Geolocation.Default.GetLocationAsync(request);
        }
        catch (FeatureNotSupportedException)
        {
            // GPS ไม่รองรับบน device นี้
        }
        catch (FeatureNotEnabledException)
        {
            // GPS ปิดอยู่
        }
        catch (PermissionException)
        {
            // ไม่ได้รับ permission
        }
        catch (Exception ex)
        {
            Console.WriteLine($"GPS error: {ex.Message}");
        }
        
        return null;
    }
    
    public double CalculateDistance(double lat1, double lon1, double lat2, double lon2)
    {
        var loc1 = new Location(lat1, lon1);
        var loc2 = new Location(lat2, lon2);
        return loc1.CalculateDistance(loc2, DistanceUnits.Kilometers);
    }
}
```

### Camera และ Media Picker

```xml
<!-- Android: AndroidManifest.xml -->
<uses-permission android:name="android.permission.CAMERA"/>
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES"/>

<!-- iOS: Info.plist -->
<key>NSCameraUsageDescription</key>
<string>แอปต้องการกล้องเพื่อถ่ายรูปประกอบโน้ต</string>
<key>NSPhotoLibraryUsageDescription</key>
<string>แอปต้องการเข้าถึงรูปภาพเพื่อแนบกับโน้ต</string>
```

```csharp
public class MediaService
{
    // เลือกรูปจาก Gallery
    public async Task<FileResult?> PickPhotoAsync()
    {
        var status = await Permissions.RequestAsync<Permissions.Photos>();
        if (status != PermissionStatus.Granted) return null;
        
        return await MediaPicker.Default.PickPhotoAsync(new MediaPickerOptions
        {
            Title = "เลือกรูปภาพ"
        });
    }
    
    // ถ่ายรูปด้วยกล้อง
    public async Task<FileResult?> TakePhotoAsync()
    {
        if (!MediaPicker.Default.IsCaptureSupported)
        {
            await Shell.Current.DisplayAlert("ไม่รองรับ", 
                "อุปกรณ์นี้ไม่รองรับการถ่ายรูป", "ตกลง");
            return null;
        }
        
        var status = await Permissions.RequestAsync<Permissions.Camera>();
        if (status != PermissionStatus.Granted) return null;
        
        return await MediaPicker.Default.CapturePhotoAsync();
    }
    
    // บันทึกรูปไปยัง app storage
    public async Task<string?> SavePhotoAsync(FileResult photo)
    {
        var filename = $"photo_{DateTime.Now:yyyyMMddHHmmss}.jpg";
        var targetPath = Path.Combine(FileSystem.AppDataDirectory, "photos", filename);
        
        Directory.CreateDirectory(Path.GetDirectoryName(targetPath)!);
        
        using var sourceStream = await photo.OpenReadAsync();
        using var targetStream = File.OpenWrite(targetPath);
        await sourceStream.CopyToAsync(targetStream);
        
        return targetPath;
    }
}
```

### Biometric Authentication

```xml
<!-- .csproj -->
<PackageReference Include="Plugin.Fingerprint" Version="2.1.5" />
```

```csharp
using Plugin.Fingerprint;
using Plugin.Fingerprint.Abstractions;

public class BiometricService
{
    public async Task<bool> IsAvailableAsync()
    {
        return await CrossFingerprint.Current.IsAvailableAsync();
    }
    
    public async Task<bool> AuthenticateAsync(string reason = "ยืนยันตัวตน")
    {
        if (!await IsAvailableAsync())
        {
            // biometric ไม่รองรับ ให้ fallback ไป PIN หรือ password
            return await FallbackAuthAsync();
        }
        
        var request = new AuthenticationRequestConfiguration(
            "ยืนยันตัวตน", reason)
        {
            FallbackTitle = "ใช้รหัสผ่าน",
            AllowAlternativeAuthentication = true,
        };
        
        var result = await CrossFingerprint.Current.AuthenticateAsync(request);
        return result.Authenticated;
    }
    
    private async Task<bool> FallbackAuthAsync()
    {
        var pin = await Shell.Current.DisplayPromptAsync(
            "ยืนยันตัวตน", "ใส่ PIN:", 
            maxLength: 6,
            keyboard: Keyboard.Numeric);
        
        // ตรวจสอบ PIN กับที่บันทึกไว้ใน SecureStorage
        var savedPin = await SecureStorage.Default.GetAsync("user_pin");
        return pin == savedPin;
    }
}
```

---

## Step 656–660: Complete MAUI App — Note-Taking App

ในส่วนนี้เราจะสร้าง Note-taking App ที่ครบถ้วนโดยรวม concept ทั้งหมดที่เรียนมา

### โครงสร้าง Project ของ App

```
NoteMauiApp/
├── MauiProgram.cs
├── App.xaml
├── AppShell.xaml
├── Models/
│   ├── Note.cs
│   └── NoteCategory.cs
├── Services/
│   ├── DatabaseService.cs
│   └── SettingsService.cs
├── ViewModels/
│   ├── BaseViewModel.cs
│   ├── NotesViewModel.cs
│   ├── NoteDetailViewModel.cs
│   └── SettingsViewModel.cs
├── Views/
│   ├── NotesPage.xaml
│   ├── NoteDetailPage.xaml
│   └── SettingsPage.xaml
├── Controls/
│   └── NoteCard.xaml
└── Converters/
    ├── BoolToColorConverter.cs
    └── RelativeTimeConverter.cs
```

### Step 656 — Models และ Database Setup

```csharp
// Models/Note.cs
using SQLite;

namespace NoteMauiApp.Models;

[Table("Notes")]
public class Note
{
    [PrimaryKey, AutoIncrement]
    public int Id { get; set; }
    
    [MaxLength(200), NotNull]
    public string Title { get; set; } = string.Empty;
    
    public string Content { get; set; } = string.Empty;
    
    [MaxLength(50)]
    public string Category { get; set; } = "ทั่วไป";
    
    public bool IsPinned { get; set; }
    
    public string? ImagePath { get; set; }
    
    public DateTime CreatedAt { get; set; } = DateTime.Now;
    
    public DateTime ModifiedAt { get; set; } = DateTime.Now;
    
    [Ignore]
    public string Preview => 
        string.IsNullOrWhiteSpace(Content) ? "(ไม่มีเนื้อหา)" :
        Content.Length > 80 ? Content[..77] + "..." : Content;
    
    [Ignore]
    public string CategoryIcon => Category switch
    {
        "งาน" => "💼",
        "ส่วนตัว" => "👤",
        "ช็อปปิ้ง" => "🛒",
        "ความคิด" => "💡",
        _ => "📝"
    };
}
```

```csharp
// Services/DatabaseService.cs
using SQLite;
using NoteMauiApp.Models;

namespace NoteMauiApp.Services;

public class DatabaseService : IDisposable
{
    private SQLiteAsyncConnection? _db;
    private bool _disposed;
    
    private static readonly string DbPath = Path.Combine(
        FileSystem.AppDataDirectory, "notes_v2.db3");
    
    private async Task<SQLiteAsyncConnection> GetDbAsync()
    {
        if (_db is null)
        {
            _db = new SQLiteAsyncConnection(DbPath,
                SQLiteOpenFlags.ReadWrite |
                SQLiteOpenFlags.Create |
                SQLiteOpenFlags.SharedCache);
            
            await _db.CreateTableAsync<Note>();
            
            // Seed data สำหรับครั้งแรก
            if (await _db.Table<Note>().CountAsync() == 0)
            {
                await SeedDataAsync();
            }
        }
        return _db;
    }
    
    private async Task SeedDataAsync()
    {
        var sampleNotes = new List<Note>
        {
            new() { Title = "ยินดีต้อนรับสู่ NoteMaui!", 
                    Content = "แอปนี้ช่วยให้คุณจดบันทึกได้ทุกที่ทุกเวลา ลองเพิ่มโน้ตแรกของคุณดูสิ!",
                    Category = "ทั่วไป", IsPinned = true },
            new() { Title = "รายการซื้อของ",
                    Content = "- นม\n- ไข่\n- ขนมปัง\n- ผักสด",
                    Category = "ช็อปปิ้ง" },
        };
        
        foreach (var note in sampleNotes)
        {
            note.CreatedAt = DateTime.Now.AddDays(-Random.Shared.Next(0, 30));
            note.ModifiedAt = note.CreatedAt.AddHours(Random.Shared.Next(0, 24));
            await _db!.InsertAsync(note);
        }
    }
    
    public async Task<List<Note>> GetAllAsync(string? searchQuery = null)
    {
        var db = await GetDbAsync();
        var query = db.Table<Note>();
        
        if (!string.IsNullOrWhiteSpace(searchQuery))
        {
            query = query.Where(n => 
                n.Title.Contains(searchQuery) || 
                n.Content.Contains(searchQuery));
        }
        
        var notes = await query.ToListAsync();
        
        // เรียงลำดับ: pinned ก่อน แล้ว modified ล่าสุด
        return notes
            .OrderByDescending(n => n.IsPinned)
            .ThenByDescending(n => n.ModifiedAt)
            .ToList();
    }
    
    public async Task<Note?> GetByIdAsync(int id)
    {
        var db = await GetDbAsync();
        return await db.Table<Note>().Where(n => n.Id == id).FirstOrDefaultAsync();
    }
    
    public async Task<int> SaveAsync(Note note)
    {
        var db = await GetDbAsync();
        note.ModifiedAt = DateTime.Now;
        
        if (note.Id == 0)
        {
            note.CreatedAt = DateTime.Now;
            return await db.InsertAsync(note);
        }
        return await db.UpdateAsync(note);
    }
    
    public async Task<int> DeleteAsync(Note note)
    {
        var db = await GetDbAsync();
        return await db.DeleteAsync(note);
    }
    
    public void Dispose()
    {
        if (!_disposed)
        {
            _db?.CloseAsync();
            _disposed = true;
        }
    }
}
```

### Step 657 — ViewModels

```csharp
// ViewModels/NotesViewModel.cs
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using System.Collections.ObjectModel;
using NoteMauiApp.Models;
using NoteMauiApp.Services;

namespace NoteMauiApp.ViewModels;

public partial class NotesViewModel : ObservableObject
{
    private readonly DatabaseService _db;
    
    [ObservableProperty]
    private string _searchText = string.Empty;
    
    [ObservableProperty]
    [NotifyPropertyChangedFor(nameof(IsNotBusy))]
    private bool _isBusy;
    
    [ObservableProperty]
    private bool _isRefreshing;
    
    [ObservableProperty]
    private int _totalNotes;
    
    public bool IsNotBusy => !IsBusy;
    
    public ObservableCollection<Note> Notes { get; } = new();
    
    public NotesViewModel(DatabaseService db)
    {
        _db = db;
    }
    
    // ค้นหาแบบ debounce เมื่อ SearchText เปลี่ยน
    partial void OnSearchTextChanged(string value)
    {
        // Cancel token เก่า แล้วรอ 300ms ก่อน search
        _searchCts?.Cancel();
        _searchCts = new CancellationTokenSource();
        var token = _searchCts.Token;
        
        Task.Delay(300, token).ContinueWith(t =>
        {
            if (!t.IsCanceled)
                MainThread.BeginInvokeOnMainThread(() => 
                    _ = LoadNotesCommand.ExecuteAsync(null));
        }, token);
    }
    
    private CancellationTokenSource? _searchCts;
    
    [RelayCommand]
    private async Task LoadNotes()
    {
        if (IsBusy) return;
        
        try
        {
            IsBusy = true;
            var notes = await _db.GetAllAsync(SearchText);
            
            Notes.Clear();
            foreach (var note in notes)
                Notes.Add(note);
            
            TotalNotes = Notes.Count;
        }
        catch (Exception ex)
        {
            await Shell.Current.DisplayAlert("Error", ex.Message, "OK");
        }
        finally
        {
            IsBusy = false;
            IsRefreshing = false;
        }
    }
    
    [RelayCommand]
    private async Task Refresh()
    {
        IsRefreshing = true;
        await LoadNotes();
    }
    
    [RelayCommand]
    private async Task AddNote()
    {
        await Shell.Current.GoToAsync(nameof(NoteDetailPage));
    }
    
    [RelayCommand]
    private async Task EditNote(Note note)
    {
        await Shell.Current.GoToAsync(nameof(NoteDetailPage),
            new Dictionary<string, object> { { "Note", note } });
    }
    
    [RelayCommand]
    private async Task TogglePin(Note note)
    {
        note.IsPinned = !note.IsPinned;
        await _db.SaveAsync(note);
        await LoadNotes(); // reload เพื่อเรียงลำดับใหม่
    }
    
    [RelayCommand]
    private async Task DeleteNote(Note note)
    {
        bool confirm = await Shell.Current.DisplayAlert(
            "ยืนยันการลบ",
            $"ลบโน้ต \"{note.Title}\" ?",
            "ลบ", "ยกเลิก");
        
        if (!confirm) return;
        
        await _db.DeleteAsync(note);
        Notes.Remove(note);
        TotalNotes--;
    }
}
```

```csharp
// ViewModels/NoteDetailViewModel.cs
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using NoteMauiApp.Models;
using NoteMauiApp.Services;

namespace NoteMauiApp.ViewModels;

[QueryProperty(nameof(Note), "Note")]
public partial class NoteDetailViewModel : ObservableObject
{
    private readonly DatabaseService _db;
    
    [ObservableProperty]
    [NotifyPropertyChangedFor(nameof(PageTitle))]
    [NotifyPropertyChangedFor(nameof(IsEditing))]
    private Note? _note;
    
    [ObservableProperty]
    [NotifyCanExecuteChangedFor(nameof(SaveCommand))]
    private string _title = string.Empty;
    
    [ObservableProperty]
    private string _content = string.Empty;
    
    [ObservableProperty]
    private string _category = "ทั่วไป";
    
    [ObservableProperty]
    private bool _isSaving;
    
    public string PageTitle => IsEditing ? "แก้ไขโน้ต" : "โน้ตใหม่";
    public bool IsEditing => Note?.Id > 0;
    
    public List<string> Categories { get; } = 
        new() { "ทั่วไป", "งาน", "ส่วนตัว", "ช็อปปิ้ง", "ความคิด" };
    
    public NoteDetailViewModel(DatabaseService db)
    {
        _db = db;
    }
    
    // เมื่อ Note เปลี่ยน (navigate มาพร้อม parameter) ให้ populate fields
    partial void OnNoteChanged(Note? value)
    {
        if (value is null) return;
        Title = value.Title;
        Content = value.Content;
        Category = value.Category;
    }
    
    [RelayCommand(CanExecute = nameof(CanSave))]
    private async Task Save()
    {
        try
        {
            IsSaving = true;
            
            var noteToSave = Note ?? new Note();
            noteToSave.Title = Title.Trim();
            noteToSave.Content = Content.Trim();
            noteToSave.Category = Category;
            
            await _db.SaveAsync(noteToSave);
            
            await Shell.Current.GoToAsync("..");
        }
        catch (Exception ex)
        {
            await Shell.Current.DisplayAlert("ข้อผิดพลาด", 
                $"บันทึกไม่ได้: {ex.Message}", "ตกลง");
        }
        finally
        {
            IsSaving = false;
        }
    }
    
    private bool CanSave() => !string.IsNullOrWhiteSpace(Title);
    
    [RelayCommand]
    private async Task Cancel()
    {
        if (HasChanges())
        {
            bool discard = await Shell.Current.DisplayAlert(
                "ยกเลิกการแก้ไข",
                "มีการเปลี่ยนแปลงที่ยังไม่ได้บันทึก ต้องการออกใช่ไหม?",
                "ออก", "อยู่ต่อ");
            
            if (!discard) return;
        }
        
        await Shell.Current.GoToAsync("..");
    }
    
    private bool HasChanges()
    {
        if (Note is null) 
            return !string.IsNullOrWhiteSpace(Title) || 
                   !string.IsNullOrWhiteSpace(Content);
        
        return Title != Note.Title || 
               Content != Note.Content || 
               Category != Note.Category;
    }
}
```

```csharp
// ViewModels/SettingsViewModel.cs
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using NoteMauiApp.Services;

namespace NoteMauiApp.ViewModels;

public partial class SettingsViewModel : ObservableObject
{
    private readonly DatabaseService _db;
    private readonly SettingsService _settings;
    
    [ObservableProperty]
    private bool _useDarkMode;
    
    [ObservableProperty]
    private double _fontSize;
    
    [ObservableProperty]
    private int _totalNotesCount;
    
    [ObservableProperty]
    private string _appVersion = AppInfo.VersionString;
    
    public SettingsViewModel(DatabaseService db, SettingsService settings)
    {
        _db = db;
        _settings = settings;
        LoadSettings();
    }
    
    private void LoadSettings()
    {
        UseDarkMode = _settings.UseDarkMode;
        FontSize = _settings.FontSize;
        _ = LoadNotesCount();
    }
    
    private async Task LoadNotesCount()
    {
        var notes = await _db.GetAllAsync();
        TotalNotesCount = notes.Count;
    }
    
    partial void OnUseDarkModeChanged(bool value)
    {
        _settings.UseDarkMode = value;
        
        // เปลี่ยน theme ทันที
        Application.Current!.UserAppTheme = value 
            ? AppTheme.Dark 
            : AppTheme.Light;
    }
    
    partial void OnFontSizeChanged(double value)
    {
        _settings.FontSize = value;
    }
    
    [RelayCommand]
    private async Task ClearAllNotes()
    {
        bool confirm = await Shell.Current.DisplayAlert(
            "⚠️ ลบโน้ตทั้งหมด",
            "การกระทำนี้ไม่สามารถย้อนกลับได้! ต้องการลบโน้ตทั้งหมดใช่ไหม?",
            "ลบทั้งหมด", "ยกเลิก");
        
        if (!confirm) return;
        
        var notes = await _db.GetAllAsync();
        foreach (var note in notes)
            await _db.DeleteAsync(note);
        
        TotalNotesCount = 0;
        await Shell.Current.DisplayAlert("เสร็จสิ้น", "ลบโน้ตทั้งหมดแล้ว", "ตกลง");
    }
    
    [RelayCommand]
    private async Task RateApp()
    {
        // เปิด App Store / Play Store
        if (AppInfo.RequestedLayout != AppTheme.Unspecified)
            await Browser.Default.OpenAsync("https://apps.apple.com/");
    }
    
    [RelayCommand]
    private async Task SendFeedback()
    {
        var email = new EmailMessage
        {
            Subject = $"Feedback - NoteMaui v{AppInfo.VersionString}",
            To = new List<string> { "feedback@example.com" }
        };
        
        await Email.Default.ComposeAsync(email);
    }
}
```

### Step 658 — Views (XAML)

```xml
<!-- Views/NotesPage.xaml -->
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             xmlns:vm="clr-namespace:NoteMauiApp.ViewModels"
             xmlns:models="clr-namespace:NoteMauiApp.Models"
             x:Class="NoteMauiApp.Views.NotesPage"
             x:DataType="vm:NotesViewModel"
             Title="โน้ตของฉัน">

    <ContentPage.ToolbarItems>
        <ToolbarItem Text="+"
                     Command="{Binding AddNoteCommand}"
                     Priority="0"/>
    </ContentPage.ToolbarItems>

    <Grid RowDefinitions="Auto,*" BackgroundColor="{AppThemeBinding 
          Light={StaticResource PageBackground}, 
          Dark={StaticResource PageBackgroundDark}}">

        <!-- Search Bar -->
        <SearchBar Grid.Row="0"
                   Placeholder="ค้นหาโน้ต..."
                   Text="{Binding SearchText}"
                   Margin="16,8"/>

        <!-- Notes List with RefreshView -->
        <RefreshView Grid.Row="1"
                     IsRefreshing="{Binding IsRefreshing}"
                     Command="{Binding RefreshCommand}">

            <CollectionView ItemsSource="{Binding Notes}"
                            SelectionMode="None">

                <!-- Empty state -->
                <CollectionView.EmptyView>
                    <VerticalStackLayout HorizontalOptions="Center"
                                         VerticalOptions="Center"
                                         Spacing="16"
                                         Padding="32">
                        <Label Text="📝" FontSize="64" HorizontalOptions="Center"/>
                        <Label Text="ยังไม่มีโน้ต" 
                               FontSize="20"
                               FontAttributes="Bold"
                               HorizontalOptions="Center"/>
                        <Label Text="กด + เพื่อสร้างโน้ตแรกของคุณ"
                               TextColor="Gray"
                               HorizontalOptions="Center"
                               HorizontalTextAlignment="Center"/>
                    </VerticalStackLayout>
                </CollectionView.EmptyView>

                <!-- Item Template -->
                <CollectionView.ItemTemplate>
                    <DataTemplate x:DataType="models:Note">
                        <SwipeView>
                            
                            <!-- Swipe Left: ลบ -->
                            <SwipeView.RightItems>
                                <SwipeItems>
                                    <SwipeItem Text="🗑️ ลบ"
                                               BackgroundColor="#FF3B30"
                                               Command="{Binding Source={RelativeSource AncestorType={x:Type ContentPage}},
                                                         Path=BindingContext.DeleteNoteCommand}"
                                               CommandParameter="{Binding .}"/>
                                </SwipeItems>
                            </SwipeView.RightItems>
                            
                            <!-- Swipe Right: Pin -->
                            <SwipeView.LeftItems>
                                <SwipeItems>
                                    <SwipeItem Text="📌 ปักหมุด"
                                               BackgroundColor="#FF9500"
                                               Command="{Binding Source={RelativeSource AncestorType={x:Type ContentPage}},
                                                         Path=BindingContext.TogglePinCommand}"
                                               CommandParameter="{Binding .}"/>
                                </SwipeItems>
                            </SwipeView.LeftItems>

                            <!-- Note Card -->
                            <Border BackgroundColor="{AppThemeBinding Light=White, Dark=#2C2C2E}"
                                    Stroke="{AppThemeBinding Light=#E5E5EA, Dark=#3A3A3C}"
                                    StrokeThickness="1"
                                    StrokeShape="RoundRectangle 12"
                                    Margin="16,4"
                                    Padding="16">
                                <Border.GestureRecognizers>
                                    <TapGestureRecognizer 
                                        Command="{Binding Source={RelativeSource AncestorType={x:Type ContentPage}},
                                                  Path=BindingContext.EditNoteCommand}"
                                        CommandParameter="{Binding .}"/>
                                </Border.GestureRecognizers>

                                <Grid RowDefinitions="Auto,Auto,Auto"
                                      ColumnDefinitions="*,Auto"
                                      RowSpacing="4">
                                    
                                    <!-- Title row -->
                                    <Label Grid.Row="0" Grid.Column="0"
                                           Text="{Binding Title}"
                                           FontSize="16"
                                           FontAttributes="Bold"
                                           LineBreakMode="TailTruncation"/>
                                    
                                    <!-- Pin indicator -->
                                    <Label Grid.Row="0" Grid.Column="1"
                                           Text="📌"
                                           IsVisible="{Binding IsPinned}"
                                           FontSize="14"/>
                                    
                                    <!-- Preview -->
                                    <Label Grid.Row="1" Grid.ColumnSpan="2"
                                           Text="{Binding Preview}"
                                           FontSize="13"
                                           TextColor="{AppThemeBinding Light=#666, Dark=#AAA}"
                                           LineBreakMode="TailTruncation"
                                           MaxLines="2"/>
                                    
                                    <!-- Footer -->
                                    <HorizontalStackLayout Grid.Row="2" Grid.Column="0"
                                                           Spacing="8">
                                        <Label Text="{Binding CategoryIcon}" FontSize="12"/>
                                        <Label Text="{Binding Category}"
                                               FontSize="11"
                                               TextColor="{AppThemeBinding Light=#999, Dark=#777}"/>
                                    </HorizontalStackLayout>
                                    
                                    <Label Grid.Row="2" Grid.Column="1"
                                           Text="{Binding ModifiedAt, StringFormat='{0:dd/MM/yy}'}"
                                           FontSize="11"
                                           TextColor="{AppThemeBinding Light=#999, Dark=#777}"/>
                                </Grid>
                            </Border>
                        </SwipeView>
                    </DataTemplate>
                </CollectionView.ItemTemplate>
                
            </CollectionView>
        </RefreshView>

        <!-- Loading indicator -->
        <ActivityIndicator Grid.Row="1"
                           IsRunning="{Binding IsBusy}"
                           IsVisible="{Binding IsBusy}"
                           Color="#007AFF"
                           VerticalOptions="Center"
                           HorizontalOptions="Center"/>

    </Grid>
</ContentPage>
```

```xml
<!-- Views/NoteDetailPage.xaml -->
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             xmlns:vm="clr-namespace:NoteMauiApp.ViewModels"
             x:Class="NoteMauiApp.Views.NoteDetailPage"
             x:DataType="vm:NoteDetailViewModel"
             Title="{Binding PageTitle}">

    <ContentPage.ToolbarItems>
        <ToolbarItem Text="บันทึก"
                     Command="{Binding SaveCommand}"
                     Priority="0"/>
    </ContentPage.ToolbarItems>

    <ScrollView>
        <VerticalStackLayout Spacing="0" Padding="16">

            <!-- Title Input -->
            <Label Text="หัวข้อ" 
                   FontSize="12" 
                   TextColor="Gray"
                   Margin="0,0,0,4"/>
            <Border StrokeShape="RoundRectangle 10"
                    BackgroundColor="{AppThemeBinding Light=#F2F2F7, Dark=#2C2C2E}"
                    Stroke="Transparent"
                    Margin="0,0,0,16">
                <Entry Text="{Binding Title}"
                       Placeholder="หัวข้อโน้ต"
                       FontSize="18"
                       FontAttributes="Bold"
                       Margin="12,8"
                       ClearButtonVisibility="WhileEditing"
                       ReturnType="Next"/>
            </Border>
            
            <!-- Category Picker -->
            <Label Text="หมวดหมู่"
                   FontSize="12"
                   TextColor="Gray"
                   Margin="0,0,0,4"/>
            <Border StrokeShape="RoundRectangle 10"
                    BackgroundColor="{AppThemeBinding Light=#F2F2F7, Dark=#2C2C2E}"
                    Stroke="Transparent"
                    Margin="0,0,0,16">
                <Picker ItemsSource="{Binding Categories}"
                        SelectedItem="{Binding Category}"
                        Margin="12,4"/>
            </Border>
            
            <!-- Content Input -->
            <Label Text="เนื้อหา"
                   FontSize="12"
                   TextColor="Gray"
                   Margin="0,0,0,4"/>
            <Border StrokeShape="RoundRectangle 10"
                    BackgroundColor="{AppThemeBinding Light=#F2F2F7, Dark=#2C2C2E}"
                    Stroke="Transparent"
                    MinimumHeightRequest="200">
                <Editor Text="{Binding Content}"
                        Placeholder="เขียนโน้ตของคุณที่นี่..."
                        AutoSize="TextChanges"
                        Margin="12,8"
                        FontSize="15"/>
            </Border>

            <!-- Cancel Button -->
            <Button Text="ยกเลิก"
                    Command="{Binding CancelCommand}"
                    BackgroundColor="Transparent"
                    TextColor="{AppThemeBinding Light=#FF3B30, Dark=#FF453A}"
                    Margin="0,24,0,0"/>

        </VerticalStackLayout>
    </ScrollView>
</ContentPage>
```

### Step 659 — Dark Mode และ Theming

```xml
<!-- Resources/Styles/Colors.xaml -->
<?xml version="1.0" encoding="UTF-8" ?>
<ResourceDictionary xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
                    xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml">

    <!-- Light Mode Colors -->
    <Color x:Key="Primary">#007AFF</Color>
    <Color x:Key="PrimaryDark">#0A84FF</Color>
    <Color x:Key="Secondary">#5856D6</Color>
    <Color x:Key="Success">#34C759</Color>
    <Color x:Key="Warning">#FF9500</Color>
    <Color x:Key="Danger">#FF3B30</Color>
    
    <!-- Page backgrounds -->
    <Color x:Key="PageBackground">#F2F2F7</Color>
    <Color x:Key="PageBackgroundDark">#000000</Color>
    
    <!-- Card colors -->
    <Color x:Key="CardBackground">White</Color>
    <Color x:Key="CardBackgroundDark">#1C1C1E</Color>
    
    <!-- Text colors -->
    <Color x:Key="TextPrimary">#000000</Color>
    <Color x:Key="TextPrimaryDark">#FFFFFF</Color>
    <Color x:Key="TextSecondary">#6C6C70</Color>
    <Color x:Key="TextSecondaryDark">#AEAEB2</Color>

</ResourceDictionary>
```

```csharp
// App.xaml.cs — จัดการ theme
public partial class App : Application
{
    private readonly SettingsService _settings;
    
    public App(SettingsService settings)
    {
        InitializeComponent();
        _settings = settings;
        
        // Apply saved theme
        UserAppTheme = _settings.UseDarkMode ? AppTheme.Dark : AppTheme.Light;
        
        MainPage = new AppShell();
    }
    
    protected override Window CreateWindow(IActivationState? activationState)
    {
        var window = base.CreateWindow(activationState);
        
        // กำหนดขนาด window สำหรับ Desktop
        window.Title = "NoteMaui";
        window.MinimumWidth = 400;
        window.MinimumHeight = 600;
        
        return window;
    }
}
```

```xml
<!-- Resources/Styles/Styles.xaml — Global styles -->
<?xml version="1.0" encoding="UTF-8" ?>
<ResourceDictionary xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
                    xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml">

    <Style TargetType="ContentPage" ApplyToDerivedTypes="True">
        <Setter Property="BackgroundColor" Value="{AppThemeBinding 
                Light={StaticResource PageBackground}, 
                Dark={StaticResource PageBackgroundDark}}"/>
    </Style>

    <Style TargetType="Label">
        <Setter Property="TextColor" Value="{AppThemeBinding 
                Light={StaticResource TextPrimary}, 
                Dark={StaticResource TextPrimaryDark}}"/>
        <Setter Property="FontFamily" Value="OpenSansRegular"/>
    </Style>

    <Style TargetType="Button">
        <Setter Property="BackgroundColor" Value="{StaticResource Primary}"/>
        <Setter Property="TextColor" Value="White"/>
        <Setter Property="CornerRadius" Value="10"/>
        <Setter Property="FontAttributes" Value="Bold"/>
        <Setter Property="HeightRequest" Value="50"/>
    </Style>
    
    <Style x:Key="SecondaryButton" TargetType="Button">
        <Setter Property="BackgroundColor" Value="Transparent"/>
        <Setter Property="TextColor" Value="{StaticResource Primary}"/>
        <Setter Property="BorderColor" Value="{StaticResource Primary}"/>
        <Setter Property="BorderWidth" Value="1.5"/>
        <Setter Property="CornerRadius" Value="10"/>
    </Style>

    <Style TargetType="Entry">
        <Setter Property="TextColor" Value="{AppThemeBinding 
                Light={StaticResource TextPrimary},
                Dark={StaticResource TextPrimaryDark}}"/>
        <Setter Property="PlaceholderColor" Value="{StaticResource TextSecondary}"/>
        <Setter Property="BackgroundColor" Value="Transparent"/>
    </Style>

</ResourceDictionary>
```

### Step 660 — MauiProgram.cs และ Final Wiring

```csharp
// MauiProgram.cs
using Microsoft.Extensions.Logging;
using NoteMauiApp.Services;
using NoteMauiApp.ViewModels;
using NoteMauiApp.Views;

namespace NoteMauiApp;

public static class MauiProgram
{
    public static MauiApp CreateMauiApp()
    {
        var builder = MauiApp.CreateBuilder();
        
        builder
            .UseMauiApp<App>()
            .ConfigureFonts(fonts =>
            {
                fonts.AddFont("OpenSans-Regular.ttf", "OpenSansRegular");
                fonts.AddFont("OpenSans-Semibold.ttf", "OpenSansSemibold");
                fonts.AddFont("fa-solid-900.ttf", "FontAwesome"); // Font Awesome icons
            });

        // Services — Singleton (สร้างครั้งเดียว)
        builder.Services.AddSingleton<DatabaseService>();
        builder.Services.AddSingleton<SettingsService>();
        builder.Services.AddSingleton<MediaService>();
        
        // ViewModels — Transient (สร้างใหม่ทุกครั้ง)
        builder.Services.AddTransient<NotesViewModel>();
        builder.Services.AddTransient<NoteDetailViewModel>();
        builder.Services.AddTransient<SettingsViewModel>();
        
        // Pages — Transient
        builder.Services.AddTransient<NotesPage>();
        builder.Services.AddTransient<NoteDetailPage>();
        builder.Services.AddTransient<SettingsPage>();

#if DEBUG
        builder.Logging.AddDebug();
#endif

        return builder.Build();
    }
}
```

```xml
<!-- AppShell.xaml — Final navigation structure -->
<?xml version="1.0" encoding="UTF-8" ?>
<Shell xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
       xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
       xmlns:views="clr-namespace:NoteMauiApp.Views"
       x:Class="NoteMauiApp.AppShell"
       Title="NoteMaui"
       FlyoutBehavior="Disabled">

    <Shell.Resources>
        <Style TargetType="TabBar">
            <Setter Property="Shell.TabBarBackgroundColor" Value="{AppThemeBinding 
                    Light=White, Dark=#1C1C1E}"/>
            <Setter Property="Shell.TabBarForegroundColor" Value="{StaticResource Primary}"/>
            <Setter Property="Shell.TabBarUnselectedColor" Value="Gray"/>
        </Style>
    </Shell.Resources>

    <TabBar>
        <ShellContent Title="โน้ต"
                      Icon="notes_tab.png"
                      ContentTemplate="{DataTemplate views:NotesPage}"
                      Route="notes"/>
        <ShellContent Title="ตั้งค่า"
                      Icon="settings_tab.png"
                      ContentTemplate="{DataTemplate views:SettingsPage}"
                      Route="settings"/>
    </TabBar>

</Shell>
```

```csharp
// AppShell.xaml.cs
public partial class AppShell : Shell
{
    public AppShell()
    {
        InitializeComponent();
        
        // ลงทะเบียน routes สำหรับ pages ที่ต้อง navigate ไป
        Routing.RegisterRoute(nameof(NoteDetailPage), typeof(NoteDetailPage));
    }
}
```

### Platform-specific Features

```csharp
// Platforms/Android/MainActivity.cs
namespace NoteMauiApp;

[Activity(Theme = "@style/Maui.SplashTheme", 
          MainLauncher = true,
          ConfigurationChanges = ConfigChanges.ScreenSize | ConfigChanges.Orientation | ConfigChanges.UiMode)]
public class MainActivity : MauiAppCompatActivity
{
    protected override void OnCreate(Bundle? savedInstanceState)
    {
        base.OnCreate(savedInstanceState);
        
        // กำหนด edge-to-edge display สำหรับ Android
        Window?.SetDecorFitsSystemWindows(false);
    }
}
```

```csharp
// Platforms/iOS/AppDelegate.cs
namespace NoteMauiApp;

[Register("AppDelegate")]
public class AppDelegate : MauiUIApplicationDelegate
{
    protected override MauiApp CreateMauiApp() => MauiProgram.CreateMauiApp();
}
```

---

## สรุปตาราง: .NET MAUI ครบถ้วน

| หัวข้อ | สิ่งที่เรียนรู้ | ใช้เมื่อ |
|--------|----------------|---------|
| **MAUI Overview** | Project structure, Shell, MauiProgram | เริ่มต้น project ใหม่ |
| **Pages & Controls** | ContentPage, CollectionView, Grid, FlexLayout | สร้าง UI layout |
| **MVVM** | ObservableProperty, RelayCommand, Binding | จัดการ state และ logic |
| **SQLite** | sqlite-net-pcl, CRUD operations | เก็บข้อมูลในเครื่อง |
| **Preferences** | Key-value storage | Settings ของ user |
| **SecureStorage** | Encrypted key-value | Token, password |
| **File System** | AppDataDirectory, CacheDirectory | ไฟล์ private ของ app |
| **Geolocation** | Permission, GetLocationAsync | ตำแหน่ง GPS |
| **Camera/Media** | MediaPicker, capture | รูปภาพ, วิดีโอ |
| **Biometric** | Plugin.Fingerprint | ยืนยันตัวตน |
| **Dark Mode** | AppThemeBinding, UserAppTheme | รองรับ theme |
| **Shell Navigation** | GoToAsync, QueryProperty | Navigation ระหว่าง pages |

---

## Key Packages สำคัญ

```xml
<PackageReference Include="CommunityToolkit.Maui" Version="7.0.1" />
<PackageReference Include="CommunityToolkit.Mvvm" Version="8.2.2" />
<PackageReference Include="sqlite-net-pcl" Version="1.9.172" />
<PackageReference Include="SQLitePCLRaw.bundle_green" Version="2.1.6" />
<PackageReference Include="Plugin.Fingerprint" Version="2.1.5" />
```

```csharp
// เพิ่ม CommunityToolkit.Maui ใน MauiProgram.cs
builder
    .UseMauiApp<App>()
    .UseMauiCommunityToolkit()  // ← เพิ่มบรรทัดนี้
    .ConfigureFonts(fonts => { ... });
```

---

## คำแนะนำเพิ่มเติม

### Performance Tips

```csharp
// 1. ใช้ CollectionView แทน ListView เสมอ
// ListView ถูก deprecated ใน MAUI

// 2. ใช้ RecycleElement สำหรับ list ยาว
<CollectionView ItemsSource="{Binding Items}"
                ItemSizingStrategy="MeasureFirstItem">  <!-- เร็วกว่า MeasureAllItems -->
```

```csharp
// 3. Compiled Bindings — เร็วกว่า reflection
// เพิ่ม x:DataType="vm:MyViewModel" ใน ContentPage
// XAML จะ compile binding ณ เวลา build แทน runtime

// 4. Image caching
<Image Source="https://example.com/image.jpg"
       Aspect="AspectFill"
       CachingStrategy="CacheItemForever"/>
```

```csharp
// 5. ใช้ MainThread.BeginInvokeOnMainThread สำหรับ UI updates จาก background thread
await Task.Run(async () =>
{
    var data = await FetchDataAsync();
    MainThread.BeginInvokeOnMainThread(() =>
    {
        Items.Clear();
        foreach (var item in data) Items.Add(item);
    });
});
```

### Testing MAUI Apps

```csharp
// ทดสอบ ViewModel โดยไม่ต้องมี UI
[TestClass]
public class NotesViewModelTests
{
    [TestMethod]
    public async Task LoadNotes_ShouldPopulateNotesList()
    {
        // Arrange
        var mockDb = new Mock<DatabaseService>();
        mockDb.Setup(db => db.GetAllAsync(null))
              .ReturnsAsync(new List<Note> { new Note { Title = "Test" } });
        
        var vm = new NotesViewModel(mockDb.Object);
        
        // Act
        await vm.LoadNotesCommand.ExecuteAsync(null);
        
        // Assert
        Assert.AreEqual(1, vm.Notes.Count);
        Assert.AreEqual("Test", vm.Notes[0].Title);
    }
}
```

---

## Troubleshooting ที่พบบ่อย

| ปัญหา | สาเหตุ | วิธีแก้ |
|-------|--------|---------|
| Build error: "Handler not found" | ลืม register service | เพิ่มใน MauiProgram.cs |
| Binding ไม่ work | ไม่มี `x:DataType` | เพิ่ม compiled binding |
| CollectionView ไม่แสดง | ItemsSource = null | ตรวจสอบ BindingContext |
| SQLite ไม่ save | ลืม await | ตรวจสอบ async/await |
| iOS build fail | Permission string ขาด | เพิ่มใน Info.plist |
| Android permission denied | ไม่ request runtime permission | ใช้ Permissions.RequestAsync |
| Dark mode ไม่ทำงาน | ใช้ hardcoded color | เปลี่ยนเป็น AppThemeBinding |

---

## Navigation Links

- ← [Part 65: ML.NET](./part65-mlnet.md)
- → [Part 67: IoT with C#](./part67-iot-csharp.md)

---

*จบ Part 66: .NET MAUI — ตอนนี้คุณสามารถสร้าง cross-platform mobile และ desktop app ด้วย C# ได้แล้ว!*
