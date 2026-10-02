# Part 23: WinForms Data Binding
## ขั้นตอนที่ 221-230: Data Binding และ DataGridView

---

## 🎯 เป้าหมายของ Part นี้
- BindingSource และ BindingList<T>
- DataGridView: columns, sorting, filtering
- Two-way Data Binding กับ TextBox, ComboBox
- INotifyPropertyChanged
- Data Validation ใน Grid
- Export to CSV/Excel
- โปรแกรม Product Catalog Manager

---

## ขั้นตอนที่ 221: BindingSource พื้นฐาน

```csharp
// BindingSource เป็น bridge ระหว่าง Data กับ Controls
var bindingSource = new BindingSource();

// ข้อมูล
var products = new BindingList<Product>
{
    new Product { Id = 1, Name = "Laptop", Price = 35000, Category = "Electronics" },
    new Product { Id = 2, Name = "Mouse", Price = 450, Category = "Electronics" },
    new Product { Id = 3, Name = "Desk", Price = 5000, Category = "Furniture" }
};

bindingSource.DataSource = products;

// Bind DataGridView
var grid = new DataGridView { Dock = DockStyle.Fill };
grid.DataSource = bindingSource;

// Bind TextBox to selected item
var txtName = new TextBox();
txtName.DataBindings.Add("Text", bindingSource, "Name", false, 
    DataSourceUpdateMode.OnPropertyChanged);

var txtPrice = new TextBox();
txtPrice.DataBindings.Add("Text", bindingSource, "Price");

// Navigate records
var btnNext = new Button { Text = "Next >" };
btnNext.Click += (s,e) => bindingSource.MoveNext();

var btnPrev = new Button { Text = "< Prev" };
btnPrev.Click += (s,e) => bindingSource.MovePrevious();

// Current position label
var lblPos = new Label();
bindingSource.PositionChanged += (s,e) =>
    lblPos.Text = $"Record {bindingSource.Position + 1} of {bindingSource.Count}";
```

---

## ขั้นตอนที่ 222: INotifyPropertyChanged

```csharp
// Model ที่แจ้งเตือนเมื่อ property เปลี่ยน
using System.ComponentModel;

class Product : INotifyPropertyChanged
{
    private int _id;
    private string _name = "";
    private decimal _price;
    private string _category = "";
    private bool _inStock = true;
    
    public event PropertyChangedEventHandler? PropertyChanged;
    
    protected virtual void OnPropertyChanged([System.Runtime.CompilerServices.CallerMemberName] string? propertyName = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
    
    public int Id
    {
        get => _id;
        set { if (_id != value) { _id = value; OnPropertyChanged(); } }
    }
    
    public string Name
    {
        get => _name;
        set { if (_name != value) { _name = value; OnPropertyChanged(); } }
    }
    
    public decimal Price
    {
        get => _price;
        set { if (_price != value) { _price = value; OnPropertyChanged(); } }
    }
    
    public string Category
    {
        get => _category;
        set { if (_category != value) { _category = value; OnPropertyChanged(); } }
    }
    
    public bool InStock
    {
        get => _inStock;
        set { if (_inStock != value) { _inStock = value; OnPropertyChanged(); } }
    }
    
    public override string ToString() => $"{Name} (฿{Price:N0})";
}
```

---

## ขั้นตอนที่ 223: DataGridView Configuration

```csharp
// DataGridView - ตารางข้อมูลขั้นสูง
var grid = new DataGridView
{
    Dock = DockStyle.Fill,
    AutoGenerateColumns = false,    // จัดการ columns เอง
    AllowUserToAddRows = false,
    AllowUserToDeleteRows = false,
    ReadOnly = false,
    SelectionMode = DataGridViewSelectionMode.FullRowSelect,
    MultiSelect = false,
    RowHeadersVisible = false,
    AlternatingRowsDefaultCellStyle = { BackColor = Color.AliceBlue },
    BackgroundColor = Color.White,
    BorderStyle = BorderStyle.None,
    ColumnHeadersDefaultCellStyle = 
    {
        BackColor = Color.FromArgb(52, 73, 94),
        ForeColor = Color.White,
        Font = new Font("Segoe UI", 9, FontStyle.Bold)
    },
    DefaultCellStyle = { Font = new Font("Segoe UI", 10) },
    RowTemplate = { Height = 32 },
    EnableHeadersVisualStyles = false
};

// Define columns
grid.Columns.AddRange(new DataGridViewColumn[]
{
    new DataGridViewTextBoxColumn
    {
        Name = "colId",
        DataPropertyName = "Id",
        HeaderText = "ID",
        Width = 50,
        ReadOnly = true
    },
    new DataGridViewTextBoxColumn
    {
        Name = "colName",
        DataPropertyName = "Name",
        HeaderText = "ชื่อสินค้า",
        Width = 200,
        AutoSizeMode = DataGridViewAutoSizeColumnMode.Fill
    },
    new DataGridViewTextBoxColumn
    {
        Name = "colPrice",
        DataPropertyName = "Price",
        HeaderText = "ราคา (฿)",
        Width = 100,
        DefaultCellStyle = { Format = "N0", Alignment = DataGridViewContentAlignment.MiddleRight }
    },
    new DataGridViewComboBoxColumn
    {
        Name = "colCategory",
        DataPropertyName = "Category",
        HeaderText = "หมวดหมู่",
        Width = 130,
        DataSource = new[] { "Electronics", "Furniture", "Clothing", "Food" }
    },
    new DataGridViewCheckBoxColumn
    {
        Name = "colInStock",
        DataPropertyName = "InStock",
        HeaderText = "มีสต็อก",
        Width = 80
    },
    new DataGridViewButtonColumn
    {
        Name = "colEdit",
        HeaderText = "",
        Text = "แก้ไข",
        UseColumnTextForButtonValue = true,
        Width = 70
    }
});

// Cell click for button column
grid.CellClick += (s,e) =>
{
    if (e.RowIndex < 0 || e.ColumnIndex != grid.Columns["colEdit"]!.Index) return;
    var product = (Product)bindingSource[e.RowIndex];
    EditProduct(product);
};

// Conditional row formatting
grid.RowPrePaint += (s,e) => { /* custom drawing */ };
grid.CellFormatting += (s,e) =>
{
    if (e.ColumnIndex == grid.Columns["colPrice"]!.Index && e.Value is decimal price)
    {
        if (price > 10000) e.CellStyle!.ForeColor = Color.DarkGreen;
        else if (price < 500) e.CellStyle!.ForeColor = Color.Gray;
    }
};
```

---

## ขั้นตอนที่ 224: Sorting และ Filtering

```csharp
// Sort ด้วย header click (auto)
grid.SortCompare += (s,e) =>
{
    if (e.Column?.Name == "colPrice")
    {
        decimal a = (decimal)(e.CellValue1 ?? 0);
        decimal b = (decimal)(e.CellValue2 ?? 0);
        e.SortResult = a.CompareTo(b);
        e.Handled = true;
    }
};

// Filter ด้วย BindingSource
var txtFilter = new TextBox { PlaceholderText = "ค้นหา..." };
var cboFilterCategory = new ComboBox();
cboFilterCategory.Items.AddRange(new[] { "ทั้งหมด", "Electronics", "Furniture", "Clothing" });
cboFilterCategory.SelectedIndex = 0;

void ApplyFilter()
{
    string search = txtFilter.Text.Trim().ToLower();
    string category = cboFilterCategory.SelectedItem?.ToString() ?? "ทั้งหมด";
    
    var filtered = allProducts.Where(p =>
        (string.IsNullOrEmpty(search) || 
         p.Name.ToLower().Contains(search) ||
         p.Id.ToString().Contains(search)) &&
        (category == "ทั้งหมด" || p.Category == category)
    ).ToList();
    
    bindingSource.DataSource = new BindingList<Product>(filtered);
    lblCount.Text = $"แสดง {filtered.Count} จาก {allProducts.Count} รายการ";
}

txtFilter.TextChanged += (s,e) => ApplyFilter();
cboFilterCategory.SelectedIndexChanged += (s,e) => ApplyFilter();
```

---

## ขั้นตอนที่ 225: Data Validation ใน Grid

```csharp
// Validate cell before commit
grid.CellValidating += (s,e) =>
{
    if (e.RowIndex < 0) return;
    
    string colName = grid.Columns[e.ColumnIndex].Name;
    string? value = e.FormattedValue?.ToString();
    
    if (colName == "colName" && string.IsNullOrWhiteSpace(value))
    {
        e.Cancel = true;
        grid.Rows[e.RowIndex].ErrorText = "ชื่อสินค้าต้องไม่ว่าง";
        return;
    }
    
    if (colName == "colPrice")
    {
        if (!decimal.TryParse(value, out decimal price) || price < 0)
        {
            e.Cancel = true;
            grid.Rows[e.RowIndex].ErrorText = "ราคาต้องเป็นตัวเลขที่ >= 0";
            return;
        }
    }
    
    grid.Rows[e.RowIndex].ErrorText = "";
};

grid.CellEndEdit += (s,e) =>
{
    grid.Rows[e.RowIndex].ErrorText = "";
};
```

---

## ขั้นตอนที่ 226-230: โปรแกรม Product Catalog Manager

```csharp
// ProductCatalog.cs - Full Application
using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Drawing;
using System.IO;
using System.Linq;
using System.Windows.Forms;

namespace ProductCatalog;

public class ProductCatalogForm : Form
{
    private DataGridView _grid = null!;
    private BindingSource _bs = null!;
    private BindingList<Product> _products = null!;
    
    private TextBox _txtSearch = null!;
    private ComboBox _cboCategory = null!;
    private Label _lblSummary = null!;
    private StatusStrip _status = null!;
    private ToolStripStatusLabel _lblStatus = null!;
    
    private readonly string[] _categories = { "Electronics", "Furniture", "Clothing", "Food", "Other" };

    public ProductCatalogForm()
    {
        InitializeUI();
        LoadSampleData();
    }

    private void InitializeUI()
    {
        Text = "Product Catalog Manager";
        Size = new Size(900, 600);
        StartPosition = FormStartPosition.CenterScreen;
        MinimumSize = new Size(700, 450);

        // Menu
        var menu = new MenuStrip();
        var fileMenu = new ToolStripMenuItem("&File");
        fileMenu.DropDownItems.Add(new ToolStripMenuItem("Import CSV...", null, ImportCsv));
        fileMenu.DropDownItems.Add(new ToolStripMenuItem("Export CSV...", null, ExportCsv));
        fileMenu.DropDownItems.Add(new ToolStripSeparator());
        fileMenu.DropDownItems.Add(new ToolStripMenuItem("Exit", null, (s,e) => Close()));
        menu.Items.Add(fileMenu);
        MainMenuStrip = menu;

        // Toolbar
        var toolbar = new Panel
        {
            Dock = DockStyle.Top,
            Height = 50,
            BackColor = Color.FromArgb(44, 62, 80),
            Padding = new Padding(8, 8, 8, 5)
        };

        _txtSearch = new TextBox
        {
            PlaceholderText = "🔍 ค้นหาสินค้า...",
            Width = 220, Height = 28,
            Location = new Point(8, 11),
            Font = new Font("Segoe UI", 10)
        };
        _txtSearch.TextChanged += (s,e) => ApplyFilter();

        _cboCategory = new ComboBox
        {
            DropDownStyle = ComboBoxStyle.DropDownList,
            Width = 150, Location = new Point(240, 11)
        };
        _cboCategory.Items.Add("ทุกหมวดหมู่");
        _cboCategory.Items.AddRange(_categories);
        _cboCategory.SelectedIndex = 0;
        _cboCategory.SelectedIndexChanged += (s,e) => ApplyFilter();

        var btnAdd = CreateBtn("➕ เพิ่ม", 410, Color.FromArgb(39, 174, 96));
        var btnDel = CreateBtn("🗑️ ลบ", 480, Color.FromArgb(231, 76, 60));
        var btnRefresh = CreateBtn("🔄 รีเฟรช", 550, Color.FromArgb(52, 152, 219));

        btnAdd.Click += (s,e) => AddProduct();
        btnDel.Click += (s,e) => DeleteSelected();
        btnRefresh.Click += (s,e) => ApplyFilter();

        toolbar.Controls.AddRange(new Control[] { _txtSearch, _cboCategory, btnAdd, btnDel, btnRefresh });

        // Summary panel
        var summaryPanel = new Panel
        {
            Dock = DockStyle.Bottom,
            Height = 35,
            BackColor = Color.WhiteSmoke,
            Padding = new Padding(10, 8, 10, 0)
        };
        _lblSummary = new Label
        {
            Dock = DockStyle.Fill,
            Font = new Font("Segoe UI", 9),
            ForeColor = Color.DimGray
        };
        summaryPanel.Controls.Add(_lblSummary);

        // Grid
        _grid = CreateGrid();

        // Status bar
        _status = new StatusStrip();
        _lblStatus = new ToolStripStatusLabel("Ready") { Spring = true };
        _status.Items.Add(_lblStatus);

        Controls.Add(_grid);
        Controls.Add(summaryPanel);
        Controls.Add(toolbar);
        Controls.Add(menu);
        Controls.Add(_status);
    }

    private Button CreateBtn(string text, int x, Color color)
    {
        var btn = new Button
        {
            Text = text, Location = new Point(x, 10), Height = 28, AutoSize = true,
            BackColor = color, ForeColor = Color.White,
            FlatStyle = FlatStyle.Flat, Cursor = Cursors.Hand
        };
        btn.FlatAppearance.BorderSize = 0;
        return btn;
    }

    private DataGridView CreateGrid()
    {
        var g = new DataGridView
        {
            Dock = DockStyle.Fill,
            AutoGenerateColumns = false,
            AllowUserToAddRows = false,
            SelectionMode = DataGridViewSelectionMode.FullRowSelect,
            MultiSelect = true,
            RowHeadersVisible = false,
            BackgroundColor = Color.White,
            BorderStyle = BorderStyle.None,
            AlternatingRowsDefaultCellStyle = { BackColor = Color.FromArgb(248, 249, 250) },
            DefaultCellStyle = { Font = new Font("Segoe UI", 10) },
            RowTemplate = { Height = 34 },
            EnableHeadersVisualStyles = false
        };

        g.ColumnHeadersDefaultCellStyle.BackColor = Color.FromArgb(44, 62, 80);
        g.ColumnHeadersDefaultCellStyle.ForeColor = Color.White;
        g.ColumnHeadersDefaultCellStyle.Font = new Font("Segoe UI", 9, FontStyle.Bold);

        g.Columns.AddRange(new DataGridViewColumn[]
        {
            new DataGridViewTextBoxColumn { Name = "colId", DataPropertyName = "Id", HeaderText = "ID", Width = 55, ReadOnly = true },
            new DataGridViewTextBoxColumn { Name = "colName", DataPropertyName = "Name", HeaderText = "ชื่อสินค้า", AutoSizeMode = DataGridViewAutoSizeColumnMode.Fill },
            new DataGridViewComboBoxColumn
            {
                Name = "colCategory", DataPropertyName = "Category", HeaderText = "หมวดหมู่",
                Width = 140, DataSource = _categories.ToArray()
            },
            new DataGridViewTextBoxColumn
            {
                Name = "colPrice", DataPropertyName = "Price", HeaderText = "ราคา (฿)", Width = 110,
                DefaultCellStyle = { Format = "N0", Alignment = DataGridViewContentAlignment.MiddleRight }
            },
            new DataGridViewTextBoxColumn
            {
                Name = "colStock", DataPropertyName = "StockQty", HeaderText = "จำนวน", Width = 80,
                DefaultCellStyle = { Alignment = DataGridViewContentAlignment.MiddleCenter }
            },
            new DataGridViewCheckBoxColumn { Name = "colActive", DataPropertyName = "IsActive", HeaderText = "ใช้งาน", Width = 70 }
        });

        g.CellFormatting += Grid_CellFormatting;
        g.CellEndEdit += (s,e) => UpdateSummary();

        return g;
    }

    private void Grid_CellFormatting(object? sender, DataGridViewCellFormattingEventArgs e)
    {
        if (e.RowIndex < 0) return;

        // Color price
        if (e.ColumnIndex == _grid.Columns["colPrice"]!.Index && e.Value is decimal price)
        {
            e.CellStyle!.ForeColor = price >= 10000 ? Color.DarkGreen :
                                     price >= 1000 ? Color.DarkBlue : Color.Gray;
        }

        // Color stock
        if (e.ColumnIndex == _grid.Columns["colStock"]!.Index && e.Value is int qty)
        {
            if (qty == 0) e.CellStyle!.ForeColor = Color.Red;
            else if (qty < 10) e.CellStyle!.ForeColor = Color.Orange;
        }
    }

    private void LoadSampleData()
    {
        _products = new BindingList<Product>
        {
            new() { Id=1, Name="MacBook Pro 14\"", Category="Electronics", Price=89000, StockQty=15, IsActive=true },
            new() { Id=2, Name="Dell XPS 15", Category="Electronics", Price=65000, StockQty=8, IsActive=true },
            new() { Id=3, Name="Logitech MX Keys", Category="Electronics", Price=4500, StockQty=42, IsActive=true },
            new() { Id=4, Name="Standing Desk", Category="Furniture", Price=12000, StockQty=5, IsActive=true },
            new() { Id=5, Name="Ergonomic Chair", Category="Furniture", Price=18000, StockQty=3, IsActive=true },
            new() { Id=6, Name="USB-C Hub", Category="Electronics", Price=1200, StockQty=0, IsActive=false },
            new() { Id=7, Name="Monitor 27\"", Category="Electronics", Price=22000, StockQty=20, IsActive=true },
            new() { Id=8, Name="Coffee Mug", Category="Other", Price=350, StockQty=100, IsActive=true },
        };

        _bs = new BindingSource { DataSource = _products };
        _grid.DataSource = _bs;
        UpdateSummary();
    }

    private void ApplyFilter()
    {
        string search = _txtSearch.Text.Trim().ToLower();
        string cat = _cboCategory.SelectedItem?.ToString() ?? "";
        bool allCat = cat == "ทุกหมวดหมู่";

        var filtered = _products.Where(p =>
            (string.IsNullOrEmpty(search) ||
             p.Name.ToLower().Contains(search) ||
             p.Id.ToString().Contains(search)) &&
            (allCat || p.Category == cat)
        ).ToList();

        _bs.DataSource = new BindingList<Product>(filtered);
        UpdateSummary(filtered);
    }

    private void UpdateSummary(IList<Product>? list = null)
    {
        list ??= _products;
        decimal total = list.Sum(p => p.Price * p.StockQty);
        int outOfStock = list.Count(p => p.StockQty == 0);
        _lblSummary.Text = $"รายการ: {list.Count} | มูลค่ารวม: ฿{total:N0} | หมดสต็อก: {outOfStock} รายการ";
    }

    private void AddProduct()
    {
        using var dlg = new ProductEditDialog(_categories);
        if (dlg.ShowDialog() == DialogResult.OK && dlg.Result != null)
        {
            int nextId = _products.Count > 0 ? _products.Max(p => p.Id) + 1 : 1;
            var p = dlg.Result with { Id = nextId };
            _products.Add(p);
            ApplyFilter();
            SetStatus($"เพิ่ม '{p.Name}' สำเร็จ");
        }
    }

    private void DeleteSelected()
    {
        var toDelete = _grid.SelectedRows.Cast<DataGridViewRow>()
            .Select(r => (Product)r.DataBoundItem!)
            .ToList();

        if (toDelete.Count == 0) return;

        var result = MessageBox.Show(
            $"ต้องการลบ {toDelete.Count} รายการ?", "ยืนยันการลบ",
            MessageBoxButtons.YesNo, MessageBoxIcon.Warning);

        if (result == DialogResult.Yes)
        {
            foreach (var p in toDelete)
                _products.Remove(p);
            ApplyFilter();
            SetStatus($"ลบ {toDelete.Count} รายการสำเร็จ");
        }
    }

    private void ImportCsv(object? s, EventArgs e)
    {
        using var dlg = new OpenFileDialog { Filter = "CSV Files|*.csv" };
        if (dlg.ShowDialog() != DialogResult.OK) return;

        try
        {
            var lines = File.ReadAllLines(dlg.FileName).Skip(1); // Skip header
            int count = 0;
            int nextId = _products.Count > 0 ? _products.Max(p => p.Id) + 1 : 1;

            foreach (var line in lines)
            {
                var parts = line.Split(',');
                if (parts.Length < 4) continue;

                _products.Add(new Product
                {
                    Id = nextId++,
                    Name = parts[0].Trim('"'),
                    Category = parts[1].Trim('"'),
                    Price = decimal.TryParse(parts[2], out var price) ? price : 0,
                    StockQty = int.TryParse(parts[3], out var qty) ? qty : 0,
                    IsActive = true
                });
                count++;
            }

            ApplyFilter();
            SetStatus($"Import สำเร็จ: {count} รายการ");
        }
        catch (Exception ex)
        {
            MessageBox.Show($"Import ล้มเหลว: {ex.Message}", "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
        }
    }

    private void ExportCsv(object? s, EventArgs e)
    {
        using var dlg = new SaveFileDialog
        {
            Filter = "CSV Files|*.csv",
            FileName = $"products_{DateTime.Now:yyyyMMdd}.csv"
        };
        if (dlg.ShowDialog() != DialogResult.OK) return;

        try
        {
            var lines = new List<string> { "Name,Category,Price,StockQty,IsActive" };
            foreach (var p in _products)
                lines.Add($"\"{p.Name}\",{p.Category},{p.Price},{p.StockQty},{p.IsActive}");

            File.WriteAllLines(dlg.FileName, lines);
            SetStatus($"Export สำเร็จ: {_products.Count} รายการ → {dlg.FileName}");
        }
        catch (Exception ex)
        {
            MessageBox.Show($"Export ล้มเหลว: {ex.Message}", "Error");
        }
    }

    private void SetStatus(string msg) => _lblStatus.Text = $"[{DateTime.Now:HH:mm:ss}] {msg}";
}

// Product model
public class Product : INotifyPropertyChanged
{
    private string _name = "";
    private decimal _price;
    private int _stockQty;
    private bool _isActive = true;
    private string _category = "";

    public event PropertyChangedEventHandler? PropertyChanged;
    void Notify([System.Runtime.CompilerServices.CallerMemberName] string? p = null) =>
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(p));

    public int Id { get; set; }
    public string Name { get => _name; set { _name = value; Notify(); } }
    public string Category { get => _category; set { _category = value; Notify(); } }
    public decimal Price { get => _price; set { _price = value; Notify(); } }
    public int StockQty { get => _stockQty; set { _stockQty = value; Notify(); } }
    public bool IsActive { get => _isActive; set { _isActive = value; Notify(); } }
}

// Edit dialog
public class ProductEditDialog : Form
{
    private TextBox _txtName = null!;
    private ComboBox _cboCategory = null!;
    private NumericUpDown _numPrice = null!, _numStock = null!;
    private CheckBox _chkActive = null!;
    public Product? Result { get; private set; }

    public ProductEditDialog(string[] categories)
    {
        Text = "เพิ่ม/แก้ไขสินค้า";
        Size = new Size(360, 300);
        StartPosition = FormStartPosition.CenterParent;
        FormBorderStyle = FormBorderStyle.FixedDialog;
        MaximizeBox = false;

        var layout = new TableLayoutPanel
        {
            Dock = DockStyle.Fill, ColumnCount = 2,
            Padding = new Padding(15), RowCount = 6
        };
        layout.ColumnStyles.Add(new ColumnStyle(SizeType.Absolute, 110));
        layout.ColumnStyles.Add(new ColumnStyle(SizeType.Percent, 100));
        for (int i = 0; i < 6; i++) layout.RowStyles.Add(new RowStyle(SizeType.Absolute, 38));

        Label L(string t) => new Label { Text = t, Anchor = AnchorStyles.Right, AutoSize = true };

        layout.Controls.Add(L("ชื่อสินค้า:"), 0, 0);
        _txtName = new TextBox { Dock = DockStyle.Fill };
        layout.Controls.Add(_txtName, 1, 0);

        layout.Controls.Add(L("หมวดหมู่:"), 0, 1);
        _cboCategory = new ComboBox { Dock = DockStyle.Fill, DropDownStyle = ComboBoxStyle.DropDownList };
        _cboCategory.Items.AddRange(categories);
        _cboCategory.SelectedIndex = 0;
        layout.Controls.Add(_cboCategory, 1, 1);

        layout.Controls.Add(L("ราคา:"), 0, 2);
        _numPrice = new NumericUpDown { Minimum = 0, Maximum = 9999999, DecimalPlaces = 2 };
        layout.Controls.Add(_numPrice, 1, 2);

        layout.Controls.Add(L("จำนวนสต็อก:"), 0, 3);
        _numStock = new NumericUpDown { Minimum = 0, Maximum = 99999 };
        layout.Controls.Add(_numStock, 1, 3);

        layout.Controls.Add(L("ใช้งาน:"), 0, 4);
        _chkActive = new CheckBox { Text = "เปิดใช้งาน", Checked = true };
        layout.Controls.Add(_chkActive, 1, 4);

        var btnOk = new Button { Text = "บันทึก", Size = new Size(80, 30) };
        var btnCancel = new Button { Text = "ยกเลิก", Size = new Size(80, 30), DialogResult = DialogResult.Cancel };
        btnOk.Click += (s, e) =>
        {
            if (string.IsNullOrWhiteSpace(_txtName.Text)) { MessageBox.Show("กรุณากรอกชื่อสินค้า"); return; }
            Result = new Product
            {
                Name = _txtName.Text,
                Category = _cboCategory.SelectedItem?.ToString() ?? "",
                Price = _numPrice.Value,
                StockQty = (int)_numStock.Value,
                IsActive = _chkActive.Checked
            };
            DialogResult = DialogResult.OK;
            Close();
        };

        var btnPanel = new FlowLayoutPanel { Dock = DockStyle.Bottom, Height = 45, FlowDirection = FlowDirection.RightToLeft, Padding = new Padding(5) };
        btnPanel.Controls.AddRange(new Control[] { btnCancel, btnOk });

        Controls.Add(layout);
        Controls.Add(btnPanel);
        AcceptButton = btnOk;
        CancelButton = btnCancel;
    }
}
```

---

## 📝 สรุป Part 23

| หัวข้อ | ใช้เมื่อ |
|--------|---------|
| BindingSource | เชื่อมต่อ data กับ controls |
| BindingList<T> | List ที่แจ้งเตือน UI เมื่อเปลี่ยน |
| INotifyPropertyChanged | Model แจ้ง UI เมื่อ property เปลี่ยน |
| DataGridView | ตารางข้อมูลขั้นสูง |
| DataBindings.Add | Two-way binding ของ control |

---

**ก่อนหน้า → [Part 22: WinForms Controls](part22-winforms-controls.md)**  
**ต่อไป → [Part 24: WinForms Dialogs & File Operations](part24-winforms-dialogs.md)**
