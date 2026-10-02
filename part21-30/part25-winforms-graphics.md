# Part 25: WinForms Graphics & Drawing
## ขั้นตอนที่ 241-250: การวาดกราฟิก

---

## 🎯 เป้าหมายของ Part นี้
- GDI+ basics: Graphics, Pen, Brush
- วาดรูปทรงพื้นฐาน: Line, Rectangle, Ellipse, Polygon
- Font rendering
- Double buffering เพื่อป้องกัน flicker
- Animation ด้วย Timer
- Custom Paint control
- โปรแกรม Drawing Application

---

## ขั้นตอนที่ 241: Graphics basics

```csharp
// Paint event - จุดเริ่มต้นของ GDI+
private void Form_Paint(object? sender, PaintEventArgs e)
{
    Graphics g = e.Graphics;
    g.SmoothingMode = System.Drawing.Drawing2D.SmoothingMode.AntiAlias;
    
    // Pen - เส้น
    using var penBlue = new Pen(Color.Blue, 2);
    using var penDash = new Pen(Color.Red, 1) { DashStyle = System.Drawing.Drawing2D.DashStyle.Dash };
    
    g.DrawLine(penBlue, 10, 10, 200, 10);
    g.DrawLine(penDash, 10, 30, 200, 30);
    
    // Rectangle
    g.DrawRectangle(penBlue, 20, 50, 150, 80);
    g.FillRectangle(Brushes.LightSkyBlue, 21, 51, 149, 79);
    
    // Ellipse (วงกลม/วงรี)
    g.DrawEllipse(new Pen(Color.Green, 2), 200, 50, 100, 80);
    g.FillEllipse(new SolidBrush(Color.FromArgb(100, Color.Green)), 200, 50, 100, 80);
    
    // Polygon (สามเหลี่ยม)
    Point[] triangle = { new(300, 160), new(350, 60), new(400, 160) };
    g.DrawPolygon(new Pen(Color.Purple, 2), triangle);
    g.FillPolygon(new SolidBrush(Color.FromArgb(80, Color.Purple)), triangle);
    
    // Arc
    g.DrawArc(new Pen(Color.Orange, 3), 20, 160, 100, 100, 0, 270);
    
    // Bezier curve
    g.DrawBezier(new Pen(Color.Teal, 2),
        new Point(10, 280), new Point(100, 200),
        new Point(200, 360), new Point(300, 280));
}
```

---

## ขั้นตอนที่ 242: Brushes

```csharp
private void DrawBrushes(Graphics g)
{
    // SolidBrush
    g.FillRectangle(new SolidBrush(Color.Coral), 10, 10, 100, 60);
    
    // LinearGradientBrush
    var gradBrush = new System.Drawing.Drawing2D.LinearGradientBrush(
        new Rectangle(120, 10, 100, 60),
        Color.Blue, Color.LightCyan,
        System.Drawing.Drawing2D.LinearGradientMode.Horizontal);
    g.FillRectangle(gradBrush, 120, 10, 100, 60);
    
    // Vertical gradient
    var vertGrad = new System.Drawing.Drawing2D.LinearGradientBrush(
        new Rectangle(230, 10, 100, 60),
        Color.DarkGreen, Color.LightGreen,
        System.Drawing.Drawing2D.LinearGradientMode.Vertical);
    g.FillRectangle(vertGrad, 230, 10, 100, 60);
    
    // HatchBrush (ลายแฮทช์)
    var hatchBrush = new System.Drawing.Drawing2D.HatchBrush(
        System.Drawing.Drawing2D.HatchStyle.DiagonalCross,
        Color.Red, Color.White);
    g.FillRectangle(hatchBrush, 340, 10, 100, 60);
    
    // TextureBrush (รูปภาพ)
    // var texBrush = new TextureBrush(Image.FromFile("texture.jpg"));
    // g.FillRectangle(texBrush, 10, 80, 100, 60);
}
```

---

## ขั้นตอนที่ 243: Text Rendering

```csharp
private void DrawText(Graphics g)
{
    // Basic text
    g.DrawString("Hello, World!", new Font("Segoe UI", 16), Brushes.Black, 10, 10);
    
    // With StringFormat
    var format = new StringFormat
    {
        Alignment = StringAlignment.Center,
        LineAlignment = StringAlignment.Center,
        FormatFlags = StringFormatFlags.NoWrap
    };
    
    var rect = new RectangleF(50, 60, 300, 50);
    g.DrawRectangle(Pens.Gray, 50, 60, 300, 50);
    g.DrawString("Centered Text", new Font("Arial", 14, FontStyle.Bold), 
        Brushes.DarkBlue, rect, format);
    
    // Right-aligned
    format.Alignment = StringAlignment.Far;
    g.DrawString("Right-aligned", new Font("Segoe UI", 11), 
        Brushes.DarkGreen, new RectangleF(50, 130, 300, 30), format);
    
    // Measure text
    var size = g.MeasureString("Hello", new Font("Segoe UI", 12));
    Console.WriteLine($"Text size: {size.Width:F0} x {size.Height:F0}");
    
    // Anti-aliased text
    g.TextRenderingHint = System.Drawing.Text.TextRenderingHint.AntiAlias;
    g.DrawString("Smooth Text", new Font("Segoe UI", 20), Brushes.Navy, 10, 200);
    
    // Shadow effect
    g.DrawString("Shadow", new Font("Segoe UI", 20, FontStyle.Bold), 
        new SolidBrush(Color.FromArgb(80, 0, 0, 0)), 52, 252);
    g.DrawString("Shadow", new Font("Segoe UI", 20, FontStyle.Bold), 
        Brushes.DodgerBlue, 50, 250);
}
```

---

## ขั้นตอนที่ 244: Double Buffering

```csharp
// Double buffering ป้องกัน flicker
public class DoubleBufferedPanel : Panel
{
    public DoubleBufferedPanel()
    {
        // Enable double buffering
        DoubleBuffered = true;
        SetStyle(ControlStyles.AllPaintingInWmPaint | 
                 ControlStyles.UserPaint | 
                 ControlStyles.OptimizedDoubleBuffer, true);
        UpdateStyles();
    }
}

// Form-level double buffering
public class AnimatedForm : Form
{
    private System.Windows.Forms.Timer _timer = null!;
    private float _angle = 0;
    private Bitmap? _buffer;
    
    public AnimatedForm()
    {
        DoubleBuffered = true;
        Size = new Size(600, 500);
        
        _timer = new System.Windows.Forms.Timer { Interval = 16 }; // ~60 FPS
        _timer.Tick += (s,e) =>
        {
            _angle += 2f;
            if (_angle >= 360) _angle = 0;
            Invalidate();
        };
        _timer.Start();
        
        Paint += OnPaint;
    }
    
    private void OnPaint(object? sender, PaintEventArgs e)
    {
        var g = e.Graphics;
        g.SmoothingMode = System.Drawing.Drawing2D.SmoothingMode.AntiAlias;
        g.Clear(Color.Black);
        
        DrawSpinner(g, ClientSize.Width / 2, ClientSize.Height / 2, 150, _angle);
        DrawClock(g, 80, 80, 60);
    }
    
    private void DrawSpinner(Graphics g, int cx, int cy, int radius, float angle)
    {
        int numDots = 12;
        for (int i = 0; i < numDots; i++)
        {
            float a = (float)(angle + i * 360.0 / numDots) * (float)Math.PI / 180f;
            float x = cx + (float)Math.Cos(a) * radius;
            float y = cy + (float)Math.Sin(a) * radius;
            
            int alpha = (int)(255 * (i + 1) / numDots);
            int size = 8 + i / 2;
            
            g.FillEllipse(new SolidBrush(Color.FromArgb(alpha, Color.DeepSkyBlue)),
                x - size/2, y - size/2, size, size);
        }
    }
    
    private void DrawClock(Graphics g, int cx, int cy, int radius)
    {
        // Clock face
        g.FillEllipse(new SolidBrush(Color.FromArgb(40, 255, 255, 255)), 
            cx - radius, cy - radius, radius * 2, radius * 2);
        g.DrawEllipse(new Pen(Color.White, 2), cx - radius, cy - radius, radius * 2, radius * 2);
        
        // Hour markers
        for (int h = 0; h < 12; h++)
        {
            double a = h * 30 * Math.PI / 180 - Math.PI / 2;
            int x1 = (int)(cx + Math.Cos(a) * (radius - 8));
            int y1 = (int)(cy + Math.Sin(a) * (radius - 8));
            int x2 = (int)(cx + Math.Cos(a) * (radius - 3));
            int y2 = (int)(cy + Math.Sin(a) * (radius - 3));
            g.DrawLine(new Pen(Color.White, 2), x1, y1, x2, y2);
        }
        
        // Hands
        var now = DateTime.Now;
        DrawHand(g, cx, cy, radius * 0.5f, 
            (now.Hour % 12 + now.Minute / 60f) * 30 - 90, 3, Color.White);
        DrawHand(g, cx, cy, radius * 0.75f, 
            now.Minute * 6 - 90, 2, Color.LightGray);
        DrawHand(g, cx, cy, radius * 0.85f, 
            now.Second * 6 - 90, 1, Color.Red);
    }
    
    private void DrawHand(Graphics g, int cx, int cy, float len, float angleDeg, float width, Color color)
    {
        double a = angleDeg * Math.PI / 180;
        int x = (int)(cx + Math.Cos(a) * len);
        int y = (int)(cy + Math.Sin(a) * len);
        g.DrawLine(new Pen(color, width), cx, cy, x, y);
    }
    
    protected override void OnFormClosed(FormClosedEventArgs e)
    {
        _timer.Stop();
        _buffer?.Dispose();
        base.OnFormClosed(e);
    }
}
```

---

## ขั้นตอนที่ 245-250: โปรแกรม Drawing Application

```csharp
// DrawingApp.cs - Paint-like application
using System;
using System.Collections.Generic;
using System.Drawing;
using System.Drawing.Drawing2D;
using System.Windows.Forms;

namespace DrawingApp;

enum DrawTool { Pencil, Line, Rectangle, Ellipse, FillRect, FillEllipse, Eraser, Text }

public class DrawingForm : Form
{
    private Panel _canvas = null!;
    private Bitmap _bitmap = null!;
    private Graphics _g = null!;
    
    // Drawing state
    private DrawTool _tool = DrawTool.Pencil;
    private Color _foreColor = Color.Black;
    private Color _backColor = Color.White;
    private int _lineWidth = 2;
    private Point _startPoint;
    private bool _drawing = false;
    private Bitmap? _snapshot;
    
    // UI
    private Panel _toolPanel = null!;
    private Panel _colorPanel = null!;
    private Label _lblColor = null!;
    private TrackBar _trackWidth = null!;
    
    // History for undo
    private readonly Stack<Bitmap> _history = new();
    
    public DrawingForm()
    {
        InitializeUI();
    }
    
    private void InitializeUI()
    {
        Text = "Drawing App";
        Size = new Size(1000, 700);
        StartPosition = FormStartPosition.CenterScreen;
        
        // Tool panel (left)
        _toolPanel = new Panel
        {
            Dock = DockStyle.Left,
            Width = 60,
            BackColor = Color.FromArgb(45, 45, 48),
            Padding = new Padding(5)
        };
        
        var tools = new[] 
        { 
            ("✏️", DrawTool.Pencil, "ดินสอ"),
            ("📏", DrawTool.Line, "เส้น"),
            ("▭", DrawTool.Rectangle, "สี่เหลี่ยม"),
            ("⬭", DrawTool.Ellipse, "วงรี"),
            ("■", DrawTool.FillRect, "สี่เหลี่ยมทึบ"),
            ("●", DrawTool.FillEllipse, "วงรีทึบ"),
            ("◻", DrawTool.Eraser, "ยางลบ"),
            ("T", DrawTool.Text, "ข้อความ")
        };
        
        int y = 10;
        foreach (var (icon, t, tip) in tools)
        {
            var btn = new Button
            {
                Text = icon,
                Size = new Size(46, 40),
                Location = new Point(7, y),
                Tag = t,
                BackColor = t == DrawTool.Pencil ? Color.FromArgb(0, 122, 204) : Color.FromArgb(62, 62, 66),
                ForeColor = Color.White,
                FlatStyle = FlatStyle.Flat,
                Font = new Font("Segoe UI Emoji", 14),
                Cursor = Cursors.Hand,
                TabStop = false
            };
            btn.FlatAppearance.BorderSize = 0;
            var toolTip = new ToolTip();
            toolTip.SetToolTip(btn, tip);
            btn.Click += ToolBtn_Click;
            _toolPanel.Controls.Add(btn);
            y += 50;
        }
        
        // Color panel (bottom of tool panel)
        y += 20;
        _lblColor = new Label
        {
            Size = new Size(46, 30),
            Location = new Point(7, y),
            BackColor = _foreColor,
            BorderStyle = BorderStyle.FixedSingle,
            Cursor = Cursors.Hand
        };
        _lblColor.Click += (s,e) =>
        {
            using var dlg = new ColorDialog { Color = _foreColor };
            if (dlg.ShowDialog() == DialogResult.OK)
            {
                _foreColor = dlg.Color;
                _lblColor.BackColor = _foreColor;
            }
        };
        _toolPanel.Controls.Add(_lblColor);
        
        y += 40;
        _trackWidth = new TrackBar
        {
            Location = new Point(5, y),
            Size = new Size(50, 100),
            Orientation = Orientation.Vertical,
            Minimum = 1, Maximum = 30, Value = 2,
            TickFrequency = 5,
            BackColor = Color.FromArgb(45, 45, 48)
        };
        _trackWidth.ValueChanged += (s,e) => _lineWidth = _trackWidth.Value;
        _toolPanel.Controls.Add(_trackWidth);
        
        // Canvas
        _canvas = new Panel
        {
            Dock = DockStyle.Fill,
            BackColor = Color.White,
            Cursor = Cursors.Cross
        };
        _canvas.Resize += Canvas_Resize;
        _canvas.Paint += Canvas_Paint;
        _canvas.MouseDown += Canvas_MouseDown;
        _canvas.MouseMove += Canvas_MouseMove;
        _canvas.MouseUp += Canvas_MouseUp;
        
        // Menu
        var menu = new MenuStrip();
        var fileMenu = new ToolStripMenuItem("File");
        fileMenu.DropDownItems.Add("New", null, (s,e) => NewCanvas());
        fileMenu.DropDownItems.Add("Save...", null, SaveImage);
        fileMenu.DropDownItems.Add("Open...", null, OpenImage);
        fileMenu.DropDownItems.Add(new ToolStripSeparator());
        fileMenu.DropDownItems.Add("Exit", null, (s,e) => Close());
        
        var editMenu = new ToolStripMenuItem("Edit");
        editMenu.DropDownItems.Add(new ToolStripMenuItem("Undo", null, (s,e) => Undo(), Keys.Control | Keys.Z));
        editMenu.DropDownItems.Add(new ToolStripMenuItem("Clear All", null, (s,e) => ClearAll()));
        
        menu.Items.AddRange(new ToolStripItem[] { fileMenu, editMenu });
        MainMenuStrip = menu;
        
        // Status
        var status = new StatusStrip();
        var lblCoords = new ToolStripStatusLabel("(0, 0)") { Spring = true };
        status.Items.Add(lblCoords);
        _canvas.MouseMove += (s,e) => lblCoords.Text = $"({e.X}, {e.Y})";
        
        Controls.Add(_canvas);
        Controls.Add(_toolPanel);
        Controls.Add(menu);
        Controls.Add(status);
        
        Load += (s,e) => InitCanvas();
    }
    
    private void InitCanvas()
    {
        if (_canvas.Width <= 0 || _canvas.Height <= 0) return;
        _bitmap?.Dispose();
        _bitmap = new Bitmap(_canvas.Width, _canvas.Height);
        _g = Graphics.FromImage(_bitmap);
        _g.SmoothingMode = SmoothingMode.AntiAlias;
        _g.Clear(Color.White);
        _canvas.Invalidate();
    }
    
    private void Canvas_Resize(object? s, EventArgs e)
    {
        if (_canvas.Width <= 0 || _canvas.Height <= 0) return;
        
        var newBitmap = new Bitmap(_canvas.Width, _canvas.Height);
        using var ng = Graphics.FromImage(newBitmap);
        ng.Clear(Color.White);
        if (_bitmap != null) ng.DrawImage(_bitmap, 0, 0);
        
        _g?.Dispose();
        _bitmap?.Dispose();
        _bitmap = newBitmap;
        _g = Graphics.FromImage(_bitmap);
        _g.SmoothingMode = SmoothingMode.AntiAlias;
        _canvas.Invalidate();
    }
    
    private void Canvas_Paint(object? s, PaintEventArgs e)
    {
        if (_bitmap != null) e.Graphics.DrawImage(_bitmap, 0, 0);
    }
    
    private void Canvas_MouseDown(object? s, MouseEventArgs e)
    {
        if (e.Button != MouseButtons.Left) return;
        
        // Save for undo
        _history.Push((Bitmap)_bitmap.Clone());
        if (_history.Count > 30) // Limit history
        {
            var old = _history.ToArray();
            _history.Clear();
            for (int i = 0; i < 20; i++) _history.Push(old[i]);
        }
        
        _drawing = true;
        _startPoint = e.Location;
        _snapshot = (Bitmap)_bitmap.Clone();
        
        if (_tool == DrawTool.Pencil || _tool == DrawTool.Eraser)
        {
            var color = _tool == DrawTool.Eraser ? Color.White : _foreColor;
            _g.DrawEllipse(new Pen(color, _lineWidth), 
                e.X - _lineWidth/2, e.Y - _lineWidth/2, _lineWidth, _lineWidth);
            _canvas.Invalidate();
        }
        else if (_tool == DrawTool.Text)
        {
            string? text = Microsoft.VisualBasic.Interaction.InputBox("Enter text:", "Add Text");
            if (!string.IsNullOrEmpty(text))
            {
                _g.DrawString(text, new Font("Segoe UI", 14), new SolidBrush(_foreColor), e.Location);
                _canvas.Invalidate();
            }
            _drawing = false;
        }
    }
    
    private void Canvas_MouseMove(object? s, MouseEventArgs e)
    {
        if (!_drawing) return;
        
        if (_tool == DrawTool.Pencil || _tool == DrawTool.Eraser)
        {
            var color = _tool == DrawTool.Eraser ? Color.White : _foreColor;
            var pen = new Pen(color, _lineWidth) { StartCap = LineCap.Round, EndCap = LineCap.Round };
            _g.DrawLine(pen, _startPoint, e.Location);
            _startPoint = e.Location;
            _canvas.Invalidate();
            return;
        }
        
        // For shape tools: restore snapshot and redraw
        if (_snapshot != null)
        {
            using var ng = Graphics.FromImage(_bitmap);
            ng.DrawImage(_snapshot, 0, 0);
        }
        
        using var g = Graphics.FromImage(_bitmap);
        g.SmoothingMode = SmoothingMode.AntiAlias;
        var rect = GetRect(_startPoint, e.Location);
        var pen = new Pen(_foreColor, _lineWidth);
        var brush = new SolidBrush(_foreColor);
        
        switch (_tool)
        {
            case DrawTool.Line:
                g.DrawLine(pen, _startPoint, e.Location);
                break;
            case DrawTool.Rectangle:
                g.DrawRectangle(pen, rect);
                break;
            case DrawTool.Ellipse:
                g.DrawEllipse(pen, rect);
                break;
            case DrawTool.FillRect:
                g.FillRectangle(brush, rect);
                break;
            case DrawTool.FillEllipse:
                g.FillEllipse(brush, rect);
                break;
        }
        
        _canvas.Invalidate();
    }
    
    private void Canvas_MouseUp(object? s, MouseEventArgs e)
    {
        _drawing = false;
        _snapshot?.Dispose();
        _snapshot = null;
    }
    
    private Rectangle GetRect(Point p1, Point p2) => new Rectangle(
        Math.Min(p1.X, p2.X), Math.Min(p1.Y, p2.Y),
        Math.Abs(p2.X - p1.X), Math.Abs(p2.Y - p1.Y));
    
    private void ToolBtn_Click(object? s, EventArgs e)
    {
        _tool = (DrawTool)((Button)s!).Tag!;
        foreach (Button btn in _toolPanel.Controls.OfType<Button>())
            btn.BackColor = (DrawTool)btn.Tag! == _tool 
                ? Color.FromArgb(0, 122, 204) 
                : Color.FromArgb(62, 62, 66);
    }
    
    private void Undo()
    {
        if (_history.Count == 0) return;
        var prev = _history.Pop();
        using var ng = Graphics.FromImage(_bitmap);
        ng.DrawImage(prev, 0, 0);
        prev.Dispose();
        _canvas.Invalidate();
    }
    
    private void ClearAll()
    {
        _history.Push((Bitmap)_bitmap.Clone());
        _g.Clear(Color.White);
        _canvas.Invalidate();
    }
    
    private void NewCanvas()
    {
        if (MessageBox.Show("สร้างใหม่? การเปลี่ยนแปลงจะหาย", "Confirm", 
            MessageBoxButtons.YesNo, MessageBoxIcon.Question) == DialogResult.Yes)
        {
            _history.Clear();
            _g.Clear(Color.White);
            _canvas.Invalidate();
        }
    }
    
    private void SaveImage(object? s, EventArgs e)
    {
        using var dlg = new SaveFileDialog { Filter = "PNG|*.png|JPEG|*.jpg|BMP|*.bmp", FileName = "drawing" };
        if (dlg.ShowDialog() != DialogResult.OK) return;
        
        var fmt = dlg.FilterIndex switch
        {
            2 => System.Drawing.Imaging.ImageFormat.Jpeg,
            3 => System.Drawing.Imaging.ImageFormat.Bmp,
            _ => System.Drawing.Imaging.ImageFormat.Png
        };
        _bitmap.Save(dlg.FileName, fmt);
        MessageBox.Show("บันทึกสำเร็จ");
    }
    
    private void OpenImage(object? s, EventArgs e)
    {
        using var dlg = new OpenFileDialog { Filter = "Images|*.png;*.jpg;*.bmp;*.gif" };
        if (dlg.ShowDialog() != DialogResult.OK) return;
        
        var img = Image.FromFile(dlg.FileName);
        _g.DrawImage(img, 0, 0, Math.Min(img.Width, _bitmap.Width), Math.Min(img.Height, _bitmap.Height));
        _canvas.Invalidate();
    }
}
```

---

## 📝 สรุป Part 25

| หัวข้อ | Key Points |
|--------|-----------|
| Graphics | Engine สำหรับ drawing |
| Pen | เส้น (สี, ขนาด, style) |
| SolidBrush | เติมสีทึบ |
| LinearGradientBrush | Gradient |
| DoubleBuffered | ป้องกัน flicker |
| Timer + Invalidate | Animation loop |
| Bitmap + Graphics.FromImage | Off-screen drawing |
| Stack<Bitmap> | Undo history |

---

**ก่อนหน้า → [Part 24: WinForms Dialogs](part24-winforms-dialogs.md)**  
**ต่อไป → [Part 26: WinForms Timer & Animation](part26-winforms-timer.md)**
