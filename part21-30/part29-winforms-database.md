# Part 29: WinForms Database Integration
## ขั้นตอนที่ 281-290: เชื่อมต่อ Database

---

## 🎯 เป้าหมายของ Part นี้
- ADO.NET พื้นฐาน: SqlConnection, SqlCommand
- SQLite ด้วย Microsoft.Data.Sqlite
- CRUD Operations ใน WinForms
- Connection string management
- DataTable กับ DataGridView
- Repository Pattern ใน WinForms
- โปรแกรม Library Management System

---

## ขั้นตอนที่ 281: SQLite Setup

```xml
<!-- .csproj - Add SQLite package -->
<ItemGroup>
  <PackageReference Include="Microsoft.Data.Sqlite" Version="8.0.0" />
</ItemGroup>
```

```csharp
// Database setup
using Microsoft.Data.Sqlite;

class DatabaseHelper
{
    private readonly string _connectionString;
    
    public DatabaseHelper(string dbPath = "library.db")
    {
        _connectionString = $"Data Source={dbPath}";
        InitializeDatabase();
    }
    
    private void InitializeDatabase()
    {
        using var conn = new SqliteConnection(_connectionString);
        conn.Open();
        
        var cmd = conn.CreateCommand();
        cmd.CommandText = @"
            CREATE TABLE IF NOT EXISTS Books (
                Id INTEGER PRIMARY KEY AUTOINCREMENT,
                Title TEXT NOT NULL,
                Author TEXT NOT NULL,
                ISBN TEXT UNIQUE,
                Category TEXT,
                Year INTEGER,
                TotalCopies INTEGER DEFAULT 1,
                AvailableCopies INTEGER DEFAULT 1,
                CreatedAt TEXT DEFAULT (datetime('now'))
            );
            
            CREATE TABLE IF NOT EXISTS Members (
                Id INTEGER PRIMARY KEY AUTOINCREMENT,
                Name TEXT NOT NULL,
                Email TEXT UNIQUE,
                Phone TEXT,
                JoinDate TEXT DEFAULT (date('now')),
                IsActive INTEGER DEFAULT 1
            );
            
            CREATE TABLE IF NOT EXISTS Loans (
                Id INTEGER PRIMARY KEY AUTOINCREMENT,
                BookId INTEGER REFERENCES Books(Id),
                MemberId INTEGER REFERENCES Members(Id),
                LoanDate TEXT DEFAULT (date('now')),
                DueDate TEXT,
                ReturnDate TEXT,
                Status TEXT DEFAULT 'Active'
            );
            
            -- Sample data if empty
            INSERT OR IGNORE INTO Books (Id, Title, Author, ISBN, Category, Year, TotalCopies, AvailableCopies)
            VALUES 
                (1, 'Clean Code', 'Robert C. Martin', '978-0132350884', 'Programming', 2008, 3, 3),
                (2, 'The Pragmatic Programmer', 'David Thomas', '978-0135957059', 'Programming', 2019, 2, 2),
                (3, 'Design Patterns', 'Gang of Four', '978-0201633610', 'Programming', 1994, 1, 1);
        ";
        cmd.ExecuteNonQuery();
    }
    
    public SqliteConnection GetConnection()
    {
        var conn = new SqliteConnection(_connectionString);
        conn.Open();
        return conn;
    }
}
```

---

## ขั้นตอนที่ 282: Repository Pattern

```csharp
// Book model
record Book(
    int Id, string Title, string Author, string ISBN,
    string Category, int Year, int TotalCopies, int AvailableCopies
);

// Book Repository
class BookRepository
{
    private readonly DatabaseHelper _db;
    
    public BookRepository(DatabaseHelper db) => _db = db;
    
    public List<Book> GetAll(string? searchTerm = null)
    {
        using var conn = _db.GetConnection();
        var cmd = conn.CreateCommand();
        
        if (string.IsNullOrEmpty(searchTerm))
        {
            cmd.CommandText = "SELECT * FROM Books ORDER BY Title";
        }
        else
        {
            cmd.CommandText = @"
                SELECT * FROM Books 
                WHERE Title LIKE @search OR Author LIKE @search OR ISBN LIKE @search
                ORDER BY Title";
            cmd.Parameters.AddWithValue("@search", $"%{searchTerm}%");
        }
        
        var books = new List<Book>();
        using var reader = cmd.ExecuteReader();
        while (reader.Read())
            books.Add(MapBook(reader));
        return books;
    }
    
    public Book? GetById(int id)
    {
        using var conn = _db.GetConnection();
        var cmd = conn.CreateCommand();
        cmd.CommandText = "SELECT * FROM Books WHERE Id = @id";
        cmd.Parameters.AddWithValue("@id", id);
        
        using var reader = cmd.ExecuteReader();
        return reader.Read() ? MapBook(reader) : null;
    }
    
    public int Create(Book book)
    {
        using var conn = _db.GetConnection();
        var cmd = conn.CreateCommand();
        cmd.CommandText = @"
            INSERT INTO Books (Title, Author, ISBN, Category, Year, TotalCopies, AvailableCopies)
            VALUES (@title, @author, @isbn, @category, @year, @total, @total);
            SELECT last_insert_rowid();";
        
        cmd.Parameters.AddWithValue("@title", book.Title);
        cmd.Parameters.AddWithValue("@author", book.Author);
        cmd.Parameters.AddWithValue("@isbn", book.ISBN ?? "");
        cmd.Parameters.AddWithValue("@category", book.Category ?? "");
        cmd.Parameters.AddWithValue("@year", book.Year);
        cmd.Parameters.AddWithValue("@total", book.TotalCopies);
        
        return Convert.ToInt32(cmd.ExecuteScalar());
    }
    
    public bool Update(Book book)
    {
        using var conn = _db.GetConnection();
        var cmd = conn.CreateCommand();
        cmd.CommandText = @"
            UPDATE Books SET 
                Title = @title, Author = @author, ISBN = @isbn,
                Category = @category, Year = @year, TotalCopies = @total
            WHERE Id = @id";
        
        cmd.Parameters.AddWithValue("@id", book.Id);
        cmd.Parameters.AddWithValue("@title", book.Title);
        cmd.Parameters.AddWithValue("@author", book.Author);
        cmd.Parameters.AddWithValue("@isbn", book.ISBN ?? "");
        cmd.Parameters.AddWithValue("@category", book.Category ?? "");
        cmd.Parameters.AddWithValue("@year", book.Year);
        cmd.Parameters.AddWithValue("@total", book.TotalCopies);
        
        return cmd.ExecuteNonQuery() > 0;
    }
    
    public bool Delete(int id)
    {
        using var conn = _db.GetConnection();
        var cmd = conn.CreateCommand();
        cmd.CommandText = "DELETE FROM Books WHERE Id = @id";
        cmd.Parameters.AddWithValue("@id", id);
        return cmd.ExecuteNonQuery() > 0;
    }
    
    public List<Book> GetByCategory(string category)
    {
        using var conn = _db.GetConnection();
        var cmd = conn.CreateCommand();
        cmd.CommandText = "SELECT * FROM Books WHERE Category = @cat ORDER BY Title";
        cmd.Parameters.AddWithValue("@cat", category);
        
        var books = new List<Book>();
        using var reader = cmd.ExecuteReader();
        while (reader.Read()) books.Add(MapBook(reader));
        return books;
    }
    
    private static Book MapBook(SqliteDataReader r) => new(
        r.GetInt32(0), r.GetString(1), r.GetString(2),
        r.IsDBNull(3) ? "" : r.GetString(3),
        r.IsDBNull(4) ? "" : r.GetString(4),
        r.IsDBNull(5) ? 0 : r.GetInt32(5),
        r.GetInt32(6), r.GetInt32(7)
    );
}
```

---

## ขั้นตอนที่ 283-290: Library Management System

```csharp
// LibraryForm.cs - Full CRUD Application with SQLite
using System;
using System.Collections.Generic;
using System.Drawing;
using System.Linq;
using System.Windows.Forms;

namespace Library;

public class LibraryForm : Form
{
    private readonly DatabaseHelper _db;
    private readonly BookRepository _bookRepo;
    
    private DataGridView _grid = null!;
    private TextBox _txtSearch = null!;
    private ComboBox _cboCategory = null!;
    private Label _lblCount = null!;
    private StatusStrip _status = null!;
    private ToolStripStatusLabel _lblStatus = null!;
    
    public LibraryForm()
    {
        _db = new DatabaseHelper();
        _bookRepo = new BookRepository(_db);
        
        InitializeUI();
        LoadBooks();
    }
    
    private void InitializeUI()
    {
        Text = "Library Management System";
        Size = new Size(950, 620);
        StartPosition = FormStartPosition.CenterScreen;
        MinimumSize = new Size(700, 450);
        
        // Menu
        var menu = new MenuStrip();
        var bookMenu = new ToolStripMenuItem("&Books");
        bookMenu.DropDownItems.Add("Add Book", null, (s,e) => AddBook());
        bookMenu.DropDownItems.Add("Import from CSV", null, ImportFromCsv);
        bookMenu.DropDownItems.Add("Export to CSV", null, ExportToCsv);
        
        var reportMenu = new ToolStripMenuItem("&Reports");
        reportMenu.DropDownItems.Add("Available Books", null, (s,e) => ShowAvailableOnly());
        reportMenu.DropDownItems.Add("Books by Category", null, (s,e) => ShowCategoryReport());
        
        menu.Items.AddRange(new ToolStripItem[] { bookMenu, reportMenu });
        MainMenuStrip = menu;
        
        // Toolbar panel
        var toolbar = new Panel { Dock = DockStyle.Top, Height = 52, BackColor = Color.FromArgb(44, 62, 80), Padding = new Padding(10) };
        
        _txtSearch = new TextBox
        {
            PlaceholderText = "🔍 ค้นหาชื่อ, ผู้แต่ง, ISBN...",
            Width = 280, Location = new Point(10, 13),
            Font = new Font("Segoe UI", 10)
        };
        _txtSearch.TextChanged += (s,e) => LoadBooks(_txtSearch.Text);
        
        _cboCategory = new ComboBox
        {
            DropDownStyle = ComboBoxStyle.DropDownList,
            Width = 160, Location = new Point(305, 13),
            Font = new Font("Segoe UI", 10)
        };
        _cboCategory.Items.AddRange(new[] { "ทุกหมวดหมู่", "Programming", "Science", "History", "Fiction", "Business", "Other" });
        _cboCategory.SelectedIndex = 0;
        _cboCategory.SelectedIndexChanged += (s,e) => LoadBooks(_txtSearch.Text);
        
        var btnAdd = MakeBtn("➕ เพิ่ม", 478, Color.FromArgb(39, 174, 96));
        var btnEdit = MakeBtn("✏️ แก้ไข", 548, Color.FromArgb(243, 156, 18));
        var btnDel = MakeBtn("🗑️ ลบ", 618, Color.FromArgb(231, 76, 60));
        
        btnAdd.Click += (s,e) => AddBook();
        btnEdit.Click += (s,e) => EditBook();
        btnDel.Click += (s,e) => DeleteBook();
        
        toolbar.Controls.AddRange(new Control[] { _txtSearch, _cboCategory, btnAdd, btnEdit, btnDel });
        
        // Grid
        _grid = new DataGridView
        {
            Dock = DockStyle.Fill,
            AutoGenerateColumns = false,
            SelectionMode = DataGridViewSelectionMode.FullRowSelect,
            ReadOnly = true,
            AllowUserToAddRows = false,
            RowHeadersVisible = false,
            BackgroundColor = Color.White,
            BorderStyle = BorderStyle.None,
            GridColor = Color.FromArgb(230, 230, 230),
            Font = new Font("Segoe UI", 10),
            AlternatingRowsDefaultCellStyle = { BackColor = Color.FromArgb(248, 249, 250) },
            RowTemplate = { Height = 34 },
            EnableHeadersVisualStyles = false,
            MultiSelect = false
        };
        
        _grid.ColumnHeadersDefaultCellStyle.BackColor = Color.FromArgb(44, 62, 80);
        _grid.ColumnHeadersDefaultCellStyle.ForeColor = Color.White;
        _grid.ColumnHeadersDefaultCellStyle.Font = new Font("Segoe UI", 9, FontStyle.Bold);
        
        _grid.Columns.AddRange(new DataGridViewColumn[]
        {
            new DataGridViewTextBoxColumn { Name = "colId", HeaderText = "ID", Width = 50 },
            new DataGridViewTextBoxColumn { Name = "colTitle", HeaderText = "ชื่อหนังสือ", AutoSizeMode = DataGridViewAutoSizeColumnMode.Fill },
            new DataGridViewTextBoxColumn { Name = "colAuthor", HeaderText = "ผู้แต่ง", Width = 150 },
            new DataGridViewTextBoxColumn { Name = "colISBN", HeaderText = "ISBN", Width = 130 },
            new DataGridViewTextBoxColumn { Name = "colCategory", HeaderText = "หมวดหมู่", Width = 110 },
            new DataGridViewTextBoxColumn { Name = "colYear", HeaderText = "ปี", Width = 55 },
            new DataGridViewTextBoxColumn
            {
                Name = "colAvail", HeaderText = "พร้อม/ทั้งหมด", Width = 110,
                DefaultCellStyle = { Alignment = DataGridViewContentAlignment.MiddleCenter }
            }
        });
        
        _grid.CellFormatting += (s,e) =>
        {
            if (e.ColumnIndex == _grid.Columns["colAvail"]!.Index && e.Value is string avail)
            {
                var parts = avail.Split('/');
                if (parts.Length == 2 && int.TryParse(parts[0].Trim(), out int a))
                    e.CellStyle!.ForeColor = a == 0 ? Color.Red : a <= 1 ? Color.Orange : Color.DarkGreen;
            }
        };
        
        _grid.CellDoubleClick += (s,e) => { if (e.RowIndex >= 0) EditBook(); };
        
        // Bottom
        var bottom = new Panel { Dock = DockStyle.Bottom, Height = 35, BackColor = Color.WhiteSmoke };
        _lblCount = new Label { Dock = DockStyle.Fill, Padding = new Padding(10, 8, 0, 0), Font = new Font("Segoe UI", 9) };
        bottom.Controls.Add(_lblCount);
        
        _status = new StatusStrip();
        _lblStatus = new ToolStripStatusLabel("Ready") { Spring = true };
        _status.Items.Add(_lblStatus);
        
        Controls.Add(_grid);
        Controls.Add(bottom);
        Controls.Add(toolbar);
        Controls.Add(menu);
        Controls.Add(_status);
    }
    
    private Button MakeBtn(string text, int x, Color color)
    {
        var b = new Button { Text = text, Location = new Point(x, 12), Height = 28, AutoSize = true, BackColor = color, ForeColor = Color.White, FlatStyle = FlatStyle.Flat, Cursor = Cursors.Hand };
        b.FlatAppearance.BorderSize = 0;
        return b;
    }
    
    private void LoadBooks(string? search = null)
    {
        string? searchTerm = string.IsNullOrWhiteSpace(search) ? null : search;
        string selectedCat = _cboCategory.SelectedItem?.ToString() ?? "ทุกหมวดหมู่";
        
        var books = searchTerm != null 
            ? _bookRepo.GetAll(searchTerm) 
            : _bookRepo.GetAll();
        
        if (selectedCat != "ทุกหมวดหมู่")
            books = books.Where(b => b.Category == selectedCat).ToList();
        
        PopulateGrid(books);
    }
    
    private void PopulateGrid(List<Book> books)
    {
        _grid.Rows.Clear();
        foreach (var b in books)
        {
            int idx = _grid.Rows.Add();
            var row = _grid.Rows[idx];
            row.Cells["colId"].Value = b.Id;
            row.Cells["colTitle"].Value = b.Title;
            row.Cells["colAuthor"].Value = b.Author;
            row.Cells["colISBN"].Value = b.ISBN;
            row.Cells["colCategory"].Value = b.Category;
            row.Cells["colYear"].Value = b.Year;
            row.Cells["colAvail"].Value = $"{b.AvailableCopies} / {b.TotalCopies}";
            row.Tag = b;
        }
        
        _lblCount.Text = $"รวม: {books.Count} เล่ม | มีพร้อม: {books.Count(b => b.AvailableCopies > 0)} เล่ม | หมด: {books.Count(b => b.AvailableCopies == 0)} เล่ม";
    }
    
    private void AddBook()
    {
        using var dlg = new BookEditDialog();
        if (dlg.ShowDialog() == DialogResult.OK && dlg.Result != null)
        {
            int id = _bookRepo.Create(dlg.Result);
            LoadBooks();
            SetStatus($"เพิ่มหนังสือ '{dlg.Result.Title}' (ID: {id}) สำเร็จ");
        }
    }
    
    private void EditBook()
    {
        if (_grid.SelectedRows.Count == 0) return;
        var selected = (Book)_grid.SelectedRows[0].Tag!;
        
        using var dlg = new BookEditDialog(selected);
        if (dlg.ShowDialog() == DialogResult.OK && dlg.Result != null)
        {
            _bookRepo.Update(dlg.Result);
            LoadBooks();
            SetStatus($"แก้ไขหนังสือ '{dlg.Result.Title}' สำเร็จ");
        }
    }
    
    private void DeleteBook()
    {
        if (_grid.SelectedRows.Count == 0) return;
        var selected = (Book)_grid.SelectedRows[0].Tag!;
        
        if (MessageBox.Show($"ต้องการลบ '{selected.Title}'?", "ยืนยันการลบ",
            MessageBoxButtons.YesNo, MessageBoxIcon.Warning) == DialogResult.Yes)
        {
            _bookRepo.Delete(selected.Id);
            LoadBooks();
            SetStatus($"ลบหนังสือ '{selected.Title}' สำเร็จ");
        }
    }
    
    private void ShowAvailableOnly()
    {
        var books = _bookRepo.GetAll().Where(b => b.AvailableCopies > 0).ToList();
        PopulateGrid(books);
        SetStatus($"แสดงเฉพาะที่พร้อมยืม: {books.Count} เล่ม");
    }
    
    private void ShowCategoryReport()
    {
        var all = _bookRepo.GetAll();
        var report = all
            .GroupBy(b => string.IsNullOrEmpty(b.Category) ? "Uncategorized" : b.Category)
            .OrderBy(g => g.Key);
        
        var sb = new System.Text.StringBuilder("รายงานตามหมวดหมู่\n===================\n\n");
        foreach (var g in report)
            sb.AppendLine($"{g.Key}: {g.Count()} เล่ม (พร้อม: {g.Sum(b => b.AvailableCopies)})");
        
        MessageBox.Show(sb.ToString(), "Category Report", MessageBoxButtons.OK, MessageBoxIcon.Information);
    }
    
    private void ImportFromCsv(object? s, EventArgs e)
    {
        using var dlg = new OpenFileDialog { Filter = "CSV|*.csv" };
        if (dlg.ShowDialog() != DialogResult.OK) return;
        
        int count = 0;
        foreach (var line in System.IO.File.ReadAllLines(dlg.FileName).Skip(1))
        {
            var p = line.Split(',');
            if (p.Length < 3) continue;
            _bookRepo.Create(new Book(0, p[0].Trim('"'), p[1].Trim('"'), p.Length > 2 ? p[2].Trim('"') : "",
                p.Length > 3 ? p[3].Trim('"') : "", p.Length > 4 ? int.Parse(p[4]) : 0, 1, 1));
            count++;
        }
        LoadBooks();
        SetStatus($"Import สำเร็จ: {count} เล่ม");
    }
    
    private void ExportToCsv(object? s, EventArgs e)
    {
        using var dlg = new SaveFileDialog { Filter = "CSV|*.csv", FileName = "books.csv" };
        if (dlg.ShowDialog() != DialogResult.OK) return;
        
        var lines = new List<string> { "Title,Author,ISBN,Category,Year,TotalCopies" };
        foreach (var b in _bookRepo.GetAll())
            lines.Add($"\"{b.Title}\",\"{b.Author}\",{b.ISBN},{b.Category},{b.Year},{b.TotalCopies}");
        
        System.IO.File.WriteAllLines(dlg.FileName, lines);
        SetStatus($"Export สำเร็จ: {lines.Count - 1} เล่ม");
    }
    
    private void SetStatus(string msg) => _lblStatus.Text = $"[{DateTime.Now:HH:mm:ss}] {msg}";
}

// Book Edit Dialog
public class BookEditDialog : Form
{
    private TextBox _txtTitle = null!, _txtAuthor = null!, _txtISBN = null!;
    private ComboBox _cboCategory = null!;
    private NumericUpDown _numYear = null!, _numCopies = null!;
    public Book? Result { get; private set; }
    
    public BookEditDialog(Book? existing = null)
    {
        Text = existing == null ? "เพิ่มหนังสือ" : "แก้ไขหนังสือ";
        Size = new Size(400, 340);
        StartPosition = FormStartPosition.CenterParent;
        FormBorderStyle = FormBorderStyle.FixedDialog;
        MaximizeBox = false;
        
        var layout = new TableLayoutPanel { Dock = DockStyle.Fill, ColumnCount = 2, Padding = new Padding(15), RowCount = 7 };
        layout.ColumnStyles.Add(new ColumnStyle(SizeType.Absolute, 110));
        layout.ColumnStyles.Add(new ColumnStyle(SizeType.Percent, 100));
        for (int i = 0; i < 7; i++) layout.RowStyles.Add(new RowStyle(SizeType.Absolute, 36));
        
        Label L(string t) => new Label { Text = t, Anchor = AnchorStyles.Right, AutoSize = true, Margin = new Padding(0, 8, 5, 0) };
        
        layout.Controls.Add(L("ชื่อหนังสือ:"), 0, 0);
        _txtTitle = new TextBox { Dock = DockStyle.Fill, Text = existing?.Title ?? "" };
        layout.Controls.Add(_txtTitle, 1, 0);
        
        layout.Controls.Add(L("ผู้แต่ง:"), 0, 1);
        _txtAuthor = new TextBox { Dock = DockStyle.Fill, Text = existing?.Author ?? "" };
        layout.Controls.Add(_txtAuthor, 1, 1);
        
        layout.Controls.Add(L("ISBN:"), 0, 2);
        _txtISBN = new TextBox { Dock = DockStyle.Fill, Text = existing?.ISBN ?? "" };
        layout.Controls.Add(_txtISBN, 1, 2);
        
        layout.Controls.Add(L("หมวดหมู่:"), 0, 3);
        _cboCategory = new ComboBox { Dock = DockStyle.Fill, DropDownStyle = ComboBoxStyle.DropDownList };
        _cboCategory.Items.AddRange(new[] { "Programming", "Science", "History", "Fiction", "Business", "Other" });
        _cboCategory.SelectedItem = existing?.Category ?? "Programming";
        if (_cboCategory.SelectedIndex < 0) _cboCategory.SelectedIndex = 0;
        layout.Controls.Add(_cboCategory, 1, 3);
        
        layout.Controls.Add(L("ปีพิมพ์:"), 0, 4);
        _numYear = new NumericUpDown { Minimum = 1900, Maximum = DateTime.Now.Year + 1, Value = existing?.Year ?? DateTime.Now.Year };
        layout.Controls.Add(_numYear, 1, 4);
        
        layout.Controls.Add(L("จำนวนสำเนา:"), 0, 5);
        _numCopies = new NumericUpDown { Minimum = 1, Maximum = 999, Value = existing?.TotalCopies ?? 1 };
        layout.Controls.Add(_numCopies, 1, 5);
        
        var btnOk = new Button { Text = "บันทึก", Size = new Size(80, 30), DialogResult = DialogResult.OK };
        btnOk.Click += (s,e) =>
        {
            if (string.IsNullOrWhiteSpace(_txtTitle.Text)) { MessageBox.Show("กรุณากรอกชื่อหนังสือ"); return; }
            if (string.IsNullOrWhiteSpace(_txtAuthor.Text)) { MessageBox.Show("กรุณากรอกชื่อผู้แต่ง"); return; }
            Result = new Book(existing?.Id ?? 0, _txtTitle.Text, _txtAuthor.Text, _txtISBN.Text,
                _cboCategory.SelectedItem?.ToString() ?? "", (int)_numYear.Value, (int)_numCopies.Value, (int)_numCopies.Value);
            DialogResult = DialogResult.OK;
            Close();
        };
        
        var btnCancel = new Button { Text = "ยกเลิก", Size = new Size(80, 30), DialogResult = DialogResult.Cancel };
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

## 📝 สรุป Part 29

| หัวข้อ | Key Points |
|--------|-----------|
| SQLite | ไม่ต้องติดตั้ง server |
| Repository Pattern | แยก Data Access จาก UI |
| SqliteConnection | เชื่อมต่อ database |
| SqliteCommand | Execute SQL |
| SqliteDataReader | อ่านข้อมูล |
| Parameters | ป้องกัน SQL Injection |

---

**ก่อนหน้า → [Part 28: Custom Controls](part28-winforms-custom-controls.md)**  
**ต่อไป → [Part 30: WinForms Final Project](part30-winforms-final.md)**
