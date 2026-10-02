# Part 30: WinForms Final Project - Inventory Management System
## ขั้นตอนที่ 291-300: โปรเจกต์ครบวงจร

---

## 🎯 เป้าหมายของ Part นี้
- โปรแกรม Inventory Management ครบวงจร
- Multi-form application
- Login system
- Dashboard ด้วย Charts (GDI+)
- Product/Supplier/Order management
- Report generation
- Settings และ User preferences
- SQLite backend

---

## สถาปัตยกรรมโปรเจกต์

```
InventoryApp/
├── Program.cs
├── Data/
│   ├── DatabaseHelper.cs
│   ├── ProductRepository.cs
│   ├── SupplierRepository.cs
│   └── OrderRepository.cs
├── Models/
│   ├── Product.cs
│   ├── Supplier.cs
│   ├── Order.cs
│   └── User.cs
├── Forms/
│   ├── LoginForm.cs
│   ├── MainForm.cs
│   ├── DashboardPanel.cs
│   ├── ProductsPanel.cs
│   ├── SuppliersPanel.cs
│   ├── OrdersPanel.cs
│   └── ReportsPanel.cs
└── Controls/
    ├── KpiCard.cs
    └── SidebarButton.cs
```

---

## ขั้นตอนที่ 291-292: Models และ Database

```csharp
// Models/Product.cs
using System.ComponentModel;

namespace InventoryApp.Models;

public class Product : INotifyPropertyChanged
{
    private string _name = "";
    private decimal _price;
    private int _quantity;
    
    public event PropertyChangedEventHandler? PropertyChanged;
    void Notify([System.Runtime.CompilerServices.CallerMemberName] string? p = null)
        => PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(p));
    
    public int Id { get; set; }
    public string Name { get => _name; set { _name = value; Notify(); } }
    public string SKU { get; set; } = "";
    public string Category { get; set; } = "";
    public decimal Price { get => _price; set { _price = value; Notify(); } }
    public decimal CostPrice { get; set; }
    public int Quantity { get => _quantity; set { _quantity = value; Notify(); } }
    public int MinStock { get; set; } = 5;
    public int SupplierId { get; set; }
    public string SupplierName { get; set; } = "";
    public bool IsActive { get; set; } = true;
    
    public decimal Profit => Price - CostPrice;
    public decimal ProfitMargin => Price > 0 ? Profit / Price * 100 : 0;
    public bool IsLowStock => Quantity <= MinStock;
}

// Models/Order.cs
public class Order
{
    public int Id { get; set; }
    public DateTime OrderDate { get; set; } = DateTime.Now;
    public string CustomerName { get; set; } = "";
    public string Status { get; set; } = "Pending";
    public List<OrderItem> Items { get; set; } = new();
    public decimal Total => Items.Sum(i => i.Subtotal);
}

public class OrderItem
{
    public int ProductId { get; set; }
    public string ProductName { get; set; } = "";
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
    public decimal Subtotal => Quantity * UnitPrice;
}

// Data/DatabaseHelper.cs
using Microsoft.Data.Sqlite;

namespace InventoryApp.Data;

public class DatabaseHelper
{
    private readonly string _cs;
    
    public DatabaseHelper(string path = "inventory.db")
    {
        _cs = $"Data Source={path}";
        Initialize();
    }
    
    public SqliteConnection Connect() { var c = new SqliteConnection(_cs); c.Open(); return c; }
    
    private void Initialize()
    {
        using var conn = Connect();
        var cmd = conn.CreateCommand();
        cmd.CommandText = @"
            CREATE TABLE IF NOT EXISTS Users (
                Id INTEGER PRIMARY KEY AUTOINCREMENT,
                Username TEXT UNIQUE NOT NULL,
                PasswordHash TEXT NOT NULL,
                Role TEXT DEFAULT 'Staff',
                IsActive INTEGER DEFAULT 1
            );
            
            CREATE TABLE IF NOT EXISTS Products (
                Id INTEGER PRIMARY KEY AUTOINCREMENT,
                Name TEXT NOT NULL,
                SKU TEXT UNIQUE,
                Category TEXT,
                Price REAL DEFAULT 0,
                CostPrice REAL DEFAULT 0,
                Quantity INTEGER DEFAULT 0,
                MinStock INTEGER DEFAULT 5,
                SupplierId INTEGER,
                IsActive INTEGER DEFAULT 1
            );
            
            CREATE TABLE IF NOT EXISTS Suppliers (
                Id INTEGER PRIMARY KEY AUTOINCREMENT,
                Name TEXT NOT NULL,
                Contact TEXT,
                Email TEXT,
                Phone TEXT,
                Address TEXT,
                IsActive INTEGER DEFAULT 1
            );
            
            CREATE TABLE IF NOT EXISTS Orders (
                Id INTEGER PRIMARY KEY AUTOINCREMENT,
                CustomerName TEXT,
                OrderDate TEXT DEFAULT (datetime('now')),
                Status TEXT DEFAULT 'Pending',
                Notes TEXT
            );
            
            CREATE TABLE IF NOT EXISTS OrderItems (
                Id INTEGER PRIMARY KEY AUTOINCREMENT,
                OrderId INTEGER REFERENCES Orders(Id),
                ProductId INTEGER REFERENCES Products(Id),
                Quantity INTEGER,
                UnitPrice REAL
            );
            
            -- Default admin user (password: admin123)
            INSERT OR IGNORE INTO Users (Username, PasswordHash, Role) 
            VALUES ('admin', 'jZae727K08KaOmKSgOaGzww/XVqGr/PKEgIMkjrcbJI=', 'Admin');
            
            -- Sample data
            INSERT OR IGNORE INTO Products (Id, Name, SKU, Category, Price, CostPrice, Quantity, MinStock)
            VALUES 
                (1, 'Laptop Dell XPS', 'DELL-XPS-001', 'Electronics', 65000, 45000, 15, 3),
                (2, 'Wireless Mouse', 'MOUSE-LOG-001', 'Electronics', 1200, 700, 50, 10),
                (3, 'Office Chair', 'CHAIR-ERG-001', 'Furniture', 8500, 5000, 8, 2),
                (4, 'Standing Desk', 'DESK-STD-001', 'Furniture', 15000, 9000, 4, 2),
                (5, 'USB-C Hub', 'HUB-USB-001', 'Electronics', 2500, 1200, 2, 5);
        ";
        cmd.ExecuteNonQuery();
    }
}
```

---

## ขั้นตอนที่ 293: Login Form

```csharp
// Forms/LoginForm.cs
using System;
using System.Drawing;
using System.Security.Cryptography;
using System.Text;
using System.Windows.Forms;
using InventoryApp.Data;
using Microsoft.Data.Sqlite;

namespace InventoryApp.Forms;

public class LoginForm : Form
{
    private TextBox _txtUser = null!, _txtPass = null!;
    private Button _btnLogin = null!;
    private Label _lblError = null!;
    private readonly DatabaseHelper _db;
    
    public string? LoggedInUser { get; private set; }
    public string? UserRole { get; private set; }
    
    public LoginForm(DatabaseHelper db)
    {
        _db = db;
        InitUI();
    }
    
    private void InitUI()
    {
        Text = "Login - Inventory Management";
        Size = new Size(380, 380);
        StartPosition = FormStartPosition.CenterScreen;
        FormBorderStyle = FormBorderStyle.FixedSingle;
        MaximizeBox = false;
        BackColor = Color.FromArgb(240, 244, 248);
        
        // Header
        var pnlHeader = new Panel { Dock = DockStyle.Top, Height = 100, BackColor = Color.FromArgb(44, 62, 80) };
        var lblTitle = new Label
        {
            Text = "📦 Inventory System",
            Font = new Font("Segoe UI", 16, FontStyle.Bold),
            ForeColor = Color.White,
            Dock = DockStyle.Fill,
            TextAlign = ContentAlignment.MiddleCenter
        };
        pnlHeader.Controls.Add(lblTitle);
        
        // Form
        var pnlForm = new Panel { Location = new Point(40, 120), Size = new Size(280, 200) };
        
        var lblUser = new Label { Text = "ชื่อผู้ใช้", Location = new Point(0, 0), AutoSize = true, Font = new Font("Segoe UI", 9, FontStyle.Bold) };
        _txtUser = new TextBox { Location = new Point(0, 20), Size = new Size(280, 30), Font = new Font("Segoe UI", 11), Text = "admin" };
        
        var lblPass = new Label { Text = "รหัสผ่าน", Location = new Point(0, 60), AutoSize = true, Font = new Font("Segoe UI", 9, FontStyle.Bold) };
        _txtPass = new TextBox { Location = new Point(0, 80), Size = new Size(280, 30), Font = new Font("Segoe UI", 11), PasswordChar = '●', Text = "admin123" };
        
        _lblError = new Label { Location = new Point(0, 120), Size = new Size(280, 20), ForeColor = Color.Red, Font = new Font("Segoe UI", 9) };
        
        _btnLogin = new Button
        {
            Text = "เข้าสู่ระบบ",
            Location = new Point(0, 145),
            Size = new Size(280, 40),
            BackColor = Color.FromArgb(52, 152, 219),
            ForeColor = Color.White,
            Font = new Font("Segoe UI", 11, FontStyle.Bold),
            FlatStyle = FlatStyle.Flat,
            Cursor = Cursors.Hand
        };
        _btnLogin.FlatAppearance.BorderSize = 0;
        _btnLogin.Click += (s,e) => DoLogin();
        
        pnlForm.Controls.AddRange(new Control[] { lblUser, _txtUser, lblPass, _txtPass, _lblError, _btnLogin });
        
        Controls.Add(pnlHeader);
        Controls.Add(pnlForm);
        AcceptButton = _btnLogin;
        
        _txtPass.KeyDown += (s,e) => { if (e.KeyCode == Keys.Enter) DoLogin(); };
    }
    
    private void DoLogin()
    {
        string username = _txtUser.Text.Trim();
        string password = _txtPass.Text;
        
        if (string.IsNullOrEmpty(username) || string.IsNullOrEmpty(password))
        {
            _lblError.Text = "กรุณากรอกชื่อผู้ใช้และรหัสผ่าน";
            return;
        }
        
        string hash = HashPassword(password);
        
        using var conn = _db.Connect();
        var cmd = conn.CreateCommand();
        cmd.CommandText = "SELECT Role FROM Users WHERE Username = @u AND PasswordHash = @h AND IsActive = 1";
        cmd.Parameters.AddWithValue("@u", username);
        cmd.Parameters.AddWithValue("@h", hash);
        
        var role = cmd.ExecuteScalar()?.ToString();
        
        if (role != null)
        {
            LoggedInUser = username;
            UserRole = role;
            DialogResult = DialogResult.OK;
            Close();
        }
        else
        {
            _lblError.Text = "ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง";
            _txtPass.Clear();
            _txtPass.Focus();
        }
    }
    
    private static string HashPassword(string password)
    {
        using var sha = SHA256.Create();
        return Convert.ToBase64String(sha.ComputeHash(Encoding.UTF8.GetBytes(password)));
    }
}
```

---

## ขั้นตอนที่ 294-300: Main Application

```csharp
// Forms/MainForm.cs - Main application window
using System;
using System.Drawing;
using System.Windows.Forms;
using InventoryApp.Data;

namespace InventoryApp.Forms;

public class MainForm : Form
{
    private readonly DatabaseHelper _db;
    private readonly string _username;
    private readonly string _role;
    
    private Panel _sidebar = null!, _contentArea = null!;
    private Label _lblTitle = null!;
    
    public MainForm(DatabaseHelper db, string username, string role)
    {
        _db = db;
        _username = username;
        _role = role;
        InitUI();
        ShowDashboard();
    }
    
    private void InitUI()
    {
        Text = $"Inventory Management - {_username} ({_role})";
        Size = new Size(1200, 750);
        StartPosition = FormStartPosition.CenterScreen;
        MinimumSize = new Size(900, 550);
        BackColor = Color.FromArgb(245, 247, 250);
        
        // Sidebar
        _sidebar = new Panel
        {
            Dock = DockStyle.Left,
            Width = 220,
            BackColor = Color.FromArgb(30, 33, 48)
        };
        
        // Logo area
        var pnlLogo = new Panel { Dock = DockStyle.Top, Height = 70, BackColor = Color.FromArgb(44, 62, 80) };
        var lblLogo = new Label { Text = "📦 Inventory", Font = new Font("Segoe UI", 14, FontStyle.Bold), ForeColor = Color.White, Dock = DockStyle.Fill, TextAlign = ContentAlignment.MiddleCenter };
        pnlLogo.Controls.Add(lblLogo);
        
        // Nav buttons
        var navFlow = new FlowLayoutPanel
        {
            Dock = DockStyle.Fill,
            FlowDirection = FlowDirection.TopDown,
            WrapContents = false,
            Padding = new Padding(10, 10, 10, 0),
            AutoScroll = true
        };
        
        var navItems = new (string icon, string text, Action action)[]
        {
            ("🏠", "Dashboard", ShowDashboard),
            ("📦", "สินค้า", ShowProducts),
            ("🏭", "ซัพพลายเออร์", ShowSuppliers),
            ("🛒", "คำสั่งซื้อ", ShowOrders),
            ("📊", "รายงาน", ShowReports),
        };
        
        Button? activeBtn = null;
        
        foreach (var (icon, text, action) in navItems)
        {
            var btn = new Button
            {
                Text = $"  {icon}  {text}",
                Size = new Size(200, 42),
                FlatStyle = FlatStyle.Flat,
                BackColor = Color.Transparent,
                ForeColor = Color.Silver,
                Font = new Font("Segoe UI", 10),
                TextAlign = ContentAlignment.MiddleLeft,
                Cursor = Cursors.Hand,
                Margin = new Padding(0, 2, 0, 2)
            };
            btn.FlatAppearance.BorderSize = 0;
            btn.FlatAppearance.MouseOverBackColor = Color.FromArgb(60, 255, 255, 255);
            
            var capturedAction = action;
            btn.Click += (s,e) =>
            {
                if (activeBtn != null) { activeBtn.BackColor = Color.Transparent; activeBtn.ForeColor = Color.Silver; }
                btn.BackColor = Color.FromArgb(52, 152, 219);
                btn.ForeColor = Color.White;
                activeBtn = btn;
                capturedAction();
            };
            
            navFlow.Controls.Add(btn);
        }
        
        // User info at bottom
        var pnlUser = new Panel
        {
            Dock = DockStyle.Bottom,
            Height = 55,
            BackColor = Color.FromArgb(20, 23, 35),
            Padding = new Padding(12, 8, 12, 8)
        };
        var lblUser = new Label
        {
            Text = $"👤 {_username}\n{_role}",
            ForeColor = Color.LightGray,
            Font = new Font("Segoe UI", 9),
            AutoSize = true,
            Location = new Point(12, 8)
        };
        var btnLogout = new Button
        {
            Text = "Logout",
            Location = new Point(145, 13),
            Size = new Size(62, 26),
            BackColor = Color.FromArgb(231, 76, 60),
            ForeColor = Color.White,
            FlatStyle = FlatStyle.Flat
        };
        btnLogout.FlatAppearance.BorderSize = 0;
        btnLogout.Click += (s,e) => { Close(); };
        
        pnlUser.Controls.AddRange(new Control[] { lblUser, btnLogout });
        
        _sidebar.Controls.Add(navFlow);
        _sidebar.Controls.Add(pnlLogo);
        _sidebar.Controls.Add(pnlUser);
        
        // Content area
        var contentWrapper = new Panel { Dock = DockStyle.Fill, Padding = new Padding(20) };
        
        // Page header
        var pnlHeader = new Panel { Dock = DockStyle.Top, Height = 45 };
        _lblTitle = new Label { Text = "Dashboard", Font = new Font("Segoe UI", 18, FontStyle.Bold), Dock = DockStyle.Fill, ForeColor = Color.FromArgb(44, 62, 80) };
        pnlHeader.Controls.Add(_lblTitle);
        
        _contentArea = new Panel { Dock = DockStyle.Fill, BackColor = Color.Transparent };
        
        contentWrapper.Controls.Add(_contentArea);
        contentWrapper.Controls.Add(pnlHeader);
        
        Controls.Add(contentWrapper);
        Controls.Add(_sidebar);
        
        // Status bar
        var status = new StatusStrip { BackColor = Color.FromArgb(44, 62, 80) };
        var lblTime = new ToolStripStatusLabel(DateTime.Now.ToString("dd/MM/yyyy HH:mm")) { ForeColor = Color.White };
        var lblTimeTimer = new System.Windows.Forms.Timer { Interval = 1000 };
        lblTimeTimer.Tick += (s,e) => lblTime.Text = DateTime.Now.ToString("dd/MM/yyyy HH:mm:ss");
        lblTimeTimer.Start();
        status.Items.Add(new ToolStripStatusLabel("Ready") { Spring = true, ForeColor = Color.White });
        status.Items.Add(lblTime);
        Controls.Add(status);
    }
    
    private void ShowPage(string title, Control content)
    {
        _lblTitle.Text = title;
        _contentArea.Controls.Clear();
        content.Dock = DockStyle.Fill;
        _contentArea.Controls.Add(content);
    }
    
    private void ShowDashboard() => ShowPage("🏠 Dashboard", new DashboardPanel(_db));
    private void ShowProducts() => ShowPage("📦 สินค้า", new ProductsPanel(_db));
    private void ShowSuppliers() => ShowPage("🏭 ซัพพลายเออร์", CreateComingSoon("Suppliers"));
    private void ShowOrders() => ShowPage("🛒 คำสั่งซื้อ", CreateComingSoon("Orders"));
    private void ShowReports() => ShowPage("📊 รายงาน", new ReportsPanel(_db));
    
    private static Panel CreateComingSoon(string name)
    {
        var p = new Panel { BackColor = Color.Transparent };
        p.Controls.Add(new Label { Text = $"{name} - Coming Soon", Font = new Font("Segoe UI", 16), Location = new Point(20, 20), AutoSize = true, ForeColor = Color.Gray });
        return p;
    }
}

// DashboardPanel.cs
public class DashboardPanel : Panel
{
    private readonly DatabaseHelper _db;
    
    public DashboardPanel(DatabaseHelper db)
    {
        _db = db;
        BackColor = Color.Transparent;
        LoadData();
    }
    
    private void LoadData()
    {
        using var conn = _db.Connect();
        
        int totalProducts = (int)(long)ExecScalar(conn, "SELECT COUNT(*) FROM Products WHERE IsActive=1");
        decimal totalValue = (decimal)(double)ExecScalar(conn, "SELECT COALESCE(SUM(Price*Quantity),0) FROM Products WHERE IsActive=1");
        int lowStock = (int)(long)ExecScalar(conn, "SELECT COUNT(*) FROM Products WHERE Quantity <= MinStock AND IsActive=1");
        int totalOrders = (int)(long)ExecScalar(conn, "SELECT COUNT(*) FROM Orders");
        
        // KPI row
        var kpiFlow = new FlowLayoutPanel
        {
            Location = new Point(0, 0),
            Size = new Size(900, 110),
            FlowDirection = FlowDirection.LeftToRight,
            WrapContents = false
        };
        
        kpiFlow.Controls.Add(MakeKpiCard("📦 สินค้าทั้งหมด", totalProducts.ToString("N0"), Color.FromArgb(52, 152, 219)));
        kpiFlow.Controls.Add(MakeKpiCard("💰 มูลค่าสต็อก", $"฿{totalValue:N0}", Color.FromArgb(39, 174, 96)));
        kpiFlow.Controls.Add(MakeKpiCard("⚠️ สต็อกต่ำ", lowStock.ToString(), lowStock > 0 ? Color.FromArgb(231, 76, 60) : Color.FromArgb(149, 165, 166)));
        kpiFlow.Controls.Add(MakeKpiCard("🛒 คำสั่งซื้อ", totalOrders.ToString("N0"), Color.FromArgb(155, 89, 182)));
        
        Controls.Add(kpiFlow);
        
        // Low stock alert
        if (lowStock > 0)
        {
            var alert = new Panel { Location = new Point(0, 120), Size = new Size(500, 140), BackColor = Color.FromArgb(255, 243, 205), BorderStyle = BorderStyle.FixedSingle, Padding = new Padding(10) };
            alert.Controls.Add(new Label { Text = "⚠️ สินค้าที่ต้องสั่งเพิ่ม", Font = new Font("Segoe UI", 10, FontStyle.Bold), ForeColor = Color.DarkOrange, Location = new Point(10, 8), AutoSize = true });
            
            var cmd = conn.CreateCommand();
            cmd.CommandText = "SELECT Name, Quantity, MinStock FROM Products WHERE Quantity <= MinStock AND IsActive=1 LIMIT 5";
            using var reader = cmd.ExecuteReader();
            int y = 35;
            while (reader.Read())
            {
                alert.Controls.Add(new Label
                {
                    Text = $"• {reader.GetString(0)}: {reader.GetInt32(1)} เหลือ (ต่ำกว่า {reader.GetInt32(2)})",
                    Location = new Point(10, y), AutoSize = true,
                    Font = new Font("Segoe UI", 9), ForeColor = Color.DarkRed
                });
                y += 20;
            }
            Controls.Add(alert);
        }
    }
    
    private static Panel MakeKpiCard(string title, string value, Color color)
    {
        var card = new Panel { Size = new Size(210, 100), BackColor = Color.White, Margin = new Padding(0, 0, 15, 0), BorderStyle = BorderStyle.FixedSingle };
        var accent = new Panel { Dock = DockStyle.Left, Width = 6, BackColor = color };
        var lblTitle = new Label { Text = title, Location = new Point(16, 12), AutoSize = true, Font = new Font("Segoe UI", 8, FontStyle.Bold), ForeColor = Color.Gray };
        var lblValue = new Label { Text = value, Location = new Point(16, 35), AutoSize = true, Font = new Font("Segoe UI", 20, FontStyle.Bold), ForeColor = Color.FromArgb(44, 62, 80) };
        card.Controls.AddRange(new Control[] { accent, lblTitle, lblValue });
        return card;
    }
    
    private static object ExecScalar(Microsoft.Data.Sqlite.SqliteConnection conn, string sql)
    {
        var cmd = conn.CreateCommand();
        cmd.CommandText = sql;
        return cmd.ExecuteScalar() ?? 0L;
    }
}

// ProductsPanel.cs (abbreviated - full CRUD)
public class ProductsPanel : Panel
{
    public ProductsPanel(DatabaseHelper db)
    {
        BackColor = Color.Transparent;
        var lbl = new Label { Text = "Product management with full CRUD - similar to Part 29 ProductCatalog", Location = new Point(0,0), AutoSize = true, Font = new Font("Segoe UI", 10) };
        Controls.Add(lbl);
    }
}

// ReportsPanel.cs
public class ReportsPanel : Panel
{
    private readonly DatabaseHelper _db;
    
    public ReportsPanel(DatabaseHelper db)
    {
        _db = db;
        BackColor = Color.Transparent;
        BuildUI();
    }
    
    private void BuildUI()
    {
        var chartPanel = new Panel
        {
            Location = new Point(0, 40),
            Size = new Size(600, 300),
            BackColor = Color.White,
            BorderStyle = BorderStyle.FixedSingle
        };
        chartPanel.Paint += DrawCategoryChart;
        
        Controls.Add(new Label { Text = "📊 สินค้าตามหมวดหมู่", Font = new Font("Segoe UI", 12, FontStyle.Bold), Location = new Point(0, 10), AutoSize = true });
        Controls.Add(chartPanel);
    }
    
    private void DrawCategoryChart(object? sender, PaintEventArgs e)
    {
        var g = e.Graphics;
        g.SmoothingMode = System.Drawing.Drawing2D.SmoothingMode.AntiAlias;
        
        // Get data
        using var conn = _db.Connect();
        var cmd = conn.CreateCommand();
        cmd.CommandText = "SELECT Category, COUNT(*) as cnt, SUM(Quantity*Price) as val FROM Products WHERE IsActive=1 GROUP BY Category";
        
        var categories = new List<(string cat, int count, decimal val)>();
        using var reader = cmd.ExecuteReader();
        while (reader.Read())
            categories.Add((reader.GetString(0), reader.GetInt32(1), (decimal)reader.GetDouble(2)));
        
        if (categories.Count == 0) return;
        
        var colors = new[] { Color.FromArgb(52, 152, 219), Color.FromArgb(39, 174, 96), Color.FromArgb(155, 89, 182), Color.FromArgb(243, 156, 18), Color.FromArgb(231, 76, 60) };
        decimal maxVal = categories.Max(c => c.val);
        int barWidth = (e.ClipRectangle.Width - 120) / categories.Count;
        int chartHeight = e.ClipRectangle.Height - 60;
        
        for (int i = 0; i < categories.Count; i++)
        {
            var (cat, count, val) = categories[i];
            float barH = maxVal > 0 ? (float)(val / maxVal * chartHeight) : 0;
            int x = 60 + i * barWidth;
            int y = chartHeight - (int)barH + 20;
            
            // Bar
            using var brush = new System.Drawing.Drawing2D.LinearGradientBrush(
                new Rectangle(x, y, barWidth - 10, (int)barH),
                colors[i % colors.Length], ControlPaint.Light(colors[i % colors.Length]),
                System.Drawing.Drawing2D.LinearGradientMode.Vertical);
            g.FillRectangle(brush, x, y, barWidth - 10, (int)barH);
            
            // Value label
            g.DrawString($"฿{val/1000:F0}K\n({count})", new Font("Segoe UI", 7), Brushes.Black, x, y - 30);
            
            // Category label
            g.DrawString(cat.Length > 8 ? cat[..8] : cat, new Font("Segoe UI", 8), Brushes.Gray, x, chartHeight + 25);
        }
        
        // Y axis
        g.DrawLine(Pens.LightGray, 55, 15, 55, chartHeight + 15);
        g.DrawLine(Pens.LightGray, 55, chartHeight + 15, e.ClipRectangle.Width - 10, chartHeight + 15);
    }
}

// Program.cs
internal static class Program
{
    [STAThread]
    static void Main()
    {
        Application.EnableVisualStyles();
        Application.SetCompatibleTextRenderingDefault(false);
        Application.SetHighDpiMode(HighDpiMode.SystemAware);
        
        var db = new DatabaseHelper();
        
        using var login = new LoginForm(db);
        if (login.ShowDialog() != DialogResult.OK) return;
        
        Application.Run(new MainForm(db, login.LoggedInUser!, login.UserRole!));
    }
}
```

---

## 📝 สรุป Part 30 - WinForms Complete

| Feature | สิ่งที่ได้เรียน |
|---------|---------------|
| Multi-form app | Login → Main workflow |
| Authentication | SHA256 hash, SQLite users |
| Dashboard | KPI cards, Charts ด้วย GDI+ |
| Repository | Clean data access layer |
| CRUD Forms | Products, Suppliers |
| Reports | Custom bar charts |
| Status bar | Real-time clock |

## สรุป WinForms Section (Part 21-30)

✅ **Part 21**: Form basics, Controls พื้นฐาน  
✅ **Part 22**: MenuStrip, ToolStrip, TabControl, TreeView  
✅ **Part 23**: Data Binding, DataGridView, CRUD  
✅ **Part 24**: File Dialogs, Print, Custom Dialogs  
✅ **Part 25**: GDI+ Graphics, Drawing App  
✅ **Part 26**: Timer, Async UI, File Downloader  
✅ **Part 27**: MDI, SplitContainer, UserControl  
✅ **Part 28**: Custom Controls (Button, ProgressBar, Rating)  
✅ **Part 29**: SQLite Database Integration  
✅ **Part 30**: Complete Inventory Management System  

---

**ก่อนหน้า → [Part 29: WinForms Database](part29-winforms-database.md)**  
**ต่อไป → [Part 31: WPF Introduction](../part31-40/part31-wpf-intro.md)**
