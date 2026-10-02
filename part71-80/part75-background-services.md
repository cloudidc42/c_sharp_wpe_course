# Part 75: Background Services & Scheduled Jobs (Steps 741-750)

บทนี้จะครอบคลุมการสร้าง Background Services และ Scheduled Jobs ใน .NET ซึ่งเป็นส่วนสำคัญของแอปพลิเคชันที่ต้องทำงานเบื้องหลัง เช่น การส่งอีเมล การประมวลผลไฟล์ การทำ cleanup และงานที่ต้องทำซ้ำตามเวลาที่กำหนด

---

## Step 741: IHostedService vs BackgroundService และ Lifecycle

### ความแตกต่างระหว่าง IHostedService และ BackgroundService

ใน .NET มี 2 วิธีหลักในการสร้าง background service:

**IHostedService** เป็น interface พื้นฐานที่ให้ control เต็มที่ในการจัดการ lifecycle ของ service
**BackgroundService** เป็น abstract class ที่ implement `IHostedService` ให้บางส่วนแล้ว เหมาะสำหรับงานที่ทำงานต่อเนื่อง

```csharp
// IHostedService - ให้ control เต็มที่
public class MyHostedService : IHostedService, IDisposable
{
    private readonly ILogger<MyHostedService> _logger;
    private Timer? _timer;

    public MyHostedService(ILogger<MyHostedService> logger)
    {
        _logger = logger;
    }

    // เรียกเมื่อ application เริ่มต้น
    public Task StartAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("MyHostedService starting...");
        
        // เริ่ม timer ทุก 5 วินาที
        _timer = new Timer(DoWork, null, TimeSpan.Zero, TimeSpan.FromSeconds(5));
        
        return Task.CompletedTask;
    }

    private void DoWork(object? state)
    {
        _logger.LogInformation("MyHostedService working at: {time}", DateTimeOffset.Now);
    }

    // เรียกเมื่อ application หยุด
    public Task StopAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("MyHostedService stopping...");
        
        // หยุด timer
        _timer?.Change(Timeout.Infinite, 0);
        
        return Task.CompletedTask;
    }

    public void Dispose()
    {
        _timer?.Dispose();
    }
}
```

```csharp
// BackgroundService - abstract class ที่ใช้งานง่ายกว่า
public class MyBackgroundService : BackgroundService
{
    private readonly ILogger<MyBackgroundService> _logger;

    public MyBackgroundService(ILogger<MyBackgroundService> logger)
    {
        _logger = logger;
    }

    // Method หลักที่ต้อง implement - ทำงานใน background loop
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("MyBackgroundService started");

        // Loop จนกว่าจะมีการ cancel
        while (!stoppingToken.IsCancellationRequested)
        {
            _logger.LogInformation("Working at: {time}", DateTimeOffset.Now);
            
            try
            {
                await DoWorkAsync(stoppingToken);
            }
            catch (OperationCanceledException)
            {
                // Normal shutdown - ไม่ต้อง log error
                break;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error in background service");
            }

            // รอ 10 วินาทีก่อนทำงานครั้งต่อไป
            await Task.Delay(TimeSpan.FromSeconds(10), stoppingToken);
        }

        _logger.LogInformation("MyBackgroundService stopped");
    }

    private async Task DoWorkAsync(CancellationToken cancellationToken)
    {
        // งานที่ต้องทำใน background
        await Task.Delay(100, cancellationToken);
    }
}
```

### Lifecycle ของ Background Service

```csharp
// Lifecycle events ของ hosted service
public class LifecycleAwareService : BackgroundService
{
    private readonly ILogger<LifecycleAwareService> _logger;
    private readonly IHostApplicationLifetime _lifetime;

    public LifecycleAwareService(
        ILogger<LifecycleAwareService> logger,
        IHostApplicationLifetime lifetime)
    {
        _logger = logger;
        _lifetime = lifetime;

        // ลงทะเบียน callbacks สำหรับ lifecycle events
        _lifetime.ApplicationStarted.Register(OnStarted);
        _lifetime.ApplicationStopping.Register(OnStopping);
        _lifetime.ApplicationStopped.Register(OnStopped);
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // รอจนกว่า application จะ started จริงๆ
        await Task.Delay(Timeout.Infinite, stoppingToken);
    }

    private void OnStarted()
    {
        _logger.LogInformation("Application has fully started");
    }

    private void OnStopping()
    {
        _logger.LogInformation("Application is stopping...");
    }

    private void OnStopped()
    {
        _logger.LogInformation("Application has stopped");
    }
}
```

### การลงทะเบียน Background Service ใน DI Container

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// ลงทะเบียน hosted services
builder.Services.AddHostedService<MyBackgroundService>();
builder.Services.AddHostedService<MyHostedService>();

// หรือใช้ factory pattern สำหรับ complex setup
builder.Services.AddHostedService(provider =>
{
    var logger = provider.GetRequiredService<ILogger<MyBackgroundService>>();
    return new MyBackgroundService(logger);
});

var app = builder.Build();
app.Run();
```

---

## Step 742: Periodic Background Worker (Polling, Cleanup Jobs)

### การใช้ PeriodicTimer (.NET 6+)

`PeriodicTimer` เป็น API ใหม่ที่ทำงานได้ดีกว่า `Timer` แบบเดิม เพราะ await-friendly และไม่ต้องกังวลเรื่อง thread safety

```csharp
public class PeriodicCleanupWorker : BackgroundService
{
    private readonly ILogger<PeriodicCleanupWorker> _logger;
    private readonly IServiceProvider _serviceProvider;
    private readonly TimeSpan _period = TimeSpan.FromHours(1);

    public PeriodicCleanupWorker(
        ILogger<PeriodicCleanupWorker> logger,
        IServiceProvider serviceProvider)
    {
        _logger = logger;
        _serviceProvider = serviceProvider;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Cleanup worker started, will run every {Period}", _period);

        // PeriodicTimer ทำงานได้ดีกว่า Task.Delay ในบางกรณี
        using var timer = new PeriodicTimer(_period);

        // ทำงานครั้งแรกทันที ไม่ต้องรอ period แรก
        await DoCleanupAsync(stoppingToken);

        // รอ timer และทำซ้ำ
        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            await DoCleanupAsync(stoppingToken);
        }
    }

    private async Task DoCleanupAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("Starting cleanup at {Time}", DateTimeOffset.Now);

        // ใช้ scoped service ผ่าน IServiceScopeFactory
        using var scope = _serviceProvider.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();

        try
        {
            // ลบ records เก่าที่หมดอายุ
            var cutoffDate = DateTime.UtcNow.AddDays(-30);
            var expiredCount = await dbContext.TempFiles
                .Where(f => f.CreatedAt < cutoffDate)
                .ExecuteDeleteAsync(cancellationToken);

            _logger.LogInformation("Cleaned up {Count} expired records", expiredCount);

            // ลบ sessions ที่หมดอายุ
            var expiredSessions = await dbContext.Sessions
                .Where(s => s.ExpiresAt < DateTime.UtcNow)
                .ExecuteDeleteAsync(cancellationToken);

            _logger.LogInformation("Cleaned up {Count} expired sessions", expiredSessions);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error during cleanup");
        }
    }
}
```

### Polling Worker - ตรวจสอบ Queue หรือ Database

```csharp
public class DatabasePollingWorker : BackgroundService
{
    private readonly ILogger<DatabasePollingWorker> _logger;
    private readonly IServiceProvider _serviceProvider;
    
    // Adaptive polling intervals
    private static readonly TimeSpan ShortInterval = TimeSpan.FromSeconds(1);
    private static readonly TimeSpan LongInterval = TimeSpan.FromSeconds(30);

    public DatabasePollingWorker(
        ILogger<DatabasePollingWorker> logger,
        IServiceProvider serviceProvider)
    {
        _logger = logger;
        _serviceProvider = serviceProvider;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var hasWork = await PollForWorkAsync(stoppingToken);
            
            // Adaptive polling: ถ้ามีงาน poll บ่อยขึ้น ถ้าไม่มีงาน poll ช้าลง
            var delay = hasWork ? ShortInterval : LongInterval;
            
            try
            {
                await Task.Delay(delay, stoppingToken);
            }
            catch (OperationCanceledException)
            {
                break;
            }
        }
    }

    private async Task<bool> PollForWorkAsync(CancellationToken cancellationToken)
    {
        using var scope = _serviceProvider.CreateScope();
        var queue = scope.ServiceProvider.GetRequiredService<IWorkQueue>();

        var workItem = await queue.DequeueAsync(cancellationToken);
        
        if (workItem == null)
            return false;

        _logger.LogInformation("Processing work item: {Id}", workItem.Id);
        
        try
        {
            await workItem.ProcessAsync(cancellationToken);
            await queue.CompleteAsync(workItem.Id, cancellationToken);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to process work item {Id}", workItem.Id);
            await queue.FailAsync(workItem.Id, ex.Message, cancellationToken);
        }

        return true;
    }
}
```

---

## Step 743: Channel-based Worker (Producer/Consumer Pattern)

### Channel ใน .NET สำหรับ Producer/Consumer

`System.Threading.Channels` ให้ thread-safe, async-friendly channel สำหรับส่งข้อมูลระหว่าง threads

```csharp
// Message model
public record WorkItem(Guid Id, string Type, string Payload);

// Channel service - เป็น singleton
public class WorkChannel
{
    private readonly Channel<WorkItem> _channel;

    public WorkChannel()
    {
        // Bounded channel - จำกัดจำนวน items ใน buffer
        _channel = Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(100)
        {
            FullMode = BoundedChannelFullMode.Wait, // รอถ้า channel เต็ม
            SingleReader = false,
            SingleWriter = false
        });
    }

    public ChannelReader<WorkItem> Reader => _channel.Reader;
    public ChannelWriter<WorkItem> Writer => _channel.Writer;
}

// Producer - ส่งงานเข้า channel
public class WorkProducer
{
    private readonly WorkChannel _channel;
    private readonly ILogger<WorkProducer> _logger;

    public WorkProducer(WorkChannel channel, ILogger<WorkProducer> logger)
    {
        _channel = channel;
        _logger = logger;
    }

    public async Task EnqueueAsync(WorkItem item, CancellationToken cancellationToken = default)
    {
        _logger.LogInformation("Enqueueing work item: {Id}", item.Id);
        
        // WriteAsync จะ block ถ้า channel เต็ม (BoundedChannel)
        await _channel.Writer.WriteAsync(item, cancellationToken);
    }

    public bool TryEnqueue(WorkItem item)
    {
        return _channel.Writer.TryWrite(item);
    }
}

// Consumer - รับงานจาก channel และประมวลผล
public class WorkConsumerService : BackgroundService
{
    private readonly WorkChannel _channel;
    private readonly ILogger<WorkConsumerService> _logger;
    private readonly IServiceProvider _serviceProvider;

    public WorkConsumerService(
        WorkChannel channel,
        ILogger<WorkConsumerService> logger,
        IServiceProvider serviceProvider)
    {
        _channel = channel;
        _logger = logger;
        _serviceProvider = serviceProvider;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Work consumer started");

        // อ่านจาก channel จนกว่าจะถูก cancel
        await foreach (var item in _channel.Reader.ReadAllAsync(stoppingToken))
        {
            await ProcessItemAsync(item, stoppingToken);
        }
    }

    private async Task ProcessItemAsync(WorkItem item, CancellationToken cancellationToken)
    {
        _logger.LogInformation("Processing: {Id} ({Type})", item.Id, item.Type);

        using var scope = _serviceProvider.CreateScope();
        
        try
        {
            var processor = scope.ServiceProvider
                .GetRequiredKeyedService<IWorkProcessor>(item.Type);
            
            await processor.ProcessAsync(item.Payload, cancellationToken);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to process item {Id}", item.Id);
        }
    }
}
```

### Multiple Consumers (Fan-out Pattern)

```csharp
// Worker pool สำหรับ parallel processing
public class WorkerPoolService : BackgroundService
{
    private readonly WorkChannel _channel;
    private readonly ILogger<WorkerPoolService> _logger;
    private readonly IServiceProvider _serviceProvider;
    private const int WorkerCount = 4; // จำนวน concurrent workers

    public WorkerPoolService(
        WorkChannel channel,
        ILogger<WorkerPoolService> logger,
        IServiceProvider serviceProvider)
    {
        _channel = channel;
        _logger = logger;
        _serviceProvider = serviceProvider;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Starting {Count} workers", WorkerCount);

        // สร้าง worker tasks ทำงานพร้อมกัน
        var workers = Enumerable.Range(0, WorkerCount)
            .Select(i => RunWorkerAsync(i, stoppingToken))
            .ToArray();

        await Task.WhenAll(workers);
        
        _logger.LogInformation("All workers stopped");
    }

    private async Task RunWorkerAsync(int workerId, CancellationToken cancellationToken)
    {
        _logger.LogInformation("Worker {Id} started", workerId);

        await foreach (var item in _channel.Reader.ReadAllAsync(cancellationToken))
        {
            _logger.LogDebug("Worker {WorkerId} processing item {ItemId}", workerId, item.Id);
            
            using var scope = _serviceProvider.CreateScope();
            
            try
            {
                var handler = scope.ServiceProvider.GetRequiredService<IWorkItemHandler>();
                await handler.HandleAsync(item, cancellationToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Worker {WorkerId} failed on item {ItemId}", workerId, item.Id);
            }
        }
    }
}
```

### การลงทะเบียนใน DI

```csharp
// Program.cs
builder.Services.AddSingleton<WorkChannel>();
builder.Services.AddSingleton<WorkProducer>();
builder.Services.AddHostedService<WorkerPoolService>();

// หรือถ้าต้องการ multiple consumers แยก instances
builder.Services.AddHostedService<WorkConsumerService>();
builder.Services.AddHostedService<WorkConsumerService>(); // เพิ่มอีก consumer
```

---

## Step 744: Quartz.NET Scheduler (IJob, CronTrigger, JobDataMap)

### การติดตั้ง Quartz.NET

```bash
dotnet add package Quartz
dotnet add package Quartz.Extensions.Hosting
dotnet add package Quartz.Extensions.DependencyInjection
```

### การสร้าง Job ด้วย IJob

```csharp
using Quartz;

// Job สำหรับส่ง report ประจำวัน
[DisallowConcurrentExecution] // ป้องกันไม่ให้ run ซ้อนกัน
public class DailyReportJob : IJob
{
    private readonly ILogger<DailyReportJob> _logger;
    private readonly IReportService _reportService;
    private readonly IEmailService _emailService;

    public DailyReportJob(
        ILogger<DailyReportJob> logger,
        IReportService reportService,
        IEmailService emailService)
    {
        _logger = logger;
        _reportService = reportService;
        _emailService = emailService;
    }

    public async Task Execute(IJobExecutionContext context)
    {
        _logger.LogInformation("DailyReportJob executing at {Time}", DateTimeOffset.Now);

        // ดึงข้อมูลจาก JobDataMap
        var jobData = context.JobDetail.JobDataMap;
        var recipients = jobData.GetString("recipients")?.Split(',') 
            ?? Array.Empty<string>();
        var reportType = jobData.GetString("reportType") ?? "summary";

        try
        {
            // สร้าง report
            var report = await _reportService.GenerateAsync(reportType);
            
            // ส่งอีเมลไปทุก recipients
            foreach (var recipient in recipients)
            {
                await _emailService.SendAsync(new EmailMessage
                {
                    To = recipient,
                    Subject = $"Daily Report - {DateTime.Today:yyyy-MM-dd}",
                    Body = report.HtmlContent
                });
            }

            _logger.LogInformation("DailyReportJob completed, sent to {Count} recipients", 
                recipients.Length);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "DailyReportJob failed");
            
            // Throw JobExecutionException เพื่อให้ Quartz จัดการ retry
            throw new JobExecutionException(ex, refireImmediately: false);
        }
    }
}
```

### การ Configure Quartz ใน Program.cs

```csharp
// Program.cs
builder.Services.AddQuartz(q =>
{
    // ใช้ DI สำหรับสร้าง jobs
    q.UseMicrosoftDependencyInjectionJobFactory();

    // กำหนด scheduler name
    q.SchedulerId = "MyAppScheduler";
    q.SchedulerName = "My Application Scheduler";

    // Job 1: Daily Report ทุกวัน เวลา 8:00 AM
    var dailyReportJobKey = new JobKey("DailyReport", "Reports");
    
    q.AddJob<DailyReportJob>(opts => opts
        .WithIdentity(dailyReportJobKey)
        .UsingJobData("recipients", "admin@example.com,cto@example.com")
        .UsingJobData("reportType", "summary")
        .StoreDurably()); // เก็บ job แม้ไม่มี trigger

    q.AddTrigger(opts => opts
        .ForJob(dailyReportJobKey)
        .WithIdentity("DailyReportTrigger", "Reports")
        .WithCronSchedule("0 0 8 * * ?") // ทุกวัน เวลา 8:00 AM
        .WithDescription("Sends daily summary report"));

    // Job 2: Cleanup ทุกคืน เวลา 2:00 AM
    var cleanupJobKey = new JobKey("DbCleanup", "Maintenance");
    
    q.AddJob<DatabaseCleanupJob>(opts => opts
        .WithIdentity(cleanupJobKey)
        .UsingJobData("retentionDays", 30));

    q.AddTrigger(opts => opts
        .ForJob(cleanupJobKey)
        .WithIdentity("DbCleanupTrigger", "Maintenance")
        .WithCronSchedule("0 0 2 * * ?") // ทุกวัน เวลา 2:00 AM
        .StartAt(DateBuilder.TomorrowAt(2, 0, 0))); // เริ่มพรุ่งนี้

    // Job 3: Health check ทุก 5 นาที
    q.AddJob<SystemHealthCheckJob>(opts => opts
        .WithIdentity("HealthCheck", "Monitoring"));

    q.AddTrigger(opts => opts
        .ForJob("HealthCheck", "Monitoring")
        .WithIdentity("HealthCheckTrigger", "Monitoring")
        .WithSimpleSchedule(x => x
            .WithIntervalInMinutes(5)
            .RepeatForever()));
});

// เพิ่ม Quartz Hosted Service
builder.Services.AddQuartzHostedService(q =>
{
    q.WaitForJobsToComplete = true; // รองานให้เสร็จก่อน shutdown
});
```

### JobDataMap และ Dynamic Job Scheduling

```csharp
// Dynamic scheduling จาก code
public class SchedulerManager
{
    private readonly ISchedulerFactory _schedulerFactory;
    private readonly ILogger<SchedulerManager> _logger;

    public SchedulerManager(
        ISchedulerFactory schedulerFactory,
        ILogger<SchedulerManager> logger)
    {
        _schedulerFactory = schedulerFactory;
        _logger = logger;
    }

    public async Task ScheduleJobAsync(
        string jobName,
        string cronExpression,
        Dictionary<string, object> data)
    {
        var scheduler = await _schedulerFactory.GetScheduler();

        var jobKey = new JobKey(jobName);
        var job = JobBuilder.Create<DynamicJob>()
            .WithIdentity(jobKey)
            .UsingJobData(new JobDataMap(data))
            .Build();

        var trigger = TriggerBuilder.Create()
            .WithIdentity($"{jobName}Trigger")
            .WithCronSchedule(cronExpression)
            .Build();

        // ลบ job เก่าถ้ามี
        await scheduler.DeleteJob(jobKey);
        
        // เพิ่ม job ใหม่
        await scheduler.ScheduleJob(job, trigger);
        
        _logger.LogInformation("Scheduled job {Name} with cron {Cron}", jobName, cronExpression);
    }

    public async Task PauseJobAsync(string jobName)
    {
        var scheduler = await _schedulerFactory.GetScheduler();
        await scheduler.PauseJob(new JobKey(jobName));
    }

    public async Task ResumeJobAsync(string jobName)
    {
        var scheduler = await _schedulerFactory.GetScheduler();
        await scheduler.ResumeJob(new JobKey(jobName));
    }

    public async Task TriggerJobNowAsync(string jobName)
    {
        var scheduler = await _schedulerFactory.GetScheduler();
        await scheduler.TriggerJob(new JobKey(jobName));
    }
}
```

---

## Step 745: Hangfire (Enqueue, Schedule, Recurring Jobs, Dashboard)

### การติดตั้ง Hangfire

```bash
dotnet add package Hangfire.Core
dotnet add package Hangfire.AspNetCore
dotnet add package Hangfire.SqlServer  # หรือ Hangfire.InMemory สำหรับ dev
```

### การ Configure Hangfire

```csharp
// Program.cs
builder.Services.AddHangfire(config => config
    .SetDataCompatibilityLevel(CompatibilityLevel.Version_180)
    .UseSimpleAssemblyNameTypeSerializer()
    .UseRecommendedSerializerSettings()
    .UseSqlServerStorage(
        builder.Configuration.GetConnectionString("HangfireConnection"),
        new SqlServerStorageOptions
        {
            CommandBatchMaxTimeout = TimeSpan.FromMinutes(5),
            SlidingInvisibilityTimeout = TimeSpan.FromMinutes(5),
            QueuePollInterval = TimeSpan.Zero,
            UseRecommendedIsolationLevel = true,
            DisableGlobalLocks = true
        }));

// เพิ่ม Hangfire Server สำหรับประมวลผล jobs
builder.Services.AddHangfireServer(options =>
{
    options.WorkerCount = Environment.ProcessorCount * 2;
    options.Queues = new[] { "critical", "default", "low" };
});

var app = builder.Build();

// เพิ่ม Hangfire Dashboard (ป้องกันด้วย Authorization)
app.UseHangfireDashboard("/hangfire", new DashboardOptions
{
    Authorization = new[] { new HangfireAuthorizationFilter() }
});
```

### การใช้งาน Hangfire Background Job Client

```csharp
public class OrderService
{
    private readonly IBackgroundJobClient _backgroundJobClient;
    private readonly IRecurringJobManager _recurringJobManager;
    private readonly ILogger<OrderService> _logger;

    public OrderService(
        IBackgroundJobClient backgroundJobClient,
        IRecurringJobManager recurringJobManager,
        ILogger<OrderService> logger)
    {
        _backgroundJobClient = backgroundJobClient;
        _recurringJobManager = recurringJobManager;
        _logger = logger;
    }

    public async Task<Guid> CreateOrderAsync(CreateOrderRequest request)
    {
        var orderId = Guid.NewGuid();
        
        // 1. Fire-and-Forget: ส่งอีเมลยืนยัน (ไม่ต้องรอผล)
        _backgroundJobClient.Enqueue<IEmailService>(
            service => service.SendOrderConfirmationAsync(orderId.ToString()));

        // 2. Scheduled Job: ส่ง reminder ถ้ายังไม่จ่ายเงินใน 24 ชั่วโมง
        _backgroundJobClient.Schedule<IEmailService>(
            service => service.SendPaymentReminderAsync(orderId.ToString()),
            TimeSpan.FromHours(24));

        // 3. Chained Job: หลังจาก job แรก เสร็จ ทำ job ถัดไป
        var emailJobId = _backgroundJobClient.Enqueue<IEmailService>(
            service => service.SendOrderConfirmationAsync(orderId.ToString()));
        
        _backgroundJobClient.ContinueJobWith<IInventoryService>(
            emailJobId,
            service => service.ReserveInventoryAsync(orderId.ToString()));

        return orderId;
    }

    public void SetupRecurringJobs()
    {
        // 4. Recurring Job: สร้างรายงานทุกวันจันทร์ เวลา 9:00 AM
        _recurringJobManager.AddOrUpdate<IReportService>(
            "weekly-sales-report",
            service => service.GenerateWeeklySalesReportAsync(),
            "0 9 * * MON"); // Cron expression

        // 5. Recurring Job: cleanup ทุกคืนเวลาเที่ยงคืน
        _recurringJobManager.AddOrUpdate<ICleanupService>(
            "nightly-cleanup",
            service => service.CleanupOldDataAsync(),
            Cron.Daily(0, 0)); // ใช้ Cron helpers

        // 6. Recurring Job ใน Queue เฉพาะ
        _recurringJobManager.AddOrUpdate<IHealthCheckService>(
            "health-check",
            service => service.CheckAllServicesAsync(),
            Cron.Minutely(),
            new RecurringJobOptions
            {
                QueueName = "critical"
            });
    }
}
```

### Hangfire Authorization Filter

```csharp
public class HangfireAuthorizationFilter : IDashboardAuthorizationFilter
{
    public bool Authorize(DashboardContext context)
    {
        var httpContext = context.GetHttpContext();
        
        // อนุญาตเฉพาะ admin users
        return httpContext.User.Identity?.IsAuthenticated == true
            && httpContext.User.IsInRole("Admin");
    }
}
```

---

## Step 746: Background Job Patterns

### Fire-and-Forget Pattern

```csharp
// Pattern: ทำงานโดยไม่สนผลลัพธ์ ไม่รอ
public class NotificationService
{
    private readonly IBackgroundJobClient _jobClient;

    public NotificationService(IBackgroundJobClient jobClient)
    {
        _jobClient = jobClient;
    }

    // Fire and forget - API response เร็ว แต่ notification ส่งใน background
    public async Task<IActionResult> RegisterUserAsync(RegisterRequest request)
    {
        var userId = await CreateUserAsync(request);
        
        // ไม่รอ - ส่ง notification ใน background
        _jobClient.Enqueue<IWelcomeEmailSender>(
            s => s.SendWelcomeEmailAsync(userId));
        
        _jobClient.Enqueue<ISlackNotifier>(
            s => s.NotifyNewUserAsync(userId));

        return Ok(new { UserId = userId, Message = "Registration successful" });
    }
}
```

### Delayed Job Pattern

```csharp
// Pattern: ทำงานหลังจากผ่านไปช่วงเวลาหนึ่ง
public class SubscriptionService
{
    private readonly IBackgroundJobClient _jobClient;

    public SubscriptionService(IBackgroundJobClient jobClient)
    {
        _jobClient = jobClient;
    }

    public async Task StartTrialAsync(Guid userId)
    {
        // ส่ง reminder 2 วันก่อนหมดอายุ trial
        var trialEndDate = DateTime.UtcNow.AddDays(14);
        var reminderDate = trialEndDate.AddDays(-2);
        
        _jobClient.Schedule<IEmailService>(
            s => s.SendTrialEndingReminderAsync(userId.ToString()),
            reminderDate);

        // ถ้าไม่ upgrade หลัง trial หมด - downgrade อัตโนมัติ
        _jobClient.Schedule<ISubscriptionManager>(
            s => s.DowngradeToFreeAsync(userId.ToString()),
            trialEndDate.AddDays(1));
    }
}
```

### Recurring Job Pattern

```csharp
// Pattern: ทำงานซ้ำตาม schedule
public class ReportingService
{
    private readonly IRecurringJobManager _recurringJobs;

    public ReportingService(IRecurringJobManager recurringJobs)
    {
        _recurringJobs = recurringJobs;
    }

    public void ConfigureReports()
    {
        // รายวัน
        _recurringJobs.AddOrUpdate<IDailyReportJob>(
            "daily-report",
            j => j.RunAsync(),
            "0 6 * * *"); // ทุกวัน 6:00 AM

        // รายสัปดาห์
        _recurringJobs.AddOrUpdate<IWeeklyReportJob>(
            "weekly-report",
            j => j.RunAsync(),
            "0 8 * * MON"); // ทุกวันจันทร์ 8:00 AM

        // รายเดือน
        _recurringJobs.AddOrUpdate<IMonthlyReportJob>(
            "monthly-report",
            j => j.RunAsync(),
            "0 9 1 * *"); // วันที่ 1 ของทุกเดือน 9:00 AM
    }
}
```

### Job Batch Pattern

```csharp
// Pattern: รวมหลาย jobs เข้าด้วยกัน
public class BatchProcessingService
{
    private readonly IBackgroundJobClient _jobClient;

    public BatchProcessingService(IBackgroundJobClient jobClient)
    {
        _jobClient = jobClient;
    }

    public void ProcessLargeDataset(IEnumerable<Guid> itemIds)
    {
        var parentJobId = _jobClient.Enqueue<IJobLogger>(
            j => j.LogBatchStarted("Processing large dataset"));

        string? lastJobId = parentJobId;
        
        // สร้าง chain ของ jobs
        foreach (var id in itemIds)
        {
            lastJobId = _jobClient.ContinueJobWith<IItemProcessor>(
                lastJobId!,
                p => p.ProcessItemAsync(id.ToString()));
        }

        // Job สุดท้ายหลังทุกอย่างเสร็จ
        _jobClient.ContinueJobWith<IJobLogger>(
            lastJobId!,
            j => j.LogBatchCompleted("Processing large dataset"));
    }
}
```

---

## Step 747: Distributed Job Locking (ป้องกัน Duplicate Runs)

### ปัญหาของ Distributed Environment

เมื่อ application ถูก scale-out หลาย instances งาน background อาจรันพร้อมกันหลายครั้ง ทำให้เกิดปัญหา เช่น ส่งอีเมลซ้ำ หรือ process ข้อมูลซ้ำ

```bash
dotnet add package StackExchange.Redis
dotnet add package RedLock.net  # Redlock algorithm implementation
```

### Redlock Pattern ด้วย Redis

```csharp
using RedLockNet;
using RedLockNet.SERedis;
using RedLockNet.SERedis.Configuration;

// Distributed lock service
public interface IDistributedLockService
{
    Task<IAsyncDisposable?> TryAcquireLockAsync(
        string resource, 
        TimeSpan expiry,
        CancellationToken cancellationToken = default);
}

public class RedisDistributedLockService : IDistributedLockService
{
    private readonly IRedLockFactory _redLockFactory;
    private readonly ILogger<RedisDistributedLockService> _logger;

    // Retry parameters สำหรับ lock acquisition
    private static readonly TimeSpan RetryTime = TimeSpan.FromMilliseconds(100);
    private static readonly TimeSpan WaitTime = TimeSpan.FromSeconds(5);
    private const int RetryCount = 3;

    public RedisDistributedLockService(
        IRedLockFactory redLockFactory,
        ILogger<RedisDistributedLockService> logger)
    {
        _redLockFactory = redLockFactory;
        _logger = logger;
    }

    public async Task<IAsyncDisposable?> TryAcquireLockAsync(
        string resource,
        TimeSpan expiry,
        CancellationToken cancellationToken = default)
    {
        _logger.LogDebug("Attempting to acquire lock on: {Resource}", resource);

        var redLock = await _redLockFactory.CreateLockAsync(
            resource,
            expiry,
            WaitTime,
            RetryTime,
            cancellationToken);

        if (redLock.IsAcquired)
        {
            _logger.LogInformation("Lock acquired on: {Resource}", resource);
            return redLock;
        }

        _logger.LogWarning("Failed to acquire lock on: {Resource}", resource);
        await redLock.DisposeAsync();
        return null;
    }
}

// การใช้งานใน Background Service
public class SynchronizedReportJob : BackgroundService
{
    private readonly IDistributedLockService _lockService;
    private readonly IReportService _reportService;
    private readonly ILogger<SynchronizedReportJob> _logger;

    public SynchronizedReportJob(
        IDistributedLockService lockService,
        IReportService reportService,
        ILogger<SynchronizedReportJob> logger)
    {
        _lockService = lockService;
        _reportService = reportService;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromHours(1));

        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            await GenerateReportWithLockAsync(stoppingToken);
        }
    }

    private async Task GenerateReportWithLockAsync(CancellationToken cancellationToken)
    {
        // Lock key ที่ unique สำหรับงานนี้
        const string lockKey = "report-generation-lock";
        var lockExpiry = TimeSpan.FromMinutes(10); // max time งานนี้ใช้

        await using var lockHandle = await _lockService.TryAcquireLockAsync(
            lockKey, 
            lockExpiry, 
            cancellationToken);

        if (lockHandle == null)
        {
            _logger.LogInformation(
                "Report generation already running on another instance, skipping");
            return;
        }

        try
        {
            _logger.LogInformation("Starting report generation (holding distributed lock)");
            await _reportService.GenerateAsync(cancellationToken);
            _logger.LogInformation("Report generation completed");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Report generation failed");
            throw;
        }
        // lock จะถูก release อัตโนมัติเมื่อ using block สิ้นสุด
    }
}
```

### การ Setup Redlock ใน DI

```csharp
// Program.cs
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
});

builder.Services.AddSingleton<IRedLockFactory>(sp =>
{
    var multiplexers = new List<RedLockMultiplexer>
    {
        ConnectionMultiplexer.Connect(
            builder.Configuration.GetConnectionString("Redis")!)
    };
    
    return RedLockFactory.Create(multiplexers);
});

builder.Services.AddSingleton<IDistributedLockService, RedisDistributedLockService>();
```

### Database-based Locking (ทางเลือกเมื่อไม่มี Redis)

```csharp
// ใช้ SQL Server Application Lock
public class SqlDistributedLockService : IDistributedLockService
{
    private readonly string _connectionString;
    private readonly ILogger<SqlDistributedLockService> _logger;

    public SqlDistributedLockService(
        IConfiguration configuration,
        ILogger<SqlDistributedLockService> logger)
    {
        _connectionString = configuration.GetConnectionString("DefaultConnection")!;
        _logger = logger;
    }

    public async Task<IAsyncDisposable?> TryAcquireLockAsync(
        string resource,
        TimeSpan expiry,
        CancellationToken cancellationToken = default)
    {
        var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync(cancellationToken);

        try
        {
            using var command = connection.CreateCommand();
            command.CommandText = "sp_getapplock";
            command.CommandType = CommandType.StoredProcedure;
            command.Parameters.AddWithValue("@Resource", resource);
            command.Parameters.AddWithValue("@LockMode", "Exclusive");
            command.Parameters.AddWithValue("@LockOwner", "Session");
            command.Parameters.AddWithValue("@LockTimeout", 
                (int)TimeSpan.FromSeconds(5).TotalMilliseconds);

            var result = (int)(await command.ExecuteScalarAsync(cancellationToken))!;

            if (result >= 0) // 0 = acquired, 1 = acquired after wait
            {
                return new SqlLockHandle(connection, resource, _logger);
            }

            await connection.DisposeAsync();
            return null;
        }
        catch
        {
            await connection.DisposeAsync();
            throw;
        }
    }
}
```

---

## Step 748: Email Queue Worker ตัวอย่างจริง

### Email Queue Architecture

```csharp
// Email models
public record EmailMessage(
    string To,
    string Subject,
    string Body,
    bool IsHtml = true,
    IReadOnlyList<string>? Attachments = null);

public class EmailQueueItem
{
    public Guid Id { get; init; } = Guid.NewGuid();
    public EmailMessage Message { get; init; } = default!;
    public int RetryCount { get; set; }
    public DateTime? NextRetryAt { get; set; }
    public string Status { get; set; } = "pending";
    public DateTime CreatedAt { get; init; } = DateTime.UtcNow;
}

// Email Queue Channel
public class EmailQueue
{
    private readonly Channel<EmailQueueItem> _channel;

    public EmailQueue()
    {
        _channel = Channel.CreateUnbounded<EmailQueueItem>(
            new UnboundedChannelOptions 
            { 
                SingleReader = true 
            });
    }

    public ChannelReader<EmailQueueItem> Reader => _channel.Reader;

    public async ValueTask EnqueueAsync(
        EmailMessage message, 
        CancellationToken cancellationToken = default)
    {
        var item = new EmailQueueItem { Message = message };
        await _channel.Writer.WriteAsync(item, cancellationToken);
    }

    public void Close() => _channel.Writer.Complete();
}

// Email Sender Interface
public interface ISmtpEmailSender
{
    Task SendAsync(EmailMessage message, CancellationToken cancellationToken);
}

// Email Queue Worker
public class EmailQueueWorker : BackgroundService
{
    private readonly EmailQueue _queue;
    private readonly ISmtpEmailSender _emailSender;
    private readonly ILogger<EmailQueueWorker> _logger;
    private const int MaxRetries = 3;

    public EmailQueueWorker(
        EmailQueue queue,
        ISmtpEmailSender emailSender,
        ILogger<EmailQueueWorker> logger)
    {
        _queue = queue;
        _emailSender = emailSender;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Email queue worker started");

        await foreach (var item in _queue.Reader.ReadAllAsync(stoppingToken))
        {
            await ProcessEmailAsync(item, stoppingToken);
        }

        _logger.LogInformation("Email queue worker stopped");
    }

    private async Task ProcessEmailAsync(EmailQueueItem item, CancellationToken cancellationToken)
    {
        _logger.LogInformation(
            "Sending email to {To} (attempt {Attempt}/{Max})",
            item.Message.To, item.RetryCount + 1, MaxRetries + 1);

        try
        {
            await _emailSender.SendAsync(item.Message, cancellationToken);
            
            _logger.LogInformation(
                "Email sent successfully to {To} (ID: {Id})", 
                item.Message.To, item.Id);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, 
                "Failed to send email to {To} (attempt {Attempt})",
                item.Message.To, item.RetryCount + 1);

            if (item.RetryCount < MaxRetries)
            {
                item.RetryCount++;
                item.NextRetryAt = DateTime.UtcNow.AddSeconds(
                    Math.Pow(2, item.RetryCount) * 10); // Exponential backoff
                
                // Re-queue หลัง delay
                _ = Task.Run(async () =>
                {
                    await Task.Delay(item.NextRetryAt.Value - DateTime.UtcNow, cancellationToken);
                    await _queue.EnqueueAsync(item.Message, cancellationToken);
                }, cancellationToken);
            }
            else
            {
                _logger.LogError(
                    "Email to {To} failed after {Max} attempts, giving up",
                    item.Message.To, MaxRetries + 1);
                
                // บันทึกลง dead letter queue หรือ database
                await HandleDeadLetterAsync(item, ex);
            }
        }
    }

    private async Task HandleDeadLetterAsync(EmailQueueItem item, Exception ex)
    {
        // TODO: บันทึกลง database สำหรับ manual review
        await Task.CompletedTask;
    }
}
```

### SMTP Email Sender Implementation

```csharp
// appsettings.json
/*
{
  "EmailSettings": {
    "Host": "smtp.gmail.com",
    "Port": 587,
    "Username": "your@gmail.com",
    "Password": "your-app-password",
    "FromName": "My App",
    "EnableSsl": true
  }
}
*/

public class SmtpEmailSender : ISmtpEmailSender
{
    private readonly EmailSettings _settings;
    private readonly ILogger<SmtpEmailSender> _logger;

    public SmtpEmailSender(
        IOptions<EmailSettings> settings,
        ILogger<SmtpEmailSender> logger)
    {
        _settings = settings.Value;
        _logger = logger;
    }

    public async Task SendAsync(EmailMessage message, CancellationToken cancellationToken)
    {
        using var client = new SmtpClient(_settings.Host, _settings.Port)
        {
            EnableSsl = _settings.EnableSsl,
            Credentials = new NetworkCredential(_settings.Username, _settings.Password)
        };

        var mailMessage = new MailMessage
        {
            From = new MailAddress(_settings.Username, _settings.FromName),
            Subject = message.Subject,
            Body = message.Body,
            IsBodyHtml = message.IsHtml
        };
        
        mailMessage.To.Add(message.To);

        await client.SendMailAsync(mailMessage, cancellationToken);
    }
}
```

---

## Step 749: File Processing Pipeline ด้วย Background Worker

### File Processing Pipeline Architecture

```csharp
// File processing stages
public enum ProcessingStage
{
    Uploaded,
    Validated,
    Processing,
    Completed,
    Failed
}

public class FileProcessingJob
{
    public Guid Id { get; init; } = Guid.NewGuid();
    public string FilePath { get; init; } = default!;
    public string FileName { get; init; } = default!;
    public string ContentType { get; init; } = default!;
    public ProcessingStage Stage { get; set; }
    public string? ErrorMessage { get; set; }
    public DateTime CreatedAt { get; init; } = DateTime.UtcNow;
    public DateTime? CompletedAt { get; set; }
}

// Pipeline stages
public interface IFileProcessingStage
{
    Task<bool> ProcessAsync(
        FileProcessingJob job, 
        CancellationToken cancellationToken);
}

// Stage 1: Validation
public class FileValidationStage : IFileProcessingStage
{
    private static readonly string[] AllowedTypes = 
    { 
        "image/jpeg", "image/png", "application/pdf" 
    };
    
    private const long MaxFileSizeBytes = 10 * 1024 * 1024; // 10 MB

    public async Task<bool> ProcessAsync(
        FileProcessingJob job, 
        CancellationToken cancellationToken)
    {
        var fileInfo = new FileInfo(job.FilePath);
        
        if (!fileInfo.Exists)
        {
            job.ErrorMessage = "File not found";
            return false;
        }

        if (fileInfo.Length > MaxFileSizeBytes)
        {
            job.ErrorMessage = $"File too large: {fileInfo.Length} bytes (max: {MaxFileSizeBytes})";
            return false;
        }

        if (!AllowedTypes.Contains(job.ContentType))
        {
            job.ErrorMessage = $"Unsupported file type: {job.ContentType}";
            return false;
        }

        job.Stage = ProcessingStage.Validated;
        return true;
    }
}

// Stage 2: Image Processing
public class ImageProcessingStage : IFileProcessingStage
{
    private readonly ILogger<ImageProcessingStage> _logger;

    public ImageProcessingStage(ILogger<ImageProcessingStage> logger)
    {
        _logger = logger;
    }

    public async Task<bool> ProcessAsync(
        FileProcessingJob job, 
        CancellationToken cancellationToken)
    {
        if (!job.ContentType.StartsWith("image/"))
            return true; // ไม่ใช่รูปภาพ ข้ามไป

        try
        {
            _logger.LogInformation("Processing image: {File}", job.FileName);

            // ตัวอย่าง: ใช้ SixLabors.ImageSharp สร้าง thumbnail
            // using var image = await Image.LoadAsync(job.FilePath, cancellationToken);
            // image.Mutate(x => x.Resize(new ResizeOptions { Size = new Size(800, 600) }));
            // var outputPath = Path.ChangeExtension(job.FilePath, ".thumb.jpg");
            // await image.SaveAsJpegAsync(outputPath, cancellationToken);

            await Task.Delay(100, cancellationToken); // Simulate processing
            
            job.Stage = ProcessingStage.Processing;
            return true;
        }
        catch (Exception ex)
        {
            job.ErrorMessage = $"Image processing failed: {ex.Message}";
            return false;
        }
    }
}

// Stage 3: Storage (อัพโหลดไปยัง cloud storage)
public class CloudStorageStage : IFileProcessingStage
{
    private readonly IBlobStorageService _blobStorage;
    private readonly ILogger<CloudStorageStage> _logger;

    public CloudStorageStage(
        IBlobStorageService blobStorage,
        ILogger<CloudStorageStage> logger)
    {
        _blobStorage = blobStorage;
        _logger = logger;
    }

    public async Task<bool> ProcessAsync(
        FileProcessingJob job, 
        CancellationToken cancellationToken)
    {
        try
        {
            _logger.LogInformation("Uploading file to cloud: {File}", job.FileName);

            await using var fileStream = File.OpenRead(job.FilePath);
            var blobName = $"uploads/{DateTime.UtcNow:yyyy/MM}/{job.Id}/{job.FileName}";
            
            await _blobStorage.UploadAsync(blobName, fileStream, cancellationToken);
            
            // ลบไฟล์ temp หลัง upload สำเร็จ
            File.Delete(job.FilePath);

            job.Stage = ProcessingStage.Completed;
            job.CompletedAt = DateTime.UtcNow;
            
            _logger.LogInformation("File uploaded successfully: {BlobName}", blobName);
            return true;
        }
        catch (Exception ex)
        {
            job.ErrorMessage = $"Cloud upload failed: {ex.Message}";
            return false;
        }
    }
}

// File Processing Pipeline Worker
public class FileProcessingPipelineWorker : BackgroundService
{
    private readonly Channel<FileProcessingJob> _jobQueue;
    private readonly IEnumerable<IFileProcessingStage> _stages;
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<FileProcessingPipelineWorker> _logger;

    public FileProcessingPipelineWorker(
        Channel<FileProcessingJob> jobQueue,
        IEnumerable<IFileProcessingStage> stages,
        IServiceProvider serviceProvider,
        ILogger<FileProcessingPipelineWorker> logger)
    {
        _jobQueue = jobQueue;
        _stages = stages;
        _serviceProvider = serviceProvider;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var job in _jobQueue.Reader.ReadAllAsync(stoppingToken))
        {
            await RunPipelineAsync(job, stoppingToken);
        }
    }

    private async Task RunPipelineAsync(
        FileProcessingJob job, 
        CancellationToken cancellationToken)
    {
        _logger.LogInformation(
            "Starting pipeline for file: {File} (ID: {Id})", 
            job.FileName, job.Id);

        foreach (var stage in _stages)
        {
            var stageName = stage.GetType().Name;
            
            try
            {
                _logger.LogDebug("Running stage: {Stage}", stageName);
                
                var success = await stage.ProcessAsync(job, cancellationToken);
                
                if (!success)
                {
                    job.Stage = ProcessingStage.Failed;
                    _logger.LogWarning(
                        "Pipeline failed at stage {Stage}: {Error}", 
                        stageName, job.ErrorMessage);
                    
                    await SaveJobStatusAsync(job);
                    return;
                }
            }
            catch (Exception ex)
            {
                job.Stage = ProcessingStage.Failed;
                job.ErrorMessage = ex.Message;
                
                _logger.LogError(ex, "Stage {Stage} threw exception", stageName);
                
                await SaveJobStatusAsync(job);
                return;
            }
        }

        _logger.LogInformation(
            "Pipeline completed successfully for: {File} (ID: {Id})", 
            job.FileName, job.Id);
        
        await SaveJobStatusAsync(job);
    }

    private async Task SaveJobStatusAsync(FileProcessingJob job)
    {
        using var scope = _serviceProvider.CreateScope();
        var repository = scope.ServiceProvider
            .GetRequiredService<IFileJobRepository>();
        
        await repository.UpdateAsync(job);
    }
}
```

---

## Step 750: Health Checks สำหรับ Background Services

### การสร้าง Health Check สำหรับ Background Service

```csharp
// Health Check Status Tracker
public class BackgroundServiceHealthTracker
{
    private readonly ConcurrentDictionary<string, ServiceHealthStatus> _statuses = new();

    public void ReportHealthy(string serviceName, string? message = null)
    {
        _statuses[serviceName] = new ServiceHealthStatus
        {
            IsHealthy = true,
            Message = message ?? "Running normally",
            LastUpdated = DateTimeOffset.UtcNow
        };
    }

    public void ReportUnhealthy(string serviceName, string reason)
    {
        _statuses[serviceName] = new ServiceHealthStatus
        {
            IsHealthy = false,
            Message = reason,
            LastUpdated = DateTimeOffset.UtcNow
        };
    }

    public ServiceHealthStatus? GetStatus(string serviceName)
    {
        return _statuses.TryGetValue(serviceName, out var status) ? status : null;
    }

    public IReadOnlyDictionary<string, ServiceHealthStatus> GetAllStatuses()
    {
        return _statuses;
    }
}

public class ServiceHealthStatus
{
    public bool IsHealthy { get; init; }
    public string Message { get; init; } = default!;
    public DateTimeOffset LastUpdated { get; init; }
    
    // ถ้าไม่มีการ report เกิน 5 นาที ถือว่าอาจมีปัญหา
    public bool IsStale => DateTimeOffset.UtcNow - LastUpdated > TimeSpan.FromMinutes(5);
}

// Background Service ที่รายงาน health
public class MonitoredWorkerService : BackgroundService
{
    private readonly BackgroundServiceHealthTracker _healthTracker;
    private readonly ILogger<MonitoredWorkerService> _logger;
    private const string ServiceName = "MonitoredWorkerService";

    public MonitoredWorkerService(
        BackgroundServiceHealthTracker healthTracker,
        ILogger<MonitoredWorkerService> logger)
    {
        _healthTracker = healthTracker;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _healthTracker.ReportHealthy(ServiceName, "Starting up");

        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(30));

        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            try
            {
                await DoWorkAsync(stoppingToken);
                _healthTracker.ReportHealthy(ServiceName, 
                    $"Last run: {DateTimeOffset.UtcNow}");
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                _healthTracker.ReportUnhealthy(ServiceName, ex.Message);
                _logger.LogError(ex, "Worker failed");
            }
        }
    }

    private async Task DoWorkAsync(CancellationToken cancellationToken)
    {
        await Task.Delay(100, cancellationToken);
    }
}

// Health Check ที่ใช้ Tracker
public class BackgroundServiceHealthCheck : IHealthCheck
{
    private readonly BackgroundServiceHealthTracker _healthTracker;

    public BackgroundServiceHealthCheck(BackgroundServiceHealthTracker healthTracker)
    {
        _healthTracker = healthTracker;
    }

    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        var allStatuses = _healthTracker.GetAllStatuses();
        
        var unhealthy = allStatuses
            .Where(s => !s.Value.IsHealthy || s.Value.IsStale)
            .ToList();

        if (!unhealthy.Any())
        {
            var data = allStatuses.ToDictionary(
                kvp => kvp.Key, 
                kvp => (object)kvp.Value.Message);
                
            return Task.FromResult(HealthCheckResult.Healthy(
                "All background services are healthy", data));
        }

        var issues = unhealthy
            .Select(s => $"{s.Key}: {s.Value.Message}")
            .ToList();
            
        return Task.FromResult(HealthCheckResult.Unhealthy(
            $"Unhealthy services: {string.Join(", ", issues)}"));
    }
}
```

### Health Checks สำหรับ External Dependencies

```csharp
// Health Check สำหรับ Redis Connection
public class RedisHealthCheck : IHealthCheck
{
    private readonly IConnectionMultiplexer _redis;

    public RedisHealthCheck(IConnectionMultiplexer redis)
    {
        _redis = redis;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var db = _redis.GetDatabase();
            await db.PingAsync();
            
            return HealthCheckResult.Healthy("Redis is responding");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy(
                "Redis is not responding", 
                exception: ex);
        }
    }
}

// Health Check สำหรับ Email Queue
public class EmailQueueHealthCheck : IHealthCheck
{
    private readonly EmailQueue _emailQueue;

    public EmailQueueHealthCheck(EmailQueue emailQueue)
    {
        _emailQueue = emailQueue;
    }

    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        // ตรวจสอบว่า channel ยังเปิดอยู่
        if (_emailQueue.Reader.Completion.IsCompleted)
        {
            return Task.FromResult(
                HealthCheckResult.Unhealthy("Email queue channel is closed"));
        }

        return Task.FromResult(
            HealthCheckResult.Healthy("Email queue is operational"));
    }
}
```

### การ Setup Health Checks ใน Program.cs

```csharp
// Program.cs - Complete Health Check Setup
builder.Services.AddSingleton<BackgroundServiceHealthTracker>();

builder.Services.AddHealthChecks()
    // Background service health
    .AddCheck<BackgroundServiceHealthCheck>(
        "background-services",
        HealthStatus.Degraded,
        tags: new[] { "background", "ready" })
    
    // Redis health
    .AddCheck<RedisHealthCheck>(
        "redis",
        HealthStatus.Unhealthy,
        tags: new[] { "db", "ready" })
    
    // Email queue health
    .AddCheck<EmailQueueHealthCheck>(
        "email-queue",
        HealthStatus.Degraded,
        tags: new[] { "messaging", "ready" })
    
    // Database health (built-in)
    .AddDbContextCheck<AppDbContext>(
        "database",
        tags: new[] { "db", "ready" });

var app = builder.Build();

// Health check endpoints
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

// Liveness probe (k8s) - ตรวจสอบว่า app ยังทำงานอยู่
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false // ไม่รัน checks, แค่ return 200 ถ้า app ยังอยู่
});

// Readiness probe (k8s) - ตรวจสอบว่าพร้อมรับ traffic
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
```

### Graceful Shutdown ด้วย CancellationToken

```csharp
// Graceful shutdown configuration
builder.Services.Configure<HostOptions>(options =>
{
    // รอให้ background services หยุดอย่างมากที่สุด 30 วินาที
    options.ShutdownTimeout = TimeSpan.FromSeconds(30);
    
    // ถ้า background service throw unhandled exception
    // application ยังคงทำงานต่อ (ไม่ crash)
    options.BackgroundServiceExceptionBehavior = 
        BackgroundServiceExceptionBehavior.Ignore;
});

// ตัวอย่าง graceful shutdown ใน background service
public class GracefulWorkerService : BackgroundService
{
    private readonly ILogger<GracefulWorkerService> _logger;

    public GracefulWorkerService(ILogger<GracefulWorkerService> logger)
    {
        _logger = logger;
    }

    public override async Task StopAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("GracefulWorkerService is stopping gracefully...");
        
        // ทำ cleanup ก่อน stop
        await CleanupAsync();
        
        // เรียก base method ซึ่งจะ cancel stoppingToken ของ ExecuteAsync
        await base.StopAsync(cancellationToken);
        
        _logger.LogInformation("GracefulWorkerService stopped");
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // ลงทะเบียน callback สำหรับเมื่อมีการ cancel
        stoppingToken.Register(() =>
        {
            _logger.LogInformation("Cancellation requested, finishing current work...");
        });

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await DoWorkAsync(stoppingToken);
            }
            catch (OperationCanceledException)
            {
                // Graceful shutdown - break out of loop
                _logger.LogInformation("Work cancelled due to shutdown");
                break;
            }

            await Task.Delay(1000, stoppingToken);
        }
    }

    private async Task DoWorkAsync(CancellationToken cancellationToken)
    {
        // งานที่ต้อง respect cancellation
        await Task.Delay(500, cancellationToken);
    }

    private async Task CleanupAsync()
    {
        _logger.LogInformation("Running cleanup tasks...");
        await Task.Delay(1000); // Simulate cleanup
    }
}
```

---

## สรุปภาพรวม Background Services

```csharp
// Complete Program.cs สำหรับ Background Services
var builder = WebApplication.CreateBuilder(args);

// === Core Services ===
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// === Channel-based Queue ===
builder.Services.AddSingleton<EmailQueue>();
builder.Services.AddSingleton<WorkChannel>();

// === Background Workers ===
builder.Services.AddHostedService<EmailQueueWorker>();
builder.Services.AddHostedService<WorkerPoolService>();
builder.Services.AddHostedService<PeriodicCleanupWorker>();

// === Distributed Locking ===
builder.Services.AddStackExchangeRedisCache(opts =>
    opts.Configuration = builder.Configuration.GetConnectionString("Redis"));
builder.Services.AddSingleton<IDistributedLockService, RedisDistributedLockService>();

// === Hangfire ===
builder.Services.AddHangfire(config => config
    .UseSqlServerStorage(builder.Configuration.GetConnectionString("HangfireConnection")));
builder.Services.AddHangfireServer();

// === Quartz ===
builder.Services.AddQuartz(q =>
{
    q.UseMicrosoftDependencyInjectionJobFactory();
    // ... เพิ่ม jobs และ triggers
});
builder.Services.AddQuartzHostedService(q => q.WaitForJobsToComplete = true);

// === Health Checks ===
builder.Services.AddSingleton<BackgroundServiceHealthTracker>();
builder.Services.AddHealthChecks()
    .AddCheck<BackgroundServiceHealthCheck>("background-services")
    .AddCheck<RedisHealthCheck>("redis")
    .AddCheck<EmailQueueHealthCheck>("email-queue")
    .AddDbContextCheck<AppDbContext>("database");

// === Graceful Shutdown ===
builder.Services.Configure<HostOptions>(opts =>
    opts.ShutdownTimeout = TimeSpan.FromSeconds(30));

var app = builder.Build();

app.UseHangfireDashboard("/hangfire");
app.MapHealthChecks("/health");

app.Run();
```

---

## แนวทางปฏิบัติที่ดี (Best Practices)

| หัวข้อ | แนวทาง |
|--------|---------|
| **Exception Handling** | จัดการทุก exception ใน ExecuteAsync ไม่ให้ worker หยุด |
| **Scoped Services** | ใช้ IServiceScopeFactory สร้าง scope ใน background service |
| **CancellationToken** | ส่ง token ไปทุก async call เพื่อ graceful shutdown |
| **PeriodicTimer** | ใช้แทน Task.Delay loop ใน .NET 6+ |
| **Distributed Lock** | ใช้เสมอเมื่อ job ห้ามรันพร้อมกันใน scaled-out environment |
| **Health Checks** | รายงาน health status อย่างสม่ำเสมอ |
| **Logging** | Log ทุก start/stop/error พร้อม structured data |
| **Channel vs Queue** | Channel สำหรับ in-process, Hangfire/Quartz สำหรับ persistent |

---

**ก่อนหน้า → [Part 74: Message Bus](part74-message-bus.md)**
**ต่อไป → [Part 76: Functional C#](part76-functional-csharp.md)**
