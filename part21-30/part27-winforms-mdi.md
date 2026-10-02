# Part 27: WinForms MDI, Docking & Layout
## ขั้นตอนที่ 261-270: MDI Applications

---

## 🎯 เป้าหมายของ Part นี้
- MDI (Multiple Document Interface)
- Panel Docking และ Splitter
- SplitContainer
- UserControl (Reusable components)
- การสร้าง IDE-like Layout
- โปรแกรม MDI Note Editor

---

## ขั้นตอนที่ 261: MDI Application

```csharp
// MDI Parent Form
public class MdiParentForm : Form
{
    private int _childCount = 0;
    
    public MdiParentForm()
    {
        Text = "MDI Application";
        Size = new Size(1200, 800);
        IsMdiContainer = true;  // <-- เปิด MDI mode
        
        // Menu
        var menu = new MenuStrip();
        
        var fileMenu = new ToolStripMenuItem("&File");
        fileMenu.DropDownItems.Add("New Document", null, (s,e) => NewDocument());
        fileMenu.DropDownItems.Add("New Image", null, (s,e) => NewImage());
        fileMenu.DropDownItems.Add(new ToolStripSeparator());
        fileMenu.DropDownItems.Add("Exit", null, (s,e) => Close());
        
        // Window menu - MDI window management
        var windowMenu = new ToolStripMenuItem("&Window");
        windowMenu.DropDownItems.Add("Cascade", null, (s,e) => LayoutMdi(MdiLayout.Cascade));
        windowMenu.DropDownItems.Add("Tile Horizontal", null, (s,e) => LayoutMdi(MdiLayout.TileHorizontal));
        windowMenu.DropDownItems.Add("Tile Vertical", null, (s,e) => LayoutMdi(MdiLayout.TileVertical));
        windowMenu.DropDownItems.Add("Arrange Icons", null, (s,e) => LayoutMdi(MdiLayout.ArrangeIcons));
        windowMenu.DropDownItems.Add(new ToolStripSeparator());
        
        // MdiWindowListItem automatically adds child window list
        var mergeMenu = new ToolStripMenuItem("&Window");
        mergeMenu.MergeAction = MergeAction.MatchOnly;
        
        menu.Items.AddRange(new ToolStripItem[] { fileMenu, windowMenu });
        MdiWindowListItem = windowMenu;  // Auto list MDI children
        MainMenuStrip = menu;
        
        // StatusBar
        var status = new StatusStrip();
        var lblStatus = new ToolStripStatusLabel("Ready") { Spring = true };
        status.Items.Add(lblStatus);
        
        MdiChildActivate += (s,e) =>
        {
            lblStatus.Text = ActiveMdiChild?.Text ?? "No document";
        };
        
        Controls.Add(menu);
        Controls.Add(status);
    }
    
    private void NewDocument()
    {
        var child = new DocumentForm($"Document {++_childCount}")
        {
            MdiParent = this  // <-- กำหนด parent
        };
        child.Show();
    }
    
    private void NewImage()
    {
        var child = new ImageForm($"Image {++_childCount}")
        {
            MdiParent = this
        };
        child.Show();
    }
}

// MDI Child Form
public class DocumentForm : Form
{
    private RichTextBox _rtb;
    
    public DocumentForm(string title)
    {
        Text = title;
        Size = new Size(600, 400);
        StartPosition = FormStartPosition.Automatic;  // MDI auto-positions
        
        _rtb = new RichTextBox { Dock = DockStyle.Fill, Font = new Font("Consolas", 11) };
        Controls.Add(_rtb);
        
        // Merge child menu with parent
        var editMenu = new ToolStripMenuItem("Edit");
        editMenu.DropDownItems.Add(new ToolStripMenuItem("Copy", null, (s,e) => _rtb.Copy()));
        editMenu.DropDownItems.Add(new ToolStripMenuItem("Paste", null, (s,e) => _rtb.Paste()));
        
        // FormBorderStyle for MDI children
        FormBorderStyle = FormBorderStyle.Sizable;
    }
}
```

---

## ขั้นตอนที่ 262: SplitContainer

```csharp
// SplitContainer - แบ่ง panel ออกเป็น 2 ส่วน
var splitMain = new SplitContainer
{
    Dock = DockStyle.Fill,
    Orientation = Orientation.Vertical,  // แนวตั้ง
    SplitterDistance = 250,  // ขนาดส่วนซ้าย
    Panel1MinSize = 150,
    Panel2MinSize = 200,
    SplitterWidth = 5
};

// Panel1 (Left) - Tree/Explorer
var tree = new TreeView { Dock = DockStyle.Fill };
splitMain.Panel1.Controls.Add(tree);

// Panel2 (Right) - Content  
// Nested split
var splitRight = new SplitContainer
{
    Dock = DockStyle.Fill,
    Orientation = Orientation.Horizontal,
    SplitterDistance = 300
};

var editor = new RichTextBox { Dock = DockStyle.Fill };
splitRight.Panel1.Controls.Add(editor);

var output = new ListBox { Dock = DockStyle.Fill };
splitRight.Panel2.Controls.Add(output);

splitMain.Panel2.Controls.Add(splitRight);

// Handle splitter moved
splitMain.SplitterMoved += (s,e) =>
    Console.WriteLine($"Splitter at: {splitMain.SplitterDistance}px");

Controls.Add(splitMain);
```

---

## ขั้นตอนที่ 263: UserControl - Reusable Components

```csharp
// UserControl - component ที่ใช้ซ้ำได้
public class CardControl : UserControl
{
    private Label _lblTitle = null!;
    private Label _lblValue = null!;
    private Label _lblIcon = null!;
    private Panel _pnlTop = null!;
    
    // Properties
    private string _title = "Card Title";
    private string _value = "0";
    private string _icon = "📊";
    private Color _accentColor = Color.SteelBlue;
    
    [System.ComponentModel.Category("Card")]
    public string CardTitle { get => _title; set { _title = value; _lblTitle.Text = value; } }
    
    [System.ComponentModel.Category("Card")]
    public string Value { get => _value; set { _value = value; _lblValue.Text = value; } }
    
    [System.ComponentModel.Category("Card")]
    public string Icon { get => _icon; set { _icon = value; _lblIcon.Text = value; } }
    
    [System.ComponentModel.Category("Card")]
    public Color AccentColor
    {
        get => _accentColor;
        set { _accentColor = value; _pnlTop.BackColor = value; }
    }
    
    public CardControl()
    {
        Size = new Size(200, 100);
        BackColor = Color.White;
        BorderStyle = BorderStyle.FixedSingle;
        
        _pnlTop = new Panel
        {
            Dock = DockStyle.Top,
            Height = 5,
            BackColor = _accentColor
        };
        
        _lblIcon = new Label
        {
            Text = _icon,
            Font = new Font("Segoe UI Emoji", 24),
            Location = new Point(10, 15),
            AutoSize = true
        };
        
        _lblTitle = new Label
        {
            Text = _title,
            Font = new Font("Segoe UI", 8, FontStyle.Bold),
            ForeColor = Color.Gray,
            Location = new Point(55, 15),
            AutoSize = true
        };
        
        _lblValue = new Label
        {
            Text = _value,
            Font = new Font("Segoe UI", 22, FontStyle.Bold),
            ForeColor = Color.FromArgb(50, 50, 50),
            Location = new Point(55, 35),
            AutoSize = true
        };
        
        Controls.AddRange(new Control[] { _pnlTop, _lblIcon, _lblTitle, _lblValue });
    }
}

// Usage
var cardSales = new CardControl
{
    CardTitle = "ยอดขาย",
    Value = "฿125,450",
    Icon = "💰",
    AccentColor = Color.Green,
    Location = new Point(10, 10)
};

var cardOrders = new CardControl
{
    CardTitle = "คำสั่งซื้อ",
    Value = "342",
    Icon = "📦",
    AccentColor = Color.Blue,
    Location = new Point(220, 10)
};

Controls.Add(cardSales);
Controls.Add(cardOrders);
```

---

## ขั้นตอนที่ 264-270: โปรแกรม MDI Note Editor

```csharp
// MdiNoteEditor.cs - Complete MDI Application
using System;
using System.Collections.Generic;
using System.Drawing;
using System.IO;
using System.Windows.Forms;

namespace MdiNotes;

public class MainForm : Form
{
    private int _docCount = 0;
    private MenuStrip _menu = null!;
    private StatusStrip _statusBar = null!;
    private ToolStripStatusLabel _lblStatus = null!;
    private ToolStripStatusLabel _lblActiveDoc = null!;
    
    public MainForm()
    {
        Text = "MDI Note Editor";
        Size = new Size(1200, 800);
        IsMdiContainer = true;
        StartPosition = FormStartPosition.CenterScreen;
        MdiChildActivate += MainForm_MdiChildActivate;
        
        BuildMenu();
        BuildToolbar();
        BuildStatusBar();
        
        // Start with one document
        NewDocument();
    }
    
    private void BuildMenu()
    {
        _menu = new MenuStrip();
        
        // File
        var file = new ToolStripMenuItem("&File");
        file.DropDownItems.AddRange(new ToolStripItem[]
        {
            new ToolStripMenuItem("&New", null, (s,e) => NewDocument(), Keys.Control | Keys.N),
            new ToolStripMenuItem("&Open...", null, (s,e) => OpenDocument(), Keys.Control | Keys.O),
            new ToolStripMenuItem("Close All", null, (s,e) => CloseAllDocuments()),
            new ToolStripSeparator(),
            new ToolStripMenuItem("E&xit", null, (s,e) => Close())
        });
        
        // Edit
        var edit = new ToolStripMenuItem("&Edit");
        edit.DropDownItems.AddRange(new ToolStripItem[]
        {
            new ToolStripMenuItem("&Undo", null, (s,e) => ActiveEditor?.Undo(), Keys.Control | Keys.Z),
            new ToolStripMenuItem("&Redo", null, (s,e) => ActiveEditor?.Redo(), Keys.Control | Keys.Y),
            new ToolStripSeparator(),
            new ToolStripMenuItem("&Find...", null, (s,e) => ShowFind(), Keys.Control | Keys.F),
            new ToolStripMenuItem("&Replace...", null, (s,e) => ShowReplace(), Keys.Control | Keys.H),
            new ToolStripSeparator(),
            new ToolStripMenuItem("Select &All", null, (s,e) => ActiveEditor?.SelectAll(), Keys.Control | Keys.A),
        });
        
        // View
        var view = new ToolStripMenuItem("&View");
        var wrapItem = new ToolStripMenuItem("Word Wrap") { CheckOnClick = true, Checked = true };
        wrapItem.Click += (s,e) => 
        {
            if (ActiveMdiChild is NoteForm note)
                note.WordWrap = wrapItem.Checked;
        };
        view.DropDownItems.Add(wrapItem);
        
        // Window
        var window = new ToolStripMenuItem("&Window");
        window.DropDownItems.AddRange(new ToolStripItem[]
        {
            new ToolStripMenuItem("Cascade", null, (s,e) => LayoutMdi(MdiLayout.Cascade)),
            new ToolStripMenuItem("Tile Horizontal", null, (s,e) => LayoutMdi(MdiLayout.TileHorizontal)),
            new ToolStripMenuItem("Tile Vertical", null, (s,e) => LayoutMdi(MdiLayout.TileVertical)),
        });
        
        _menu.Items.AddRange(new ToolStripItem[] { file, edit, view, window });
        MdiWindowListItem = window;
        MainMenuStrip = _menu;
        Controls.Add(_menu);
    }
    
    private void BuildToolbar()
    {
        var toolbar = new ToolStrip { Dock = DockStyle.Top };
        
        ToolStripButton Btn(string txt, string tip, Action act) 
        {
            var b = new ToolStripButton(txt) { ToolTipText = tip, AutoSize = true };
            b.Click += (s,e) => act();
            return b;
        }
        
        toolbar.Items.AddRange(new ToolStripItem[]
        {
            Btn("New", "New document", NewDocument),
            Btn("Open", "Open file", OpenDocument),
            Btn("Save", "Save file", () => (ActiveMdiChild as NoteForm)?.Save()),
            new ToolStripSeparator(),
            Btn("Bold", "Bold", () => ToggleFormat(FontStyle.Bold)),
            Btn("Italic", "Italic", () => ToggleFormat(FontStyle.Italic)),
            Btn("Underline", "Underline", () => ToggleFormat(FontStyle.Underline)),
            new ToolStripSeparator(),
            Btn("Cascade", "Cascade windows", () => LayoutMdi(MdiLayout.Cascade)),
            Btn("Tile H", "Tile horizontal", () => LayoutMdi(MdiLayout.TileHorizontal)),
            Btn("Tile V", "Tile vertical", () => LayoutMdi(MdiLayout.TileVertical)),
        });
        
        Controls.Add(toolbar);
    }
    
    private void BuildStatusBar()
    {
        _statusBar = new StatusStrip();
        _lblActiveDoc = new ToolStripStatusLabel("No document") { Width = 200 };
        _lblStatus = new ToolStripStatusLabel("Ready") { Spring = true };
        var lblTime = new ToolStripStatusLabel(DateTime.Now.ToString("HH:mm"));
        
        var timer = new System.Windows.Forms.Timer { Interval = 1000 };
        timer.Tick += (s,e) => lblTime.Text = DateTime.Now.ToString("HH:mm:ss");
        timer.Start();
        
        _statusBar.Items.AddRange(new ToolStripItem[] { _lblActiveDoc, _lblStatus, lblTime });
        Controls.Add(_statusBar);
    }
    
    private RichTextBox? ActiveEditor =>
        (ActiveMdiChild as NoteForm)?.Editor;
    
    private void NewDocument()
    {
        var note = new NoteForm($"Note {++_docCount}") { MdiParent = this };
        note.Modified += (s,e) => UpdateStatus();
        note.Show();
        UpdateStatus();
    }
    
    private void OpenDocument()
    {
        using var dlg = new OpenFileDialog { Filter = "Text Files|*.txt|Rich Text|*.rtf|All Files|*.*" };
        if (dlg.ShowDialog() != DialogResult.OK) return;
        
        var note = new NoteForm(Path.GetFileName(dlg.FileName)) { MdiParent = this };
        note.Load(dlg.FileName);
        note.Modified += (s,e) => UpdateStatus();
        note.Show();
    }
    
    private void CloseAllDocuments()
    {
        foreach (Form child in MdiChildren.ToArray())
            child.Close();
    }
    
    private void ToggleFormat(FontStyle style)
    {
        var editor = ActiveEditor;
        if (editor == null) return;
        var f = editor.SelectionFont ?? editor.Font;
        editor.SelectionFont = new Font(f, f.Style.HasFlag(style) ? f.Style & ~style : f.Style | style);
    }
    
    private void ShowFind()
    {
        var editor = ActiveEditor;
        if (editor == null) return;
        
        using var dlg = new FindReplaceDialog(editor, isReplace: false);
        dlg.ShowDialog(this);
    }
    
    private void ShowReplace()
    {
        var editor = ActiveEditor;
        if (editor == null) return;
        
        using var dlg = new FindReplaceDialog(editor, isReplace: true);
        dlg.ShowDialog(this);
    }
    
    private void MainForm_MdiChildActivate(object? sender, EventArgs e)
    {
        UpdateStatus();
    }
    
    private void UpdateStatus()
    {
        _lblActiveDoc.Text = ActiveMdiChild?.Text ?? "No document";
        int count = MdiChildren.Length;
        _lblStatus.Text = $"{count} document{(count != 1 ? "s" : "")} open";
    }
}

// Note Form (MDI Child)
public class NoteForm : Form
{
    public RichTextBox Editor { get; } = null!;
    private string _filePath = "";
    private bool _isModified = false;
    
    public event EventHandler? Modified;
    
    public bool WordWrap
    {
        get => Editor.WordWrap;
        set => Editor.WordWrap = value;
    }
    
    public NoteForm(string title)
    {
        Text = title;
        Size = new Size(600, 450);
        StartPosition = FormStartPosition.Automatic;
        
        Editor = new RichTextBox
        {
            Dock = DockStyle.Fill,
            Font = new Font("Consolas", 11),
            AcceptsTab = true,
            WordWrap = true
        };
        
        Editor.TextChanged += (s,e) =>
        {
            if (!_isModified)
            {
                _isModified = true;
                Text = "* " + (string.IsNullOrEmpty(_filePath) 
                    ? Text.TrimStart('*', ' ') 
                    : Path.GetFileName(_filePath));
                Modified?.Invoke(this, EventArgs.Empty);
            }
        };
        
        Editor.KeyDown += (s,e) =>
        {
            if (e.Control && e.KeyCode == Keys.S) { Save(); e.SuppressKeyPress = true; }
        };
        
        Controls.Add(Editor);
        FormClosing += (s,e) =>
        {
            if (_isModified)
            {
                var result = MessageBox.Show($"Save '{Text.TrimStart('*', ' ')}' before closing?",
                    "Unsaved Changes", MessageBoxButtons.YesNoCancel, MessageBoxIcon.Question);
                if (result == DialogResult.Cancel) { e.Cancel = true; return; }
                if (result == DialogResult.Yes) Save();
            }
        };
    }
    
    public void Load(string path)
    {
        _filePath = path;
        Text = Path.GetFileName(path);
        if (path.EndsWith(".rtf", StringComparison.OrdinalIgnoreCase))
            Editor.LoadFile(path, RichTextBoxStreamType.RichText);
        else
            Editor.Text = File.ReadAllText(path);
        _isModified = false;
    }
    
    public void Save()
    {
        if (string.IsNullOrEmpty(_filePath))
        {
            using var dlg = new SaveFileDialog { Filter = "Text Files|*.txt|Rich Text|*.rtf" };
            if (dlg.ShowDialog() != DialogResult.OK) return;
            _filePath = dlg.FileName;
        }
        
        if (_filePath.EndsWith(".rtf"))
            Editor.SaveFile(_filePath, RichTextBoxStreamType.RichText);
        else
            File.WriteAllText(_filePath, Editor.Text);
        
        _isModified = false;
        Text = Path.GetFileName(_filePath);
    }
}

// Find & Replace Dialog
public class FindReplaceDialog : Form
{
    private TextBox _txtFind, _txtReplace;
    private CheckBox _chkMatchCase, _chkWholeWord;
    private RichTextBox _editor;
    
    public FindReplaceDialog(RichTextBox editor, bool isReplace)
    {
        _editor = editor;
        Text = isReplace ? "Find & Replace" : "Find";
        Size = new Size(380, isReplace ? 200 : 160);
        StartPosition = FormStartPosition.CenterParent;
        FormBorderStyle = FormBorderStyle.FixedDialog;
        MaximizeBox = false;
        
        var lblFind = new Label { Text = "Find:", Location = new Point(12, 15), AutoSize = true };
        _txtFind = new TextBox { Location = new Point(75, 12), Size = new Size(200, 23) };
        var btnFind = new Button { Text = "Find Next", Location = new Point(285, 10), Size = new Size(80, 26) };
        btnFind.Click += (s,e) => FindNext();
        
        _txtReplace = new TextBox { Location = new Point(75, 45), Size = new Size(200, 23) };
        var btnReplace = new Button { Text = "Replace", Location = new Point(285, 43), Size = new Size(80, 26) };
        var btnReplaceAll = new Button { Text = "Replace All", Location = new Point(285, 75), Size = new Size(80, 26) };
        
        if (isReplace)
        {
            var lblReplace = new Label { Text = "Replace:", Location = new Point(12, 48), AutoSize = true };
            Controls.Add(lblReplace);
            Controls.Add(_txtReplace);
            Controls.Add(btnReplace);
            Controls.Add(btnReplaceAll);
            btnReplace.Click += (s,e) => Replace();
            btnReplaceAll.Click += (s,e) => ReplaceAll();
        }
        
        _chkMatchCase = new CheckBox { Text = "Match case", Location = new Point(12, 110), AutoSize = true };
        _chkWholeWord = new CheckBox { Text = "Whole word", Location = new Point(12, 130), AutoSize = true };
        
        Controls.AddRange(new Control[] { lblFind, _txtFind, btnFind, _chkMatchCase, _chkWholeWord });
        AcceptButton = btnFind;
    }
    
    private void FindNext()
    {
        if (string.IsNullOrEmpty(_txtFind.Text)) return;
        
        var comparison = _chkMatchCase.Checked 
            ? StringComparison.Ordinal 
            : StringComparison.OrdinalIgnoreCase;
        
        int start = _editor.SelectionStart + _editor.SelectionLength;
        int idx = _editor.Text.IndexOf(_txtFind.Text, start, comparison);
        
        if (idx < 0) idx = _editor.Text.IndexOf(_txtFind.Text, 0, comparison);
        if (idx >= 0) { _editor.Select(idx, _txtFind.Text.Length); _editor.ScrollToCaret(); }
        else MessageBox.Show("ไม่พบข้อความ");
    }
    
    private void Replace()
    {
        if (_editor.SelectedText.Equals(_txtFind.Text, StringComparison.OrdinalIgnoreCase))
            _editor.SelectedText = _txtReplace.Text;
        FindNext();
    }
    
    private void ReplaceAll()
    {
        var comparison = _chkMatchCase.Checked ? StringComparison.Ordinal : StringComparison.OrdinalIgnoreCase;
        string newText = _editor.Text.Replace(_txtFind.Text, _txtReplace.Text, comparison);
        _editor.Text = newText;
        MessageBox.Show("แทนที่เสร็จสิ้น");
    }
}
```

---

## 📝 สรุป Part 27

| หัวข้อ | Key Points |
|--------|-----------|
| MDI | IsMdiContainer = true, MdiParent |
| MdiLayout | Cascade, TileH, TileV |
| SplitContainer | แบ่ง 2 panel, ปรับ splitter |
| UserControl | Component ที่ใช้ซ้ำ |
| MdiWindowListItem | Auto list open documents |

---

**ก่อนหน้า → [Part 26: WinForms Timer](part26-winforms-timer.md)**  
**ต่อไป → [Part 28: WinForms Custom Controls](part28-winforms-custom-controls.md)**
