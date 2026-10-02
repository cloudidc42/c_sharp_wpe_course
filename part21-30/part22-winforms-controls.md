# Part 22: WinForms Controls Advanced
## ขั้นตอนที่ 211-220: Controls ขั้นสูง

---

## 🎯 เป้าหมายของ Part นี้
- MenuStrip และ ContextMenuStrip
- ToolStrip (Toolbar)
- TabControl สำหรับ multi-page UI
- TreeView และ สร้าง File Explorer
- NumericUpDown, DateTimePicker, TrackBar
- ProgressBar
- PictureBox
- โปรแกรม Settings Manager แบบครบครัน

---

## ขั้นตอนที่ 211: MenuStrip

```csharp
// MenuStrip - เมนูบาร์หลัก
var menuStrip = new MenuStrip();

// File menu
var fileMenu = new ToolStripMenuItem("&File");
fileMenu.DropDownItems.AddRange(new ToolStripItem[]
{
    new ToolStripMenuItem("&New", null, (s,e) => NewFile(), Keys.Control | Keys.N),
    new ToolStripMenuItem("&Open...", null, (s,e) => OpenFile(), Keys.Control | Keys.O),
    new ToolStripMenuItem("&Save", null, (s,e) => SaveFile(), Keys.Control | Keys.S),
    new ToolStripMenuItem("Save &As...", null, (s,e) => SaveAs()),
    new ToolStripSeparator(),
    new ToolStripMenuItem("E&xit", null, (s,e) => Application.Exit(), Keys.Alt | Keys.F4)
});

// Edit menu
var editMenu = new ToolStripMenuItem("&Edit");
var pasteItem = new ToolStripMenuItem("&Paste", null, (s,e) => Paste(), Keys.Control | Keys.V)
{
    Enabled = Clipboard.ContainsText()
};
editMenu.DropDownItems.AddRange(new ToolStripItem[]
{
    new ToolStripMenuItem("&Undo", null, (s,e) => Undo(), Keys.Control | Keys.Z),
    new ToolStripMenuItem("&Redo", null, (s,e) => Redo(), Keys.Control | Keys.Y),
    new ToolStripSeparator(),
    new ToolStripMenuItem("Cu&t", null, (s,e) => Cut(), Keys.Control | Keys.X),
    new ToolStripMenuItem("&Copy", null, (s,e) => Copy(), Keys.Control | Keys.C),
    pasteItem
});

// Help menu with icon
var helpMenu = new ToolStripMenuItem("&Help");
helpMenu.DropDownItems.Add(new ToolStripMenuItem("&About", 
    SystemIcons.Information.ToBitmap(), (s,e) =>
    {
        MessageBox.Show("My App v1.0\nBuilt with C# WinForms", "About",
            MessageBoxButtons.OK, MessageBoxIcon.Information);
    }));

menuStrip.Items.AddRange(new ToolStripItem[] { fileMenu, editMenu, helpMenu });
MainMenuStrip = menuStrip;
Controls.Add(menuStrip);

// ContextMenuStrip - Right-click menu
var contextMenu = new ContextMenuStrip();
contextMenu.Items.AddRange(new ToolStripItem[]
{
    new ToolStripMenuItem("Cut", null, (s,e) => Cut()),
    new ToolStripMenuItem("Copy", null, (s,e) => Copy()),
    new ToolStripMenuItem("Paste", null, (s,e) => Paste()),
    new ToolStripSeparator(),
    new ToolStripMenuItem("Properties", null, (s,e) => ShowProperties())
});

// Attach to control
richTextBox.ContextMenuStrip = contextMenu;
```

---

## ขั้นตอนที่ 212: ToolStrip (Toolbar)

```csharp
// ToolStrip - แถบเครื่องมือ
var toolStrip = new ToolStrip { Dock = DockStyle.Top };

// Buttons with images
var btnNew = new ToolStripButton("New")
{
    Image = SystemIcons.Application.ToBitmap(),
    DisplayStyle = ToolStripItemDisplayStyle.ImageAndText,
    ToolTipText = "Create new file (Ctrl+N)"
};
btnNew.Click += (s,e) => NewFile();

// Separator
toolStrip.Items.Add(new ToolStripSeparator());

// ComboBox in toolbar
var cboFont = new ToolStripComboBox("Font")
{
    Width = 150,
    ToolTipText = "Select font"
};
cboFont.Items.AddRange(new[] { "Consolas", "Segoe UI", "Arial", "Times New Roman" });
cboFont.SelectedIndex = 0;
cboFont.SelectedIndexChanged += (s,e) => ChangeFont(cboFont.Text);

// TextBox in toolbar
var txtFind = new ToolStripTextBox { PlaceholderText = "Find...", Width = 120 };
var btnFind = new ToolStripButton("Find");
btnFind.Click += (s,e) => FindText(txtFind.Text);

toolStrip.Items.AddRange(new ToolStripItem[]
{
    btnNew,
    new ToolStripSeparator(),
    cboFont,
    new ToolStripSeparator(),
    txtFind, btnFind
});

Controls.Add(toolStrip);
```

---

## ขั้นตอนที่ 213: TabControl

```csharp
// TabControl - หลาย Tab
var tabControl = new TabControl { Dock = DockStyle.Fill };

// Tab 1: Text Editor
var tabEditor = new TabPage("📝 Editor");
var rtbEditor = new RichTextBox { Dock = DockStyle.Fill, Font = new Font("Consolas", 11) };
tabEditor.Controls.Add(rtbEditor);

// Tab 2: Settings
var tabSettings = new TabPage("⚙️ Settings");
var settingsPanel = CreateSettingsPanel();
tabSettings.Controls.Add(settingsPanel);

// Tab 3: Log
var tabLog = new TabPage("📋 Log");
var lstLog = new ListBox { Dock = DockStyle.Fill, Font = new Font("Consolas", 9) };
tabLog.Controls.Add(lstLog);

tabControl.TabPages.AddRange(new[] { tabEditor, tabSettings, tabLog });

// Tab events
tabControl.SelectedIndexChanged += (s,e) =>
{
    string tabName = tabControl.SelectedTab?.Text ?? "";
    Console.WriteLine($"Switched to: {tabName}");
};

// Tab with close button (custom)
tabControl.DrawMode = TabDrawMode.OwnerDrawFixed;
tabControl.DrawItem += (s,e) =>
{
    var tab = tabControl.TabPages[e.Index];
    e.Graphics.DrawString(tab.Text, e.Font!, Brushes.Black, e.Bounds);
    // Draw X button
    e.Graphics.DrawString("×", new Font("Arial", 8), Brushes.Red, 
        new Point(e.Bounds.Right - 15, e.Bounds.Top + 3));
};
```

---

## ขั้นตอนที่ 214: TreeView

```csharp
// TreeView - แสดงโครงสร้างแบบต้นไม้
var treeView = new TreeView
{
    Dock = DockStyle.Left,
    Width = 250,
    ShowLines = true,
    ShowPlusMinus = true,
    ShowRootLines = true,
    ImageList = CreateImageList()
};

// สร้าง nodes
var rootNode = new TreeNode("📁 My Projects") { ImageIndex = 0 };

var backendNode = new TreeNode("Backend") { ImageIndex = 1 };
backendNode.Nodes.AddRange(new[]
{
    new TreeNode("Controllers") { ImageIndex = 2 },
    new TreeNode("Services") { ImageIndex = 2 },
    new TreeNode("Models") { ImageIndex = 2 }
});

var frontendNode = new TreeNode("Frontend") { ImageIndex = 1 };
frontendNode.Nodes.AddRange(new[]
{
    new TreeNode("Components") { ImageIndex = 2 },
    new TreeNode("Pages") { ImageIndex = 2 }
});

rootNode.Nodes.AddRange(new[] { backendNode, frontendNode });
treeView.Nodes.Add(rootNode);
rootNode.Expand();

// Events
treeView.AfterSelect += (s,e) =>
{
    string path = GetFullPath(e.Node!);
    Console.WriteLine($"Selected: {path}");
};

treeView.AfterExpand += (s,e) =>
{
    // Lazy load children
    if (e.Node?.Tag is string dirPath && e.Node.Nodes.Count == 1 && e.Node.Nodes[0].Text == "...")
    {
        e.Node.Nodes.Clear();
        LoadDirectory(e.Node, dirPath);
    }
};

// Double-click to rename
treeView.NodeMouseDoubleClick += (s, e) =>
{
    if (e.Node != null) e.Node.BeginEdit();
};

string GetFullPath(TreeNode node)
{
    var parts = new List<string>();
    var current = node;
    while (current != null)
    {
        parts.Insert(0, current.Text);
        current = current.Parent;
    }
    return string.Join(" > ", parts);
}
```

---

## ขั้นตอนที่ 215: NumericUpDown, DateTimePicker, TrackBar

```csharp
// NumericUpDown - ตัวเลขที่ปรับได้
var numAge = new NumericUpDown
{
    Minimum = 0,
    Maximum = 150,
    Value = 25,
    DecimalPlaces = 0,
    Increment = 1,
    Font = new Font("Segoe UI", 11)
};
numAge.ValueChanged += (s,e) => Console.WriteLine($"Age: {numAge.Value}");

// สำหรับราคา
var numPrice = new NumericUpDown
{
    Minimum = 0,
    Maximum = 1000000,
    Value = 100,
    DecimalPlaces = 2,
    Increment = 0.5m,
    ThousandsSeparator = true
};

// DateTimePicker
var dtpBirthDate = new DateTimePicker
{
    Format = DateTimePickerFormat.Short,
    Value = DateTime.Today.AddYears(-25),
    MinDate = new DateTime(1900, 1, 1),
    MaxDate = DateTime.Today
};
dtpBirthDate.ValueChanged += (s,e) => 
{
    int age = DateTime.Today.Year - dtpBirthDate.Value.Year;
    Console.WriteLine($"Age: {age} years");
};

// DateTimePicker แบบเวลา
var dtpTime = new DateTimePicker
{
    Format = DateTimePickerFormat.Time,
    ShowUpDown = true
};

// TrackBar - slider
var trackVolume = new TrackBar
{
    Minimum = 0,
    Maximum = 100,
    Value = 50,
    TickFrequency = 10,
    SmallChange = 1,
    LargeChange = 10
};
var lblVolume = new Label { Text = "Volume: 50%" };
trackVolume.ValueChanged += (s,e) => lblVolume.Text = $"Volume: {trackVolume.Value}%";
```

---

## ขั้นตอนที่ 216: ProgressBar และ Async Operations

```csharp
// ProgressBar
var progressBar = new ProgressBar
{
    Minimum = 0,
    Maximum = 100,
    Value = 0,
    Style = ProgressBarStyle.Continuous  // หรือ Marquee สำหรับ indeterminate
};

var lblProgress = new Label { Text = "Ready" };
var btnStart = new Button { Text = "Start Task" };

btnStart.Click += async (s,e) =>
{
    btnStart.Enabled = false;
    progressBar.Value = 0;
    
    // Run long task on background thread
    var progress = new Progress<(int percent, string message)>(update =>
    {
        progressBar.Value = update.percent;
        lblProgress.Text = update.message;
    });
    
    await Task.Run(() => LongRunningTask(progress));
    
    lblProgress.Text = "✅ Complete!";
    btnStart.Enabled = true;
};

void LongRunningTask(IProgress<(int, string)> progress)
{
    for (int i = 0; i <= 100; i += 10)
    {
        Thread.Sleep(300);  // simulate work
        progress.Report((i, $"Processing... {i}%"));
    }
}

// Marquee style (indeterminate)
var progressMarquee = new ProgressBar
{
    Style = ProgressBarStyle.Marquee,
    MarqueeAnimationSpeed = 30
};
```

---

## ขั้นตอนที่ 217: PictureBox และ ImageViewer

```csharp
// PictureBox - แสดงรูปภาพ
var pictureBox = new PictureBox
{
    SizeMode = PictureBoxSizeMode.Zoom,  // Fit with aspect ratio
    Dock = DockStyle.Fill,
    BackColor = Color.Black,
    BorderStyle = BorderStyle.FixedSingle
};

// Load from file
pictureBox.Image = Image.FromFile("photo.jpg");

// Load from URL async
async Task LoadImageFromUrl(string url)
{
    pictureBox.Image = null;
    var progressMarquee = new ProgressBar { Style = ProgressBarStyle.Marquee };
    
    using var client = new System.Net.Http.HttpClient();
    var bytes = await client.GetByteArrayAsync(url);
    using var ms = new System.IO.MemoryStream(bytes);
    pictureBox.Image = Image.FromStream(ms);
}

// PictureBox modes
// StretchImage - stretch to fit (may distort)
// Zoom - fit with aspect ratio
// CenterImage - center without scaling
// AutoSize - resize PictureBox to image size
// Normal - top-left no scaling

// Mouse zoom/pan
float _zoom = 1.0f;
Point _pan = Point.Empty;
Point _lastMouse = Point.Empty;

pictureBox.MouseWheel += (s,e) =>
{
    _zoom += e.Delta > 0 ? 0.1f : -0.1f;
    _zoom = Math.Clamp(_zoom, 0.1f, 10f);
    pictureBox.Refresh();
};

pictureBox.Paint += (s,e) =>
{
    if (pictureBox.Image == null) return;
    var g = e.Graphics;
    g.TranslateTransform(_pan.X, _pan.Y);
    g.ScaleTransform(_zoom, _zoom);
    g.DrawImage(pictureBox.Image, 0, 0);
};
```

---

## ขั้นตอนที่ 218-220: โปรแกรม Settings Manager

```csharp
// SettingsManager.cs - Full application
using System;
using System.Collections.Generic;
using System.Drawing;
using System.IO;
using System.Text.Json;
using System.Windows.Forms;

namespace SettingsApp;

record AppSettings
{
    public string UserName { get; init; } = "";
    public string Theme { get; init; } = "Light";
    public int FontSize { get; init; } = 12;
    public bool AutoSave { get; init; } = true;
    public bool ShowToolbar { get; init; } = true;
    public string Language { get; init; } = "Thai";
    public string BackupPath { get; init; } = "";
    public int AutoSaveInterval { get; init; } = 5;
    public DateTime LastModified { get; init; } = DateTime.Now;
}

public class SettingsForm : Form
{
    private TabControl _tabs = null!;
    private AppSettings _settings;
    private readonly string _settingsPath;
    
    // General tab controls
    private TextBox _txtUserName = null!;
    private ComboBox _cboTheme = null!;
    private NumericUpDown _numFontSize = null!;
    private CheckBox _chkAutoSave = null!;
    private CheckBox _chkShowToolbar = null!;
    
    // Language tab
    private RadioButton _rdoThai = null!, _rdoEnglish = null!, _rdoJapanese = null!;
    
    // Backup tab
    private TextBox _txtBackupPath = null!;
    private NumericUpDown _numInterval = null!;
    
    // Buttons
    private Button _btnOk = null!, _btnCancel = null!, _btnApply = null!, _btnReset = null!;
    
    public SettingsForm()
    {
        _settingsPath = Path.Combine(
            Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData),
            "MyApp", "settings.json");
        _settings = LoadSettings();
        
        InitializeUI();
        LoadToUI();
    }
    
    private AppSettings LoadSettings()
    {
        try
        {
            if (File.Exists(_settingsPath))
            {
                string json = File.ReadAllText(_settingsPath);
                return JsonSerializer.Deserialize<AppSettings>(json) ?? new AppSettings();
            }
        }
        catch { /* Use defaults */ }
        return new AppSettings();
    }
    
    private void SaveSettings(AppSettings settings)
    {
        try
        {
            Directory.CreateDirectory(Path.GetDirectoryName(_settingsPath)!);
            string json = JsonSerializer.Serialize(settings, new JsonSerializerOptions { WriteIndented = true });
            File.WriteAllText(_settingsPath, json);
        }
        catch (Exception ex)
        {
            MessageBox.Show($"บันทึกการตั้งค่าล้มเหลว: {ex.Message}", "Error", 
                MessageBoxButtons.OK, MessageBoxIcon.Error);
        }
    }
    
    private void InitializeUI()
    {
        Text = "Settings";
        Size = new Size(550, 500);
        StartPosition = FormStartPosition.CenterParent;
        FormBorderStyle = FormBorderStyle.FixedDialog;
        MaximizeBox = false;
        MinimizeBox = false;
        
        _tabs = new TabControl { Dock = DockStyle.Fill, Padding = new Point(10, 5) };
        
        _tabs.TabPages.Add(CreateGeneralTab());
        _tabs.TabPages.Add(CreateLanguageTab());
        _tabs.TabPages.Add(CreateBackupTab());
        _tabs.TabPages.Add(CreateAboutTab());
        
        // Button panel
        var btnPanel = new Panel { Dock = DockStyle.Bottom, Height = 50 };
        
        _btnOk = new Button
        {
            Text = "ตกลง", Size = new Size(80, 30),
            Location = new Point(360, 10), DialogResult = DialogResult.OK
        };
        _btnCancel = new Button
        {
            Text = "ยกเลิก", Size = new Size(80, 30),
            Location = new Point(450, 10), DialogResult = DialogResult.Cancel
        };
        _btnApply = new Button { Text = "ใช้งาน", Size = new Size(80, 30), Location = new Point(270, 10) };
        _btnReset = new Button { Text = "รีเซ็ต", Size = new Size(80, 30), Location = new Point(10, 10) };
        
        _btnOk.Click += (s,e) => { SaveFromUI(); Close(); };
        _btnApply.Click += (s,e) => SaveFromUI();
        _btnReset.Click += (s,e) => ResetToDefaults();
        
        btnPanel.Controls.AddRange(new Control[] { _btnReset, _btnApply, _btnCancel, _btnOk });
        
        Controls.Add(_tabs);
        Controls.Add(btnPanel);
        AcceptButton = _btnOk;
        CancelButton = _btnCancel;
    }
    
    private TabPage CreateGeneralTab()
    {
        var tab = new TabPage("⚙️ ทั่วไป");
        var layout = new TableLayoutPanel
        {
            Dock = DockStyle.Fill,
            ColumnCount = 2,
            Padding = new Padding(15, 15, 15, 0),
            RowCount = 6
        };
        layout.ColumnStyles.Add(new ColumnStyle(SizeType.Absolute, 140));
        layout.ColumnStyles.Add(new ColumnStyle(SizeType.Percent, 100));
        
        for (int i = 0; i < 6; i++)
            layout.RowStyles.Add(new RowStyle(SizeType.Absolute, 40));
        
        // User Name
        layout.Controls.Add(MakeLabel("ชื่อผู้ใช้:"), 0, 0);
        _txtUserName = new TextBox { Dock = DockStyle.Fill, Font = new Font("Segoe UI", 10) };
        layout.Controls.Add(_txtUserName, 1, 0);
        
        // Theme
        layout.Controls.Add(MakeLabel("ธีม:"), 0, 1);
        _cboTheme = new ComboBox { Dock = DockStyle.Fill, DropDownStyle = ComboBoxStyle.DropDownList };
        _cboTheme.Items.AddRange(new[] { "Light", "Dark", "System" });
        layout.Controls.Add(_cboTheme, 1, 1);
        
        // Font Size
        layout.Controls.Add(MakeLabel("ขนาดตัวอักษร:"), 0, 2);
        _numFontSize = new NumericUpDown { Minimum = 8, Maximum = 32, Increment = 1 };
        layout.Controls.Add(_numFontSize, 1, 2);
        
        // Auto Save
        layout.Controls.Add(MakeLabel("บันทึกอัตโนมัติ:"), 0, 3);
        _chkAutoSave = new CheckBox { Text = "เปิดใช้งาน", Dock = DockStyle.Fill };
        layout.Controls.Add(_chkAutoSave, 1, 3);
        
        // Show Toolbar
        layout.Controls.Add(MakeLabel("แถบเครื่องมือ:"), 0, 4);
        _chkShowToolbar = new CheckBox { Text = "แสดง Toolbar", Dock = DockStyle.Fill };
        layout.Controls.Add(_chkShowToolbar, 1, 4);
        
        tab.Controls.Add(layout);
        return tab;
    }
    
    private TabPage CreateLanguageTab()
    {
        var tab = new TabPage("🌐 ภาษา");
        var groupBox = new GroupBox
        {
            Text = "เลือกภาษา",
            Dock = DockStyle.Fill,
            Padding = new Padding(20)
        };
        
        _rdoThai = new RadioButton { Text = "🇹🇭 ภาษาไทย", Location = new Point(20, 30), AutoSize = true };
        _rdoEnglish = new RadioButton { Text = "🇬🇧 English", Location = new Point(20, 65), AutoSize = true };
        _rdoJapanese = new RadioButton { Text = "🇯🇵 日本語", Location = new Point(20, 100), AutoSize = true };
        
        groupBox.Controls.AddRange(new Control[] { _rdoThai, _rdoEnglish, _rdoJapanese });
        tab.Controls.Add(groupBox);
        return tab;
    }
    
    private TabPage CreateBackupTab()
    {
        var tab = new TabPage("💾 สำรองข้อมูล");
        var layout = new TableLayoutPanel
        {
            Dock = DockStyle.Fill,
            ColumnCount = 3,
            Padding = new Padding(15),
            RowCount = 3
        };
        layout.ColumnStyles.Add(new ColumnStyle(SizeType.Absolute, 140));
        layout.ColumnStyles.Add(new ColumnStyle(SizeType.Percent, 100));
        layout.ColumnStyles.Add(new ColumnStyle(SizeType.Absolute, 80));
        
        for (int i = 0; i < 3; i++)
            layout.RowStyles.Add(new RowStyle(SizeType.Absolute, 40));
        
        layout.Controls.Add(MakeLabel("โฟลเดอร์สำรอง:"), 0, 0);
        _txtBackupPath = new TextBox { Dock = DockStyle.Fill };
        layout.Controls.Add(_txtBackupPath, 1, 0);
        
        var btnBrowse = new Button { Text = "เลือก..." };
        btnBrowse.Click += (s,e) =>
        {
            using var dlg = new FolderBrowserDialog { Description = "เลือกโฟลเดอร์สำรอง" };
            if (dlg.ShowDialog() == DialogResult.OK)
                _txtBackupPath.Text = dlg.SelectedPath;
        };
        layout.Controls.Add(btnBrowse, 2, 0);
        
        layout.Controls.Add(MakeLabel("บันทึกทุก (นาที):"), 0, 1);
        _numInterval = new NumericUpDown { Minimum = 1, Maximum = 60, Value = 5 };
        layout.Controls.Add(_numInterval, 1, 1);
        
        var btnBackupNow = new Button { Text = "สำรองทันที", Height = 32 };
        btnBackupNow.Click += (s,e) =>
        {
            if (string.IsNullOrEmpty(_txtBackupPath.Text))
            {
                MessageBox.Show("กรุณาเลือกโฟลเดอร์สำรองก่อน");
                return;
            }
            MessageBox.Show("สำรองข้อมูลสำเร็จ ✓", "Success", MessageBoxButtons.OK, MessageBoxIcon.Information);
        };
        layout.Controls.Add(btnBackupNow, 0, 2);
        
        tab.Controls.Add(layout);
        return tab;
    }
    
    private TabPage CreateAboutTab()
    {
        var tab = new TabPage("ℹ️ เกี่ยวกับ");
        var panel = new Panel { Dock = DockStyle.Fill };
        
        var lblApp = new Label
        {
            Text = "MyApp",
            Font = new Font("Segoe UI", 24, FontStyle.Bold),
            Location = new Point(20, 30),
            AutoSize = true,
            ForeColor = Color.Navy
        };
        
        var lblVersion = new Label
        {
            Text = "Version 1.0.0",
            Location = new Point(20, 75),
            AutoSize = true,
            Font = new Font("Segoe UI", 11)
        };
        
        var lblDesc = new Label
        {
            Text = "โปรแกรมตัวอย่าง WinForms\nพัฒนาด้วย C# .NET 8",
            Location = new Point(20, 105),
            AutoSize = true,
            Font = new Font("Segoe UI", 10)
        };
        
        panel.Controls.AddRange(new Control[] { lblApp, lblVersion, lblDesc });
        tab.Controls.Add(panel);
        return tab;
    }
    
    private Label MakeLabel(string text) => new Label
    {
        Text = text,
        Anchor = AnchorStyles.Right | AnchorStyles.Top,
        AutoSize = true,
        Margin = new Padding(0, 10, 5, 0)
    };
    
    private void LoadToUI()
    {
        _txtUserName.Text = _settings.UserName;
        _cboTheme.SelectedItem = _settings.Theme;
        if (_cboTheme.SelectedIndex < 0) _cboTheme.SelectedIndex = 0;
        _numFontSize.Value = _settings.FontSize;
        _chkAutoSave.Checked = _settings.AutoSave;
        _chkShowToolbar.Checked = _settings.ShowToolbar;
        
        _rdoThai.Checked = _settings.Language == "Thai";
        _rdoEnglish.Checked = _settings.Language == "English";
        _rdoJapanese.Checked = _settings.Language == "Japanese";
        
        _txtBackupPath.Text = _settings.BackupPath;
        _numInterval.Value = _settings.AutoSaveInterval;
    }
    
    private void SaveFromUI()
    {
        string lang = _rdoEnglish.Checked ? "English" : _rdoJapanese.Checked ? "Japanese" : "Thai";
        
        _settings = new AppSettings
        {
            UserName = _txtUserName.Text,
            Theme = _cboTheme.SelectedItem?.ToString() ?? "Light",
            FontSize = (int)_numFontSize.Value,
            AutoSave = _chkAutoSave.Checked,
            ShowToolbar = _chkShowToolbar.Checked,
            Language = lang,
            BackupPath = _txtBackupPath.Text,
            AutoSaveInterval = (int)_numInterval.Value,
            LastModified = DateTime.Now
        };
        
        SaveSettings(_settings);
        MessageBox.Show("บันทึกการตั้งค่าสำเร็จ ✓", "Success", 
            MessageBoxButtons.OK, MessageBoxIcon.Information);
    }
    
    private void ResetToDefaults()
    {
        var result = MessageBox.Show("รีเซ็ตการตั้งค่าทั้งหมดเป็นค่าเริ่มต้น?", "ยืนยัน",
            MessageBoxButtons.YesNo, MessageBoxIcon.Question);
        if (result == DialogResult.Yes)
        {
            _settings = new AppSettings();
            LoadToUI();
        }
    }
}
```

---

## 📝 สรุป Part 22

| Control | ใช้เมื่อ |
|---------|---------|
| MenuStrip | เมนูบาร์หลัก |
| ContextMenuStrip | Right-click menu |
| ToolStrip | Toolbar |
| TabControl | หลาย Tab |
| TreeView | โครงสร้างต้นไม้ |
| ProgressBar | แสดงความคืบหน้า |
| DateTimePicker | เลือกวันที่/เวลา |
| NumericUpDown | ตัวเลขที่ปรับได้ |

---

**ก่อนหน้า → [Part 21: WinForms Intro](part21-winforms-intro.md)**  
**ต่อไป → [Part 23: WinForms Data Binding](part23-winforms-databinding.md)**
