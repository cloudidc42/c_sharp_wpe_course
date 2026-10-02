# Part 78: WinForms Advanced Development

## ขั้นตอนที่ 771-780: การพัฒนา WinForms ขั้นสูง

---

## Step 771: WinForms Architecture Review & GDI+ Basics

### โครงสร้างของ WinForms และการทำงานเบื้องต้น

WinForms (Windows Forms) คือ UI framework ของ .NET ที่สร้างขึ้นบน Win32 API
แต่ละ Form และ Control คือ window ที่มี Handle (HWND) เป็นของตัวเอง

#### Message Loop และ Message Pump

```csharp
// โปรแกรม WinForms เริ่มต้นด้วย Application.Run()
// ซึ่งสร้าง Message Loop ขึ้นมาเพื่อรับ Windows Messages
static class Program
{
    [STAThread]
    static void Main()
    {
        Application.EnableVisualStyles();
        Application.SetCompatibleTextRenderingDefault(false);
        Application.SetHighDpiMode(HighDpiMode.SystemAware);
        Application.Run(new MainForm());
    }
}
```

#### การทำงานของ Control Hierarchy

```csharp
// WinForms ใช้ลำดับชั้นของ Control
// Form -> Panel -> Button/Label/TextBox ...
public class ArchitectureDemo : Form
{
    public ArchitectureDemo()
    {
        this.Text = "Architecture Demo";
        this.Size = new Size(600, 400);

        // Controls collection เก็บ child controls ทั้งหมด
        var panel = new Panel
        {
            Dock = DockStyle.Fill,
            BackColor = Color.LightGray
        };

        var label = new Label
        {
            Text = "WinForms Architecture",
            Location = new Point(10, 10),
            AutoSize = true
        };

        panel.Controls.Add(label);
        this.Controls.Add(panel);

        // Handle คือตัวแทน Win32 Window
        this.Load += (s, e) =>
        {
            Console.WriteLine($"Form Handle: {this.Handle}");
            Console.WriteLine($"Panel Handle: {panel.Handle}");
        };
    }
}
```

#### GDI+ Basics - ระบบกราฟิกของ WinForms

GDI+ (Graphics Device Interface Plus) คือ API สำหรับวาดกราฟิกใน WinForms
มันทำงานผ่าน `System.Drawing` namespace

```csharp
// GDI+ ทำงานผ่าน Graphics object ที่ได้จาก PaintEventArgs
public class GdiBasicsForm : Form
{
    public GdiBasicsForm()
    {
        this.Text = "GDI+ Basics";
        this.Size = new Size(800, 600);
        this.DoubleBuffered = true; // ป้องกัน flickering
    }

    protected override void OnPaint(PaintEventArgs e)
    {
        base.OnPaint(e);
        Graphics g = e.Graphics;

        // ตั้งค่า rendering quality
        g.SmoothingMode = System.Drawing.Drawing2D.SmoothingMode.AntiAlias;
        g.TextRenderingHint = System.Drawing.Text.TextRenderingHint.ClearTypeGridFit;

        // วาดพื้นหลัง
        g.Clear(Color.White);

        // วาดเส้น
        using var pen = new Pen(Color.Blue, 2);
        g.DrawLine(pen, 10, 10, 200, 10);

        // วาดสี่เหลี่ยม
        using var rectPen = new Pen(Color.Red, 1);
        g.DrawRectangle(rectPen, 10, 30, 150, 80);

        // วาดวงกลม
        using var ellipsePen = new Pen(Color.Green, 2);
        g.DrawEllipse(ellipsePen, 10, 130, 100, 60);

        // เขียนข้อความ
        using var font = new Font("Arial", 14, FontStyle.Bold);
        using var brush = new SolidBrush(Color.Black);
        g.DrawString("Hello GDI+", font, brush, 10, 210);
    }
}
```

#### Coordinate System ของ GDI+

```csharp
// GDI+ ใช้ระบบพิกัดที่ (0,0) อยู่ที่มุมซ้ายบน
// X เพิ่มไปทางขวา, Y เพิ่มลงล่าง
// หน่วยเป็น Pixels (ค่าเริ่มต้น)

public partial class CoordinateDemo : Form
{
    protected override void OnPaint(PaintEventArgs e)
    {
        base.OnPaint(e);
        var g = e.Graphics;

        // แสดง coordinate system
        var centerX = this.Width / 2;
        var centerY = this.Height / 2;

        // วาดแกน X
        g.DrawLine(Pens.Black, 0, centerY, this.Width, centerY);
        // วาดแกน Y
        g.DrawLine(Pens.Black, centerX, 0, centerX, this.Height);

        // Label
        g.DrawString("(0,0)", SystemFonts.DefaultFont, Brushes.Red, 5, 5);
        g.DrawString($"({this.Width},{this.Height})",
            SystemFonts.DefaultFont, Brushes.Red,
            this.Width - 80, this.Height - 20);
    }
}
```

---

## Step 772: Custom Controls (UserControl & Custom Painting)

### การสร้าง Custom Control

#### UserControl - การรวม Controls เข้าด้วยกัน

```csharp
// UserControl ใช้สำหรับสร้าง Control ที่ประกอบด้วย Controls หลายตัว
public class RatingControl : UserControl
{
    private int _maxStars = 5;
    private int _rating = 0;
    private List<PictureBox> _stars = new List<PictureBox>();

    // Property พร้อม event notification
    public int Rating
    {
        get => _rating;
        set
        {
            if (_rating != value && value >= 0 && value <= _maxStars)
            {
                _rating = value;
                UpdateStarDisplay();
                RatingChanged?.Invoke(this, EventArgs.Empty);
            }
        }
    }

    public event EventHandler? RatingChanged;

    public RatingControl()
    {
        InitializeStars();
    }

    private void InitializeStars()
    {
        this.Height = 30;
        this.Width = _maxStars * 35;

        for (int i = 0; i < _maxStars; i++)
        {
            var star = new PictureBox
            {
                Size = new Size(30, 30),
                Location = new Point(i * 35, 0),
                Cursor = Cursors.Hand,
                Tag = i + 1
            };

            int starIndex = i + 1;
            star.Click += (s, e) => Rating = starIndex;
            star.MouseEnter += (s, e) => HighlightStars((int)((Control)s!).Tag!);
            star.MouseLeave += (s, e) => UpdateStarDisplay();

            _stars.Add(star);
            this.Controls.Add(star);
        }

        UpdateStarDisplay();
    }

    private void HighlightStars(int count)
    {
        for (int i = 0; i < _stars.Count; i++)
        {
            _stars[i].BackColor = i < count ? Color.Gold : Color.LightGray;
        }
    }

    private void UpdateStarDisplay()
    {
        for (int i = 0; i < _stars.Count; i++)
        {
            _stars[i].BackColor = i < _rating ? Color.Gold : Color.LightGray;
        }
    }
}
```

#### Custom Control ด้วย OnPaint

```csharp
// Custom Control ที่วาดกราฟิกเองทั้งหมดผ่าน OnPaint
[ToolboxItem(true)]
public class CircularProgressBar : Control
{
    private int _value = 0;
    private int _maximum = 100;
    private Color _progressColor = Color.DodgerBlue;
    private Color _trackColor = Color.LightGray;
    private int _thickness = 8;

    public int Value
    {
        get => _value;
        set
        {
            _value = Math.Clamp(value, 0, _maximum);
            Invalidate(); // สั่งให้ repaint
        }
    }

    public int Maximum
    {
        get => _maximum;
        set { _maximum = value; Invalidate(); }
    }

    public Color ProgressColor
    {
        get => _progressColor;
        set { _progressColor = value; Invalidate(); }
    }

    public CircularProgressBar()
    {
        // ตั้งค่าสำคัญสำหรับ custom drawing
        this.SetStyle(
            ControlStyles.OptimizedDoubleBuffer |
            ControlStyles.AllPaintingInWmPaint |
            ControlStyles.UserPaint |
            ControlStyles.ResizeRedraw,
            true);
        this.Size = new Size(100, 100);
    }

    protected override void OnPaint(PaintEventArgs e)
    {
        base.OnPaint(e);
        var g = e.Graphics;
        g.SmoothingMode = System.Drawing.Drawing2D.SmoothingMode.AntiAlias;

        var rect = new Rectangle(
            _thickness, _thickness,
            Width - _thickness * 2,
            Height - _thickness * 2);

        // วาด track (วงกลมพื้นหลัง)
        using var trackPen = new Pen(_trackColor, _thickness);
        g.DrawEllipse(trackPen, rect);

        // วาด progress arc
        if (_value > 0)
        {
            float sweepAngle = 360f * _value / _maximum;
            using var progressPen = new Pen(_progressColor, _thickness);
            progressPen.StartCap = System.Drawing.Drawing2D.LineCap.Round;
            progressPen.EndCap = System.Drawing.Drawing2D.LineCap.Round;
            // เริ่มจากด้านบน (-90 องศา)
            g.DrawArc(progressPen, rect, -90, sweepAngle);
        }

        // เขียนเปอร์เซ็นต์ตรงกลาง
        string text = $"{_value}%";
        using var font = new Font("Arial", Width * 0.18f, FontStyle.Bold);
        var textSize = g.MeasureString(text, font);
        var textPoint = new PointF(
            (Width - textSize.Width) / 2,
            (Height - textSize.Height) / 2);

        using var textBrush = new SolidBrush(ForeColor);
        g.DrawString(text, font, textBrush, textPoint);
    }
}
```

#### Toggle Switch Custom Control

```csharp
public class ToggleSwitch : Control
{
    private bool _checked = false;
    private bool _isAnimating = false;
    private float _animPosition = 0f;
    private System.Windows.Forms.Timer _animTimer;

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

    public event EventHandler? CheckedChanged;

    public ToggleSwitch()
    {
        this.Size = new Size(60, 30);
        this.Cursor = Cursors.Hand;

        _animTimer = new System.Windows.Forms.Timer { Interval = 16 };
        _animTimer.Tick += AnimationTick;

        this.SetStyle(
            ControlStyles.OptimizedDoubleBuffer |
            ControlStyles.AllPaintingInWmPaint |
            ControlStyles.UserPaint, true);

        this.Click += (s, e) => Checked = !Checked;
    }

    private void StartAnimation()
    {
        _animPosition = _checked ? 0f : 1f;
        _isAnimating = true;
        _animTimer.Start();
    }

    private void AnimationTick(object? sender, EventArgs e)
    {
        float target = _checked ? 1f : 0f;
        _animPosition += (_checked ? 0.1f : -0.1f);
        _animPosition = Math.Clamp(_animPosition, 0f, 1f);

        if (Math.Abs(_animPosition - target) < 0.01f)
        {
            _animPosition = target;
            _animTimer.Stop();
            _isAnimating = false;
        }
        Invalidate();
    }

    protected override void OnPaint(PaintEventArgs e)
    {
        base.OnPaint(e);
        var g = e.Graphics;
        g.SmoothingMode = System.Drawing.Drawing2D.SmoothingMode.AntiAlias;

        // วาดพื้นหลัง
        var trackColor = ColorInterpolate(Color.LightGray, Color.MediumSeaGreen, _animPosition);
        using var trackBrush = new SolidBrush(trackColor);
        g.FillRoundedRectangle(trackBrush, 0, 0, Width, Height, Height / 2);

        // วาดปุ่ม (thumb)
        int thumbSize = Height - 4;
        float thumbX = 2 + _animPosition * (Width - thumbSize - 4);

        using var thumbBrush = new SolidBrush(Color.White);
        g.FillEllipse(thumbBrush, thumbX, 2, thumbSize, thumbSize);
    }

    private Color ColorInterpolate(Color from, Color to, float t)
    {
        return Color.FromArgb(
            (int)(from.A + (to.A - from.A) * t),
            (int)(from.R + (to.R - from.R) * t),
            (int)(from.G + (to.G - from.G) * t),
            (int)(from.B + (to.B - from.B) * t));
    }
}

// Extension method สำหรับวาดสี่เหลี่ยมมุมโค้ง
public static class GraphicsExtensions
{
    public static void FillRoundedRectangle(this Graphics g, Brush brush,
        float x, float y, float width, float height, float radius)
    {
        using var path = new System.Drawing.Drawing2D.GraphicsPath();
        path.AddArc(x, y, radius * 2, radius * 2, 180, 90);
        path.AddArc(x + width - radius * 2, y, radius * 2, radius * 2, 270, 90);
        path.AddArc(x + width - radius * 2, y + height - radius * 2, radius * 2, radius * 2, 0, 90);
        path.AddArc(x, y + height - radius * 2, radius * 2, radius * 2, 90, 90);
        path.CloseAllFigures();
        g.FillPath(brush, path);
    }
}
```

---

## Step 773: GDI+ Drawing - Graphics, Pen, Brush, Shapes, Text, Images

### การวาดกราฟิกด้วย GDI+

#### Pen และ Brush ประเภทต่างๆ

```csharp
public class GdiAdvancedForm : Form
{
    public GdiAdvancedForm()
    {
        this.Size = new Size(900, 700);
        this.DoubleBuffered = true;
        this.Text = "GDI+ Advanced Drawing";
    }

    protected override void OnPaint(PaintEventArgs e)
    {
        base.OnPaint(e);
        var g = e.Graphics;
        g.SmoothingMode = System.Drawing.Drawing2D.SmoothingMode.HighQuality;

        DemoPens(g);
        DemoBrushes(g);
        DemoShapes(g);
        DemoText(g);
        DemoTransform(g);
    }

    private void DemoPens(Graphics g)
    {
        int y = 20;
        g.DrawString("=== Pens ===", SystemFonts.DefaultFont, Brushes.Black, 10, y);

        // Solid Pen
        using var solidPen = new Pen(Color.Blue, 3);
        g.DrawLine(solidPen, 10, y + 20, 200, y + 20);
        g.DrawString("Solid Pen", SystemFonts.DefaultFont, Brushes.Black, 210, y + 12);

        // Dashed Pen
        using var dashPen = new Pen(Color.Red, 2);
        dashPen.DashStyle = System.Drawing.Drawing2D.DashStyle.Dash;
        g.DrawLine(dashPen, 10, y + 45, 200, y + 45);
        g.DrawString("Dash Pen", SystemFonts.DefaultFont, Brushes.Black, 210, y + 37);

        // DotDash Pen
        using var dotDashPen = new Pen(Color.Green, 2);
        dotDashPen.DashStyle = System.Drawing.Drawing2D.DashStyle.DashDot;
        g.DrawLine(dotDashPen, 10, y + 70, 200, y + 70);
        g.DrawString("DashDot Pen", SystemFonts.DefaultFont, Brushes.Black, 210, y + 62);

        // Custom dash pattern
        using var customPen = new Pen(Color.Purple, 2);
        customPen.DashStyle = System.Drawing.Drawing2D.DashStyle.Custom;
        customPen.DashPattern = new float[] { 5, 2, 1, 2 };
        g.DrawLine(customPen, 10, y + 95, 200, y + 95);
        g.DrawString("Custom Pattern", SystemFonts.DefaultFont, Brushes.Black, 210, y + 87);
    }

    private void DemoBrushes(Graphics g)
    {
        int startX = 350;
        int y = 20;
        g.DrawString("=== Brushes ===", SystemFonts.DefaultFont, Brushes.Black, startX, y);

        var rect = new Rectangle(startX, y + 20, 80, 50);

        // SolidBrush
        using (var solidBrush = new SolidBrush(Color.Coral))
            g.FillRectangle(solidBrush, rect);
        g.DrawString("Solid", SystemFonts.DefaultFont, Brushes.Black, startX, y + 75);

        rect.X += 100;
        // HatchBrush
        using (var hatchBrush = new System.Drawing.Drawing2D.HatchBrush(
            System.Drawing.Drawing2D.HatchStyle.CrossedDiagonal, Color.Navy, Color.LightBlue))
            g.FillRectangle(hatchBrush, rect);
        g.DrawString("Hatch", SystemFonts.DefaultFont, Brushes.Black, rect.X, y + 75);

        rect.X += 100;
        // LinearGradientBrush
        using (var gradBrush = new System.Drawing.Drawing2D.LinearGradientBrush(
            rect, Color.Orange, Color.Purple,
            System.Drawing.Drawing2D.LinearGradientMode.Horizontal))
            g.FillRectangle(gradBrush, rect);
        g.DrawString("Gradient", SystemFonts.DefaultFont, Brushes.Black, rect.X, y + 75);

        rect.X += 100;
        // PathGradientBrush
        var path = new System.Drawing.Drawing2D.GraphicsPath();
        path.AddEllipse(rect);
        using (var pathBrush = new System.Drawing.Drawing2D.PathGradientBrush(path))
        {
            pathBrush.CenterColor = Color.White;
            pathBrush.SurroundColors = new[] { Color.Red };
            g.FillEllipse(pathBrush, rect);
        }
        g.DrawString("PathGradient", SystemFonts.DefaultFont, Brushes.Black, rect.X, y + 75);
    }

    private void DemoShapes(Graphics g)
    {
        int y = 150;
        g.DrawString("=== Shapes ===", SystemFonts.DefaultFont, Brushes.Black, 10, y);
        y += 20;

        using var pen = new Pen(Color.DarkBlue, 2);
        using var fillBrush = new SolidBrush(Color.LightSkyBlue);

        // Rectangle
        g.FillRectangle(fillBrush, 10, y, 80, 60);
        g.DrawRectangle(pen, 10, y, 80, 60);
        g.DrawString("Rect", SystemFonts.DefaultFont, Brushes.Black, 30, y + 65);

        // Ellipse
        g.FillEllipse(fillBrush, 110, y, 80, 60);
        g.DrawEllipse(pen, 110, y, 80, 60);
        g.DrawString("Ellipse", SystemFonts.DefaultFont, Brushes.Black, 125, y + 65);

        // Polygon (Triangle)
        var trianglePoints = new Point[]
        {
            new Point(250, y + 60),
            new Point(290, y),
            new Point(330, y + 60)
        };
        g.FillPolygon(fillBrush, trianglePoints);
        g.DrawPolygon(pen, trianglePoints);
        g.DrawString("Polygon", SystemFonts.DefaultFont, Brushes.Black, 265, y + 65);

        // Arc
        g.DrawArc(pen, 360, y, 80, 80, -45, 270);
        g.DrawString("Arc", SystemFonts.DefaultFont, Brushes.Black, 390, y + 65);

        // Bezier Curve
        g.DrawBezier(pen,
            new Point(460, y + 60),   // start
            new Point(490, y),         // control 1
            new Point(530, y + 80),    // control 2
            new Point(560, y + 20));   // end
        g.DrawString("Bezier", SystemFonts.DefaultFont, Brushes.Black, 490, y + 65);
    }

    private void DemoText(Graphics g)
    {
        int y = 300;
        g.DrawString("=== Text Rendering ===", SystemFonts.DefaultFont, Brushes.Black, 10, y);
        y += 25;

        // ขนาดต่างๆ
        using var smallFont = new Font("Arial", 10);
        using var medFont = new Font("Arial", 16, FontStyle.Bold);
        using var largeFont = new Font("Arial", 24, FontStyle.Italic);

        g.DrawString("Small text (10pt)", smallFont, Brushes.Black, 10, y);
        y += 20;
        g.DrawString("Medium Bold (16pt)", medFont, Brushes.DarkBlue, 10, y);
        y += 30;
        g.DrawString("Large Italic (24pt)", largeFont, Brushes.DarkRed, 10, y);
        y += 40;

        // Text alignment
        var rect = new RectangleF(10, y, 300, 60);
        g.DrawRectangle(Pens.Gray, 10, y, 300, 60);

        var format = new StringFormat
        {
            Alignment = StringAlignment.Center,
            LineAlignment = StringAlignment.Center
        };
        g.DrawString("Centered Text\nIn a Rectangle", medFont, Brushes.DarkGreen, rect, format);
    }

    private void DemoTransform(Graphics g)
    {
        int y = 480;
        g.DrawString("=== Transform ===", SystemFonts.DefaultFont, Brushes.Black, 10, y);

        // บันทึก state ก่อน transform
        var state = g.Save();

        g.TranslateTransform(150, y + 60);
        g.RotateTransform(30);
        g.ScaleTransform(1.5f, 0.8f);

        g.FillRectangle(Brushes.LightGreen, -40, -20, 80, 40);
        g.DrawRectangle(Pens.DarkGreen, -40, -20, 80, 40);
        g.DrawString("Transformed!", SystemFonts.DefaultFont, Brushes.DarkGreen, -38, -10);

        // คืนค่า state เดิม
        g.Restore(state);
    }
}
```

#### การวาดรูปภาพด้วย GDI+

```csharp
public class ImageDrawingDemo : Form
{
    private Bitmap? _bitmap;

    public ImageDrawingDemo()
    {
        this.Size = new Size(800, 600);
        this.DoubleBuffered = true;

        // สร้าง bitmap ในหน่วยความจำ
        _bitmap = new Bitmap(400, 300);
        DrawToBitmap(_bitmap);
    }

    private void DrawToBitmap(Bitmap bmp)
    {
        using var g = Graphics.FromImage(bmp);
        g.SmoothingMode = System.Drawing.Drawing2D.SmoothingMode.AntiAlias;

        // วาด gradient background
        using var grad = new System.Drawing.Drawing2D.LinearGradientBrush(
            new Rectangle(0, 0, bmp.Width, bmp.Height),
            Color.MidnightBlue, Color.DarkOrchid,
            System.Drawing.Drawing2D.LinearGradientMode.Diagonal);
        g.FillRectangle(grad, 0, 0, bmp.Width, bmp.Height);

        // วาดดาว
        var random = new Random(42);
        for (int i = 0; i < 50; i++)
        {
            int x = random.Next(bmp.Width);
            int y = random.Next(bmp.Height);
            int size = random.Next(2, 6);
            g.FillEllipse(Brushes.White, x, y, size, size);
        }

        // วาดข้อความ
        using var font = new Font("Arial", 20, FontStyle.Bold);
        using var textBrush = new SolidBrush(Color.Gold);
        g.DrawString("Night Sky", font, textBrush, 140, 130);
    }

    protected override void OnPaint(PaintEventArgs e)
    {
        base.OnPaint(e);
        var g = e.Graphics;

        if (_bitmap != null)
        {
            // วาด bitmap ตามขนาดเดิม
            g.DrawImage(_bitmap, 10, 10);

            // วาด bitmap ด้วยขนาดที่กำหนด
            g.DrawImage(_bitmap, new Rectangle(420, 10, 200, 150));

            // วาดเฉพาะบางส่วนของ bitmap
            var srcRect = new Rectangle(50, 50, 200, 150);
            var destRect = new Rectangle(10, 320, 300, 200);
            g.DrawImage(_bitmap, destRect, srcRect, GraphicsUnit.Pixel);
        }

        // Image attributes - ปรับสี
        if (_bitmap != null)
        {
            var colorMatrix = new System.Drawing.Imaging.ColorMatrix(new float[][]
            {
                new float[] { 0.3f, 0.3f, 0.3f, 0, 0 },
                new float[] { 0.59f, 0.59f, 0.59f, 0, 0 },
                new float[] { 0.11f, 0.11f, 0.11f, 0, 0 },
                new float[] { 0, 0, 0, 1, 0 },
                new float[] { 0, 0, 0, 0, 1 }
            });

            var attrs = new System.Drawing.Imaging.ImageAttributes();
            attrs.SetColorMatrix(colorMatrix);

            var destRect2 = new Rectangle(330, 320, 200, 150);
            g.DrawImage(_bitmap, destRect2,
                0, 0, _bitmap.Width, _bitmap.Height,
                GraphicsUnit.Pixel, attrs);
            g.DrawString("Grayscale", SystemFonts.DefaultFont, Brushes.Black, 330, 475);
        }
    }

    protected override void Dispose(bool disposing)
    {
        if (disposing) _bitmap?.Dispose();
        base.Dispose(disposing);
    }
}
```

---

## Step 774: Data Binding ใน WinForms (BindingSource, BindingList\<T\>)

### การเชื่อมต่อข้อมูลกับ UI

#### BindingSource พื้นฐาน

```csharp
// Model class
public class Employee
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string Department { get; set; } = "";
    public decimal Salary { get; set; }
    public DateTime HireDate { get; set; }
}

// Form ที่ใช้ BindingSource
public class DataBindingForm : Form
{
    private BindingSource _bindingSource = new BindingSource();
    private BindingList<Employee> _employees = new BindingList<Employee>();

    private TextBox txtId = new TextBox();
    private TextBox txtName = new TextBox();
    private TextBox txtDepartment = new TextBox();
    private TextBox txtSalary = new TextBox();
    private DateTimePicker dtpHireDate = new DateTimePicker();
    private DataGridView dgvEmployees = new DataGridView();
    private Button btnAdd = new Button { Text = "Add" };
    private Button btnUpdate = new Button { Text = "Update" };
    private Button btnDelete = new Button { Text = "Delete" };

    public DataBindingForm()
    {
        SetupUI();
        LoadData();
        SetupBindings();
    }

    private void LoadData()
    {
        _employees.Add(new Employee { Id = 1, Name = "สมชาย ใจดี", Department = "IT", Salary = 50000, HireDate = new DateTime(2020, 1, 15) });
        _employees.Add(new Employee { Id = 2, Name = "สมหญิง รักงาน", Department = "HR", Salary = 45000, HireDate = new DateTime(2019, 6, 1) });
        _employees.Add(new Employee { Id = 3, Name = "มานะ ขยัน", Department = "Finance", Salary = 55000, HireDate = new DateTime(2021, 3, 20) });

        _bindingSource.DataSource = _employees;
    }

    private void SetupBindings()
    {
        // Bind TextBox กับ Property ของ object ปัจจุบัน
        txtId.DataBindings.Add("Text", _bindingSource, "Id", true,
            DataSourceUpdateMode.OnPropertyChanged);

        txtName.DataBindings.Add("Text", _bindingSource, "Name", true,
            DataSourceUpdateMode.OnPropertyChanged);

        txtDepartment.DataBindings.Add("Text", _bindingSource, "Department", true,
            DataSourceUpdateMode.OnPropertyChanged);

        txtSalary.DataBindings.Add("Text", _bindingSource, "Salary", true,
            DataSourceUpdateMode.OnPropertyChanged);

        dtpHireDate.DataBindings.Add("Value", _bindingSource, "HireDate", true,
            DataSourceUpdateMode.OnPropertyChanged);

        // Bind DataGridView
        dgvEmployees.DataSource = _bindingSource;

        // เมื่อเลือก row ใน grid จะ sync กับ detail fields โดยอัตโนมัติ
    }

    private void SetupUI()
    {
        this.Text = "Employee Management";
        this.Size = new Size(800, 600);

        // layout code...
        var panel = new TableLayoutPanel
        {
            Dock = DockStyle.Top,
            Height = 200,
            ColumnCount = 2,
            RowCount = 5
        };

        panel.Controls.Add(new Label { Text = "ID:" }, 0, 0);
        panel.Controls.Add(txtId, 1, 0);
        panel.Controls.Add(new Label { Text = "Name:" }, 0, 1);
        panel.Controls.Add(txtName, 1, 1);
        panel.Controls.Add(new Label { Text = "Department:" }, 0, 2);
        panel.Controls.Add(txtDepartment, 1, 2);
        panel.Controls.Add(new Label { Text = "Salary:" }, 0, 3);
        panel.Controls.Add(txtSalary, 1, 3);
        panel.Controls.Add(new Label { Text = "Hire Date:" }, 0, 4);
        panel.Controls.Add(dtpHireDate, 1, 4);

        var btnPanel = new FlowLayoutPanel { Dock = DockStyle.Top, Height = 40 };
        btnPanel.Controls.AddRange(new Control[] { btnAdd, btnUpdate, btnDelete });

        dgvEmployees.Dock = DockStyle.Fill;

        this.Controls.Add(dgvEmployees);
        this.Controls.Add(btnPanel);
        this.Controls.Add(panel);

        btnAdd.Click += BtnAdd_Click;
        btnUpdate.Click += BtnUpdate_Click;
        btnDelete.Click += BtnDelete_Click;
    }

    private void BtnAdd_Click(object? sender, EventArgs e)
    {
        int nextId = _employees.Count > 0 ? _employees.Max(x => x.Id) + 1 : 1;
        _employees.Add(new Employee
        {
            Id = nextId,
            Name = txtName.Text,
            Department = txtDepartment.Text,
            Salary = decimal.TryParse(txtSalary.Text, out var sal) ? sal : 0,
            HireDate = dtpHireDate.Value
        });
        // BindingList<T> แจ้ง UI โดยอัตโนมัติผ่าน INotifyCollectionChanged
    }

    private void BtnUpdate_Click(object? sender, EventArgs e)
    {
        // _bindingSource.Current คือ object ปัจจุบันที่ binding
        if (_bindingSource.Current is Employee emp)
        {
            emp.Name = txtName.Text;
            emp.Department = txtDepartment.Text;
            emp.Salary = decimal.TryParse(txtSalary.Text, out var sal) ? sal : emp.Salary;
            emp.HireDate = dtpHireDate.Value;
            _bindingSource.ResetCurrentItem(); // แจ้ง UI ให้ refresh
        }
    }

    private void BtnDelete_Click(object? sender, EventArgs e)
    {
        if (_bindingSource.Current is Employee emp)
        {
            _employees.Remove(emp);
        }
    }
}
```

#### INotifyPropertyChanged สำหรับ Two-way Binding

```csharp
// Model ที่ implement INotifyPropertyChanged
public class ProductModel : INotifyPropertyChanged
{
    private string _name = "";
    private decimal _price;
    private int _stock;

    public event PropertyChangedEventHandler? PropertyChanged;

    protected virtual void OnPropertyChanged([System.Runtime.CompilerServices.CallerMemberName] string? propertyName = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }

    public string Name
    {
        get => _name;
        set { if (_name != value) { _name = value; OnPropertyChanged(); } }
    }

    public decimal Price
    {
        get => _price;
        set { if (_price != value) { _price = value; OnPropertyChanged(); OnPropertyChanged(nameof(TotalValue)); } }
    }

    public int Stock
    {
        get => _stock;
        set { if (_stock != value) { _stock = value; OnPropertyChanged(); OnPropertyChanged(nameof(TotalValue)); } }
    }

    // Computed property
    public decimal TotalValue => Price * Stock;
}

// Form ที่ใช้ประโยชน์จาก INotifyPropertyChanged
public class ProductForm : Form
{
    private ProductModel _product = new ProductModel
    {
        Name = "Test Product",
        Price = 100.00m,
        Stock = 50
    };

    public ProductForm()
    {
        var nameBox = new TextBox();
        var priceBox = new TextBox();
        var stockBox = new TextBox();
        var totalLabel = new Label();

        // One-way binding สำหรับ computed property
        totalLabel.DataBindings.Add("Text", _product, "TotalValue",
            true, DataSourceUpdateMode.Never, "N2");

        // Two-way binding
        nameBox.DataBindings.Add("Text", _product, "Name", true,
            DataSourceUpdateMode.OnPropertyChanged);

        priceBox.DataBindings.Add("Text", _product, "Price", true,
            DataSourceUpdateMode.OnPropertyChanged, "0.00");

        stockBox.DataBindings.Add("Text", _product, "Stock", true,
            DataSourceUpdateMode.OnPropertyChanged);

        // เมื่อ Price หรือ Stock เปลี่ยน TotalValue จะ update อัตโนมัติ
        var layout = new TableLayoutPanel { Dock = DockStyle.Fill };
        layout.Controls.Add(new Label { Text = "Name:" });
        layout.Controls.Add(nameBox);
        layout.Controls.Add(new Label { Text = "Price:" });
        layout.Controls.Add(priceBox);
        layout.Controls.Add(new Label { Text = "Stock:" });
        layout.Controls.Add(stockBox);
        layout.Controls.Add(new Label { Text = "Total Value:" });
        layout.Controls.Add(totalLabel);

        this.Controls.Add(layout);
        this.Size = new Size(400, 300);
    }
}
```

---

## Step 775: DataGridView Advanced

### การใช้งาน DataGridView ขั้นสูง

#### Custom Columns และ Cell Rendering

```csharp
public class AdvancedGridForm : Form
{
    private DataGridView dgv = new DataGridView();

    public AdvancedGridForm()
    {
        this.Size = new Size(900, 600);
        this.Text = "Advanced DataGridView";

        SetupGrid();
        LoadData();
    }

    private void SetupGrid()
    {
        dgv.Dock = DockStyle.Fill;
        dgv.AutoGenerateColumns = false;
        dgv.AllowUserToAddRows = false;
        dgv.RowHeadersVisible = false;
        dgv.SelectionMode = DataGridViewSelectionMode.FullRowSelect;
        dgv.AlternatingRowsDefaultCellStyle.BackColor = Color.AliceBlue;

        // TextBox Column
        var nameCol = new DataGridViewTextBoxColumn
        {
            Name = "Name",
            HeaderText = "ชื่อสินค้า",
            DataPropertyName = "Name",
            Width = 180
        };

        // ComboBox Column
        var categoryCol = new DataGridViewComboBoxColumn
        {
            Name = "Category",
            HeaderText = "หมวดหมู่",
            DataPropertyName = "Category",
            Width = 120
        };
        categoryCol.Items.AddRange("Electronics", "Clothing", "Food", "Books");

        // Custom Number Column with formatting
        var priceCol = new DataGridViewTextBoxColumn
        {
            Name = "Price",
            HeaderText = "ราคา (บาท)",
            DataPropertyName = "Price",
            Width = 120,
            DefaultCellStyle = new DataGridViewCellStyle
            {
                Alignment = DataGridViewContentAlignment.MiddleRight,
                Format = "N2"
            }
        };

        // CheckBox Column
        var activeCol = new DataGridViewCheckBoxColumn
        {
            Name = "IsActive",
            HeaderText = "Active",
            DataPropertyName = "IsActive",
            Width = 70
        };

        // Button Column
        var actionCol = new DataGridViewButtonColumn
        {
            Name = "Action",
            HeaderText = "Action",
            Text = "แก้ไข",
            UseColumnTextForButtonValue = true,
            Width = 80
        };

        // Image Column
        var imageCol = new DataGridViewImageColumn
        {
            Name = "StatusIcon",
            HeaderText = "Status",
            DataPropertyName = "StatusIcon",
            Width = 60
        };

        dgv.Columns.AddRange(nameCol, categoryCol, priceCol, activeCol, actionCol, imageCol);

        // Event handlers
        dgv.CellClick += Dgv_CellClick;
        dgv.CellPainting += Dgv_CellPainting;
        dgv.ColumnHeaderMouseClick += Dgv_ColumnHeaderMouseClick;

        this.Controls.Add(dgv);
    }

    private void Dgv_CellClick(object? sender, DataGridViewCellEventArgs e)
    {
        // จัดการ Button column click
        if (e.ColumnIndex == dgv.Columns["Action"]!.Index && e.RowIndex >= 0)
        {
            var row = dgv.Rows[e.RowIndex];
            var name = row.Cells["Name"].Value?.ToString();
            MessageBox.Show($"แก้ไข: {name}");
        }
    }

    private void Dgv_CellPainting(object? sender, DataGridViewCellPaintingEventArgs e)
    {
        // Custom rendering สำหรับ Price column
        if (e.ColumnIndex == dgv.Columns["Price"]?.Index && e.RowIndex >= 0)
        {
            if (e.Value is decimal price)
            {
                e.Handled = true;
                e.PaintBackground(e.ClipBounds, true);

                // เปลี่ยนสีตามราคา
                Color textColor = price > 1000 ? Color.DarkRed :
                                  price > 500 ? Color.DarkOrange : Color.DarkGreen;

                var textRect = e.CellBounds;
                textRect.Inflate(-2, -2);

                TextRenderer.DrawText(e.Graphics, price.ToString("N2"),
                    e.CellStyle.Font, textRect, textColor,
                    TextFormatFlags.Right | TextFormatFlags.VerticalCenter);

                // วาด progress bar mini แสดงราคาเทียบกับ max
                float maxPrice = 2000f;
                float ratio = Math.Min((float)price / maxPrice, 1f);
                int barWidth = (int)(e.CellBounds.Width * ratio * 0.8f);

                using var barBrush = new SolidBrush(Color.FromArgb(30, textColor));
                g.FillRectangle(barBrush,
                    e.CellBounds.Left + 2,
                    e.CellBounds.Bottom - 5,
                    barWidth, 3);
            }
        }

        // Custom rendering สำหรับ Status column
        if (e.ColumnIndex == dgv.Columns["IsActive"]?.Index && e.RowIndex >= 0)
        {
            // handled โดย DataGridViewCheckBoxColumn อยู่แล้ว
        }
    }

    // Sorting แบบ custom
    private Dictionary<string, bool> _sortAscending = new Dictionary<string, bool>();
    private void Dgv_ColumnHeaderMouseClick(object? sender, DataGridViewCellMouseEventArgs e)
    {
        string colName = dgv.Columns[e.ColumnIndex].DataPropertyName ?? "";
        if (string.IsNullOrEmpty(colName)) return;

        bool ascending = !_sortAscending.GetValueOrDefault(colName, false);
        _sortAscending[colName] = ascending;

        // Sort ด้วย LINQ
        if (dgv.DataSource is BindingList<ProductItem> items)
        {
            var sorted = ascending
                ? items.OrderBy(x => GetPropertyValue(x, colName)).ToList()
                : items.OrderByDescending(x => GetPropertyValue(x, colName)).ToList();

            dgv.DataSource = new BindingList<ProductItem>(sorted);
        }

        // แสดง sort indicator
        foreach (DataGridViewColumn col in dgv.Columns)
            col.HeaderCell.SortGlyphDirection = SortOrder.None;

        dgv.Columns[e.ColumnIndex].HeaderCell.SortGlyphDirection =
            ascending ? SortOrder.Ascending : SortOrder.Descending;
    }

    private object? GetPropertyValue(object obj, string propertyName)
    {
        return obj.GetType().GetProperty(propertyName)?.GetValue(obj);
    }

    private void LoadData()
    {
        var products = new BindingList<ProductItem>
        {
            new ProductItem { Name = "iPhone 15 Pro", Category = "Electronics", Price = 45000, IsActive = true },
            new ProductItem { Name = "เสื้อยืด Basic", Category = "Clothing", Price = 299, IsActive = true },
            new ProductItem { Name = "C# Programming Guide", Category = "Books", Price = 750, IsActive = true },
            new ProductItem { Name = "ข้าวหอมมะลิ 5กก.", Category = "Food", Price = 180, IsActive = false },
            new ProductItem { Name = "Samsung TV 55\"", Category = "Electronics", Price = 28000, IsActive = true },
        };
        dgv.DataSource = products;
    }
}

public class ProductItem
{
    public string Name { get; set; } = "";
    public string Category { get; set; } = "";
    public decimal Price { get; set; }
    public bool IsActive { get; set; }
}
```

---

## Step 776: Async Operations ใน WinForms

### การทำงานแบบ Asynchronous ใน WinForms

#### async/await กับ UI Thread

```csharp
// WinForms ทำงานบน UI thread เดียว
// ต้องระวัง cross-thread exception เมื่อ update UI จาก background thread

public class AsyncDemoForm : Form
{
    private Button btnStart = new Button { Text = "Start", Width = 100 };
    private Button btnCancel = new Button { Text = "Cancel", Width = 100, Enabled = false };
    private ProgressBar progressBar = new ProgressBar { Width = 400, Maximum = 100 };
    private Label lblStatus = new Label { AutoSize = true, Text = "Ready" };
    private ListBox lstLog = new ListBox { Width = 500, Height = 200 };

    private CancellationTokenSource? _cts;

    public AsyncDemoForm()
    {
        SetupUI();
        btnStart.Click += BtnStart_Click;
        btnCancel.Click += BtnCancel_Click;
    }

    private void SetupUI()
    {
        this.Size = new Size(600, 400);
        var layout = new FlowLayoutPanel { Dock = DockStyle.Fill, FlowDirection = FlowDirection.TopDown, Padding = new Padding(10) };
        var btnPanel = new FlowLayoutPanel { Width = 500, Height = 40 };
        btnPanel.Controls.AddRange(new Control[] { btnStart, btnCancel });
        layout.Controls.Add(btnPanel);
        layout.Controls.Add(progressBar);
        layout.Controls.Add(lblStatus);
        layout.Controls.Add(lstLog);
        this.Controls.Add(layout);
    }

    private async void BtnStart_Click(object? sender, EventArgs e)
    {
        btnStart.Enabled = false;
        btnCancel.Enabled = true;
        lstLog.Items.Clear();

        _cts = new CancellationTokenSource();

        // Progress<T> ทำงานบน UI thread โดยอัตโนมัติ
        var progress = new Progress<ProgressInfo>(info =>
        {
            progressBar.Value = info.Percentage;
            lblStatus.Text = info.Message;
            lstLog.Items.Add($"[{DateTime.Now:HH:mm:ss}] {info.Message}");
        });

        try
        {
            await DoLongWorkAsync(progress, _cts.Token);
            lblStatus.Text = "Completed!";
            MessageBox.Show("งานเสร็จสิ้น!", "Success");
        }
        catch (OperationCanceledException)
        {
            lblStatus.Text = "Cancelled";
            progressBar.Value = 0;
        }
        catch (Exception ex)
        {
            lblStatus.Text = $"Error: {ex.Message}";
            MessageBox.Show($"เกิดข้อผิดพลาด: {ex.Message}", "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
        }
        finally
        {
            btnStart.Enabled = true;
            btnCancel.Enabled = false;
            _cts?.Dispose();
            _cts = null;
        }
    }

    private void BtnCancel_Click(object? sender, EventArgs e)
    {
        _cts?.Cancel();
    }

    private async Task DoLongWorkAsync(IProgress<ProgressInfo> progress, CancellationToken ct)
    {
        int total = 10;
        for (int i = 1; i <= total; i++)
        {
            ct.ThrowIfCancellationRequested();

            // จำลองงานหนัก (I/O bound)
            await Task.Delay(500, ct);

            int percentage = i * 100 / total;
            progress.Report(new ProgressInfo
            {
                Percentage = percentage,
                Message = $"Processing item {i} of {total}..."
            });
        }
    }
}

public class ProgressInfo
{
    public int Percentage { get; set; }
    public string Message { get; set; } = "";
}
```

#### Control.Invoke() vs BeginInvoke()

```csharp
public class InvokeDemo : Form
{
    private Label lblStatus = new Label { AutoSize = true, Text = "Status: Idle" };
    private TextBox txtLog = new TextBox
    {
        Multiline = true,
        ScrollBars = ScrollBars.Vertical,
        Height = 200,
        ReadOnly = true
    };
    private Button btnStartThread = new Button { Text = "Start Background Thread" };

    public InvokeDemo()
    {
        this.Size = new Size(600, 350);
        var layout = new FlowLayoutPanel { Dock = DockStyle.Fill, FlowDirection = FlowDirection.TopDown, Padding = new Padding(10) };
        layout.Controls.Add(btnStartThread);
        layout.Controls.Add(lblStatus);
        layout.Controls.Add(txtLog);
        this.Controls.Add(layout);

        btnStartThread.Click += BtnStartThread_Click;
    }

    private void BtnStartThread_Click(object? sender, EventArgs e)
    {
        btnStartThread.Enabled = false;

        Thread thread = new Thread(() =>
        {
            for (int i = 1; i <= 5; i++)
            {
                Thread.Sleep(1000);

                // Control.Invoke() - รอจนกว่า UI thread จะทำงานเสร็จ (synchronous)
                this.Invoke(() =>
                {
                    lblStatus.Text = $"Status: Processing {i}/5...";
                });

                // Control.BeginInvoke() - ไม่รอ (asynchronous) ดีกว่าสำหรับ logging
                this.BeginInvoke(() =>
                {
                    txtLog.AppendText($"[{DateTime.Now:HH:mm:ss.fff}] Item {i} done\r\n");
                });
            }

            // กลับมาที่ UI thread หลังเสร็จ
            this.Invoke(() =>
            {
                lblStatus.Text = "Status: Done!";
                btnStartThread.Enabled = true;
            });
        });

        thread.IsBackground = true; // thread จะหยุดเมื่อ app ปิด
        thread.Start();
    }
}
```

#### BackgroundWorker (วิธีเก่า แต่ยังใช้ได้)

```csharp
public class BackgroundWorkerDemo : Form
{
    private BackgroundWorker _worker = new BackgroundWorker
    {
        WorkerReportsProgress = true,
        WorkerSupportsCancellation = true
    };

    private ProgressBar progressBar = new ProgressBar { Width = 400 };
    private Label lblStatus = new Label { AutoSize = true };
    private Button btnStart = new Button { Text = "Start" };
    private Button btnCancel = new Button { Text = "Cancel", Enabled = false };

    public BackgroundWorkerDemo()
    {
        SetupUI();
        SetupWorker();
    }

    private void SetupUI()
    {
        this.Size = new Size(500, 200);
        var layout = new FlowLayoutPanel { Dock = DockStyle.Fill, FlowDirection = FlowDirection.TopDown, Padding = new Padding(10) };
        var btns = new FlowLayoutPanel { Height = 40 };
        btns.Controls.AddRange(new Control[] { btnStart, btnCancel });
        layout.Controls.Add(btns);
        layout.Controls.Add(progressBar);
        layout.Controls.Add(lblStatus);
        this.Controls.Add(layout);
        btnStart.Click += (s, e) => { _worker.RunWorkerAsync(); btnStart.Enabled = false; btnCancel.Enabled = true; };
        btnCancel.Click += (s, e) => _worker.CancelAsync();
    }

    private void SetupWorker()
    {
        _worker.DoWork += (sender, e) =>
        {
            // ทำงานบน background thread ห้าม access UI โดยตรง
            for (int i = 1; i <= 100; i++)
            {
                if (_worker.CancellationPending)
                {
                    e.Cancel = true;
                    return;
                }
                Thread.Sleep(50);
                _worker.ReportProgress(i, $"Processing {i}%");
            }
            e.Result = "Work completed successfully!";
        };

        // ProgressChanged รันบน UI thread
        _worker.ProgressChanged += (sender, e) =>
        {
            progressBar.Value = e.ProgressPercentage;
            lblStatus.Text = e.UserState?.ToString();
        };

        // RunWorkerCompleted รันบน UI thread
        _worker.RunWorkerCompleted += (sender, e) =>
        {
            btnStart.Enabled = true;
            btnCancel.Enabled = false;

            if (e.Cancelled)
                lblStatus.Text = "Cancelled!";
            else if (e.Error != null)
                lblStatus.Text = $"Error: {e.Error.Message}";
            else
                lblStatus.Text = e.Result?.ToString();
        };
    }
}
```

---

## Step 777: Drag and Drop ใน WinForms

### การจัดการ Drag and Drop

#### พื้นฐาน Drag and Drop

```csharp
public class DragDropDemo : Form
{
    private ListBox lstSource = new ListBox { Width = 200, Height = 300 };
    private ListBox lstDest = new ListBox { Width = 200, Height = 300 };
    private Label lblDrop = new Label
    {
        Text = "Drop files here",
        BorderStyle = BorderStyle.FixedSingle,
        Width = 200,
        Height = 100,
        TextAlign = ContentAlignment.MiddleCenter
    };

    public DragDropDemo()
    {
        this.Text = "Drag and Drop Demo";
        this.Size = new Size(700, 450);
        this.AllowDrop = true; // อนุญาตให้รับ drop จาก outside

        SetupSourceList();
        SetupDestList();
        SetupDropZone();
        SetupFormDrop();

        var layout = new FlowLayoutPanel
        {
            Dock = DockStyle.Fill,
            FlowDirection = FlowDirection.LeftToRight,
            Padding = new Padding(10),
            WrapContents = false
        };

        var leftPanel = new GroupBox { Text = "Source", Width = 220, Height = 320 };
        leftPanel.Controls.Add(lstSource);
        lstSource.Location = new Point(8, 20);

        var rightPanel = new GroupBox { Text = "Destination", Width = 220, Height = 320 };
        rightPanel.Controls.Add(lstDest);
        lstDest.Location = new Point(8, 20);

        var filePanel = new GroupBox { Text = "File Drop Zone", Width = 220, Height = 320 };
        filePanel.Controls.Add(lblDrop);
        lblDrop.Location = new Point(8, 20);

        layout.Controls.Add(leftPanel);
        layout.Controls.Add(rightPanel);
        layout.Controls.Add(filePanel);
        this.Controls.Add(layout);
    }

    private void SetupSourceList()
    {
        lstSource.Items.AddRange(new string[] { "Item A", "Item B", "Item C", "Item D", "Item E" });

        // เริ่ม drag เมื่อ mouse down
        lstSource.MouseDown += (s, e) =>
        {
            if (lstSource.SelectedItem == null) return;
            // DoDragDrop เริ่ม drag operation
            lstSource.DoDragDrop(lstSource.SelectedItem, DragDropEffects.Move | DragDropEffects.Copy);
        };
    }

    private void SetupDestList()
    {
        lstDest.AllowDrop = true;

        lstDest.DragEnter += (s, e) =>
        {
            // ตรวจสอบประเภทข้อมูลที่ drag มา
            if (e.Data?.GetDataPresent(DataFormats.StringFormat) == true)
                e.Effect = e.KeyState == 8 ? DragDropEffects.Copy : DragDropEffects.Move;
            else
                e.Effect = DragDropEffects.None;
        };

        lstDest.DragOver += (s, e) =>
        {
            // แสดง visual feedback ระหว่าง drag
            if (e.Data?.GetDataPresent(DataFormats.StringFormat) == true)
            {
                e.Effect = (e.KeyState & 8) != 0 ? DragDropEffects.Copy : DragDropEffects.Move;
            }
        };

        lstDest.DragDrop += (s, e) =>
        {
            // รับข้อมูลเมื่อ drop
            string? item = e.Data?.GetData(DataFormats.StringFormat) as string;
            if (item != null)
            {
                lstDest.Items.Add(item);
                // ถ้า Move ให้ลบจาก source
                if (e.Effect == DragDropEffects.Move)
                    lstSource.Items.Remove(item);
            }
        };
    }

    private void SetupDropZone()
    {
        lblDrop.AllowDrop = true;

        lblDrop.DragEnter += (s, e) =>
        {
            if (e.Data?.GetDataPresent(DataFormats.FileDrop) == true)
            {
                e.Effect = DragDropEffects.Copy;
                lblDrop.BackColor = Color.LightYellow;
            }
        };

        lblDrop.DragLeave += (s, e) => lblDrop.BackColor = SystemColors.Control;

        lblDrop.DragDrop += (s, e) =>
        {
            lblDrop.BackColor = SystemColors.Control;
            string[]? files = (string[]?)e.Data?.GetData(DataFormats.FileDrop);
            if (files != null)
            {
                lblDrop.Text = string.Join("\n", files.Select(f => Path.GetFileName(f)));
            }
        };
    }

    private void SetupFormDrop()
    {
        // รับ drop บน Form เอง
        this.DragEnter += (s, e) =>
        {
            if (e.Data?.GetDataPresent(DataFormats.FileDrop) == true)
                e.Effect = DragDropEffects.Copy;
        };

        this.DragDrop += (s, e) =>
        {
            string[]? files = (string[]?)e.Data?.GetData(DataFormats.FileDrop);
            if (files != null)
            {
                MessageBox.Show($"Dropped {files.Length} file(s) on form:\n{string.Join("\n", files)}");
            }
        };
    }
}
```

#### Drag and Drop พร้อม Custom Data

```csharp
// Custom data class สำหรับ drag
[Serializable]
public class DragData
{
    public int Id { get; set; }
    public string Text { get; set; } = "";
    public Color Color { get; set; }
}

// Panel ที่รองรับ drag items
public class DragPanel : Panel
{
    private List<DragData> _items = new List<DragData>();
    private DragData? _draggingItem;

    public DragPanel()
    {
        this.AllowDrop = true;
        this.BorderStyle = BorderStyle.FixedSingle;
        this.BackColor = Color.White;

        // เพิ่ม sample items
        _items.Add(new DragData { Id = 1, Text = "Red Card", Color = Color.Salmon });
        _items.Add(new DragData { Id = 2, Text = "Blue Card", Color = Color.SkyBlue });
        _items.Add(new DragData { Id = 3, Text = "Green Card", Color = Color.LightGreen });

        this.MouseDown += OnMouseDown;
        this.DragOver += OnDragOver;
        this.DragDrop += OnDragDrop;

        this.SetStyle(ControlStyles.OptimizedDoubleBuffer, true);
    }

    private void OnMouseDown(object? sender, MouseEventArgs e)
    {
        // หา item ที่ click
        _draggingItem = HitTest(e.Location);
        if (_draggingItem != null)
        {
            DoDragDrop(_draggingItem, DragDropEffects.Move);
        }
    }

    private DragData? HitTest(Point p)
    {
        int itemHeight = 40;
        int index = p.Y / itemHeight;
        return index < _items.Count ? _items[index] : null;
    }

    private void OnDragOver(object? sender, DragEventArgs e)
    {
        if (e.Data?.GetDataPresent(typeof(DragData)) == true)
            e.Effect = DragDropEffects.Move;
    }

    private void OnDragDrop(object? sender, DragEventArgs e)
    {
        var data = e.Data?.GetData(typeof(DragData)) as DragData;
        if (data != null)
        {
            var dropPoint = this.PointToClient(new Point(e.X, e.Y));
            int newIndex = Math.Min(dropPoint.Y / 40, _items.Count - 1);
            int oldIndex = _items.IndexOf(data);

            if (oldIndex != newIndex)
            {
                _items.RemoveAt(oldIndex);
                _items.Insert(newIndex, data);
                Invalidate();
            }
        }
    }

    protected override void OnPaint(PaintEventArgs e)
    {
        base.OnPaint(e);
        var g = e.Graphics;

        for (int i = 0; i < _items.Count; i++)
        {
            var item = _items[i];
            var rect = new Rectangle(4, i * 40 + 4, Width - 8, 34);

            using var brush = new SolidBrush(item.Color);
            g.FillRoundedRectangle(brush, rect.X, rect.Y, rect.Width, rect.Height, 5);
            g.DrawString(item.Text, SystemFonts.DefaultFont, Brushes.Black, rect.X + 8, rect.Y + 9);
        }
    }
}
```

---

## Step 778: MDI Applications (Multiple Document Interface)

### การสร้าง MDI Application

#### MDI Parent Form

```csharp
public class MdiParentForm : Form
{
    private MenuStrip menuStrip = new MenuStrip();
    private StatusStrip statusStrip = new StatusStrip();
    private ToolStripStatusLabel statusLabel = new ToolStripStatusLabel("Ready");
    private int _docCount = 0;

    public MdiParentForm()
    {
        this.IsMdiContainer = true; // ทำให้เป็น MDI Parent
        this.Text = "MDI Application";
        this.Size = new Size(1000, 700);
        this.WindowState = FormWindowState.Maximized;

        SetupMenu();
        SetupStatus();
        SetupToolbar();
    }

    private void SetupMenu()
    {
        // File Menu
        var fileMenu = new ToolStripMenuItem("&File");
        fileMenu.DropDownItems.Add("&New Document", null, (s, e) => NewDocument());
        fileMenu.DropDownItems.Add("New &Spreadsheet", null, (s, e) => NewSpreadsheet());
        fileMenu.DropDownItems.Add("-");
        fileMenu.DropDownItems.Add("E&xit", null, (s, e) => Close());

        // View Menu
        var viewMenu = new ToolStripMenuItem("&View");
        viewMenu.DropDownItems.Add("&Cascade", null, (s, e) =>
            this.LayoutMdi(MdiLayout.Cascade));
        viewMenu.DropDownItems.Add("Tile &Horizontal", null, (s, e) =>
            this.LayoutMdi(MdiLayout.TileHorizontal));
        viewMenu.DropDownItems.Add("Tile &Vertical", null, (s, e) =>
            this.LayoutMdi(MdiLayout.TileVertical));
        viewMenu.DropDownItems.Add("Arrange &Icons", null, (s, e) =>
            this.LayoutMdi(MdiLayout.ArrangeIcons));

        // Window Menu - แสดงรายการ child windows โดยอัตโนมัติ
        var windowMenu = new ToolStripMenuItem("&Window");
        windowMenu.MdiWindowListItem = windowMenu; // ลิสต์ window อัตโนมัติ

        menuStrip.Items.Add(fileMenu);
        menuStrip.Items.Add(viewMenu);
        menuStrip.Items.Add(windowMenu);
        this.MainMenuStrip = menuStrip;
        this.Controls.Add(menuStrip);

        statusStrip.Items.Add(statusLabel);
        this.Controls.Add(statusStrip);

        // Track active MDI child
        this.MdiChildActivate += (s, e) =>
        {
            if (this.ActiveMdiChild != null)
                statusLabel.Text = $"Active: {this.ActiveMdiChild.Text}";
            else
                statusLabel.Text = "No document open";
        };
    }

    private void SetupStatus()
    {
        statusStrip.Items.Add(statusLabel);
        this.Controls.Add(statusStrip);
    }

    private void SetupToolbar()
    {
        var toolbar = new ToolStrip();
        toolbar.Items.Add(new ToolStripButton("New Doc", null, (s, e) => NewDocument()));
        toolbar.Items.Add(new ToolStripButton("New Sheet", null, (s, e) => NewSpreadsheet()));
        toolbar.Items.Add(new ToolStripSeparator());
        toolbar.Items.Add(new ToolStripButton("Cascade", null, (s, e) => LayoutMdi(MdiLayout.Cascade)));
        this.Controls.Add(toolbar);
    }

    private void NewDocument()
    {
        _docCount++;
        var child = new MdiTextDocument($"Document {_docCount}")
        {
            MdiParent = this // ตั้งให้เป็น MDI Child
        };
        child.Show();
    }

    private void NewSpreadsheet()
    {
        _docCount++;
        var child = new MdiSpreadsheet($"Spreadsheet {_docCount}")
        {
            MdiParent = this
        };
        child.Show();
    }
}

// MDI Child - Text Document
public class MdiTextDocument : Form
{
    private RichTextBox rtb = new RichTextBox { Dock = DockStyle.Fill };
    private bool _modified = false;

    public MdiTextDocument(string title)
    {
        this.Text = title;
        this.Size = new Size(600, 400);
        this.Controls.Add(rtb);

        rtb.TextChanged += (s, e) =>
        {
            if (!_modified)
            {
                _modified = true;
                this.Text = this.Text + " *";
            }
        };

        this.FormClosing += (s, e) =>
        {
            if (_modified)
            {
                var result = MessageBox.Show(
                    "มีการแก้ไขที่ยังไม่ได้บันทึก บันทึกก่อนปิดหรือไม่?",
                    "Confirm Close",
                    MessageBoxButtons.YesNoCancel);

                if (result == DialogResult.Cancel)
                    e.Cancel = true;
                else if (result == DialogResult.Yes)
                    SaveDocument();
            }
        };
    }

    private void SaveDocument()
    {
        using var dlg = new SaveFileDialog { Filter = "Text Files|*.txt" };
        if (dlg.ShowDialog() == DialogResult.OK)
        {
            File.WriteAllText(dlg.FileName, rtb.Text);
            _modified = false;
            this.Text = Path.GetFileName(dlg.FileName);
        }
    }
}

// MDI Child - Spreadsheet
public class MdiSpreadsheet : Form
{
    private DataGridView grid = new DataGridView { Dock = DockStyle.Fill };

    public MdiSpreadsheet(string title)
    {
        this.Text = title;
        this.Size = new Size(700, 400);

        // สร้าง grid ธรรมดา
        grid.ColumnCount = 10;
        grid.RowCount = 20;
        for (int i = 0; i < grid.ColumnCount; i++)
            grid.Columns[i].HeaderText = ((char)('A' + i)).ToString();

        this.Controls.Add(grid);
    }
}
```

---

## Step 779: WinForms ใน .NET 8

### คุณสมบัติใหม่ใน WinForms .NET 8

#### High DPI และ Dark Mode

```csharp
// Program.cs สำหรับ .NET 8 WinForms
static class Program
{
    [STAThread]
    static void Main()
    {
        // ตั้งค่า High DPI ใน .NET 8
        Application.SetHighDpiMode(HighDpiMode.PerMonitorV2);
        Application.EnableVisualStyles();
        Application.SetCompatibleTextRenderingDefault(false);

        // .NET 8: ตั้งค่า dark mode
        Application.SetColorScheme(SystemColorScheme.Dark);

        Application.Run(new MainForm());
    }
}

// Form ที่รองรับ Dark Mode
public class DarkModeForm : Form
{
    public DarkModeForm()
    {
        this.Text = ".NET 8 WinForms Dark Mode";
        this.Size = new Size(600, 400);

        // Dark mode colors
        ApplyDarkTheme();
        SetupUI();
    }

    private void ApplyDarkTheme()
    {
        // ใช้ Application.IsDarkModeEnabled ใน .NET 8
        bool isDark = SystemInformation.HighContrast ||
                      Application.IsDarkModeEnabled;

        if (isDark)
        {
            this.BackColor = Color.FromArgb(32, 32, 32);
            this.ForeColor = Color.White;
        }
    }

    private void SetupUI()
    {
        bool isDark = Application.IsDarkModeEnabled;

        var panel = new Panel
        {
            Dock = DockStyle.Fill,
            BackColor = isDark ? Color.FromArgb(45, 45, 48) : SystemColors.Control
        };

        var label = new Label
        {
            Text = isDark ? "Dark Mode Active" : "Light Mode Active",
            ForeColor = isDark ? Color.White : Color.Black,
            AutoSize = true,
            Location = new Point(20, 20),
            Font = new Font("Segoe UI", 16)
        };

        var btn = new Button
        {
            Text = "Toggle Theme",
            Location = new Point(20, 60),
            Size = new Size(150, 40),
            BackColor = isDark ? Color.FromArgb(63, 63, 70) : SystemColors.Control,
            ForeColor = isDark ? Color.White : Color.Black,
            FlatStyle = isDark ? FlatStyle.Flat : FlatStyle.Standard
        };

        if (isDark)
            btn.FlatAppearance.BorderColor = Color.Gray;

        panel.Controls.Add(label);
        panel.Controls.Add(btn);
        this.Controls.Add(panel);
    }
}
```

#### .NET 8 WinForms ใหม่ - TaskDialog

```csharp
public class TaskDialogDemo : Form
{
    public TaskDialogDemo()
    {
        this.Size = new Size(400, 300);
        this.Text = "TaskDialog Demo (.NET 8)";

        var btn = new Button
        {
            Text = "Show TaskDialog",
            Location = new Point(20, 20),
            Size = new Size(150, 40)
        };
        btn.Click += ShowTaskDialog;

        var btn2 = new Button
        {
            Text = "Show Progress Dialog",
            Location = new Point(20, 70),
            Size = new Size(150, 40)
        };
        btn2.Click += ShowProgressDialog;

        this.Controls.Add(btn);
        this.Controls.Add(btn2);
    }

    private void ShowTaskDialog(object? sender, EventArgs e)
    {
        // TaskDialog - modern dialog ใน .NET 6+
        var page = new TaskDialogPage
        {
            Caption = "My Application",
            Heading = "ยืนยันการดำเนินการ",
            Text = "คุณต้องการดำเนินการนี้หรือไม่?\nการกระทำนี้ไม่สามารถย้อนกลับได้",
            Icon = TaskDialogIcon.Warning,

            Buttons = new TaskDialogButtonCollection
            {
                new TaskDialogButton("ยืนยัน", isDefault: true),
                new TaskDialogButton("ยกเลิก")
            },

            Footnote = new TaskDialogFootnote
            {
                Text = "โปรดอ่านรายละเอียดก่อนยืนยัน",
                Icon = TaskDialogFootnoteIcon.Information
            },

            Expander = new TaskDialogExpander
            {
                Text = "รายละเอียดเพิ่มเติม:\n- การดำเนินการนี้จะลบข้อมูลทั้งหมด\n- ไม่สามารถกู้คืนได้",
                CollapsedButtonText = "แสดงรายละเอียด",
                ExpandedButtonText = "ซ่อนรายละเอียด"
            },

            Verification = new TaskDialogVerificationCheckBox
            {
                Text = "ฉันเข้าใจและยืนยันการดำเนินการนี้"
            }
        };

        var result = TaskDialog.ShowDialog(this, page);
        MessageBox.Show($"Result: {result.Text}");
    }

    private async void ShowProgressDialog(object? sender, EventArgs e)
    {
        var page = new TaskDialogPage
        {
            Caption = "Processing",
            Heading = "กำลังประมวลผล...",
            Text = "โปรดรอสักครู่",
            Icon = TaskDialogIcon.Information,

            ProgressBar = new TaskDialogProgressBar
            {
                Minimum = 0,
                Maximum = 100,
                Value = 0,
                State = TaskDialogProgressBarState.Normal
            },

            Buttons = { TaskDialogButton.Cancel }
        };

        page.Created += async (s, ev) =>
        {
            for (int i = 0; i <= 100; i += 5)
            {
                await Task.Delay(100);
                if (page.ProgressBar != null)
                    page.ProgressBar.Value = i;
                page.Text = $"กำลังประมวลผล... {i}%";
            }
            page.Navigate(new TaskDialogPage
            {
                Icon = TaskDialogIcon.ShieldSuccessGreenBar,
                Heading = "เสร็จสิ้น!",
                Text = "การดำเนินการเสร็จสมบูรณ์",
                Buttons = { TaskDialogButton.OK }
            });
        };

        TaskDialog.ShowDialog(this, page);
    }
}
```

---

## Step 780: WinForms + WPF Interop และ Win32 API

### การใช้งาน WinForms ร่วมกับ WPF และ Win32 API

#### Win32 API ผ่าน DllImport (P/Invoke)

```csharp
using System.Runtime.InteropServices;

public class Win32ApiDemo : Form
{
    // Win32 API declarations ด้วย DllImport
    [DllImport("user32.dll")]
    private static extern bool FlashWindow(IntPtr hwnd, bool bInvert);

    [DllImport("user32.dll")]
    private static extern int FlashWindowEx(ref FLASHWINFO pfwi);

    [DllImport("user32.dll")]
    private static extern bool SetWindowPos(IntPtr hWnd, IntPtr hWndInsertAfter,
        int X, int Y, int cx, int cy, uint uFlags);

    [DllImport("user32.dll")]
    private static extern IntPtr GetForegroundWindow();

    [DllImport("user32.dll")]
    private static extern bool SetForegroundWindow(IntPtr hWnd);

    [DllImport("kernel32.dll")]
    private static extern bool Beep(uint dwFreq, uint dwDuration);

    [DllImport("shell32.dll", CharSet = CharSet.Unicode)]
    private static extern int SHGetKnownFolderPath(
        [MarshalAs(UnmanagedType.LPStruct)] Guid rfid,
        uint dwFlags, IntPtr hToken,
        out IntPtr pszPath);

    // Structs สำหรับ Win32 API
    [StructLayout(LayoutKind.Sequential)]
    private struct FLASHWINFO
    {
        public uint cbSize;
        public IntPtr hwnd;
        public uint dwFlags;
        public uint uCount;
        public uint dwTimeout;
    }

    // Constants
    private static readonly IntPtr HWND_TOPMOST = new IntPtr(-1);
    private static readonly IntPtr HWND_NOTOPMOST = new IntPtr(-2);
    private const uint SWP_NOMOVE = 0x0002;
    private const uint SWP_NOSIZE = 0x0001;
    private const uint FLASHW_TRAY = 0x00000002;
    private const uint FLASHW_TIMER = 0x00000004;
    private const uint FLASHW_STOP = 0;

    public Win32ApiDemo()
    {
        this.Text = "Win32 API Demo";
        this.Size = new Size(500, 350);

        var btnFlash = new Button { Text = "Flash Taskbar", Location = new Point(20, 20), Width = 150 };
        var btnTopmost = new Button { Text = "Always on Top", Location = new Point(20, 60), Width = 150 };
        var btnBeep = new Button { Text = "Beep", Location = new Point(20, 100), Width = 150 };
        var btnBringFront = new Button { Text = "Bring to Front", Location = new Point(20, 140), Width = 150 };

        btnFlash.Click += (s, e) => FlashTaskbar();
        btnTopmost.Click += (s, e) => ToggleAlwaysOnTop();
        btnBeep.Click += (s, e) => PlayBeep();
        btnBringFront.Click += (s, e) => BringToFront();

        this.Controls.AddRange(new Control[] { btnFlash, btnTopmost, btnBeep, btnBringFront });
    }

    private void FlashTaskbar()
    {
        var fwi = new FLASHWINFO
        {
            cbSize = (uint)Marshal.SizeOf(typeof(FLASHWINFO)),
            hwnd = this.Handle,
            dwFlags = FLASHW_TRAY | FLASHW_TIMER,
            uCount = 5,
            dwTimeout = 0
        };
        FlashWindowEx(ref fwi);

        // หยุด flash หลัง 3 วินาที
        Task.Delay(3000).ContinueWith(_ =>
        {
            fwi.dwFlags = FLASHW_STOP;
            FlashWindowEx(ref fwi);
        });
    }

    private bool _isTopmost = false;
    private void ToggleAlwaysOnTop()
    {
        _isTopmost = !_isTopmost;
        var hWndInsertAfter = _isTopmost ? HWND_TOPMOST : HWND_NOTOPMOST;
        SetWindowPos(this.Handle, hWndInsertAfter, 0, 0, 0, 0, SWP_NOMOVE | SWP_NOSIZE);
        this.Text = _isTopmost ? "Win32 API Demo [TOPMOST]" : "Win32 API Demo";
    }

    private void PlayBeep()
    {
        // เล่นเสียง beep ด้วย Windows Beep API
        Task.Run(() =>
        {
            Beep(440, 200);  // A4
            Thread.Sleep(100);
            Beep(494, 200);  // B4
            Thread.Sleep(100);
            Beep(523, 400);  // C5
        });
    }

    private void BringToFront()
    {
        SetForegroundWindow(this.Handle);
    }
}
```

#### WinForms + WPF ElementHost (Interop)

```csharp
// เพิ่ม NuGet package: System.Windows.Forms.Integration
// เพิ่ม Reference: WindowsBase, PresentationCore, PresentationFramework

// ในไฟล์ .csproj ต้องเพิ่ม:
// <UseWPF>true</UseWPF>
// <UseWindowsForms>true</UseWindowsForms>

using System.Windows.Forms.Integration;
// using System.Windows.Controls; // WPF Controls

public class WinFormsWpfInteropForm : Form
{
    public WinFormsWpfInteropForm()
    {
        this.Text = "WinForms + WPF Interop";
        this.Size = new Size(800, 600);

        SetupLayout();
    }

    private void SetupLayout()
    {
        var splitContainer = new SplitContainer
        {
            Dock = DockStyle.Fill,
            Orientation = Orientation.Horizontal
        };

        // ส่วน WinForms
        var winformsPanel = new Panel { Dock = DockStyle.Fill, BackColor = Color.LightBlue };
        winformsPanel.Controls.Add(new Label
        {
            Text = "This is WinForms Panel",
            Dock = DockStyle.Top,
            Height = 30,
            TextAlign = ContentAlignment.MiddleCenter
        });
        winformsPanel.Controls.Add(new Button
        {
            Text = "WinForms Button",
            Location = new Point(10, 40),
            Size = new Size(150, 40)
        });

        splitContainer.Panel1.Controls.Add(winformsPanel);

        // ส่วน WPF ผ่าน ElementHost
        try
        {
            var elementHost = new ElementHost
            {
                Dock = DockStyle.Fill,
            };

            // สร้าง WPF Control แบบ programmatic
            var wpfControl = CreateWpfControl();
            elementHost.Child = wpfControl;

            var wpfWrapper = new Panel { Dock = DockStyle.Fill };
            wpfWrapper.Controls.Add(new Label
            {
                Text = "This is WPF Control (hosted in WinForms)",
                Dock = DockStyle.Top,
                Height = 30,
                TextAlign = ContentAlignment.MiddleCenter,
                BackColor = Color.LightYellow
            });
            wpfWrapper.Controls.Add(elementHost);

            splitContainer.Panel2.Controls.Add(wpfWrapper);
        }
        catch (Exception ex)
        {
            splitContainer.Panel2.Controls.Add(new Label
            {
                Text = $"WPF interop not available: {ex.Message}",
                Dock = DockStyle.Fill,
                TextAlign = ContentAlignment.MiddleCenter
            });
        }

        this.Controls.Add(splitContainer);
    }

    private System.Windows.UIElement CreateWpfControl()
    {
        // สร้าง WPF StackPanel แบบ code-behind
        var stackPanel = new System.Windows.Controls.StackPanel
        {
            Orientation = System.Windows.Controls.Orientation.Vertical,
            Margin = new System.Windows.Thickness(10)
        };

        var wpfLabel = new System.Windows.Controls.Label
        {
            Content = "WPF Label with styling",
            FontSize = 16,
            Foreground = System.Windows.Media.Brushes.DarkBlue
        };

        var wpfButton = new System.Windows.Controls.Button
        {
            Content = "WPF Button",
            Width = 200,
            Height = 40,
            Margin = new System.Windows.Thickness(0, 10, 0, 0)
        };
        wpfButton.Click += (s, e) =>
            System.Windows.MessageBox.Show("This is a WPF MessageBox from WinForms app!");

        var wpfSlider = new System.Windows.Controls.Slider
        {
            Minimum = 0,
            Maximum = 100,
            Value = 50,
            Width = 200,
            Margin = new System.Windows.Thickness(0, 10, 0, 0)
        };

        var wpfProgressBar = new System.Windows.Controls.ProgressBar
        {
            Width = 200,
            Height = 20,
            Margin = new System.Windows.Thickness(0, 5, 0, 0)
        };

        // Bind ProgressBar ค่าของ Slider
        wpfSlider.ValueChanged += (s, e) => wpfProgressBar.Value = e.NewValue;
        wpfProgressBar.Value = 50;

        stackPanel.Children.Add(wpfLabel);
        stackPanel.Children.Add(wpfButton);
        stackPanel.Children.Add(wpfSlider);
        stackPanel.Children.Add(wpfProgressBar);

        return stackPanel;
    }
}
```

#### WindowsFormsHost ใน WPF (Reverse Interop)

```xml
<!-- ใน WPF Window XAML -->
<!-- เพิ่ม namespace: xmlns:wf="clr-namespace:System.Windows.Forms;assembly=System.Windows.Forms" -->
<!-- เพิ่ม namespace: xmlns:wfi="clr-namespace:System.Windows.Forms.Integration;assembly=WindowsFormsIntegration" -->

<!--
<Grid>
    <Grid.RowDefinitions>
        <RowDefinition Height="Auto"/>
        <RowDefinition Height="*"/>
    </Grid.RowDefinitions>

    <TextBlock Grid.Row="0" Text="WPF + WinForms Interop" FontSize="20" Margin="10"/>

    <wfi:WindowsFormsHost Grid.Row="1" Margin="10">
        <wf:DataGridView x:Name="winformsGrid"/>
    </wfi:WindowsFormsHost>
</Grid>
-->
```

```csharp
// Code-behind สำหรับ WPF Window ที่ host WinForms control
public partial class WpfHostingWinForms : System.Windows.Window
{
    private System.Windows.Forms.DataGridView? _dgv;

    public WpfHostingWinForms()
    {
        InitializeComponent();
        SetupWinFormsControl();
    }

    private void SetupWinFormsControl()
    {
        // หาก _dgv ถูก inject ผ่าน WindowsFormsHost ใน XAML
        if (_dgv != null)
        {
            _dgv.AutoGenerateColumns = true;
            _dgv.DataSource = GetSampleData();
            _dgv.AutoSizeColumnsMode = System.Windows.Forms.DataGridViewAutoSizeColumnsMode.Fill;
        }
    }

    private List<dynamic> GetSampleData()
    {
        return new List<dynamic>
        {
            new { Name = "Item 1", Value = 100 },
            new { Name = "Item 2", Value = 200 },
            new { Name = "Item 3", Value = 300 }
        };
    }
}
```

#### สรุปการเลือก WinForms vs WPF

```csharp
// Helper class สำหรับตัดสินใจเลือก framework
public static class UiFrameworkGuide
{
    /*
     * เลือก WinForms เมื่อ:
     * - ต้องการความเร็วในการพัฒนา
     * - แอปใช้งาน Controls มาตรฐาน (TextBox, Button, Grid)
     * - ต้องการ Win32 API interop ง่ายๆ
     * - ทีมมีประสบการณ์ WinForms
     * - แอป Legacy ที่ต้องการ maintain
     *
     * เลือก WPF เมื่อ:
     * - ต้องการ UI ที่ customize ได้สูง
     * - ใช้ MVVM pattern
     * - ต้องการ animation และ visual effects
     * - ต้องการ data binding ที่ซับซ้อน
     * - ใหม่ development ที่ต้องการ modern UI
     *
     * ใช้ร่วมกัน (Interop) เมื่อ:
     * - Migrate WinForms app ไป WPF ทีละส่วน
     * - ต้องการ WinForms control เฉพาะใน WPF app
     * - ต้องการ WPF visual ใน WinForms app
     */

    // Best practices สำหรับ modern WinForms development
    public static class BestPractices
    {
        // 1. ใช้ async/await แทน BackgroundWorker เสมอ
        // 2. ใช้ Progress<T> สำหรับ UI progress updates
        // 3. ใช้ DoubleBuffered = true ใน custom painting
        // 4. ใช้ BindingList<T> กับ INotifyPropertyChanged
        // 5. จัดการ Dispose ของ GDI+ objects ทุกครั้ง
        // 6. ใช้ using statement สำหรับ Pen, Brush, Font, Image
        // 7. ตั้งค่า ControlStyles สำหรับ custom controls
        // 8. ใช้ TableLayoutPanel / FlowLayoutPanel แทน absolute positioning
    }
}
```

---

## สรุปบทเรียน Part 78

### สิ่งที่เรียนรู้ในส่วนนี้

| Step | หัวข้อ | ความสำคัญ |
|------|--------|-----------|
| 771 | WinForms Architecture & GDI+ Basics | พื้นฐานสำคัญ |
| 772 | Custom Controls (UserControl & OnPaint) | สร้าง UI ที่ reusable |
| 773 | GDI+ Drawing (Pen, Brush, Shapes, Text, Images) | วาดกราฟิกขั้นสูง |
| 774 | Data Binding (BindingSource, BindingList<T>) | เชื่อมต่อข้อมูลกับ UI |
| 775 | DataGridView Advanced | แสดงข้อมูลตาราง |
| 776 | Async Operations (async/await, Progress<T>) | ป้องกัน UI freeze |
| 777 | Drag and Drop | UX ที่ดีขึ้น |
| 778 | MDI Applications | จัดการหลาย document |
| 779 | WinForms in .NET 8 | คุณสมบัติใหม่ |
| 780 | WinForms + WPF Interop | ทำงานร่วมกัน |

### Key Points ที่ต้องจำ

```csharp
// 1. DoubleBuffered ป้องกัน flickering ใน custom drawing
this.DoubleBuffered = true;

// 2. ControlStyles สำหรับ custom controls
this.SetStyle(ControlStyles.OptimizedDoubleBuffer |
              ControlStyles.AllPaintingInWmPaint |
              ControlStyles.UserPaint, true);

// 3. Invalidate() สั่งให้ repaint
this.Invalidate();          // repaint ทั้งหมด
this.Invalidate(region);    // repaint เฉพาะพื้นที่

// 4. Dispose GDI+ objects เสมอ
using var pen = new Pen(Color.Red, 2);
using var brush = new SolidBrush(Color.Blue);

// 5. Cross-thread UI update
this.Invoke(() => label.Text = "Updated");
this.BeginInvoke(() => listBox.Items.Add("Log"));

// 6. Progress<T> update UI จาก async task
var progress = new Progress<int>(pct => progressBar.Value = pct);
await Task.Run(() => DoWork(progress, ct));

// 7. BindingList<T> auto-notifies grid เมื่อข้อมูลเปลี่ยน
var list = new BindingList<MyModel>();
dgv.DataSource = list;
list.Add(new MyModel()); // grid update อัตโนมัติ
```

---

**ก่อนหน้า → [Part 77: WPF Advanced](part77-wpf-advanced.md)**
**ต่อไป → [Part 79: Code Quality](part79-code-quality.md)**
