# Part 28: WinForms Custom Controls
## ขั้นตอนที่ 271-280: สร้าง Controls เอง

---

## 🎯 เป้าหมายของ Part นี้
- สร้าง Custom Control โดย extend Control
- Owner-drawn controls
- Custom Button, Label, TextBox
- Circular Progress Bar
- Rating Control (ดาว)
- Toggle Switch (iOS-style)
- Notification Panel
- โปรแกรม Dashboard UI

---

## ขั้นตอนที่ 271: Custom Button

```csharp
// Modern flat button with hover and click effects
public class ModernButton : Button
{
    private Color _normalColor;
    private Color _hoverColor;
    private Color _pressedColor;
    private bool _isHovered = false;
    private bool _isPressed = false;
    
    public ModernButton()
    {
        FlatStyle = FlatStyle.Flat;
        FlatAppearance.BorderSize = 0;
        ForeColor = Color.White;
        Font = new Font("Segoe UI", 10, FontStyle.Regular);
        Cursor = Cursors.Hand;
        
        _normalColor = Color.FromArgb(52, 152, 219);
        _hoverColor = Color.FromArgb(41, 128, 185);
        _pressedColor = Color.FromArgb(31, 97, 141);
        BackColor = _normalColor;
        
        MouseEnter += (s,e) => { _isHovered = true; BackColor = _hoverColor; Invalidate(); };
        MouseLeave += (s,e) => { _isHovered = false; _isPressed = false; BackColor = _normalColor; Invalidate(); };
        MouseDown += (s,e) => { _isPressed = true; BackColor = _pressedColor; Invalidate(); };
        MouseUp += (s,e) => { _isPressed = false; BackColor = _isHovered ? _hoverColor : _normalColor; Invalidate(); };
    }
    
    [System.ComponentModel.Category("Appearance")]
    public Color NormalColor { get => _normalColor; set { _normalColor = value; BackColor = value; } }
    
    [System.ComponentModel.Category("Appearance")]
    public Color HoverColor { get => _hoverColor; set => _hoverColor = value; }
    
    [System.ComponentModel.Category("Appearance")]
    public int CornerRadius { get; set; } = 8;
    
    protected override void OnPaint(PaintEventArgs e)
    {
        var g = e.Graphics;
        g.SmoothingMode = System.Drawing.Drawing2D.SmoothingMode.AntiAlias;
        
        // Rounded rectangle
        var path = RoundedRect(ClientRectangle, CornerRadius);
        
        using var brush = new System.Drawing.Drawing2D.LinearGradientBrush(
            ClientRectangle,
            BackColor,
            ControlPaint.Dark(BackColor, 0.1f),
            System.Drawing.Drawing2D.LinearGradientMode.Vertical);
        
        g.FillPath(brush, path);
        
        // Shadow (if not pressed)
        if (!_isPressed)
        {
            using var shadowBrush = new SolidBrush(Color.FromArgb(30, 0, 0, 0));
            g.FillRectangle(shadowBrush, new Rectangle(2, Height - 4, Width - 4, 3));
        }
        
        // Text
        var format = new StringFormat
        {
            Alignment = StringAlignment.Center,
            LineAlignment = StringAlignment.Center
        };
        
        g.DrawString(Text, Font, new SolidBrush(ForeColor), ClientRectangle, format);
        
        // Focus indicator
        if (Focused)
        {
            using var focusPen = new Pen(Color.White, 1) { DashStyle = System.Drawing.Drawing2D.DashStyle.Dot };
            var focusRect = new Rectangle(4, 4, Width - 9, Height - 9);
            g.DrawRectangle(focusPen, focusRect);
        }
    }
    
    private static System.Drawing.Drawing2D.GraphicsPath RoundedRect(Rectangle rect, int radius)
    {
        var path = new System.Drawing.Drawing2D.GraphicsPath();
        int d = radius * 2;
        path.AddArc(rect.X, rect.Y, d, d, 180, 90);
        path.AddArc(rect.Right - d, rect.Y, d, d, 270, 90);
        path.AddArc(rect.Right - d, rect.Bottom - d, d, d, 0, 90);
        path.AddArc(rect.X, rect.Bottom - d, d, d, 90, 90);
        path.CloseFigure();
        return path;
    }
}
```

---

## ขั้นตอนที่ 272: Circular ProgressBar

```csharp
// Circular progress bar
public class CircularProgress : Control
{
    private int _value = 0;
    private int _minimum = 0;
    private int _maximum = 100;
    private Color _progressColor = Color.DeepSkyBlue;
    private Color _trackColor = Color.FromArgb(230, 230, 230);
    private int _strokeWidth = 8;
    private bool _showText = true;
    
    public int Value
    {
        get => _value;
        set { _value = Math.Clamp(value, _minimum, _maximum); Invalidate(); }
    }
    
    public int Maximum { get => _maximum; set { _maximum = value; Invalidate(); } }
    public Color ProgressColor { get => _progressColor; set { _progressColor = value; Invalidate(); } }
    public int StrokeWidth { get => _strokeWidth; set { _strokeWidth = value; Invalidate(); } }
    public bool ShowText { get => _showText; set { _showText = value; Invalidate(); } }
    
    public CircularProgress()
    {
        Size = new Size(100, 100);
        DoubleBuffered = true;
        SetStyle(ControlStyles.AllPaintingInWmPaint | ControlStyles.OptimizedDoubleBuffer, true);
    }
    
    protected override void OnPaint(PaintEventArgs e)
    {
        var g = e.Graphics;
        g.SmoothingMode = System.Drawing.Drawing2D.SmoothingMode.AntiAlias;
        
        int margin = _strokeWidth / 2 + 2;
        var rect = new Rectangle(margin, margin, Width - margin * 2, Height - margin * 2);
        
        // Track (background circle)
        using var trackPen = new Pen(_trackColor, _strokeWidth);
        trackPen.StartCap = System.Drawing.Drawing2D.LineCap.Round;
        trackPen.EndCap = System.Drawing.Drawing2D.LineCap.Round;
        g.DrawEllipse(trackPen, rect);
        
        // Progress arc
        float percent = (float)(_value - _minimum) / (_maximum - _minimum);
        float sweepAngle = 360f * percent;
        
        if (sweepAngle > 0)
        {
            using var progressPen = new Pen(_progressColor, _strokeWidth);
            progressPen.StartCap = System.Drawing.Drawing2D.LineCap.Round;
            progressPen.EndCap = System.Drawing.Drawing2D.LineCap.Round;
            g.DrawArc(progressPen, rect, -90, sweepAngle);
        }
        
        // Center text
        if (_showText)
        {
            string text = $"{(int)(percent * 100)}%";
            var font = new Font("Segoe UI", Width / 6f, FontStyle.Bold, GraphicsUnit.Pixel);
            var sf = new StringFormat { Alignment = StringAlignment.Center, LineAlignment = StringAlignment.Center };
            g.DrawString(text, font, new SolidBrush(ForeColor), ClientRectangle, sf);
        }
    }
    
    // Animated fill
    public async Task AnimateToAsync(int target, int durationMs = 1000)
    {
        int start = _value;
        int steps = Math.Abs(target - start);
        if (steps == 0) return;
        
        int delay = durationMs / steps;
        int direction = target > start ? 1 : -1;
        
        while (_value != target)
        {
            Value += direction;
            await Task.Delay(Math.Max(1, delay));
        }
    }
}

// Usage
var cp = new CircularProgress
{
    Value = 0,
    ProgressColor = Color.FromArgb(46, 204, 113),
    Size = new Size(120, 120),
    Location = new Point(20, 20)
};
Controls.Add(cp);
await cp.AnimateToAsync(75, 1500);
```

---

## ขั้นตอนที่ 273: Rating Control

```csharp
// Star Rating Control
public class RatingControl : Control
{
    private int _rating = 0;
    private int _maxRating = 5;
    private int _hoverRating = 0;
    private Color _filledColor = Color.Gold;
    private Color _emptyColor = Color.LightGray;
    
    public event EventHandler? RatingChanged;
    
    public int Rating
    {
        get => _rating;
        set { _rating = Math.Clamp(value, 0, _maxRating); Invalidate(); RatingChanged?.Invoke(this, EventArgs.Empty); }
    }
    
    public int MaxRating { get => _maxRating; set { _maxRating = value; Size = new Size(value * 28 + 4, 28); Invalidate(); } }
    
    public RatingControl()
    {
        Size = new Size(_maxRating * 28 + 4, 28);
        DoubleBuffered = true;
        SetStyle(ControlStyles.AllPaintingInWmPaint | ControlStyles.OptimizedDoubleBuffer, true);
        Cursor = Cursors.Hand;
        
        MouseMove += (s,e) =>
        {
            int hover = e.X / 28 + 1;
            if (hover != _hoverRating) { _hoverRating = hover; Invalidate(); }
        };
        
        MouseLeave += (s,e) => { _hoverRating = 0; Invalidate(); };
        MouseClick += (s,e) => Rating = e.X / 28 + 1;
    }
    
    protected override void OnPaint(PaintEventArgs e)
    {
        var g = e.Graphics;
        g.SmoothingMode = System.Drawing.Drawing2D.SmoothingMode.AntiAlias;
        
        int display = _hoverRating > 0 ? _hoverRating : _rating;
        
        for (int i = 0; i < _maxRating; i++)
        {
            bool filled = i < display;
            var starPoints = GetStarPoints(new PointF(i * 28 + 14, 14), 12, 5, 5);
            
            using var brush = new SolidBrush(filled ? _filledColor : _emptyColor);
            g.FillPolygon(brush, starPoints);
            
            if (filled)
            {
                using var pen = new Pen(ControlPaint.Dark(_filledColor, 0.2f), 1);
                g.DrawPolygon(pen, starPoints);
            }
        }
    }
    
    private static PointF[] GetStarPoints(PointF center, float outerR, float innerR, int points)
    {
        var pts = new PointF[points * 2];
        double angle = -Math.PI / 2;
        double step = Math.PI / points;
        
        for (int i = 0; i < points * 2; i++)
        {
            float r = i % 2 == 0 ? outerR : innerR;
            pts[i] = new PointF(
                center.X + (float)(Math.Cos(angle) * r),
                center.Y + (float)(Math.Sin(angle) * r)
            );
            angle += step;
        }
        return pts;
    }
}
```

---

## ขั้นตอนที่ 274: Toggle Switch

```csharp
// iOS-style toggle switch
public class ToggleSwitch : Control
{
    private bool _checked = false;
    private Color _onColor = Color.FromArgb(52, 199, 89);
    private Color _offColor = Color.FromArgb(204, 204, 204);
    private float _animationPos;
    private System.Windows.Forms.Timer _animTimer = null!;
    
    public event EventHandler? CheckedChanged;
    
    public bool Checked
    {
        get => _checked;
        set
        {
            if (_checked != value)
            {
                _checked = value;
                StartAnimation();
                CheckedChanged?.Invoke(this, EventArgs.Empty);
            }
        }
    }
    
    public ToggleSwitch()
    {
        Size = new Size(60, 30);
        DoubleBuffered = true;
        SetStyle(ControlStyles.AllPaintingInWmPaint | ControlStyles.OptimizedDoubleBuffer, true);
        Cursor = Cursors.Hand;
        
        _animationPos = 3f;
        
        _animTimer = new System.Windows.Forms.Timer { Interval = 16 };
        _animTimer.Tick += (s,e) =>
        {
            float target = _checked ? Width - Height + 3f : 3f;
            float diff = target - _animationPos;
            
            if (Math.Abs(diff) < 1) { _animationPos = target; _animTimer.Stop(); }
            else _animationPos += diff * 0.3f;
            
            Invalidate();
        };
        
        Click += (s,e) => Checked = !Checked;
    }
    
    private void StartAnimation() => _animTimer.Start();
    
    protected override void OnPaint(PaintEventArgs e)
    {
        var g = e.Graphics;
        g.SmoothingMode = System.Drawing.Drawing2D.SmoothingMode.AntiAlias;
        
        // Track
        float target = _checked ? Width - Height + 3f : 3f;
        float blend = ((_animationPos - 3f) / (Width - Height)) ;
        blend = Math.Clamp(blend, 0, 1);
        
        Color trackColor = Color.FromArgb(
            (int)(_offColor.R + (_onColor.R - _offColor.R) * blend),
            (int)(_offColor.G + (_onColor.G - _offColor.G) * blend),
            (int)(_offColor.B + (_onColor.B - _offColor.B) * blend)
        );
        
        using var trackBrush = new SolidBrush(trackColor);
        g.FillEllipse(trackBrush, 0, 0, Height, Height);
        g.FillEllipse(trackBrush, Width - Height, 0, Height, Height);
        g.FillRectangle(trackBrush, Height / 2, 0, Width - Height, Height);
        
        // Thumb
        float thumbSize = Height - 6;
        g.FillEllipse(Brushes.White, _animationPos, 3, thumbSize, thumbSize);
    }
}
```

---

## ขั้นตอนที่ 275-280: Dashboard UI

```csharp
// Dashboard.cs - Complete Dashboard Application
using System;
using System.Collections.Generic;
using System.Drawing;
using System.Linq;
using System.Windows.Forms;

namespace Dashboard;

public class DashboardForm : Form
{
    private Panel _sidebar = null!, _content = null!;
    private Label _lblPage = null!;
    
    public DashboardForm()
    {
        Text = "Dashboard";
        Size = new Size(1100, 700);
        StartPosition = FormStartPosition.CenterScreen;
        MinimumSize = new Size(800, 500);
        BackColor = Color.White;
        
        BuildSidebar();
        BuildContent();
        ShowHomePage();
    }
    
    private void BuildSidebar()
    {
        _sidebar = new Panel
        {
            Dock = DockStyle.Left,
            Width = 220,
            BackColor = Color.FromArgb(30, 33, 48),
            Padding = new Padding(0, 20, 0, 0)
        };
        
        // Logo
        var logo = new Label
        {
            Text = "📊 Dashboard",
            Font = new Font("Segoe UI", 14, FontStyle.Bold),
            ForeColor = Color.White,
            Dock = DockStyle.Top,
            Height = 50,
            TextAlign = ContentAlignment.MiddleCenter
        };
        _sidebar.Controls.Add(logo);
        
        // Nav items
        var navItems = new[]
        {
            ("🏠", "หน้าหลัก", (Action)ShowHomePage),
            ("📦", "สินค้า", ShowProductsPage),
            ("👥", "ลูกค้า", ShowCustomersPage),
            ("📈", "รายงาน", ShowReportsPage),
            ("⚙️", "ตั้งค่า", ShowSettingsPage),
        };
        
        var navContainer = new FlowLayoutPanel
        {
            Dock = DockStyle.Top,
            FlowDirection = FlowDirection.TopDown,
            WrapContents = false,
            AutoSize = true,
            Padding = new Padding(10, 10, 10, 0)
        };
        
        foreach (var (icon, text, action) in navItems)
        {
            var btn = new NavButton(icon, text);
            btn.Click += (s,e) =>
            {
                action();
                foreach (NavButton nb in navContainer.Controls.OfType<NavButton>())
                    nb.IsActive = false;
                btn.IsActive = true;
            };
            navContainer.Controls.Add(btn);
        }
        
        _sidebar.Controls.Add(navContainer);
        Controls.Add(_sidebar);
    }
    
    private void BuildContent()
    {
        _content = new Panel
        {
            Dock = DockStyle.Fill,
            BackColor = Color.FromArgb(245, 247, 250),
            Padding = new Padding(20)
        };
        Controls.Add(_content);
    }
    
    private void ShowPage(Control page)
    {
        _content.Controls.Clear();
        page.Dock = DockStyle.Fill;
        _content.Controls.Add(page);
    }
    
    private void ShowHomePage() => ShowPage(new HomePage());
    private void ShowProductsPage() => ShowPage(CreateSimplePage("📦 สินค้า", "Product Management Page"));
    private void ShowCustomersPage() => ShowPage(CreateSimplePage("👥 ลูกค้า", "Customer Management Page"));
    private void ShowReportsPage() => ShowPage(CreateSimplePage("📈 รายงาน", "Reports Page"));
    private void ShowSettingsPage() => ShowPage(CreateSimplePage("⚙️ ตั้งค่า", "Settings Page"));
    
    private static Panel CreateSimplePage(string title, string subtitle)
    {
        var p = new Panel { BackColor = Color.Transparent };
        p.Controls.Add(new Label { Text = title, Font = new Font("Segoe UI", 18, FontStyle.Bold), Location = new Point(0, 0), AutoSize = true });
        p.Controls.Add(new Label { Text = subtitle, Font = new Font("Segoe UI", 11), ForeColor = Color.Gray, Location = new Point(0, 40), AutoSize = true });
        return p;
    }
}

// Home Page with KPI cards
class HomePage : Panel
{
    public HomePage()
    {
        BackColor = Color.Transparent;
        
        var title = new Label { Text = "🏠 ภาพรวมธุรกิจ", Font = new Font("Segoe UI", 18, FontStyle.Bold), Location = new Point(0, 0), AutoSize = true };
        Controls.Add(title);
        
        // KPI Cards
        var cards = new FlowLayoutPanel
        {
            Location = new Point(0, 50),
            Size = new Size(900, 120),
            FlowDirection = FlowDirection.LeftToRight,
            WrapContents = false
        };
        
        cards.Controls.Add(new KpiCard("💰 ยอดขาย", "฿1,250,400", "+12%", Color.FromArgb(46, 204, 113)));
        cards.Controls.Add(new KpiCard("📦 คำสั่งซื้อ", "342", "+8%", Color.FromArgb(52, 152, 219)));
        cards.Controls.Add(new KpiCard("👥 ลูกค้าใหม่", "128", "+23%", Color.FromArgb(155, 89, 182)));
        cards.Controls.Add(new KpiCard("⭐ Rating", "4.8/5.0", "+0.2", Color.FromArgb(243, 156, 18)));
        
        Controls.Add(cards);
        Controls.Add(title);
    }
}

// KPI Card
class KpiCard : Panel
{
    public KpiCard(string title, string value, string change, Color accentColor)
    {
        Size = new Size(205, 105);
        BackColor = Color.White;
        Margin = new Padding(0, 0, 15, 0);
        
        // Accent bar
        var bar = new Panel { Dock = DockStyle.Left, Width = 5, BackColor = accentColor };
        
        var lblTitle = new Label { Text = title, Font = new Font("Segoe UI", 9, FontStyle.Bold), ForeColor = Color.Gray, Location = new Point(15, 12), AutoSize = true };
        var lblValue = new Label { Text = value, Font = new Font("Segoe UI", 18, FontStyle.Bold), ForeColor = Color.FromArgb(40, 40, 40), Location = new Point(15, 35), AutoSize = true };
        var lblChange = new Label
        {
            Text = change,
            Font = new Font("Segoe UI", 9),
            ForeColor = change.StartsWith("+") ? Color.Green : Color.Red,
            Location = new Point(15, 75),
            AutoSize = true
        };
        
        Controls.AddRange(new Control[] { bar, lblTitle, lblValue, lblChange });
    }
}

// Nav Button
class NavButton : Control
{
    private readonly string _icon, _text;
    private bool _isActive;
    
    public bool IsActive
    {
        get => _isActive;
        set { _isActive = value; Invalidate(); }
    }
    
    public NavButton(string icon, string text)
    {
        _icon = icon;
        _text = text;
        Size = new Size(200, 42);
        Margin = new Padding(0, 2, 0, 2);
        Cursor = Cursors.Hand;
        DoubleBuffered = true;
        SetStyle(ControlStyles.AllPaintingInWmPaint | ControlStyles.OptimizedDoubleBuffer, true);
        
        MouseEnter += (s,e) => Invalidate();
        MouseLeave += (s,e) => Invalidate();
    }
    
    protected override void OnPaint(PaintEventArgs e)
    {
        var g = e.Graphics;
        bool hovered = ClientRectangle.Contains(PointToClient(MousePosition));
        
        if (_isActive)
            g.FillRectangle(new SolidBrush(Color.FromArgb(60, 255, 255, 255)), ClientRectangle);
        else if (hovered)
            g.FillRectangle(new SolidBrush(Color.FromArgb(30, 255, 255, 255)), ClientRectangle);
        
        if (_isActive)
        {
            g.FillRectangle(new SolidBrush(Color.FromArgb(52, 152, 219)), new Rectangle(0, 8, 4, Height - 16));
        }
        
        g.DrawString(_icon, new Font("Segoe UI Emoji", 14), Brushes.White, new RectangleF(12, 8, 28, 26));
        g.DrawString(_text, new Font("Segoe UI", 10, _isActive ? FontStyle.Bold : FontStyle.Regular),
            new SolidBrush(_isActive ? Color.White : Color.Silver), new RectangleF(44, 11, 140, 22));
    }
}
```

---

## 📝 สรุป Part 28

| Custom Control | ใช้เมื่อ |
|---------------|---------|
| ModernButton | Button สวยงาม |
| CircularProgress | แสดงความคืบหน้าแบบวงกลม |
| RatingControl | ให้คะแนนดาว |
| ToggleSwitch | เปิด/ปิด |
| UserControl | Component ที่ใช้ซ้ำ |

---

**ก่อนหน้า → [Part 27: WinForms MDI](part27-winforms-mdi.md)**  
**ต่อไป → [Part 29: WinForms Database Integration](part29-winforms-database.md)**
