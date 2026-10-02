# Part 21: WinForms Introduction
## ขั้นตอนที่ 201-210: เริ่มต้น Windows Forms

---

## 🎯 เป้าหมายของ Part นี้
- ติดตั้งและตั้งค่า WinForms project
- เข้าใจ Form lifecycle
- Controls พื้นฐาน: Button, Label, TextBox
- Layout: Panels, TableLayoutPanel, FlowLayoutPanel
- Event handling ใน WinForms
- สร้าง Contact Form จริง

---

## ขั้นตอนที่ 201: สร้าง WinForms Project

### Setup WinForms Project
```xml
<!-- MyApp.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net8.0-windows</TargetFramework>
    <Nullable>enable</Nullable>
    <UseWindowsForms>true</UseWindowsForms>
    <ImplicitUsings>enable</ImplicitUsings>
    <ApplicationIcon>app.ico</ApplicationIcon>
  </PropertyGroup>
</Project>
```

### Program.cs Entry Point
```csharp
// Program.cs
using System.Windows.Forms;

namespace MyApp;

internal static class Program
{
    [STAThread]  // Single-Threaded Apartment - required for WinForms
    static void Main()
    {
        // Enable modern DPI awareness
        Application.EnableVisualStyles();
        Application.SetCompatibleTextRenderingDefault(false);
        Application.SetHighDpiMode(HighDpiMode.SystemAware);
        
        // Run the application with main form
        Application.Run(new MainForm());
    }
}
```

---

## ขั้นตอนที่ 202: Form Basics

### สร้าง Form แรก
```csharp
// MainForm.cs
using System.Drawing;
using System.Windows.Forms;

namespace MyApp;

public class MainForm : Form
{
    // Controls as fields
    private Label _lblTitle = null!;
    private TextBox _txtName = null!;
    private Button _btnGreet = null!;
    private RichTextBox _rtbOutput = null!;
    
    public MainForm()
    {
        InitializeComponent();
        SetupForm();
    }
    
    private void InitializeComponent()
    {
        // Form settings
        this.Text = "My First WinForms App";
        this.Size = new Size(500, 400);
        this.StartPosition = FormStartPosition.CenterScreen;
        this.MinimumSize = new Size(400, 300);
        this.FormBorderStyle = FormBorderStyle.Sizable;
        
        // Title Label
        _lblTitle = new Label
        {
            Text = "ยินดีต้อนรับ",
            Font = new Font("Segoe UI", 18, FontStyle.Bold),
            ForeColor = Color.Navy,
            AutoSize = true,
            Location = new Point(20, 20)
        };
        
        // Name TextBox
        _txtName = new TextBox
        {
            PlaceholderText = "กรอกชื่อของคุณ...",
            Location = new Point(20, 70),
            Size = new Size(300, 25),
            Font = new Font("Segoe UI", 11)
        };
        
        // Greet Button
        _btnGreet = new Button
        {
            Text = "👋 ทักทาย",
            Location = new Point(340, 68),
            Size = new Size(120, 30),
            BackColor = Color.SteelBlue,
            ForeColor = Color.White,
            FlatStyle = FlatStyle.Flat,
            Cursor = Cursors.Hand
        };
        _btnGreet.FlatAppearance.BorderSize = 0;
        
        // Output RichTextBox
        _rtbOutput = new RichTextBox
        {
            Location = new Point(20, 120),
            Size = new Size(440, 200),
            ReadOnly = true,
            BackColor = Color.White,
            Font = new Font("Consolas", 10),
            BorderStyle = BorderStyle.FixedSingle
        };
        
        // Add to form
        this.Controls.AddRange(new Control[] 
        { 
            _lblTitle, _txtName, _btnGreet, _rtbOutput 
        });
        
        // Wire up events
        _btnGreet.Click += BtnGreet_Click;
        _txtName.KeyPress += TxtName_KeyPress;
        this.Resize += MainForm_Resize;
        this.Load += MainForm_Load;
    }
    
    private void SetupForm()
    {
        // Additional setup after InitializeComponent
        this.Icon = SystemIcons.Application;
    }
    
    // Event Handlers
    private void MainForm_Load(object? sender, EventArgs e)
    {
        AppendOutput("โปรแกรมเริ่มทำงาน!", Color.Gray);
        _txtName.Focus();
    }
    
    private void BtnGreet_Click(object? sender, EventArgs e)
    {
        string name = _txtName.Text.Trim();
        
        if (string.IsNullOrEmpty(name))
        {
            MessageBox.Show("กรุณากรอกชื่อก่อน", "แจ้งเตือน", 
                MessageBoxButtons.OK, MessageBoxIcon.Warning);
            _txtName.Focus();
            return;
        }
        
        string greeting = $"สวัสดี {name}! ยินดีต้อนรับสู่ WinForms 🎉";
        AppendOutput(greeting, Color.DarkGreen);
        
        _txtName.Clear();
        _txtName.Focus();
    }
    
    private void TxtName_KeyPress(object? sender, KeyPressEventArgs e)
    {
        // Enter key triggers button
        if (e.KeyChar == (char)Keys.Enter)
        {
            _btnGreet.PerformClick();
            e.Handled = true;
        }
    }
    
    private void MainForm_Resize(object? sender, EventArgs e)
    {
        // Responsive layout on resize
        if (_rtbOutput != null)
        {
            _rtbOutput.Size = new Size(
                this.ClientSize.Width - 40, 
                this.ClientSize.Height - 140
            );
        }
    }
    
    private void AppendOutput(string text, Color color)
    {
        _rtbOutput.SelectionStart = _rtbOutput.TextLength;
        _rtbOutput.SelectionLength = 0;
        _rtbOutput.SelectionColor = color;
        _rtbOutput.AppendText($"[{DateTime.Now:HH:mm:ss}] {text}\n");
        _rtbOutput.SelectionColor = _rtbOutput.ForeColor;
        _rtbOutput.ScrollToCaret();
    }
}
```

---

## ขั้นตอนที่ 203: Common Controls

### Label, TextBox, Button Variants
```csharp
// TextBox variants
var txtSingle = new TextBox
{
    Multiline = false,
    PlaceholderText = "Single line..."
};

var txtMultiline = new TextBox
{
    Multiline = true,
    ScrollBars = ScrollBars.Vertical,
    Size = new Size(300, 100)
};

var txtPassword = new TextBox
{
    PasswordChar = '●',  // หรือ '*'
    PlaceholderText = "Password..."
};

var txtNumber = new TextBox
{
    PlaceholderText = "Numbers only..."
};
// Validate input
txtNumber.KeyPress += (s, e) =>
{
    if (!char.IsDigit(e.KeyChar) && e.KeyChar != (char)Keys.Back)
        e.Handled = true;
};

// Label variants
var lblNormal = new Label { Text = "Normal Label" };
var lblLink = new LinkLabel 
{ 
    Text = "Click here",
    LinkColor = Color.Blue
};
lblLink.LinkClicked += (s, e) => System.Diagnostics.Process.Start("explorer", "https://example.com");

// Button variants
var btnNormal = new Button { Text = "Normal Button" };
var btnCheckBox = new CheckBox { Text = "Check me" };
var radioBtn = new RadioButton { Text = "Option A" };

// CheckBox
checkBox.CheckedChanged += (s, e) =>
{
    bool isChecked = ((CheckBox)s!).Checked;
    Console.WriteLine($"Checked: {isChecked}");
};
```

---

## ขั้นตอนที่ 204: ListBox, ComboBox, ListView

### Selection Controls
```csharp
// ListBox
var listBox = new ListBox
{
    Size = new Size(200, 150),
    SelectionMode = SelectionMode.MultiExtended  // Multiple selection
};

// Add items
listBox.Items.AddRange(new[] { "Item 1", "Item 2", "Item 3", "Item 4", "Item 5" });
// หรือ
listBox.DataSource = new List<string> { "Option A", "Option B", "Option C" };

listBox.SelectedIndexChanged += (s, e) =>
{
    if (listBox.SelectedItem != null)
        Console.WriteLine($"Selected: {listBox.SelectedItem}");
};

// ComboBox
var comboBox = new ComboBox
{
    DropDownStyle = ComboBoxStyle.DropDownList,
    Size = new Size(200, 25)
};
comboBox.Items.AddRange(new[] { "Thailand", "Japan", "USA", "UK" });
comboBox.SelectedIndex = 0;

// ListView
var listView = new ListView
{
    View = View.Details,
    FullRowSelect = true,
    GridLines = true,
    Size = new Size(400, 200)
};

// Columns
listView.Columns.Add("Name", 150);
listView.Columns.Add("Age", 60);
listView.Columns.Add("City", 130);

// Add items
listView.Items.Add(new ListViewItem(new[] { "Alice", "30", "Bangkok" }));
listView.Items.Add(new ListViewItem(new[] { "Bob", "25", "Chiang Mai" }));

listView.SelectedIndexChanged += (s, e) =>
{
    if (listView.SelectedItems.Count > 0)
    {
        var item = listView.SelectedItems[0];
        Console.WriteLine($"Selected: {item.Text}");
    }
};
```

---

## ขั้นตอนที่ 205: Layouts

### TableLayoutPanel, FlowLayoutPanel
```csharp
// TableLayoutPanel - Grid layout
var tableLayout = new TableLayoutPanel
{
    ColumnCount = 2,
    RowCount = 4,
    Dock = DockStyle.Fill,
    Padding = new Padding(10),
    CellBorderStyle = TableLayoutPanelCellBorderStyle.None
};

// Column styles
tableLayout.ColumnStyles.Add(new ColumnStyle(SizeType.Absolute, 120));
tableLayout.ColumnStyles.Add(new ColumnStyle(SizeType.Percent, 100));

// Row styles
tableLayout.RowStyles.Add(new RowStyle(SizeType.Absolute, 40));
tableLayout.RowStyles.Add(new RowStyle(SizeType.Absolute, 40));

// Add controls to cells
tableLayout.Controls.Add(new Label { Text = "ชื่อ:", Anchor = AnchorStyles.Right }, 0, 0);
tableLayout.Controls.Add(new TextBox { Dock = DockStyle.Fill }, 1, 0);
tableLayout.Controls.Add(new Label { Text = "อีเมล:", Anchor = AnchorStyles.Right }, 0, 1);
tableLayout.Controls.Add(new TextBox { Dock = DockStyle.Fill }, 1, 1);

// FlowLayoutPanel - Flow layout
var flowLayout = new FlowLayoutPanel
{
    FlowDirection = FlowDirection.LeftToRight,
    WrapContents = true,
    Dock = DockStyle.Top,
    AutoSize = true,
    Padding = new Padding(5)
};

// Toolbar buttons
string[] actions = { "New", "Open", "Save", "Close", "Settings" };
foreach (string action in actions)
{
    flowLayout.Controls.Add(new Button
    {
        Text = action,
        Size = new Size(70, 30),
        Margin = new Padding(2)
    });
}

// Panel - basic container
var panel = new Panel
{
    Dock = DockStyle.Fill,
    BorderStyle = BorderStyle.FixedSingle,
    BackColor = Color.WhiteSmoke
};
```

---

## ขั้นตอนที่ 206-210: โปรแกรมตัวอย่าง - Contact Manager Form

```csharp
// ContactForm.cs
using System;
using System.Collections.Generic;
using System.Drawing;
using System.Linq;
using System.Windows.Forms;

namespace ContactManager;

record Contact(int Id, string Name, string Email, string Phone, string City);

public class ContactForm : Form
{
    // Controls
    private TextBox _txtSearch = null!;
    private Button _btnSearch = null!;
    private Button _btnAdd = null!;
    private Button _btnEdit = null!;
    private Button _btnDelete = null!;
    private ListView _lvContacts = null!;
    private Panel _pnlToolbar = null!;
    private StatusStrip _statusBar = null!;
    private ToolStripStatusLabel _lblStatus = null!;
    
    // Data
    private List<Contact> _contacts = new();
    private int _nextId = 1;
    
    public ContactForm()
    {
        InitializeUI();
        LoadSampleData();
        RefreshList();
    }
    
    private void InitializeUI()
    {
        // Form
        Text = "Contact Manager";
        Size = new Size(800, 600);
        StartPosition = FormStartPosition.CenterScreen;
        MinimumSize = new Size(600, 400);
        
        // Toolbar
        _pnlToolbar = new Panel
        {
            Dock = DockStyle.Top,
            Height = 50,
            BackColor = Color.FromArgb(52, 73, 94),
            Padding = new Padding(10, 10, 10, 5)
        };
        
        _txtSearch = new TextBox
        {
            PlaceholderText = "ค้นหา...",
            Width = 250,
            Height = 28,
            Location = new Point(10, 11),
            Font = new Font("Segoe UI", 10)
        };
        _txtSearch.TextChanged += (s, e) => RefreshList(_txtSearch.Text);
        
        _btnSearch = CreateToolButton("🔍", 270, Color.FromArgb(41, 128, 185));
        _btnAdd = CreateToolButton("➕ เพิ่ม", 310, Color.FromArgb(39, 174, 96));
        _btnEdit = CreateToolButton("✏️ แก้ไข", 380, Color.FromArgb(243, 156, 18));
        _btnDelete = CreateToolButton("🗑️ ลบ", 450, Color.FromArgb(231, 76, 60));
        
        _pnlToolbar.Controls.AddRange(new Control[] { _txtSearch, _btnSearch, _btnAdd, _btnEdit, _btnDelete });
        
        // ListView
        _lvContacts = new ListView
        {
            Dock = DockStyle.Fill,
            View = View.Details,
            FullRowSelect = true,
            GridLines = false,
            Font = new Font("Segoe UI", 10),
            BorderStyle = BorderStyle.None,
            MultiSelect = false
        };
        
        _lvContacts.Columns.Add("ID", 50);
        _lvContacts.Columns.Add("ชื่อ", 180);
        _lvContacts.Columns.Add("อีเมล", 200);
        _lvContacts.Columns.Add("โทรศัพท์", 130);
        _lvContacts.Columns.Add("เมือง", 120);
        
        _lvContacts.DoubleClick += (s, e) => EditSelected();
        
        // Status bar
        _statusBar = new StatusStrip();
        _lblStatus = new ToolStripStatusLabel("Ready") { Spring = true };
        _statusBar.Items.Add(_lblStatus);
        
        // Wire button events
        _btnAdd.Click += (s, e) => AddContact();
        _btnEdit.Click += (s, e) => EditSelected();
        _btnDelete.Click += (s, e) => DeleteSelected();
        
        // Layout
        Controls.Add(_lvContacts);
        Controls.Add(_pnlToolbar);
        Controls.Add(_statusBar);
    }
    
    private Button CreateToolButton(string text, int x, Color backColor)
    {
        var btn = new Button
        {
            Text = text,
            Location = new Point(x, 10),
            Height = 28,
            AutoSize = true,
            BackColor = backColor,
            ForeColor = Color.White,
            FlatStyle = FlatStyle.Flat,
            Cursor = Cursors.Hand,
            Font = new Font("Segoe UI", 9)
        };
        btn.FlatAppearance.BorderSize = 0;
        return btn;
    }
    
    private void LoadSampleData()
    {
        var samples = new[]
        {
            ("Alice Smith", "alice@example.com", "081-111-1111", "Bangkok"),
            ("Bob Jones", "bob@test.org", "082-222-2222", "Chiang Mai"),
            ("Charlie Brown", "charlie@mail.com", "083-333-3333", "Phuket"),
            ("Diana Prince", "diana@example.com", "084-444-4444", "Bangkok"),
            ("Eve Wilson", "eve@test.com", "085-555-5555", "Pattaya"),
        };
        
        foreach (var (name, email, phone, city) in samples)
            _contacts.Add(new Contact(_nextId++, name, email, phone, city));
    }
    
    private void RefreshList(string filter = "")
    {
        _lvContacts.BeginUpdate();
        _lvContacts.Items.Clear();
        
        var filtered = string.IsNullOrWhiteSpace(filter)
            ? _contacts
            : _contacts.Where(c => 
                c.Name.Contains(filter, StringComparison.OrdinalIgnoreCase) ||
                c.Email.Contains(filter, StringComparison.OrdinalIgnoreCase) ||
                c.Phone.Contains(filter) ||
                c.City.Contains(filter, StringComparison.OrdinalIgnoreCase));
        
        foreach (var contact in filtered)
        {
            var item = new ListViewItem(contact.Id.ToString());
            item.SubItems.AddRange(new[] { contact.Name, contact.Email, contact.Phone, contact.City });
            item.Tag = contact;
            _lvContacts.Items.Add(item);
        }
        
        _lvContacts.EndUpdate();
        _lblStatus.Text = $"แสดง {_lvContacts.Items.Count} จาก {_contacts.Count} รายการ";
    }
    
    private void AddContact()
    {
        using var dialog = new ContactDialog();
        if (dialog.ShowDialog() == DialogResult.OK && dialog.Result != null)
        {
            var c = dialog.Result;
            _contacts.Add(new Contact(_nextId++, c.Name, c.Email, c.Phone, c.City));
            RefreshList();
            SetStatus($"เพิ่ม '{c.Name}' สำเร็จ", Color.Green);
        }
    }
    
    private void EditSelected()
    {
        if (_lvContacts.SelectedItems.Count == 0)
        {
            MessageBox.Show("กรุณาเลือกรายการก่อน", "แจ้งเตือน", 
                MessageBoxButtons.OK, MessageBoxIcon.Information);
            return;
        }
        
        var selected = (Contact)_lvContacts.SelectedItems[0].Tag!;
        using var dialog = new ContactDialog(selected);
        
        if (dialog.ShowDialog() == DialogResult.OK && dialog.Result != null)
        {
            var updated = dialog.Result;
            int idx = _contacts.FindIndex(c => c.Id == selected.Id);
            _contacts[idx] = updated with { Id = selected.Id };
            RefreshList(_txtSearch.Text);
            SetStatus($"แก้ไข '{updated.Name}' สำเร็จ", Color.DarkOrange);
        }
    }
    
    private void DeleteSelected()
    {
        if (_lvContacts.SelectedItems.Count == 0) return;
        
        var selected = (Contact)_lvContacts.SelectedItems[0].Tag!;
        var result = MessageBox.Show(
            $"ต้องการลบ '{selected.Name}' ใช่หรือไม่?",
            "ยืนยันการลบ",
            MessageBoxButtons.YesNo,
            MessageBoxIcon.Question
        );
        
        if (result == DialogResult.Yes)
        {
            _contacts.RemoveAll(c => c.Id == selected.Id);
            RefreshList(_txtSearch.Text);
            SetStatus($"ลบ '{selected.Name}' สำเร็จ", Color.Red);
        }
    }
    
    private void SetStatus(string message, Color color)
    {
        _lblStatus.ForeColor = color;
        _lblStatus.Text = $"[{DateTime.Now:HH:mm:ss}] {message}";
    }
}

// Add/Edit Dialog
public class ContactDialog : Form
{
    private TextBox _txtName = null!, _txtEmail = null!, _txtPhone = null!, _txtCity = null!;
    private Button _btnOk = null!, _btnCancel = null!;
    public Contact? Result { get; private set; }
    
    public ContactDialog(Contact? existing = null)
    {
        Text = existing == null ? "เพิ่มติดต่อใหม่" : "แก้ไขติดต่อ";
        Size = new Size(380, 300);
        StartPosition = FormStartPosition.CenterParent;
        FormBorderStyle = FormBorderStyle.FixedDialog;
        MaximizeBox = false;
        MinimizeBox = false;
        
        var layout = new TableLayoutPanel
        {
            Dock = DockStyle.Fill,
            ColumnCount = 2,
            Padding = new Padding(15),
        };
        layout.ColumnStyles.Add(new ColumnStyle(SizeType.Absolute, 80));
        layout.ColumnStyles.Add(new ColumnStyle(SizeType.Percent, 100));
        
        _txtName = AddField(layout, "ชื่อ:", 0, existing?.Name);
        _txtEmail = AddField(layout, "อีเมล:", 1, existing?.Email);
        _txtPhone = AddField(layout, "โทร:", 2, existing?.Phone);
        _txtCity = AddField(layout, "เมือง:", 3, existing?.City);
        
        var btnPanel = new FlowLayoutPanel
        {
            Dock = DockStyle.Bottom,
            FlowDirection = FlowDirection.RightToLeft,
            Height = 45,
            Padding = new Padding(5)
        };
        
        _btnOk = new Button { Text = "ตกลง", DialogResult = DialogResult.OK, Size = new Size(80, 30) };
        _btnCancel = new Button { Text = "ยกเลิก", DialogResult = DialogResult.Cancel, Size = new Size(80, 30) };
        
        _btnOk.Click += (s, e) =>
        {
            if (string.IsNullOrWhiteSpace(_txtName.Text))
            {
                MessageBox.Show("กรุณากรอกชื่อ", "แจ้งเตือน");
                return;
            }
            Result = new Contact(0, _txtName.Text, _txtEmail.Text, _txtPhone.Text, _txtCity.Text);
            DialogResult = DialogResult.OK;
            Close();
        };
        
        btnPanel.Controls.AddRange(new Control[] { _btnCancel, _btnOk });
        
        Controls.Add(layout);
        Controls.Add(btnPanel);
        AcceptButton = _btnOk;
        CancelButton = _btnCancel;
    }
    
    private TextBox AddField(TableLayoutPanel layout, string label, int row, string? value)
    {
        layout.Controls.Add(new Label { Text = label, Anchor = AnchorStyles.Right, AutoSize = true }, 0, row);
        var txt = new TextBox { Dock = DockStyle.Fill, Text = value ?? "", Font = new Font("Segoe UI", 10) };
        layout.Controls.Add(txt, 1, row);
        return txt;
    }
}
```

---

## 📝 สรุป Part 21

| หัวข้อ | Key Points |
|--------|-----------|
| Form class | หน้าต่าง WinForms |
| Controls | Label, TextBox, Button, ListBox, ComboBox |
| Dock/Anchor | Responsive layout |
| TableLayoutPanel | Grid-based layout |
| FlowLayoutPanel | Flow layout |
| Events | Click, TextChanged, Resize, Load |
| MessageBox | Dialog สำหรับ message/confirm |
| ShowDialog() | Modal dialog |

---

**ก่อนหน้า → [Part 20: Attributes & Reflection](../part11-20/part20-attributes-reflection.md)**  
**ต่อไป → [Part 22: WinForms Controls Advanced](part22-winforms-controls.md)**
