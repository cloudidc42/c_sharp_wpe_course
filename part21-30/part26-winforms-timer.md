# Part 26: WinForms Timer, Background Tasks & Async UI
## ขั้นตอนที่ 251-260: Timer และ Background Processing

---

## 🎯 เป้าหมายของ Part นี้
- System.Windows.Forms.Timer (UI thread)
- System.Timers.Timer (background thread)
- BackgroundWorker pattern
- Task + Invoke สำหรับ UI update
- CancellationToken ใน WinForms
- Progress<T> ใน WinForms
- โปรแกรม File Downloader

---

## ขั้นตอนที่ 251: Windows Forms Timer

```csharp
// System.Windows.Forms.Timer - runs on UI thread, safe for UI updates
var uiTimer = new System.Windows.Forms.Timer { Interval = 1000 };
int seconds = 0;
var lblTimer = new Label { Text = "00:00:00", Font = new Font("Consolas", 18) };

uiTimer.Tick += (s,e) =>
{
    seconds++;
    TimeSpan ts = TimeSpan.FromSeconds(seconds);
    lblTimer.Text = ts.ToString(@"hh\:mm\:ss");
};

uiTimer.Start();

// Stop and reset
var btnStop = new Button { Text = "Stop" };
btnStop.Click += (s,e) =>
{
    uiTimer.Stop();
    seconds = 0;
};

// One-shot timer (fire once after delay)
void DelayAction(int ms, Action action)
{
    var t = new System.Windows.Forms.Timer { Interval = ms };
    t.Tick += (s,e) => { t.Stop(); t.Dispose(); action(); };
    t.Start();
}

// Show tooltip after 2 seconds
DelayAction(2000, () => toolTip.Show("Welcome!", this, 5000));

// Debounce search (wait for user to stop typing)
System.Windows.Forms.Timer? debounceTimer = null;
txtSearch.TextChanged += (s,e) =>
{
    debounceTimer?.Stop();
    debounceTimer = new System.Windows.Forms.Timer { Interval = 300 };
    debounceTimer.Tick += (s2,e2) =>
    {
        debounceTimer.Stop();
        PerformSearch(txtSearch.Text);
    };
    debounceTimer.Start();
};
```

---

## ขั้นตอนที่ 252: System.Timers.Timer

```csharp
// System.Timers.Timer - background thread, NOT safe for UI
var bgTimer = new System.Timers.Timer(5000); // every 5 seconds
bgTimer.AutoReset = true;
bgTimer.Elapsed += (s,e) =>
{
    // This runs on thread pool - need Invoke for UI
    var data = FetchDataFromServer();
    
    // Marshal back to UI thread
    this.Invoke(() =>
    {
        lblData.Text = data;
        listBox.Items.Add($"[{DateTime.Now:HH:mm:ss}] Updated");
    });
};
bgTimer.Start();

// Thread-safe UI update helper
void SafeInvoke(Control control, Action action)
{
    if (control.IsHandleCreated)
    {
        if (control.InvokeRequired)
            control.Invoke(action);
        else
            action();
    }
}

// Or using BeginInvoke (async - doesn't block caller)
bgTimer.Elapsed += (s,e) =>
{
    BeginInvoke(() =>
    {
        lblStatus.Text = "Updated!";
    });
};
```

---

## ขั้นตอนที่ 253: Async/Await ใน WinForms

```csharp
// async button click
private async void BtnLoad_Click(object? sender, EventArgs e)
{
    var btn = (Button)sender!;
    btn.Enabled = false;
    lblStatus.Text = "กำลังโหลด...";
    progressBar.Style = ProgressBarStyle.Marquee;
    
    try
    {
        // Long operation on background thread
        var data = await Task.Run(() => LoadDataFromDatabase());
        
        // Back on UI thread after await
        dataGridView.DataSource = data;
        lblStatus.Text = $"โหลดสำเร็จ: {data.Count} รายการ";
    }
    catch (Exception ex)
    {
        MessageBox.Show($"โหลดล้มเหลว: {ex.Message}", "Error", 
            MessageBoxButtons.OK, MessageBoxIcon.Error);
        lblStatus.Text = "Error";
    }
    finally
    {
        btn.Enabled = true;
        progressBar.Style = ProgressBarStyle.Continuous;
        progressBar.Value = 100;
    }
}

// With CancellationToken
private CancellationTokenSource? _cts;

private async void BtnProcess_Click(object? sender, EventArgs e)
{
    _cts?.Cancel();
    _cts = new CancellationTokenSource();
    var token = _cts.Token;
    
    btnProcess.Enabled = false;
    btnCancel.Enabled = true;
    
    var progress = new Progress<(int pct, string msg)>(p =>
    {
        progressBar.Value = p.pct;
        lblStatus.Text = p.msg;
    });
    
    try
    {
        await ProcessFilesAsync(progress, token);
        MessageBox.Show("เสร็จสิ้น!");
    }
    catch (OperationCanceledException)
    {
        lblStatus.Text = "ยกเลิกแล้ว";
    }
    finally
    {
        btnProcess.Enabled = true;
        btnCancel.Enabled = false;
        progressBar.Value = 0;
    }
}

private void BtnCancel_Click(object? sender, EventArgs e) => _cts?.Cancel();

async Task ProcessFilesAsync(IProgress<(int, string)> progress, CancellationToken ct)
{
    string[] files = Directory.GetFiles(@"C:\Data");
    for (int i = 0; i < files.Length; i++)
    {
        ct.ThrowIfCancellationRequested();
        await Task.Run(() => ProcessFile(files[i]), ct);
        progress.Report(((i + 1) * 100 / files.Length, $"Processing {i+1}/{files.Length}"));
    }
}
```

---

## ขั้นตอนที่ 254: BackgroundWorker (legacy but useful)

```csharp
// BackgroundWorker - older pattern but simpler for basic cases
var worker = new System.ComponentModel.BackgroundWorker
{
    WorkerReportsProgress = true,
    WorkerSupportsCancellation = true
};

worker.DoWork += (s,e) =>
{
    for (int i = 0; i <= 100; i += 10)
    {
        if (worker.CancellationPending)
        {
            e.Cancel = true;
            return;
        }
        Thread.Sleep(300);
        worker.ReportProgress(i, $"Processing {i}%");
    }
    e.Result = "Done!";
};

worker.ProgressChanged += (s,e) =>
{
    // UI thread - safe!
    progressBar.Value = e.ProgressPercentage;
    lblStatus.Text = e.UserState?.ToString() ?? "";
};

worker.RunWorkerCompleted += (s,e) =>
{
    btnStart.Enabled = true;
    if (e.Cancelled)
        lblStatus.Text = "Cancelled";
    else if (e.Error != null)
        MessageBox.Show(e.Error.Message);
    else
        MessageBox.Show(e.Result?.ToString());
};

btnStart.Click += (s,e) =>
{
    if (!worker.IsBusy)
    {
        btnStart.Enabled = false;
        worker.RunWorkerAsync();
    }
};

btnCancel.Click += (s,e) => worker.CancelAsync();
```

---

## ขั้นตอนที่ 255-260: โปรแกรม File Downloader

```csharp
// FileDownloader.cs
using System;
using System.Collections.Generic;
using System.Drawing;
using System.IO;
using System.Linq;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;
using System.Windows.Forms;

namespace FileDownloader;

record DownloadItem(string Url, string FileName, long TotalBytes = 0, long Downloaded = 0, string Status = "Waiting");

public class DownloaderForm : Form
{
    private ListView _lvDownloads = null!;
    private TextBox _txtUrl = null!;
    private Button _btnAdd = null!, _btnStart = null!, _btnCancelAll = null!, _btnClear = null!;
    private Label _lblOverall = null!;
    private ProgressBar _barOverall = null!;
    private StatusStrip _status = null!;
    private ToolStripStatusLabel _lblStatus = null!;
    
    private readonly List<(DownloadItem item, CancellationTokenSource cts, int rowIndex)> _queue = new();
    private readonly HttpClient _http = new();
    private int _activeCount = 0;
    private const int MaxParallel = 3;
    
    public DownloaderForm()
    {
        InitializeUI();
        AddSampleUrls();
    }
    
    private void InitializeUI()
    {
        Text = "File Downloader";
        Size = new Size(800, 550);
        StartPosition = FormStartPosition.CenterScreen;
        
        // URL input panel
        var inputPanel = new Panel { Dock = DockStyle.Top, Height = 50, Padding = new Padding(8) };
        _txtUrl = new TextBox { Location = new Point(8, 12), Width = 550, PlaceholderText = "https://example.com/file.zip" };
        _btnAdd = new Button { Text = "➕ Add", Location = new Point(570, 10), Size = new Size(80, 26), BackColor = Color.SteelBlue, ForeColor = Color.White, FlatStyle = FlatStyle.Flat };
        _btnAdd.FlatAppearance.BorderSize = 0;
        _btnAdd.Click += (s,e) => AddUrl(_txtUrl.Text.Trim());
        inputPanel.Controls.AddRange(new Control[] { _txtUrl, _btnAdd });
        
        // Toolbar
        var toolbar = new Panel { Dock = DockStyle.Top, Height = 42, BackColor = Color.WhiteSmoke, Padding = new Padding(5) };
        _btnStart = MakeBtn("▶ Download All", Color.FromArgb(39, 174, 96), 5);
        _btnCancelAll = MakeBtn("⏹ Cancel All", Color.FromArgb(231, 76, 60), 160);
        _btnClear = MakeBtn("🗑 Clear Done", Color.FromArgb(149, 165, 166), 315);
        _btnStart.Click += (s,e) => StartAll();
        _btnCancelAll.Click += (s,e) => CancelAll();
        _btnClear.Click += (s,e) => ClearCompleted();
        toolbar.Controls.AddRange(new Control[] { _btnStart, _btnCancelAll, _btnClear });
        
        // ListView
        _lvDownloads = new ListView
        {
            Dock = DockStyle.Fill,
            View = View.Details,
            FullRowSelect = true,
            GridLines = false,
            RowHeadersVisible = false,
            Font = new Font("Segoe UI", 9)
        };
        _lvDownloads.Columns.Add("File Name", 220);
        _lvDownloads.Columns.Add("Size", 90);
        _lvDownloads.Columns.Add("Progress", 160);
        _lvDownloads.Columns.Add("Speed", 90);
        _lvDownloads.Columns.Add("Status", 130);
        
        // Custom draw progress bar in list
        _lvDownloads.OwnerDraw = true;
        _lvDownloads.DrawColumnHeader += (s,e) => e.DrawDefault = true;
        _lvDownloads.DrawItem += (s,e) => e.DrawDefault = true;
        _lvDownloads.DrawSubItem += LvDownloads_DrawSubItem;
        
        // Overall progress
        var bottomPanel = new Panel { Dock = DockStyle.Bottom, Height = 40, Padding = new Padding(8, 5, 8, 5) };
        _lblOverall = new Label { Text = "รวม: 0/0", Location = new Point(8, 8), AutoSize = true };
        _barOverall = new ProgressBar { Location = new Point(100, 7), Size = new Size(450, 22), Style = ProgressBarStyle.Continuous };
        bottomPanel.Controls.AddRange(new Control[] { _lblOverall, _barOverall });
        
        // Status
        _status = new StatusStrip();
        _lblStatus = new ToolStripStatusLabel("Ready") { Spring = true };
        _status.Items.Add(_lblStatus);
        
        Controls.Add(_lvDownloads);
        Controls.Add(bottomPanel);
        Controls.Add(toolbar);
        Controls.Add(inputPanel);
        Controls.Add(_status);
    }
    
    private Button MakeBtn(string text, Color color, int x)
    {
        var b = new Button { Text = text, Location = new Point(x, 6), Size = new Size(145, 30), BackColor = color, ForeColor = Color.White, FlatStyle = FlatStyle.Flat, Cursor = Cursors.Hand };
        b.FlatAppearance.BorderSize = 0;
        return b;
    }
    
    private void LvDownloads_DrawSubItem(object? sender, DrawListViewSubItemEventArgs e)
    {
        if (e.ColumnIndex != 2) { e.DrawDefault = true; return; }
        
        // Draw progress bar
        var item = (DownloadItem?)e.Item?.Tag;
        if (item == null) { e.DrawDefault = true; return; }
        
        var rect = e.Bounds;
        rect.Inflate(-2, -3);
        
        e.Graphics.FillRectangle(new SolidBrush(Color.LightGray), rect);
        
        if (item.TotalBytes > 0)
        {
            float pct = (float)item.Downloaded / item.TotalBytes;
            int fillW = (int)(rect.Width * pct);
            if (fillW > 0)
                e.Graphics.FillRectangle(new SolidBrush(Color.SteelBlue), rect.X, rect.Y, fillW, rect.Height);
            
            string text = $"{pct:P0}";
            var sf = new StringFormat { Alignment = StringAlignment.Center, LineAlignment = StringAlignment.Center };
            e.Graphics.DrawString(text, e.Item!.Font, Brushes.Black, rect, sf);
        }
        
        e.Graphics.DrawRectangle(Pens.Silver, rect);
    }
    
    private void AddSampleUrls()
    {
        // Sample files for testing
        AddUrl("https://speed.hetzner.de/100MB.bin");
        AddUrl("https://proof.ovh.net/files/10Mb.dat");
    }
    
    private void AddUrl(string url)
    {
        if (string.IsNullOrWhiteSpace(url)) return;
        
        string fileName = Uri.TryCreate(url, UriKind.Absolute, out var uri) 
            ? Path.GetFileName(uri.LocalPath) 
            : $"file_{_queue.Count + 1}";
        if (string.IsNullOrEmpty(fileName)) fileName = $"download_{_queue.Count + 1}";
        
        var item = new DownloadItem(url, fileName);
        var lvi = new ListViewItem(fileName);
        lvi.SubItems.AddRange(new[] { "Unknown", "", "", "⏳ Waiting" });
        lvi.Tag = item;
        
        _lvDownloads.Items.Add(lvi);
        _queue.Add((item, new CancellationTokenSource(), _lvDownloads.Items.Count - 1));
        
        _txtUrl.Clear();
        UpdateOverall();
    }
    
    private async void StartAll()
    {
        var waiting = _queue.Where(q => q.item.Status == "Waiting").ToList();
        
        var semaphore = new SemaphoreSlim(MaxParallel);
        var tasks = waiting.Select(async q =>
        {
            await semaphore.WaitAsync(q.cts.Token);
            try
            {
                Interlocked.Increment(ref _activeCount);
                await DownloadFileAsync(q.item, q.cts.Token, q.rowIndex);
            }
            finally
            {
                Interlocked.Decrement(ref _activeCount);
                semaphore.Release();
                UpdateOverall();
            }
        });
        
        await Task.WhenAll(tasks);
        _lblStatus.Text = "All downloads completed";
    }
    
    private async Task DownloadFileAsync(DownloadItem item, CancellationToken ct, int rowIndex)
    {
        UpdateRow(rowIndex, null, null, null, "⬇️ Downloading");
        
        try
        {
            string savePath = Path.Combine(
                Environment.GetFolderPath(Environment.SpecialFolder.Desktop),
                item.FileName);
            
            using var response = await _http.GetAsync(item.Url, HttpCompletionOption.ResponseHeadersRead, ct);
            response.EnsureSuccessStatusCode();
            
            long total = response.Content.Headers.ContentLength ?? 0;
            UpdateRow(rowIndex, FormatSize(total), null, null, null);
            
            using var stream = await response.Content.ReadAsStreamAsync(ct);
            using var file = File.Create(savePath);
            
            var buffer = new byte[8192];
            long downloaded = 0;
            var startTime = DateTime.Now;
            int read;
            
            while ((read = await stream.ReadAsync(buffer, ct)) > 0)
            {
                await file.WriteAsync(buffer.AsMemory(0, read), ct);
                downloaded += read;
                
                double seconds = (DateTime.Now - startTime).TotalSeconds;
                string speed = seconds > 0 ? FormatSize((long)(downloaded / seconds)) + "/s" : "";
                
                UpdateRow(rowIndex, FormatSize(total),
                    new DownloadItem(item.Url, item.FileName, total, downloaded, "Downloading"),
                    speed, null);
            }
            
            UpdateRow(rowIndex, FormatSize(downloaded),
                new DownloadItem(item.Url, item.FileName, downloaded, downloaded, "Done"),
                "", "✅ Done");
        }
        catch (OperationCanceledException)
        {
            UpdateRow(rowIndex, null, null, null, "⏹ Cancelled");
        }
        catch (Exception ex)
        {
            UpdateRow(rowIndex, null, null, null, $"❌ {ex.Message[..Math.Min(20, ex.Message.Length)]}");
        }
    }
    
    private void UpdateRow(int rowIndex, string? size, DownloadItem? progressItem, string? speed, string? status)
    {
        if (InvokeRequired) { Invoke(() => UpdateRow(rowIndex, size, progressItem, speed, status)); return; }
        if (rowIndex >= _lvDownloads.Items.Count) return;
        
        var lvi = _lvDownloads.Items[rowIndex];
        if (size != null) lvi.SubItems[1].Text = size;
        if (progressItem != null) { lvi.Tag = progressItem; _lvDownloads.Invalidate(lvi.Bounds); }
        if (speed != null) lvi.SubItems[3].Text = speed;
        if (status != null) lvi.SubItems[4].Text = status;
    }
    
    private void CancelAll()
    {
        foreach (var (_, cts, _) in _queue) cts.Cancel();
    }
    
    private void ClearCompleted()
    {
        var done = _queue.Where(q => 
            q.item.Status == "Done" || q.item.Status == "Cancelled").ToList();
        foreach (var q in done)
        {
            _lvDownloads.Items.RemoveAt(q.rowIndex);
            _queue.Remove(q);
        }
    }
    
    private void UpdateOverall()
    {
        if (InvokeRequired) { Invoke(UpdateOverall); return; }
        int done = _queue.Count(q => q.item.Status == "Done");
        int total = _queue.Count;
        _lblOverall.Text = $"รวม: {done}/{total}";
        _barOverall.Value = total > 0 ? done * 100 / total : 0;
        _lblStatus.Text = $"Active: {_activeCount} | Done: {done}/{total}";
    }
    
    private static string FormatSize(long bytes) => bytes switch
    {
        >= 1_073_741_824 => $"{bytes / 1_073_741_824.0:F1} GB",
        >= 1_048_576 => $"{bytes / 1_048_576.0:F1} MB",
        >= 1_024 => $"{bytes / 1_024.0:F1} KB",
        _ => $"{bytes} B"
    };
}
```

---

## 📝 สรุป Part 26

| หัวข้อ | ใช้เมื่อ |
|--------|---------|
| WinForms Timer | Animation, periodic UI updates |
| Timers.Timer | Background polling |
| async/await | Long operations ไม่ block UI |
| InvokeRequired/Invoke | Update UI จาก background thread |
| Progress<T> | รายงาน progress จาก async |
| CancellationToken | ยกเลิก async operation |
| SemaphoreSlim | จำกัด parallel tasks |

---

**ก่อนหน้า → [Part 25: WinForms Graphics](part25-winforms-graphics.md)**  
**ต่อไป → [Part 27: WinForms MDI & Docking](part27-winforms-mdi.md)**
