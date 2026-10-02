# Part 51: Clean Architecture
## ขั้นตอนที่ 501-510: สถาปัตยกรรมซอฟต์แวร์ระดับมืออาชีพ

---

## 🎯 เป้าหมายของ Part นี้
- Clean Architecture คืออะไร
- Layers และ Dependencies
- Domain Layer (Entities, Value Objects)
- Application Layer (Use Cases, CQRS)
- Infrastructure Layer (EF Core, External APIs)
- Presentation Layer (WPF, API)
- โปรแกรม Task Management System

---

## ขั้นตอนที่ 501: Clean Architecture Overview

```
Clean Architecture Layers:
┌─────────────────────────────────────────────────────────────┐
│                    Presentation Layer                        │
│              (WPF, API Controllers, CLI)                     │
├─────────────────────────────────────────────────────────────┤
│                  Infrastructure Layer                        │
│         (EF Core, File I/O, External APIs, Email)           │
├─────────────────────────────────────────────────────────────┤
│                  Application Layer                           │
│            (Use Cases, CQRS, Validators, DTOs)              │
├─────────────────────────────────────────────────────────────┤
│                    Domain Layer                              │
│           (Entities, Value Objects, Domain Events)           │
└─────────────────────────────────────────────────────────────┘

Dependency Rule: ลูกศรชี้เข้าด้านใน (Inward only)
- Presentation → Application → Domain ✓
- Domain → Infrastructure ✗ (Domain ไม่รู้จัก EF Core)
```

```
Project Structure:
TaskManager/
├── TaskManager.Domain/           (no dependencies)
│   ├── Entities/
│   ├── ValueObjects/
│   ├── Events/
│   ├── Exceptions/
│   └── Interfaces/               (IRepository abstractions)
├── TaskManager.Application/      (depends on Domain only)
│   ├── Common/
│   │   ├── Interfaces/
│   │   └── Behaviors/
│   ├── Tasks/
│   │   ├── Commands/
│   │   ├── Queries/
│   │   └── Validators/
│   └── DependencyInjection.cs
├── TaskManager.Infrastructure/   (implements Domain interfaces)
│   ├── Persistence/
│   ├── Services/
│   └── DependencyInjection.cs
└── TaskManager.WPF/              (depends on Application)
    ├── ViewModels/
    └── Views/
```

---

## ขั้นตอนที่ 502: Domain Layer - Entities & Value Objects

```csharp
// Domain/Entities/TaskItem.cs
namespace TaskManager.Domain.Entities;

public class TaskItem : BaseEntity
{
    private TaskItem() { } // EF Core requires parameterless constructor
    
    public static TaskItem Create(string title, string description, Priority priority, UserId assignedTo)
    {
        if (string.IsNullOrWhiteSpace(title)) throw new DomainException("Title cannot be empty");
        if (title.Length > 200) throw new DomainException("Title too long (max 200 chars)");
        
        var task = new TaskItem
        {
            Title = title,
            Description = description,
            Priority = priority,
            AssignedTo = assignedTo,
            Status = TaskStatus.Todo,
            CreatedAt = DateTime.UtcNow
        };
        task.AddEvent(new TaskCreatedEvent(task.Id, title, assignedTo));
        return task;
    }
    
    public string Title { get; private set; } = "";
    public string Description { get; private set; } = "";
    public Priority Priority { get; private set; }
    public TaskStatus Status { get; private set; }
    public UserId AssignedTo { get; private set; } = UserId.Empty;
    public DueDate? DueDate { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? CompletedAt { get; private set; }
    
    public void UpdateTitle(string title)
    {
        if (string.IsNullOrWhiteSpace(title)) throw new DomainException("Title cannot be empty");
        Title = title;
        AddEvent(new TaskUpdatedEvent(Id));
    }
    
    public void SetDueDate(DateTime date)
    {
        if (date < DateTime.UtcNow.Date) throw new DomainException("Due date cannot be in the past");
        DueDate = new DueDate(date);
    }
    
    public void Complete()
    {
        if (Status == TaskStatus.Done) throw new DomainException("Task already completed");
        Status = TaskStatus.Done;
        CompletedAt = DateTime.UtcNow;
        AddEvent(new TaskCompletedEvent(Id, AssignedTo));
    }
    
    public void Assign(UserId userId)
    {
        AssignedTo = userId;
        AddEvent(new TaskAssignedEvent(Id, userId));
    }
    
    public bool IsOverdue => DueDate != null && DueDate.Value < DateTime.UtcNow && Status != TaskStatus.Done;
}

// Domain/Entities/BaseEntity.cs
public abstract class BaseEntity
{
    public Guid Id { get; protected set; } = Guid.NewGuid();
    
    private readonly List<IDomainEvent> _events = new();
    public IReadOnlyList<IDomainEvent> DomainEvents => _events.AsReadOnly();
    
    protected void AddEvent(IDomainEvent evt) => _events.Add(evt);
    public void ClearEvents() => _events.Clear();
}

// Domain/ValueObjects/UserId.cs
public record UserId(Guid Value)
{
    public static UserId Empty => new(Guid.Empty);
    public static UserId New() => new(Guid.NewGuid());
    public static UserId From(string id) => new(Guid.Parse(id));
    public override string ToString() => Value.ToString();
}

// Domain/ValueObjects/DueDate.cs
public record DueDate(DateTime Value)
{
    public bool IsOverdue => Value.Date < DateTime.UtcNow.Date;
    public int DaysRemaining => (Value.Date - DateTime.UtcNow.Date).Days;
    public string Display => Value.ToString("dd/MM/yyyy");
}

// Domain/Enums
public enum Priority { Low, Medium, High, Critical }
public enum TaskStatus { Todo, InProgress, Review, Done, Cancelled }

// Domain/Exceptions/DomainException.cs
public class DomainException : Exception
{
    public DomainException(string message) : base(message) { }
}
```

---

## ขั้นตอนที่ 503: Domain Interfaces (Ports)

```csharp
// Domain/Interfaces/ITaskRepository.cs
public interface ITaskRepository
{
    Task<TaskItem?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<IReadOnlyList<TaskItem>> GetAllAsync(CancellationToken ct = default);
    Task<IReadOnlyList<TaskItem>> GetByUserAsync(UserId userId, CancellationToken ct = default);
    Task<IReadOnlyList<TaskItem>> GetOverdueAsync(CancellationToken ct = default);
    Task AddAsync(TaskItem task, CancellationToken ct = default);
    void Update(TaskItem task);
    void Delete(TaskItem task);
}

// Domain/Interfaces/IUnitOfWork.cs
public interface IUnitOfWork
{
    ITaskRepository Tasks { get; }
    IUserRepository Users { get; }
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}
```

---

## ขั้นตอนที่ 504: Application Layer - CQRS

```csharp
// Application layer uses MediatR for CQRS
// Install: dotnet add package MediatR
// Install: dotnet add package FluentValidation

// Application/Tasks/Commands/CreateTask/CreateTaskCommand.cs
public record CreateTaskCommand(
    string Title,
    string Description,
    Priority Priority,
    string AssignedToUserId,
    DateTime? DueDate
) : IRequest<Guid>;

// Application/Tasks/Commands/CreateTask/CreateTaskCommandHandler.cs
public class CreateTaskCommandHandler : IRequestHandler<CreateTaskCommand, Guid>
{
    private readonly IUnitOfWork _uow;
    private readonly ICurrentUserService _currentUser;
    private readonly IPublisher _publisher;
    
    public CreateTaskCommandHandler(IUnitOfWork uow, ICurrentUserService currentUser, IPublisher publisher)
    { _uow = uow; _currentUser = currentUser; _publisher = publisher; }
    
    public async Task<Guid> Handle(CreateTaskCommand request, CancellationToken ct)
    {
        var assignedTo = UserId.From(request.AssignedToUserId);
        
        // Verify user exists
        var user = await _uow.Users.GetByIdAsync(assignedTo, ct);
        if (user == null) throw new NotFoundException(nameof(User), request.AssignedToUserId);
        
        var task = TaskItem.Create(request.Title, request.Description, request.Priority, assignedTo);
        
        if (request.DueDate.HasValue)
            task.SetDueDate(request.DueDate.Value);
        
        await _uow.Tasks.AddAsync(task, ct);
        await _uow.SaveChangesAsync(ct);
        
        // Publish domain events
        foreach (var evt in task.DomainEvents)
            await _publisher.Publish(evt, ct);
        task.ClearEvents();
        
        return task.Id;
    }
}

// Application/Tasks/Commands/CreateTask/CreateTaskCommandValidator.cs
public class CreateTaskCommandValidator : AbstractValidator<CreateTaskCommand>
{
    public CreateTaskCommandValidator()
    {
        RuleFor(x => x.Title)
            .NotEmpty().WithMessage("กรุณาระบุชื่องาน")
            .MaximumLength(200).WithMessage("ชื่องานต้องไม่เกิน 200 ตัวอักษร");
        
        RuleFor(x => x.AssignedToUserId)
            .NotEmpty().WithMessage("กรุณาระบุผู้รับผิดชอบ")
            .Must(id => Guid.TryParse(id, out _)).WithMessage("รูปแบบ UserId ไม่ถูกต้อง");
        
        RuleFor(x => x.DueDate)
            .GreaterThan(DateTime.UtcNow).When(x => x.DueDate.HasValue)
            .WithMessage("วันครบกำหนดต้องอยู่ในอนาคต");
    }
}
```

---

## ขั้นตอนที่ 505: Application Layer - Queries

```csharp
// Application/Tasks/Queries/GetTaskList/GetTaskListQuery.cs
public record GetTaskListQuery(
    string? AssignedToUserId = null,
    TaskStatus? Status = null,
    Priority? Priority = null,
    bool OverdueOnly = false,
    int Page = 1,
    int PageSize = 20
) : IRequest<PagedResult<TaskItemDto>>;

// Application/Tasks/Queries/GetTaskList/TaskItemDto.cs
public record TaskItemDto(
    Guid Id,
    string Title,
    string Description,
    string Priority,
    string Status,
    string AssignedToName,
    string? DueDate,
    bool IsOverdue,
    int? DaysRemaining
);

// Application/Tasks/Queries/GetTaskList/GetTaskListQueryHandler.cs
public class GetTaskListQueryHandler : IRequestHandler<GetTaskListQuery, PagedResult<TaskItemDto>>
{
    private readonly ITaskRepository _tasks;
    private readonly IUserRepository _users;
    
    public GetTaskListQueryHandler(ITaskRepository tasks, IUserRepository users)
    { _tasks = tasks; _users = users; }
    
    public async Task<PagedResult<TaskItemDto>> Handle(GetTaskListQuery request, CancellationToken ct)
    {
        var tasks = await _tasks.GetAllAsync(ct);
        
        // Filter
        IEnumerable<TaskItem> filtered = tasks;
        if (!string.IsNullOrEmpty(request.AssignedToUserId))
            filtered = filtered.Where(t => t.AssignedTo == UserId.From(request.AssignedToUserId));
        if (request.Status.HasValue)
            filtered = filtered.Where(t => t.Status == request.Status.Value);
        if (request.Priority.HasValue)
            filtered = filtered.Where(t => t.Priority == request.Priority.Value);
        if (request.OverdueOnly)
            filtered = filtered.Where(t => t.IsOverdue);
        
        // Paginate
        var list = filtered.ToList();
        var pagedItems = list.Skip((request.Page - 1) * request.PageSize).Take(request.PageSize);
        
        // Map to DTO
        var userIds = pagedItems.Select(t => t.AssignedTo).Distinct();
        var users = await _users.GetByIdsAsync(userIds, ct);
        var userMap = users.ToDictionary(u => u.Id);
        
        var dtos = pagedItems.Select(t => new TaskItemDto(
            t.Id, t.Title, t.Description, t.Priority.ToString(), t.Status.ToString(),
            userMap.GetValueOrDefault(t.AssignedTo)?.Name ?? "Unknown",
            t.DueDate?.Display, t.IsOverdue, t.DueDate?.DaysRemaining
        )).ToList();
        
        return new PagedResult<TaskItemDto>(dtos, list.Count, request.Page, request.PageSize);
    }
}
```

---

## ขั้นตอนที่ 506: Application Behaviors (Cross-cutting Concerns)

```csharp
// Application/Common/Behaviors/ValidationBehavior.cs
// Automatically validate commands before handling
public class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;
    
    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators) => _validators = validators;
    
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        if (!_validators.Any()) return await next();
        
        var context = new ValidationContext<TRequest>(request);
        var results = await Task.WhenAll(_validators.Select(v => v.ValidateAsync(context, ct)));
        var failures = results.SelectMany(r => r.Errors).Where(f => f != null).ToList();
        
        if (failures.Any())
            throw new ValidationException(failures);
        
        return await next();
    }
}

// Application/Common/Behaviors/LoggingBehavior.cs
public class LoggingBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly ILogger<LoggingBehavior<TRequest, TResponse>> _logger;
    
    public LoggingBehavior(ILogger<LoggingBehavior<TRequest, TResponse>> logger) => _logger = logger;
    
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var name = typeof(TRequest).Name;
        _logger.LogInformation("Handling {Name}: {@Request}", name, request);
        var sw = Stopwatch.StartNew();
        
        try
        {
            var response = await next();
            _logger.LogInformation("Handled {Name} in {Ms}ms", name, sw.ElapsedMilliseconds);
            return response;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error handling {Name} after {Ms}ms", name, sw.ElapsedMilliseconds);
            throw;
        }
    }
}

// Application/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        var assembly = Assembly.GetExecutingAssembly();
        
        services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(assembly));
        services.AddValidatorsFromAssembly(assembly);
        
        services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
        services.AddTransient(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
        
        return services;
    }
}
```

---

## ขั้นตอนที่ 507: Infrastructure Layer

```csharp
// Infrastructure/Persistence/AppDbContext.cs
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }
    
    public DbSet<TaskItem> Tasks => Set<TaskItem>();
    public DbSet<User> Users => Set<User>();
    
    protected override void OnModelCreating(ModelBuilder mb)
    {
        mb.ApplyConfigurationsFromAssembly(Assembly.GetExecutingAssembly());
    }
}

// Infrastructure/Persistence/Configurations/TaskItemConfiguration.cs
public class TaskItemConfiguration : IEntityTypeConfiguration<TaskItem>
{
    public void Configure(EntityTypeBuilder<TaskItem> b)
    {
        b.HasKey(t => t.Id);
        b.Property(t => t.Title).HasMaxLength(200).IsRequired();
        b.Property(t => t.Status).HasConversion<string>();
        b.Property(t => t.Priority).HasConversion<string>();
        
        // Value Object mapping (DueDate as owned entity)
        b.OwnsOne(t => t.DueDate, d =>
        {
            d.Property(x => x.Value).HasColumnName("DueDate");
        });
        
        // Value Object mapping (UserId stored as Guid)
        b.Property(t => t.AssignedTo)
            .HasConversion(id => id.Value, v => new UserId(v));
        
        // Ignore domain events (not persisted)
        b.Ignore(t => t.DomainEvents);
        
        b.HasIndex(t => t.AssignedTo);
        b.HasIndex(t => t.Status);
    }
}

// Infrastructure/Persistence/Repositories/TaskRepository.cs
public class TaskRepository : ITaskRepository
{
    private readonly AppDbContext _ctx;
    
    public TaskRepository(AppDbContext ctx) => _ctx = ctx;
    
    public async Task<TaskItem?> GetByIdAsync(Guid id, CancellationToken ct = default)
        => await _ctx.Tasks.FirstOrDefaultAsync(t => t.Id == id, ct);
    
    public async Task<IReadOnlyList<TaskItem>> GetAllAsync(CancellationToken ct = default)
        => await _ctx.Tasks.AsNoTracking().ToListAsync(ct);
    
    public async Task<IReadOnlyList<TaskItem>> GetByUserAsync(UserId userId, CancellationToken ct = default)
        => await _ctx.Tasks.AsNoTracking().Where(t => t.AssignedTo == userId).ToListAsync(ct);
    
    public async Task<IReadOnlyList<TaskItem>> GetOverdueAsync(CancellationToken ct = default)
    {
        var now = DateTime.UtcNow;
        return await _ctx.Tasks.AsNoTracking()
            .Where(t => t.DueDate != null && t.DueDate.Value < now && t.Status != TaskStatus.Done)
            .ToListAsync(ct);
    }
    
    public async Task AddAsync(TaskItem task, CancellationToken ct = default)
        => await _ctx.Tasks.AddAsync(task, ct);
    
    public void Update(TaskItem task) => _ctx.Tasks.Update(task);
    
    public void Delete(TaskItem task) => _ctx.Tasks.Remove(task);
}

// Infrastructure/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(this IServiceCollection services, string connectionString)
    {
        services.AddDbContext<AppDbContext>(opt => opt.UseSqlite(connectionString));
        services.AddScoped<IUnitOfWork, UnitOfWork>();
        services.AddScoped<ITaskRepository, TaskRepository>();
        services.AddScoped<IUserRepository, UserRepository>();
        return services;
    }
}
```

---

## ขั้นตอนที่ 508-510: WPF Presentation Layer

```csharp
// WPF/ViewModels/TaskListViewModel.cs
public class TaskListViewModel : ViewModelBase
{
    private readonly IMediator _mediator;
    
    public TaskListViewModel(IMediator mediator)
    {
        _mediator = mediator;
        LoadCommand = new AsyncRelayCommand(LoadAsync);
        CreateCommand = new AsyncRelayCommand(CreateAsync);
        CompleteCommand = new AsyncRelayCommand<TaskItemDto>(CompleteAsync);
    }
    
    private ObservableCollection<TaskItemDto> _tasks = new();
    public ObservableCollection<TaskItemDto> Tasks
    {
        get => _tasks;
        set => SetProperty(ref _tasks, value);
    }
    
    private bool _isLoading;
    public bool IsLoading { get => _isLoading; set => SetProperty(ref _isLoading, value); }
    
    public IAsyncRelayCommand LoadCommand { get; }
    public IAsyncRelayCommand CreateCommand { get; }
    public IAsyncRelayCommand<TaskItemDto> CompleteCommand { get; }
    
    private async Task LoadAsync()
    {
        IsLoading = true;
        try
        {
            var result = await _mediator.Send(new GetTaskListQuery());
            Tasks = new ObservableCollection<TaskItemDto>(result.Items);
        }
        finally { IsLoading = false; }
    }
    
    private async Task CreateAsync()
    {
        // Show dialog and create task via MediatR
        var id = await _mediator.Send(new CreateTaskCommand(
            "งานใหม่", "", Priority.Medium, 
            "user-id-here", DateTime.UtcNow.AddDays(7)));
        await LoadAsync();
    }
    
    private async Task CompleteAsync(TaskItemDto? task)
    {
        if (task == null) return;
        await _mediator.Send(new CompleteTaskCommand(task.Id));
        await LoadAsync();
    }
}

// WPF/App.xaml.cs - DI Setup
public partial class App : Application
{
    private IServiceProvider? _services;
    
    protected override async void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);
        
        var services = new ServiceCollection();
        services.AddApplication();  // Application layer
        services.AddInfrastructure("Data Source=taskmanager.db");  // Infrastructure
        
        // WPF ViewModels
        services.AddTransient<TaskListViewModel>();
        services.AddTransient<MainWindow>();
        
        _services = services.BuildServiceProvider();
        
        // Ensure DB
        using var scope = _services.CreateScope();
        var ctx = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await ctx.Database.MigrateAsync();
        
        var window = _services.GetRequiredService<MainWindow>();
        window.Show();
    }
}
```

---

## 📝 สรุป Part 51

| Layer | หน้าที่ | Dependency |
|-------|---------|------------|
| Domain | Business rules, Entities, Value Objects | ไม่ขึ้นกับใคร |
| Application | Use cases, CQRS, Validation | → Domain เท่านั้น |
| Infrastructure | DB, APIs, Files | → Application, Domain |
| Presentation | UI, ViewModels | → Application เท่านั้น |

ข้อดีของ Clean Architecture:
- **Testability**: Test business logic โดยไม่ต้องการ DB
- **Flexibility**: เปลี่ยน DB หรือ UI ได้โดยไม่กระทบ Domain
- **Maintainability**: แต่ละ Layer มีหน้าที่ชัดเจน

---

**ก่อนหน้า → [Part 50: Design Patterns Final](../part41-50/part50-design-patterns-final.md)**  
**ต่อไป → [Part 52: Unit Testing](part52-unit-testing.md)**
