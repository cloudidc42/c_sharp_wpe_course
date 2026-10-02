# Part 24: WinForms Dialogs & File Operations
## ขั้นตอนที่ 231-240: Dialogs และการทำงานกับไฟล์

---

## 🎯 เป้าหมายของ Part นี้
- OpenFileDialog, SaveFileDialog, FolderBrowserDialog
- ColorDialog, FontDialog
- PrintDialog, PrintDocument
- Custom Modal และ Modeless Dialogs
- Splash Screen
- โปรแกรม Text Editor พร้อม Print

---

## ขั้นตอนที่ 231: OpenFileDialog และ SaveFileDialog

```csharp
// OpenFileDialog - เปิดไฟล์
var openDlg = new OpenFileDialog
{
    Title = "เลือกไฟล์",
    InitialDirectory = Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments),
    Filter = "Text Files|*.txt|All Files|*.*",
    FilterIndex = 1,
    Multiselect = false,
    CheckFileExists = true
};

if (openDlg.ShowDialog() == DialogResult.OK)
{
    string path = openDlg.FileName;
    string content = File.ReadAllText(path, System.Text.Encoding.UTF8);
    richTextBox.Text = content;
    Text = $"Editor - {Path.GetFileName(path)}";
}

// Open multiple files
openDlg.Multiselect = true;
if (openDlg.ShowDialog() == DialogResult.OK)
{
    foreach (string file in openDlg.FileNames)
        Console.WriteLine(file);
}

// SaveFileDialog
var saveDlg = new SaveFileDialog
{
    Title = "บันทึกไฟล์",
    Filter = "Text Files|*.txt|Rich Text|*.rtf|All Files|*.*",
    DefaultExt = "txt",
    FileName = "document",
    OverwritePrompt = true
};

if (saveDlg.ShowDialog() == DialogResult.OK)
{
    File.WriteAllText(saveDlg.FileName, richTextBox.Text, System.Text.Encoding.UTF8);
    MessageBox.Show($"บันทึกที่ {saveDlg.FileName}", "Success");
}

// FolderBrowserDialog
var folderDlg = new FolderBrowserDialog
{
    Description = "เลือกโฟลเดอร์สำหรับบันทึก",
    ShowNewFolderButton = true,
    SelectedPath = Environment.GetFolderPath(Environment.SpecialFolder.Desktop)
};

if (folderDlg.ShowDialog() == DialogResult.OK)
{
    string folder = folderDlg.SelectedPath;
    Console.WriteLine($"Selected: {folder}");
}
```

---

## ขั้นตอนที่ 232: ColorDialog และ FontDialog

```csharp
// ColorDialog - เลือกสี
var colorDlg = new ColorDialog
{
    AllowFullOpen = true,
    FullOpen = true,
    Color = richTextBox.SelectionColor
};

var btnFontColor = new Button { Text = "🎨 สี" };
btnFontColor.Click += (s,e) =>
{
    if (colorDlg.ShowDialog() == DialogResult.OK)
    {
        richTextBox.SelectionColor = colorDlg.Color;
    }
};

// FontDialog - เลือก Font
var fontDlg = new FontDialog
{
    ShowColor = true,
    ShowEffects = true,
    Font = richTextBox.SelectionFont ?? richTextBox.Font,
    Color = richTextBox.SelectionColor
};

var btnFont = new Button { Text = "🔤 ฟอนต์" };
btnFont.Click += (s,e) =>
{
    if (fontDlg.ShowDialog() == DialogResult.OK)
    {
        richTextBox.SelectionFont = fontDlg.Font;
        if (fontDlg.ShowColor)
            richTextBox.SelectionColor = fontDlg.Color;
    }
};

// Color picker button (custom - shows color swatch)
var btnColor = new Button { Text = "     Color", Width = 100 };
btnColor.Paint += (s,e) =>
{
    e.Graphics.FillRectangle(new SolidBrush(selectedColor), new Rectangle(5, 7, 20, 16));
};
btnColor.Click += (s,e) =>
{
    colorDlg.Color = selectedColor;
    if (colorDlg.ShowDialog() == DialogResult.OK)
    {
        selectedColor = colorDlg.Color;
        btnColor.Invalidate();
    }
};
```

---

## ขั้นตอนที่ 233: Printing

```csharp
// PrintDocument
using System.Drawing.Printing;

var printDoc = new PrintDocument();
var pageSettings = new PageSettings
{
    Margins = new Margins(50, 50, 50, 50)
};
printDoc.DefaultPageSettings = pageSettings;

string _textToPrint = "";
int _printLine = 0;
Font _printFont = new Font("Consolas", 11);

printDoc.BeginPrint += (s,e) =>
{
    _printLine = 0;
};

printDoc.PrintPage += (s,e) =>
{
    var lines = _textToPrint.Split('\n');
    float yPos = e.MarginBounds.Top;
    float lineHeight = _printFont.GetHeight(e.Graphics);
    
    while (_printLine < lines.Length)
    {
        if (yPos + lineHeight > e.MarginBounds.Bottom)
        {
            e.HasMorePages = true;
            return;
        }
        
        e.Graphics!.DrawString(lines[_printLine], _printFont, Brushes.Black,
            e.MarginBounds.Left, yPos);
        
        yPos += lineHeight;
        _printLine++;
    }
    
    e.HasMorePages = false;
};

// Print buttons
var btnPrint = new Button { Text = "🖨️ Print" };
btnPrint.Click += (s,e) =>
{
    _textToPrint = richTextBox.Text;
    
    using var printDlg = new PrintDialog
    {
        Document = printDoc,
        AllowSelection = true,
        AllowSomePages = true
    };
    
    if (printDlg.ShowDialog() == DialogResult.OK)
        printDoc.Print();
};

var btnPreview = new Button { Text = "👁️ Preview" };
btnPreview.Click += (s,e) =>
{
    _textToPrint = richTextBox.Text;
    
    using var previewDlg = new PrintPreviewDialog
    {
        Document = printDoc,
        Width = 900, Height = 700
    };
    previewDlg.ShowDialog();
};

var btnPageSetup = new Button { Text = "📄 Page Setup" };
btnPageSetup.Click += (s,e) =>
{
    using var pageSetupDlg = new PageSetupDialog { Document = printDoc };
    pageSetupDlg.ShowDialog();
};
```

---

## ขั้นตอนที่ 234: Custom Dialogs

```csharp
// Custom input dialog
public static class CustomDialog
{
    public static string? ShowInput(string prompt, string title = "Input", string defaultValue = "")
    {
        using var form = new Form
        {
            Text = title,
            Size = new Size(380, 150),
            StartPosition = FormStartPosition.CenterParent,
            FormBorderStyle = FormBorderStyle.FixedDialog,
            MaximizeBox = false, MinimizeBox = false
        };
        
        var lbl = new Label { Text = prompt, Location = new Point(12, 20), AutoSize = true };
        var txt = new TextBox { Location = new Point(12, 45), Size = new Size(340, 25), Text = defaultValue };
        var btnOk = new Button { Text = "ตกลง", DialogResult = DialogResult.OK, Location = new Point(195, 80), Size = new Size(75, 28) };
        var btnCancel = new Button { Text = "ยกเลิก", DialogResult = DialogResult.Cancel, Location = new Point(280, 80), Size = new Size(75, 28) };
        
        form.Controls.AddRange(new Control[] { lbl, txt, btnOk, btnCancel });
        form.AcceptButton = btnOk;
        form.CancelButton = btnCancel;
        
        txt.SelectAll();
        txt.Focus();
        
        return form.ShowDialog() == DialogResult.OK ? txt.Text : null;
    }
    
    public static bool ShowConfirm(string message, string title = "Confirm")
    {
        return MessageBox.Show(message, title, MessageBoxButtons.YesNo, 
            MessageBoxIcon.Question) == DialogResult.Yes;
    }
    
    public static void ShowError(string message, string title = "Error")
    {
        MessageBox.Show(message, title, MessageBoxButtons.OK, MessageBoxIcon.Error);
    }
    
    public static void ShowInfo(string message, string title = "Info")
    {
        MessageBox.Show(message, title, MessageBoxButtons.OK, MessageBoxIcon.Information);
    }
}

// Progress Dialog (modeless)
public class ProgressDialog : Form
{
    private ProgressBar _bar;
    private Label _lblMessage;
    private Button _btnCancel;
    private CancellationTokenSource _cts = new();
    
    public CancellationToken CancellationToken => _cts.Token;
    
    public ProgressDialog(string title = "Processing...")
    {
        Text = title;
        Size = new Size(400, 140);
        StartPosition = FormStartPosition.CenterParent;
        FormBorderStyle = FormBorderStyle.FixedDialog;
        MaximizeBox = false;
        ControlBox = false;
        
        _lblMessage = new Label
        {
            Text = "กำลังดำเนินการ...",
            Location = new Point(15, 15),
            Size = new Size(360, 20)
        };
        
        _bar = new ProgressBar
        {
            Location = new Point(15, 45),
            Size = new Size(360, 22),
            Style = ProgressBarStyle.Continuous
        };
        
        _btnCancel = new Button
        {
            Text = "ยกเลิก",
            Location = new Point(310, 80),
            Size = new Size(68, 28)
        };
        _btnCancel.Click += (s,e) => { _cts.Cancel(); _btnCancel.Enabled = false; };
        
        Controls.AddRange(new Control[] { _lblMessage, _bar, _btnCancel });
    }
    
    public void SetProgress(int percent, string? message = null)
    {
        if (InvokeRequired)
        {
            Invoke(() => SetProgress(percent, message));
            return;
        }
        _bar.Value = Math.Clamp(percent, 0, 100);
        if (message != null) _lblMessage.Text = message;
    }
}

// Usage
async void ProcessFiles()
{
    using var dlg = new ProgressDialog("กำลังประมวลผล...");
    dlg.Show();
    
    var files = Directory.GetFiles(@"C:\Data", "*.csv");
    for (int i = 0; i < files.Length; i++)
    {
        if (dlg.CancellationToken.IsCancellationRequested) break;
        
        dlg.SetProgress((i + 1) * 100 / files.Length, $"กำลังประมวลผล {Path.GetFileName(files[i])}");
        await Task.Run(() => ProcessFile(files[i]), dlg.CancellationToken);
    }
    
    dlg.Close();
    MessageBox.Show("เสร็จสิ้น!");
}
```

---

## ขั้นตอนที่ 235: Splash Screen

```csharp
// SplashScreen.cs
public class SplashScreen : Form
{
    private ProgressBar _bar;
    private Label _lblStatus;
    
    public SplashScreen()
    {
        FormBorderStyle = FormBorderStyle.None;
        Size = new Size(500, 300);
        StartPosition = FormStartPosition.CenterScreen;
        BackColor = Color.FromArgb(30, 30, 30);
        ForeColor = Color.White;
        
        var lblTitle = new Label
        {
            Text = "My Application",
            Font = new Font("Segoe UI", 28, FontStyle.Bold),
            ForeColor = Color.White,
            Location = new Point(40, 80),
            AutoSize = true
        };
        
        var lblVersion = new Label
        {
            Text = "Version 2.0",
            Font = new Font("Segoe UI", 12),
            ForeColor = Color.Silver,
            Location = new Point(43, 130),
            AutoSize = true
        };
        
        _bar = new ProgressBar
        {
            Location = new Point(40, 220),
            Size = new Size(420, 8),
            Style = ProgressBarStyle.Continuous,
            ForeColor = Color.DeepSkyBlue
        };
        
        _lblStatus = new Label
        {
            Text = "กำลังเริ่มต้น...",
            ForeColor = Color.Silver,
            Location = new Point(40, 240),
            Size = new Size(420, 20)
        };
        
        Controls.AddRange(new Control[] { lblTitle, lblVersion, _bar, _lblStatus });
        
        // Fade in
        Opacity = 0;
    }
    
    public async Task ShowAndLoadAsync(Func<IProgress<(int, string)>, Task> loadAction)
    {
        Show();
        
        // Fade in animation
        for (double op = 0; op <= 1; op += 0.05)
        {
            Opacity = op;
            await Task.Delay(20);
        }
        
        var progress = new Progress<(int percent, string message)>(update =>
        {
            _bar.Value = update.percent;
            _lblStatus.Text = update.message;
        });
        
        await loadAction(progress);
        
        // Fade out
        for (double op = 1; op >= 0; op -= 0.05)
        {
            Opacity = op;
            await Task.Delay(20);
        }
        
        Close();
    }
}

// Program.cs
[STAThread]
static async Task Main()
{
    Application.EnableVisualStyles();
    Application.SetCompatibleTextRenderingDefault(false);
    
    using var splash = new SplashScreen();
    await splash.ShowAndLoadAsync(async progress =>
    {
        progress.Report((10, "โหลด configuration..."));
        await Task.Delay(500);
        
        progress.Report((40, "เชื่อมต่อ database..."));
        await Task.Delay(800);
        
        progress.Report((70, "โหลด resources..."));
        await Task.Delay(600);
        
        progress.Report((100, "พร้อมใช้งาน!"));
        await Task.Delay(200);
    });
    
    Application.Run(new MainForm());
}
```

---

## ขั้นตอนที่ 236-240: โปรแกรม Text Editor สมบูรณ์

```csharp
// TextEditor.cs - Full Text Editor
using System;
using System.Drawing;
using System.Drawing.Printing;
using System.IO;
using System.Windows.Forms;

namespace TextEditor;

public class TextEditorForm : Form
{
    private RichTextBox _rtb = null!;
    private string _currentFile = "";
    private bool _modified = false;
    private PrintDocument _printDoc = null!;
    
    public TextEditorForm()
    {
        InitializeUI();
        SetupPrint();
        UpdateTitle();
    }
    
    private void InitializeUI()
    {
        Text = "Text Editor";
        Size = new Size(900, 650);
        StartPosition = FormStartPosition.CenterScreen;
        
        // Menu
        var menu = new MenuStrip();
        
        var fileMenu = new ToolStripMenuItem("&File");
        fileMenu.DropDownItems.AddRange(new ToolStripItem[]
        {
            new ToolStripMenuItem("&New", null, (s,e) => NewFile(), Keys.Control | Keys.N),
            new ToolStripMenuItem("&Open...", null, (s,e) => OpenFile(), Keys.Control | Keys.O),
            new ToolStripMenuItem("&Save", null, (s,e) => SaveFile(), Keys.Control | Keys.S),
            new ToolStripMenuItem("Save &As...", null, (s,e) => SaveAs()),
            new ToolStripSeparator(),
            new ToolStripMenuItem("&Print...", null, (s,e) => Print(), Keys.Control | Keys.P),
            new ToolStripMenuItem("Print Pre&view", null, (s,e) => PrintPreview()),
            new ToolStripSeparator(),
            new ToolStripMenuItem("E&xit", null, (s,e) => Close())
        });
        
        var editMenu = new ToolStripMenuItem("&Edit");
        editMenu.DropDownItems.AddRange(new ToolStripItem[]
        {
            new ToolStripMenuItem("&Undo", null, (s,e) => _rtb.Undo(), Keys.Control | Keys.Z),
            new ToolStripMenuItem("&Redo", null, (s,e) => _rtb.Redo(), Keys.Control | Keys.Y),
            new ToolStripSeparator(),
            new ToolStripMenuItem("Cu&t", null, (s,e) => _rtb.Cut(), Keys.Control | Keys.X),
            new ToolStripMenuItem("&Copy", null, (s,e) => _rtb.Copy(), Keys.Control | Keys.C),
            new ToolStripMenuItem("&Paste", null, (s,e) => _rtb.Paste(), Keys.Control | Keys.V),
            new ToolStripMenuItem("Select &All", null, (s,e) => _rtb.SelectAll(), Keys.Control | Keys.A),
            new ToolStripSeparator(),
            new ToolStripMenuItem("&Find...", null, (s,e) => ShowFind(), Keys.Control | Keys.F)
        });
        
        var formatMenu = new ToolStripMenuItem("F&ormat");
        formatMenu.DropDownItems.AddRange(new ToolStripItem[]
        {
            new ToolStripMenuItem("&Font...", null, (s,e) => ChangeFont()),
            new ToolStripMenuItem("&Color...", null, (s,e) => ChangeColor()),
            new ToolStripSeparator(),
            new ToolStripMenuItem("&Bold", null, (s,e) => ToggleBold(), Keys.Control | Keys.B),
            new ToolStripMenuItem("&Italic", null, (s,e) => ToggleItalic(), Keys.Control | Keys.I),
            new ToolStripMenuItem("&Underline", null, (s,e) => ToggleUnderline(), Keys.Control | Keys.U),
            new ToolStripSeparator(),
            new ToolStripMenuItem("Word &Wrap", null, ToggleWordWrap) { Checked = true, CheckOnClick = true }
        });
        
        menu.Items.AddRange(new ToolStripItem[] { fileMenu, editMenu, formatMenu });
        MainMenuStrip = menu;
        
        // Toolbar
        var toolbar = new ToolStrip { Dock = DockStyle.Top };
        toolbar.Items.AddRange(new ToolStripItem[]
        {
            new ToolStripButton("New") { ToolTipText = "New (Ctrl+N)" },
            new ToolStripButton("Open") { ToolTipText = "Open (Ctrl+O)" },
            new ToolStripButton("Save") { ToolTipText = "Save (Ctrl+S)" },
            new ToolStripSeparator(),
            new ToolStripButton("Cut") { ToolTipText = "Cut" },
            new ToolStripButton("Copy") { ToolTipText = "Copy" },
            new ToolStripButton("Paste") { ToolTipText = "Paste" },
            new ToolStripSeparator(),
            new ToolStripButton("Bold") { ToolTipText = "Bold" },
            new ToolStripButton("Italic") { ToolTipText = "Italic" },
            new ToolStripButton("Underline") { ToolTipText = "Underline" },
        });
        
        // RichTextBox
        _rtb = new RichTextBox
        {
            Dock = DockStyle.Fill,
            Font = new Font("Consolas", 12),
            WordWrap = true,
            ScrollBars = RichTextBoxScrollBars.Both
        };
        _rtb.TextChanged += (s,e) => { _modified = true; UpdateTitle(); };
        
        // Status bar
        var status = new StatusStrip();
        var lblPos = new ToolStripStatusLabel("Line 1, Col 1") { Spring = true };
        var lblLen = new ToolStripStatusLabel("0 chars");
        status.Items.AddRange(new ToolStripItem[] { lblPos, new ToolStripSeparator(), lblLen });
        
        _rtb.SelectionChanged += (s,e) =>
        {
            int line = _rtb.GetLineFromCharIndex(_rtb.SelectionStart) + 1;
            int col = _rtb.SelectionStart - _rtb.GetFirstCharIndexFromLine(line - 1) + 1;
            lblPos.Text = $"Line {line}, Col {col}";
            lblLen.Text = $"{_rtb.TextLength:N0} chars";
        };
        
        Controls.Add(_rtb);
        Controls.Add(toolbar);
        Controls.Add(menu);
        Controls.Add(status);
        
        FormClosing += (s,e) => { if (!ConfirmSave()) e.Cancel = true; };
    }
    
    private void SetupPrint()
    {
        _printDoc = new PrintDocument();
        int printLine = 0;
        string[] lines = Array.Empty<string>();
        
        _printDoc.BeginPrint += (s,e) => { lines = _rtb.Text.Split('\n'); printLine = 0; };
        _printDoc.PrintPage += (s,e) =>
        {
            var font = new Font("Consolas", 11);
            float y = e.MarginBounds.Top;
            float h = font.GetHeight(e.Graphics);
            
            while (printLine < lines.Length && y + h <= e.MarginBounds.Bottom)
            {
                e.Graphics!.DrawString(lines[printLine++], font, Brushes.Black, e.MarginBounds.Left, y);
                y += h;
            }
            e.HasMorePages = printLine < lines.Length;
        };
    }
    
    // Actions
    private void NewFile()
    {
        if (!ConfirmSave()) return;
        _rtb.Clear();
        _currentFile = "";
        _modified = false;
        UpdateTitle();
    }
    
    private void OpenFile()
    {
        if (!ConfirmSave()) return;
        using var dlg = new OpenFileDialog { Filter = "Text Files|*.txt|Rich Text|*.rtf|All Files|*.*" };
        if (dlg.ShowDialog() != DialogResult.OK) return;
        
        _currentFile = dlg.FileName;
        if (Path.GetExtension(_currentFile).ToLower() == ".rtf")
            _rtb.LoadFile(_currentFile, RichTextBoxStreamType.RichText);
        else
            _rtb.Text = File.ReadAllText(_currentFile);
        
        _modified = false;
        UpdateTitle();
    }
    
    private bool SaveFile()
    {
        if (string.IsNullOrEmpty(_currentFile)) return SaveAs();
        DoSave(_currentFile);
        return true;
    }
    
    private bool SaveAs()
    {
        using var dlg = new SaveFileDialog { Filter = "Text Files|*.txt|Rich Text|*.rtf" };
        if (dlg.ShowDialog() != DialogResult.OK) return false;
        _currentFile = dlg.FileName;
        DoSave(_currentFile);
        return true;
    }
    
    private void DoSave(string path)
    {
        if (Path.GetExtension(path).ToLower() == ".rtf")
            _rtb.SaveFile(path, RichTextBoxStreamType.RichText);
        else
            File.WriteAllText(path, _rtb.Text);
        _modified = false;
        UpdateTitle();
    }
    
    private bool ConfirmSave()
    {
        if (!_modified) return true;
        var r = MessageBox.Show("บันทึกการเปลี่ยนแปลง?", "ยืนยัน",
            MessageBoxButtons.YesNoCancel, MessageBoxIcon.Question);
        if (r == DialogResult.Cancel) return false;
        if (r == DialogResult.Yes) SaveFile();
        return true;
    }
    
    private void Print()
    {
        using var dlg = new PrintDialog { Document = _printDoc };
        if (dlg.ShowDialog() == DialogResult.OK) _printDoc.Print();
    }
    
    private void PrintPreview()
    {
        using var dlg = new PrintPreviewDialog { Document = _printDoc, Width = 900, Height = 700 };
        dlg.ShowDialog();
    }
    
    private void ChangeFont()
    {
        using var dlg = new FontDialog { Font = _rtb.SelectionFont ?? _rtb.Font, ShowColor = true };
        if (dlg.ShowDialog() == DialogResult.OK) { _rtb.SelectionFont = dlg.Font; _rtb.SelectionColor = dlg.Color; }
    }
    
    private void ChangeColor()
    {
        using var dlg = new ColorDialog { Color = _rtb.SelectionColor };
        if (dlg.ShowDialog() == DialogResult.OK) _rtb.SelectionColor = dlg.Color;
    }
    
    private void ToggleBold()
    {
        var f = _rtb.SelectionFont ?? _rtb.Font;
        _rtb.SelectionFont = new Font(f, f.Bold ? f.Style & ~FontStyle.Bold : f.Style | FontStyle.Bold);
    }
    
    private void ToggleItalic()
    {
        var f = _rtb.SelectionFont ?? _rtb.Font;
        _rtb.SelectionFont = new Font(f, f.Italic ? f.Style & ~FontStyle.Italic : f.Style | FontStyle.Italic);
    }
    
    private void ToggleUnderline()
    {
        var f = _rtb.SelectionFont ?? _rtb.Font;
        _rtb.SelectionFont = new Font(f, f.Underline ? f.Style & ~FontStyle.Underline : f.Style | FontStyle.Underline);
    }
    
    private void ToggleWordWrap(object? s, EventArgs e)
    {
        _rtb.WordWrap = !_rtb.WordWrap;
    }
    
    private void ShowFind()
    {
        var findDlg = new Form { Text = "Find", Size = new Size(350, 120), StartPosition = FormStartPosition.CenterParent, FormBorderStyle = FormBorderStyle.FixedDialog };
        var txt = new TextBox { Location = new Point(12, 15), Size = new Size(220, 25) };
        var btnFind = new Button { Text = "Find Next", Location = new Point(245, 13), Size = new Size(85, 27) };
        
        btnFind.Click += (s2, e2) =>
        {
            int idx = _rtb.Text.IndexOf(txt.Text, _rtb.SelectionStart + _rtb.SelectionLength, StringComparison.OrdinalIgnoreCase);
            if (idx >= 0) { _rtb.Select(idx, txt.Text.Length); _rtb.ScrollToCaret(); }
            else MessageBox.Show("ไม่พบข้อความ");
        };
        
        findDlg.Controls.AddRange(new Control[] { new Label { Text = "ค้นหา:", Location = new Point(12, -3), AutoSize = true }, txt, btnFind });
        findDlg.Show(this);
    }
    
    private void UpdateTitle()
    {
        string name = string.IsNullOrEmpty(_currentFile) ? "Untitled" : Path.GetFileName(_currentFile);
        Text = $"{(_modified ? "* " : "")}{name} - Text Editor";
    }
}
```

---

## 📝 สรุป Part 24

| Dialog | ใช้เมื่อ |
|--------|---------|
| OpenFileDialog | เปิดไฟล์ |
| SaveFileDialog | บันทึกไฟล์ |
| FolderBrowserDialog | เลือกโฟลเดอร์ |
| ColorDialog | เลือกสี |
| FontDialog | เลือกฟอนต์ |
| PrintDialog | พิมพ์ |
| Custom Form | Dialog ที่กำหนดเอง |

---

**ก่อนหน้า → [Part 23: WinForms Data Binding](part23-winforms-databinding.md)**  
**ต่อไป → [Part 25: WinForms Graphics & Drawing](part25-winforms-graphics.md)**
