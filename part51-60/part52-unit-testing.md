# Part 52: Unit Testing ใน C# - Steps 511-520

> **หลักสูตร C# Web Programming Enterprise**
> ส่วนที่ 52: การทดสอบอัตโนมัติด้วย xUnit, Moq, FluentAssertions และ EF Core InMemory

---

## สารบัญ

| Step | หัวข้อ |
|------|--------|
| 511 | xUnit Basics - พื้นฐานการเขียน Unit Test |
| 512 | Testing Domain Entities - ทดสอบ Domain Objects |
| 513 | Moq - การ Mock Dependencies |
| 514 | Testing Application Layer - ทดสอบ Command/Query Handlers |
| 515 | FluentAssertions - Assertion แบบอ่านง่าย |
| 516 | Integration Tests ด้วย EF Core InMemory |
| 517 | Test Coverage และ End-to-End Flow |
| 518 | Builder Pattern สำหรับ Test Data |
| 519 | Parameterized Tests ด้วย MemberData และ ClassData |
| 520 | สรุปและ Best Practices |

---

## Step 511: xUnit Basics - พื้นฐานการเขียน Unit Test

### ทำไมต้องเขียน Unit Test?

Unit Test คือการทดสอบโค้ดทีละหน่วยเล็กๆ (method หรือ class) ในแบบแยกอิสระจากส่วนอื่น เพื่อให้มั่นใจว่าโค้ดทำงานถูกต้องตามที่คาดหวัง

ประโยชน์หลัก:
- **ป้องกัน Regression** - ตรวจจับ bug ที่เกิดจากการแก้ไขโค้ดในภายหลัง
- **Documentation** - Test คือเอกสารที่บอกพฤติกรรมของโค้ด
- **Design Feedback** - โค้ดที่ทดสอบยากมักหมายความว่า Design ไม่ดี
- **Confidence** - ให้ความมั่นใจในการ Refactor

### การติดตั้ง xUnit

```xml
<!-- TaskManager.Tests.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <IsPackable>false</IsPackable>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="xunit" Version="2.9.0" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.8.2" />
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.11.1" />
    <PackageReference Include="Moq" Version="4.20.72" />
    <PackageReference Include="FluentAssertions" Version="6.12.0" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.InMemory" Version="8.0.10" />
    <PackageReference Include="Bogus" Version="35.5.1" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\TaskManager.Domain\TaskManager.Domain.csproj" />
    <ProjectReference Include="..\TaskManager.Application\TaskManager.Application.csproj" />
    <ProjectReference Include="..\TaskManager.Infrastructure\TaskManager.Infrastructure.csproj" />
  </ItemGroup>
</Project>
```

### Attribute [Fact] และ [Theory]

`[Fact]` ใช้สำหรับ Test ที่มี input ค่าเดียวและคาดหวัง output เดียว
`[Theory]` ใช้สำหรับ Test ที่ต้องการทดสอบหลาย input/output combinations

```csharp
// Tests/Unit/CalculatorTests.cs
using Xunit;
using FluentAssertions;

namespace TaskManager.Tests.Unit;

public class CalculatorTests
{
    // [Fact] - ทดสอบกรณีเดียว
    [Fact]
    public void Add_TwoPositiveNumbers_ReturnsSum()
    {
        // Arrange - เตรียมข้อมูลและ dependencies
        var calculator = new Calculator();
        int a = 5;
        int b = 3;

        // Act - เรียก method ที่ต้องการทดสอบ
        int result = calculator.Add(a, b);

        // Assert - ตรวจสอบผลลัพธ์
        Assert.Equal(8, result);
    }

    // [Theory] พร้อม [InlineData] - ทดสอบหลายกรณี
    [Theory]
    [InlineData(1, 2, 3)]
    [InlineData(0, 0, 0)]
    [InlineData(-1, -1, -2)]
    [InlineData(int.MaxValue - 1, 1, int.MaxValue)]
    public void Add_VariousInputs_ReturnsCorrectSum(int a, int b, int expected)
    {
        // Arrange
        var calculator = new Calculator();

        // Act
        int result = calculator.Add(a, b);

        // Assert
        Assert.Equal(expected, result);
    }
}
```

### Test Naming Convention

รูปแบบมาตรฐาน: **MethodName_Scenario_ExpectedResult**

```csharp
// ตัวอย่างชื่อ Test ที่ดี
public void CreateTask_WithValidTitle_ReturnsNewTask()          // ดี
public void CreateTask_WithEmptyTitle_ThrowsDomainException()  // ดี
public void Complete_OnCompletedTask_ThrowsInvalidOperation()  // ดี
public void IsOverdue_WhenDueDateIsPast_ReturnsTrue()         // ดี

// ตัวอย่างชื่อ Test ที่ไม่ดี
public void TestCreate()      // ไม่ดี - ไม่รู้ว่าทดสอบอะไร
public void Test1()           // ไม่ดี - ไม่มีความหมาย
public void ShouldWork()      // ไม่ดี - ไม่ชัดเจน
```

### Assert Methods หลักของ xUnit

```csharp
public class AssertExamplesTests
{
    [Fact]
    public void DemonstrateAssertMethods()
    {
        // Assert.Equal - ตรวจสอบค่าเท่ากัน
        Assert.Equal(5, 2 + 3);
        Assert.Equal("hello", "hello");

        // Assert.NotEqual - ตรวจสอบค่าไม่เท่ากัน
        Assert.NotEqual(5, 2 + 2);

        // Assert.True / Assert.False
        Assert.True(5 > 3);
        Assert.False(5 < 3);

        // Assert.Null / Assert.NotNull
        string? nullValue = null;
        Assert.Null(nullValue);
        Assert.NotNull("not null");

        // Assert.Same / Assert.NotSame (reference equality)
        var obj1 = new object();
        var obj2 = obj1;
        Assert.Same(obj1, obj2);

        // Assert.IsType<T>
        object value = 42;
        Assert.IsType<int>(value);
        Assert.IsAssignableFrom<IComparable>(value);

        // Assert.Contains / Assert.DoesNotContain
        var list = new List<int> { 1, 2, 3, 4, 5 };
        Assert.Contains(3, list);
        Assert.DoesNotContain(6, list);

        // Assert.Empty / Assert.NotEmpty
        Assert.Empty(new List<int>());
        Assert.NotEmpty(list);
    }

    [Fact]
    public void Assert_Throws_CatchesException()
    {
        // Assert.Throws<T> - ตรวจสอบว่า exception ถูก throw
        var exception = Assert.Throws<ArgumentException>(() =>
        {
            throw new ArgumentException("Invalid argument", "paramName");
        });

        Assert.Equal("paramName", exception.ParamName);
        Assert.Contains("Invalid argument", exception.Message);
    }

    [Fact]
    public async Task Assert_ThrowsAsync_ForAsyncMethods()
    {
        // Assert.ThrowsAsync<T> สำหรับ async methods
        await Assert.ThrowsAsync<InvalidOperationException>(async () =>
        {
            await Task.Run(() => throw new InvalidOperationException("Async error"));
        });
    }

    [Fact]
    public void Assert_Collection_ChecksEachElement()
    {
        var items = new List<string> { "Apple", "Banana", "Cherry" };

        // Assert.Collection ตรวจสอบแต่ละ element ตามลำดับ
        Assert.Collection(items,
            item => Assert.Equal("Apple", item),
            item => Assert.StartsWith("Ban", item),
            item => Assert.EndsWith("erry", item)
        );
    }
}
```

---

## Step 512: Testing Domain Entities

### Domain Entity ที่จะทดสอบ

```csharp
// Domain/Entities/TaskItem.cs
public class TaskItem
{
    public Guid Id { get; private set; }
    public string Title { get; private set; }
    public string? Description { get; private set; }
    public TaskStatus Status { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? CompletedAt { get; private set; }
    public DateTime? DueDate { get; private set; }
    public Priority Priority { get; private set; }

    private TaskItem() { } // สำหรับ EF Core

    public static TaskItem Create(string title, string? description = null,
        DateTime? dueDate = null, Priority priority = Priority.Medium)
    {
        if (string.IsNullOrWhiteSpace(title))
            throw new DomainException("Title cannot be empty");

        if (title.Length < 3)
            throw new DomainException("Title must be at least 3 characters");

        if (title.Length > 200)
            throw new DomainException("Title cannot exceed 200 characters");

        if (dueDate.HasValue && dueDate.Value < DateTime.UtcNow.Date)
            throw new DomainException("Due date cannot be in the past");

        return new TaskItem
        {
            Id = Guid.NewGuid(),
            Title = title.Trim(),
            Description = description?.Trim(),
            Status = TaskStatus.Pending,
            CreatedAt = DateTime.UtcNow,
            DueDate = dueDate,
            Priority = priority
        };
    }

    public void Complete()
    {
        if (Status == TaskStatus.Completed)
            throw new DomainException("Task is already completed");

        if (Status == TaskStatus.Cancelled)
            throw new DomainException("Cannot complete a cancelled task");

        Status = TaskStatus.Completed;
        CompletedAt = DateTime.UtcNow;
    }

    public void Cancel(string reason)
    {
        if (Status == TaskStatus.Completed)
            throw new DomainException("Cannot cancel a completed task");

        Status = TaskStatus.Cancelled;
    }

    public bool IsOverdue()
    {
        if (!DueDate.HasValue) return false;
        if (Status == TaskStatus.Completed || Status == TaskStatus.Cancelled) return false;
        return DateTime.UtcNow > DueDate.Value;
    }

    public void UpdateTitle(string newTitle)
    {
        if (string.IsNullOrWhiteSpace(newTitle))
            throw new DomainException("Title cannot be empty");

        if (newTitle.Length < 3)
            throw new DomainException("Title must be at least 3 characters");

        if (Status == TaskStatus.Completed)
            throw new DomainException("Cannot update a completed task");

        Title = newTitle.Trim();
    }
}
```

### ทดสอบ TaskItem.Create

```csharp
// Tests/Unit/Domain/TaskItemTests.cs
using FluentAssertions;
using TaskManager.Domain.Entities;
using TaskManager.Domain.Exceptions;
using Xunit;

namespace TaskManager.Tests.Unit.Domain;

public class TaskItemCreateTests
{
    [Fact]
    public void Create_WithValidTitle_ReturnsTaskWithCorrectProperties()
    {
        // Arrange
        string title = "Complete project documentation";
        string description = "Write detailed API docs";
        var dueDate = DateTime.UtcNow.Date.AddDays(7);

        // Act
        var task = TaskItem.Create(title, description, dueDate);

        // Assert
        task.Should().NotBeNull();
        task.Id.Should().NotBe(Guid.Empty);
        task.Title.Should().Be(title);
        task.Description.Should().Be(description);
        task.Status.Should().Be(TaskStatus.Pending);
        task.DueDate.Should().Be(dueDate);
        task.CreatedAt.Should().BeCloseTo(DateTime.UtcNow, TimeSpan.FromSeconds(5));
        task.CompletedAt.Should().BeNull();
    }

    [Fact]
    public void Create_WithEmptyTitle_ThrowsDomainException()
    {
        // Arrange
        string emptyTitle = "";

        // Act
        Action act = () => TaskItem.Create(emptyTitle);

        // Assert
        act.Should().Throw<DomainException>()
           .WithMessage("Title cannot be empty");
    }

    [Fact]
    public void Create_WithWhitespaceTitle_ThrowsDomainException()
    {
        // Arrange & Act
        Action act = () => TaskItem.Create("   ");

        // Assert
        act.Should().Throw<DomainException>()
           .WithMessage("Title cannot be empty");
    }

    [Theory]
    [InlineData("ab")]     // สั้นเกินไป - 2 ตัวอักษร
    [InlineData("a")]      // สั้นเกินไป - 1 ตัวอักษร
    public void Create_WithTitleTooShort_ThrowsDomainException(string shortTitle)
    {
        Action act = () => TaskItem.Create(shortTitle);

        act.Should().Throw<DomainException>()
           .WithMessage("Title must be at least 3 characters");
    }

    [Fact]
    public void Create_WithTitleTooLong_ThrowsDomainException()
    {
        // Arrange - สร้าง title ยาว 201 ตัวอักษร
        string longTitle = new string('A', 201);

        // Act
        Action act = () => TaskItem.Create(longTitle);

        // Assert
        act.Should().Throw<DomainException>()
           .WithMessage("Title cannot exceed 200 characters");
    }

    [Fact]
    public void Create_WithPastDueDate_ThrowsDomainException()
    {
        // Arrange
        var pastDate = DateTime.UtcNow.Date.AddDays(-1);

        // Act
        Action act = () => TaskItem.Create("Valid Title", dueDate: pastDate);

        // Assert
        act.Should().Throw<DomainException>()
           .WithMessage("Due date cannot be in the past");
    }

    [Fact]
    public void Create_WithTitleHavingLeadingWhitespace_TrimsTitle()
    {
        // Arrange
        string titleWithSpaces = "  My Task Title  ";

        // Act
        var task = TaskItem.Create(titleWithSpaces);

        // Assert
        task.Title.Should().Be("My Task Title");
    }
}
```

### ทดสอบ TaskItem.Complete

```csharp
public class TaskItemCompleteTests
{
    [Fact]
    public void Complete_OnPendingTask_SetsStatusToCompleted()
    {
        // Arrange
        var task = TaskItem.Create("My Task");

        // Act
        task.Complete();

        // Assert
        task.Status.Should().Be(TaskStatus.Completed);
        task.CompletedAt.Should().NotBeNull();
        task.CompletedAt.Should().BeCloseTo(DateTime.UtcNow, TimeSpan.FromSeconds(5));
    }

    [Fact]
    public void Complete_AlreadyCompletedTask_ThrowsDomainException()
    {
        // Arrange
        var task = TaskItem.Create("My Task");
        task.Complete(); // complete ครั้งแรก

        // Act
        Action act = () => task.Complete(); // ลอง complete ซ้ำ

        // Assert
        act.Should().Throw<DomainException>()
           .WithMessage("Task is already completed");
    }

    [Fact]
    public void Complete_CancelledTask_ThrowsDomainException()
    {
        // Arrange
        var task = TaskItem.Create("My Task");
        task.Cancel("No longer needed");

        // Act
        Action act = () => task.Complete();

        // Assert
        act.Should().Throw<DomainException>()
           .WithMessage("Cannot complete a cancelled task");
    }
}
```

### ทดสอบ TaskItem.IsOverdue

```csharp
public class TaskItemIsOverdueTests
{
    [Fact]
    public void IsOverdue_WithNoDueDate_ReturnsFalse()
    {
        // Arrange
        var task = TaskItem.Create("Task without due date");

        // Act
        bool result = task.IsOverdue();

        // Assert
        result.Should().BeFalse();
    }

    [Fact]
    public void IsOverdue_WithFutureDueDate_ReturnsFalse()
    {
        // Arrange
        var task = TaskItem.Create("Task", dueDate: DateTime.UtcNow.Date.AddDays(1));

        // Act
        bool result = task.IsOverdue();

        // Assert
        result.Should().BeFalse();
    }

    [Fact]
    public void IsOverdue_PastDueDateAndPending_ReturnsTrue()
    {
        // Arrange - ต้องใช้ reflection หรือ factory method ที่รับ date ได้
        // เนื่องจาก Create ไม่ยอมรับ past date ในปกติ เราต้องใช้วิธีพิเศษ
        var task = CreateTaskWithPastDueDate(DateTime.UtcNow.AddDays(-1));

        // Act
        bool result = task.IsOverdue();

        // Assert
        result.Should().BeTrue();
    }

    [Fact]
    public void IsOverdue_CompletedTaskWithPastDueDate_ReturnsFalse()
    {
        // Arrange
        var task = CreateTaskWithPastDueDate(DateTime.UtcNow.AddDays(-1));
        // ต้องใช้ reflection เพื่อ set status ก่อน complete
        task.Complete();

        // Act
        bool result = task.IsOverdue();

        // Assert
        result.Should().BeFalse("completed tasks are not considered overdue");
    }

    // Helper method - ใช้ reflection สร้าง task ที่มี past due date
    private static TaskItem CreateTaskWithPastDueDate(DateTime pastDate)
    {
        var task = TaskItem.Create("Overdue Task");
        var dueDateProp = typeof(TaskItem).GetProperty("DueDate");
        dueDateProp?.SetValue(task, pastDate);
        return task;
    }
}
```

---

## Step 513: Moq - การ Mock Dependencies

### ทำความเข้าใจ Mocking

Mocking คือการสร้าง "ตัวแทน" ของ dependency เพื่อให้เราทดสอบ class เป้าหมายในแบบแยกส่วน โดยไม่ต้องพึ่งพา implementation จริง

```csharp
// Interface ที่จะ mock
public interface ITaskRepository
{
    Task<TaskItem?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<List<TaskItem>> GetAllAsync(CancellationToken ct = default);
    Task AddAsync(TaskItem task, CancellationToken ct = default);
    Task UpdateAsync(TaskItem task, CancellationToken ct = default);
    Task DeleteAsync(Guid id, CancellationToken ct = default);
    Task<bool> ExistsAsync(Guid id, CancellationToken ct = default);
}
```

### การสร้าง Mock พื้นฐาน

```csharp
// Tests/Unit/Mocking/BasicMockingTests.cs
using Moq;
using Xunit;
using FluentAssertions;

namespace TaskManager.Tests.Unit.Mocking;

public class BasicMockingTests
{
    [Fact]
    public async Task GetByIdAsync_MockReturnsTask_ServiceReceivesIt()
    {
        // Arrange
        var expectedTask = TaskItem.Create("Test Task");
        var mockRepo = new Mock<ITaskRepository>();

        // Setup - กำหนดค่าที่ mock จะ return เมื่อถูกเรียก
        mockRepo.Setup(r => r.GetByIdAsync(expectedTask.Id, It.IsAny<CancellationToken>()))
                .ReturnsAsync(expectedTask);

        // Act - เรียก method ผ่าน mock
        var result = await mockRepo.Object.GetByIdAsync(expectedTask.Id);

        // Assert
        result.Should().NotBeNull();
        result.Should().Be(expectedTask);
    }

    [Fact]
    public async Task AddAsync_WhenCalled_VerifyWasCalledOnce()
    {
        // Arrange
        var task = TaskItem.Create("New Task");
        var mockRepo = new Mock<ITaskRepository>();

        // Setup - กำหนดให้ AddAsync ทำงานโดยไม่ return ค่า
        mockRepo.Setup(r => r.AddAsync(It.IsAny<TaskItem>(), It.IsAny<CancellationToken>()))
                .Returns(Task.CompletedTask);

        // Act
        await mockRepo.Object.AddAsync(task);

        // Assert - ตรวจสอบว่า AddAsync ถูกเรียกครั้งเดียว
        mockRepo.Verify(r => r.AddAsync(task, It.IsAny<CancellationToken>()), Times.Once);
    }

    [Fact]
    public async Task GetAllAsync_WhenNoTasks_ReturnsEmptyList()
    {
        // Arrange
        var mockRepo = new Mock<ITaskRepository>();
        mockRepo.Setup(r => r.GetAllAsync(It.IsAny<CancellationToken>()))
                .ReturnsAsync(new List<TaskItem>());

        // Act
        var result = await mockRepo.Object.GetAllAsync();

        // Assert
        result.Should().BeEmpty();
    }
}
```

### It.IsAny<T> และ It.Is<T>

```csharp
public class MockMatcherTests
{
    [Fact]
    public async Task It_IsAny_MatchesAnyValue()
    {
        var mockRepo = new Mock<ITaskRepository>();

        // It.IsAny<T> - ยอมรับ argument ค่าใดก็ได้
        mockRepo.Setup(r => r.GetByIdAsync(It.IsAny<Guid>(), It.IsAny<CancellationToken>()))
                .ReturnsAsync((TaskItem?)null);

        // เรียกด้วย Guid ใดก็ได้
        var result1 = await mockRepo.Object.GetByIdAsync(Guid.NewGuid());
        var result2 = await mockRepo.Object.GetByIdAsync(Guid.NewGuid());

        result1.Should().BeNull();
        result2.Should().BeNull();
    }

    [Fact]
    public async Task It_Is_MatchesWithPredicate()
    {
        var mockRepo = new Mock<ITaskRepository>();
        var specificId = Guid.NewGuid();

        // It.Is<T>(predicate) - ยอมรับเฉพาะ argument ที่ตรงตาม predicate
        mockRepo.Setup(r => r.ExistsAsync(
                    It.Is<Guid>(id => id == specificId),
                    It.IsAny<CancellationToken>()))
                .ReturnsAsync(true);

        // ตรงกับ predicate - return true
        var exists = await mockRepo.Object.ExistsAsync(specificId);
        exists.Should().BeTrue();

        // ไม่ตรงกับ predicate - return false (default)
        var notExists = await mockRepo.Object.ExistsAsync(Guid.NewGuid());
        notExists.Should().BeFalse();
    }
}
```

### Mock Sequences และ Callbacks

```csharp
public class MockAdvancedTests
{
    [Fact]
    public async Task SetupSequence_ReturnsDifferentValuesEachCall()
    {
        var mockRepo = new Mock<ITaskRepository>();
        var task1 = TaskItem.Create("Task 1");
        var task2 = TaskItem.Create("Task 2");

        // SetupSequence - คืนค่าต่างกันในแต่ละการเรียก
        mockRepo.SetupSequence(r => r.GetByIdAsync(It.IsAny<Guid>(), It.IsAny<CancellationToken>()))
                .ReturnsAsync(task1)   // ครั้งที่ 1
                .ReturnsAsync(task2)   // ครั้งที่ 2
                .ReturnsAsync(null);   // ครั้งที่ 3 และหลังจากนั้น

        var result1 = await mockRepo.Object.GetByIdAsync(Guid.NewGuid());
        var result2 = await mockRepo.Object.GetByIdAsync(Guid.NewGuid());
        var result3 = await mockRepo.Object.GetByIdAsync(Guid.NewGuid());

        result1.Should().Be(task1);
        result2.Should().Be(task2);
        result3.Should().BeNull();
    }

    [Fact]
    public async Task Callback_ExecutesCustomLogic()
    {
        var mockRepo = new Mock<ITaskRepository>();
        var addedTasks = new List<TaskItem>();

        // Callback - รัน logic พิเศษเมื่อ method ถูกเรียก
        mockRepo.Setup(r => r.AddAsync(It.IsAny<TaskItem>(), It.IsAny<CancellationToken>()))
                .Callback<TaskItem, CancellationToken>((task, ct) => addedTasks.Add(task))
                .Returns(Task.CompletedTask);

        var task = TaskItem.Create("Callback Task");
        await mockRepo.Object.AddAsync(task);

        addedTasks.Should().HaveCount(1);
        addedTasks[0].Should().Be(task);
    }

    [Fact]
    public async Task MockThrows_SimulatesExceptions()
    {
        var mockRepo = new Mock<ITaskRepository>();

        // Setup ให้ throw exception
        mockRepo.Setup(r => r.GetByIdAsync(It.IsAny<Guid>(), It.IsAny<CancellationToken>()))
                .ThrowsAsync(new RepositoryException("Database connection failed"));

        // Act & Assert
        Func<Task> act = () => mockRepo.Object.GetByIdAsync(Guid.NewGuid());

        await act.Should().ThrowAsync<RepositoryException>()
                 .WithMessage("Database connection failed");
    }

    [Fact]
    public void Verify_NeverCalled_PassesAssertion()
    {
        var mockRepo = new Mock<ITaskRepository>();

        // ไม่ได้เรียก AddAsync เลย
        mockRepo.Verify(r => r.AddAsync(It.IsAny<TaskItem>(), It.IsAny<CancellationToken>()),
                        Times.Never);
    }

    [Fact]
    public async Task Verify_CalledNTimes_CountMatches()
    {
        var mockRepo = new Mock<ITaskRepository>();
        mockRepo.Setup(r => r.AddAsync(It.IsAny<TaskItem>(), It.IsAny<CancellationToken>()))
                .Returns(Task.CompletedTask);

        // เรียก 3 ครั้ง
        await mockRepo.Object.AddAsync(TaskItem.Create("Task 1"));
        await mockRepo.Object.AddAsync(TaskItem.Create("Task 2"));
        await mockRepo.Object.AddAsync(TaskItem.Create("Task 3"));

        // Verify ว่าถูกเรียก 3 ครั้งพอดี
        mockRepo.Verify(r => r.AddAsync(It.IsAny<TaskItem>(), It.IsAny<CancellationToken>()),
                        Times.Exactly(3));
    }
}
```

---

## Step 514: Testing Application Layer (Command Handlers)

### Command Handler ที่จะทดสอบ

```csharp
// Application/Commands/CreateTask/CreateTaskCommand.cs
public record CreateTaskCommand(
    string Title,
    string? Description,
    DateTime? DueDate,
    Priority Priority
) : IRequest<Guid>;

// Application/Commands/CreateTask/CreateTaskCommandHandler.cs
public class CreateTaskCommandHandler : IRequestHandler<CreateTaskCommand, Guid>
{
    private readonly ITaskRepository _taskRepository;
    private readonly IUnitOfWork _unitOfWork;
    private readonly ILogger<CreateTaskCommandHandler> _logger;

    public CreateTaskCommandHandler(
        ITaskRepository taskRepository,
        IUnitOfWork unitOfWork,
        ILogger<CreateTaskCommandHandler> logger)
    {
        _taskRepository = taskRepository;
        _unitOfWork = unitOfWork;
        _logger = logger;
    }

    public async Task<Guid> Handle(CreateTaskCommand request, CancellationToken ct)
    {
        var task = TaskItem.Create(request.Title, request.Description,
                                   request.DueDate, request.Priority);

        await _taskRepository.AddAsync(task, ct);
        await _unitOfWork.SaveChangesAsync(ct);

        _logger.LogInformation("Task {TaskId} created", task.Id);

        return task.Id;
    }
}
```

### ทดสอบ Command Handler

```csharp
// Tests/Unit/Application/CreateTaskCommandHandlerTests.cs
public class CreateTaskCommandHandlerTests
{
    private readonly Mock<ITaskRepository> _mockTaskRepo;
    private readonly Mock<IUnitOfWork> _mockUnitOfWork;
    private readonly Mock<ILogger<CreateTaskCommandHandler>> _mockLogger;
    private readonly CreateTaskCommandHandler _handler;

    public CreateTaskCommandHandlerTests()
    {
        _mockTaskRepo = new Mock<ITaskRepository>();
        _mockUnitOfWork = new Mock<IUnitOfWork>();
        _mockLogger = new Mock<ILogger<CreateTaskCommandHandler>>();

        // Setup defaults
        _mockTaskRepo.Setup(r => r.AddAsync(It.IsAny<TaskItem>(), It.IsAny<CancellationToken>()))
                     .Returns(Task.CompletedTask);
        _mockUnitOfWork.Setup(u => u.SaveChangesAsync(It.IsAny<CancellationToken>()))
                       .ReturnsAsync(1);

        _handler = new CreateTaskCommandHandler(
            _mockTaskRepo.Object,
            _mockUnitOfWork.Object,
            _mockLogger.Object);
    }

    [Fact]
    public async Task Handle_ValidCommand_CreatesTaskAndReturnsId()
    {
        // Arrange
        var command = new CreateTaskCommand(
            "Complete unit tests",
            "Write comprehensive tests",
            DateTime.UtcNow.Date.AddDays(7),
            Priority.High);

        // Act
        var taskId = await _handler.Handle(command, CancellationToken.None);

        // Assert
        taskId.Should().NotBe(Guid.Empty);

        // Verify repository was called
        _mockTaskRepo.Verify(r => r.AddAsync(
            It.Is<TaskItem>(t => t.Title == command.Title),
            It.IsAny<CancellationToken>()), Times.Once);

        // Verify unit of work was saved
        _mockUnitOfWork.Verify(u => u.SaveChangesAsync(It.IsAny<CancellationToken>()), Times.Once);
    }

    [Fact]
    public async Task Handle_InvalidTitle_ThrowsDomainException()
    {
        // Arrange - command ที่มี title ไม่ถูกต้อง
        var command = new CreateTaskCommand("ab", null, null, Priority.Low);

        // Act
        Func<Task> act = () => _handler.Handle(command, CancellationToken.None);

        // Assert
        await act.Should().ThrowAsync<DomainException>()
                 .WithMessage("Title must be at least 3 characters");

        // ตรวจสอบว่า repository และ unit of work ไม่ถูกเรียกเลย
        _mockTaskRepo.Verify(r => r.AddAsync(It.IsAny<TaskItem>(), It.IsAny<CancellationToken>()),
                             Times.Never);
        _mockUnitOfWork.Verify(u => u.SaveChangesAsync(It.IsAny<CancellationToken>()),
                               Times.Never);
    }

    [Fact]
    public async Task Handle_RepositoryThrows_PropagatesException()
    {
        // Arrange
        var command = new CreateTaskCommand("Valid Task", null, null, Priority.Medium);
        _mockTaskRepo.Setup(r => r.AddAsync(It.IsAny<TaskItem>(), It.IsAny<CancellationToken>()))
                     .ThrowsAsync(new RepositoryException("DB Error"));

        // Act
        Func<Task> act = () => _handler.Handle(command, CancellationToken.None);

        // Assert
        await act.Should().ThrowAsync<RepositoryException>()
                 .WithMessage("DB Error");

        // UnitOfWork ไม่ควรถูกเรียกถ้า repository throw ก่อน
        _mockUnitOfWork.Verify(u => u.SaveChangesAsync(It.IsAny<CancellationToken>()),
                               Times.Never);
    }
}
```

### ทดสอบ Query Handler

```csharp
// Application/Queries/GetTaskList/GetTaskListQuery.cs
public record GetTaskListQuery(
    TaskStatus? Status = null,
    Priority? Priority = null,
    bool OnlyOverdue = false
) : IRequest<List<TaskListItemDto>>;

public class GetTaskListQueryHandler : IRequestHandler<GetTaskListQuery, List<TaskListItemDto>>
{
    private readonly ITaskRepository _taskRepository;
    private readonly IMapper _mapper;

    public GetTaskListQueryHandler(ITaskRepository taskRepository, IMapper mapper)
    {
        _taskRepository = taskRepository;
        _mapper = mapper;
    }

    public async Task<List<TaskListItemDto>> Handle(GetTaskListQuery request, CancellationToken ct)
    {
        var tasks = await _taskRepository.GetAllAsync(ct);

        var filtered = tasks.AsQueryable();

        if (request.Status.HasValue)
            filtered = filtered.Where(t => t.Status == request.Status.Value);

        if (request.Priority.HasValue)
            filtered = filtered.Where(t => t.Priority == request.Priority.Value);

        if (request.OnlyOverdue)
            filtered = filtered.Where(t => t.IsOverdue());

        return _mapper.Map<List<TaskListItemDto>>(filtered.ToList());
    }
}
```

```csharp
// Tests/Unit/Application/GetTaskListQueryHandlerTests.cs
public class GetTaskListQueryHandlerTests
{
    private readonly Mock<ITaskRepository> _mockRepo;
    private readonly Mock<IMapper> _mockMapper;
    private readonly GetTaskListQueryHandler _handler;
    private readonly List<TaskItem> _testTasks;

    public GetTaskListQueryHandlerTests()
    {
        _mockRepo = new Mock<ITaskRepository>();
        _mockMapper = new Mock<IMapper>();

        // สร้าง test data
        _testTasks = new List<TaskItem>
        {
            TaskItem.Create("High Priority Task", priority: Priority.High),
            TaskItem.Create("Low Priority Task", priority: Priority.Low),
            TaskItem.Create("Medium Priority Task", priority: Priority.Medium),
        };

        // Task 1 จะ complete
        _testTasks[0].Complete();

        _mockRepo.Setup(r => r.GetAllAsync(It.IsAny<CancellationToken>()))
                 .ReturnsAsync(_testTasks);

        // Setup mapper - map TaskItem to DTO
        _mockMapper.Setup(m => m.Map<List<TaskListItemDto>>(It.IsAny<List<TaskItem>>()))
                   .Returns((List<TaskItem> items) => items.Select(t => new TaskListItemDto
                   {
                       Id = t.Id,
                       Title = t.Title,
                       Status = t.Status,
                       Priority = t.Priority
                   }).ToList());

        _handler = new GetTaskListQueryHandler(_mockRepo.Object, _mockMapper.Object);
    }

    [Fact]
    public async Task Handle_NoFilter_ReturnsAllTasks()
    {
        var query = new GetTaskListQuery();

        var result = await _handler.Handle(query, CancellationToken.None);

        result.Should().HaveCount(3);
    }

    [Fact]
    public async Task Handle_FilterByStatus_ReturnsOnlyMatchingTasks()
    {
        var query = new GetTaskListQuery(Status: TaskStatus.Completed);

        var result = await _handler.Handle(query, CancellationToken.None);

        result.Should().HaveCount(1);
        result.Should().AllSatisfy(t => t.Status.Should().Be(TaskStatus.Completed));
    }

    [Fact]
    public async Task Handle_FilterByHighPriority_ReturnsOnlyHighPriorityTasks()
    {
        var query = new GetTaskListQuery(Priority: Priority.High);

        var result = await _handler.Handle(query, CancellationToken.None);

        result.Should().HaveCount(1);
        result[0].Title.Should().Be("High Priority Task");
    }
}
```

---

## Step 515: FluentAssertions - การ Assert แบบอ่านง่าย

### ทำไม FluentAssertions ถึงดีกว่า Assert ธรรมดา?

```csharp
// xUnit Assert ธรรมดา - อ่านยาก ข้อความ error ไม่ชัดเจน
Assert.Equal(3, list.Count);
Assert.True(list.Any(x => x.Title == "Test"));

// FluentAssertions - อ่านง่ายเหมือนภาษาธรรมชาติ
list.Should().HaveCount(3);
list.Should().Contain(x => x.Title == "Test");
```

### Assertion สำหรับค่าพื้นฐาน

```csharp
public class FluentAssertionBasicsTests
{
    [Fact]
    public void BasicValueAssertions()
    {
        int number = 42;
        string text = "Hello World";
        bool flag = true;

        // Numeric assertions
        number.Should().Be(42);
        number.Should().NotBe(0);
        number.Should().BeGreaterThan(40);
        number.Should().BeLessThanOrEqualTo(42);
        number.Should().BeInRange(40, 50);
        number.Should().BePositive();

        // String assertions
        text.Should().Be("Hello World");
        text.Should().StartWith("Hello");
        text.Should().EndWith("World");
        text.Should().Contain("lo Wo");
        text.Should().HaveLength(11);
        text.Should().NotBeNullOrEmpty();
        text.Should().NotBeNullOrWhiteSpace();
        text.Should().MatchRegex(@"^Hello \w+$");

        // Boolean assertions
        flag.Should().BeTrue();
        (!flag).Should().BeFalse();
    }

    [Fact]
    public void NullableAndTypeAssertions()
    {
        object? nullObj = null;
        object nonNullObj = new();

        nullObj.Should().BeNull();
        nonNullObj.Should().NotBeNull();

        // Type assertions
        object value = 42;
        value.Should().BeOfType<int>();
        value.Should().BeAssignableTo<IComparable>();
    }

    [Fact]
    public void DateTimeAssertions()
    {
        var now = DateTime.UtcNow;
        var past = now.AddDays(-1);
        var future = now.AddDays(1);

        now.Should().BeAfter(past);
        now.Should().BeBefore(future);
        now.Should().BeCloseTo(DateTime.UtcNow, TimeSpan.FromSeconds(5));
        now.Should().BeWithin(TimeSpan.FromMinutes(1)).Before(future);
    }
}
```

### Assertion สำหรับ Collections

```csharp
public class FluentAssertionCollectionTests
{
    private readonly List<TaskItem> _tasks = new()
    {
        TaskItem.Create("Task Alpha", priority: Priority.High),
        TaskItem.Create("Task Beta", priority: Priority.Medium),
        TaskItem.Create("Task Gamma", priority: Priority.Low),
    };

    [Fact]
    public void CollectionBasicAssertions()
    {
        _tasks.Should().NotBeNull();
        _tasks.Should().HaveCount(3);
        _tasks.Should().NotBeEmpty();
        _tasks.Should().HaveCountGreaterThan(1);
        _tasks.Should().HaveCountLessThanOrEqualTo(10);
    }

    [Fact]
    public void CollectionElementAssertions()
    {
        _tasks.Should().Contain(t => t.Title == "Task Alpha");
        _tasks.Should().NotContain(t => t.Title == "Nonexistent Task");
        _tasks.Should().ContainSingle(t => t.Priority == Priority.High);
        _tasks.Should().OnlyContain(t => t.Status == TaskStatus.Pending);
    }

    [Fact]
    public void CollectionOrderingAssertions()
    {
        var ordered = _tasks.OrderBy(t => t.Title).ToList();

        ordered.Should().BeInAscendingOrder(t => t.Title);
        ordered.Should().HaveElementAt(0, ordered[0]);
        ordered.Should().StartWith(ordered[0]);
        ordered.Should().EndWith(ordered[^1]);
    }

    [Fact]
    public void CollectionEquivalenceAssertions()
    {
        var ids = _tasks.Select(t => t.Id).ToList();
        var sameIds = _tasks.Select(t => t.Id).ToList(); // copy

        // BeEquivalentTo - ตรวจสอบว่า collections มีเนื้อหาเหมือนกัน (ไม่สนใจลำดับ)
        ids.Should().BeEquivalentTo(sameIds);
    }

    [Fact]
    public void ChainedAssertions_WithAnd()
    {
        // ใช้ .And เพื่อ chain assertions
        _tasks.Should()
              .HaveCount(3)
              .And.Contain(t => t.Priority == Priority.High)
              .And.NotContain(t => t.Status == TaskStatus.Completed);
    }
}
```

### Assertion สำหรับ Exceptions

```csharp
public class FluentAssertionExceptionTests
{
    [Fact]
    public void Exception_Synchronous_Assertions()
    {
        // Arrange
        Action act = () => TaskItem.Create("ab");

        // Assert - Throw ถูกต้องและมี message ถูกต้อง
        act.Should().Throw<DomainException>()
           .WithMessage("Title must be at least 3 characters")
           .And.Message.Should().Contain("3 characters");
    }

    [Fact]
    public async Task Exception_Async_Assertions()
    {
        // Arrange
        Func<Task> act = async () =>
        {
            await Task.Run(() => throw new InvalidOperationException("Async error"));
        };

        // Assert
        await act.Should().ThrowAsync<InvalidOperationException>()
                 .WithMessage("Async error");
    }

    [Fact]
    public void Exception_NotThrown_Assertions()
    {
        Action act = () => TaskItem.Create("Valid Task");

        act.Should().NotThrow();
    }

    [Fact]
    public void Exception_InnerException_Assertions()
    {
        Action act = () =>
        {
            try { throw new DomainException("Original"); }
            catch (Exception ex) { throw new ApplicationException("Wrapper", ex); }
        };

        act.Should().Throw<ApplicationException>()
           .WithMessage("Wrapper")
           .WithInnerException<DomainException>()
           .WithMessage("Original");
    }
}
```

### Assertion สำหรับ Objects

```csharp
public class FluentAssertionObjectTests
{
    [Fact]
    public void ObjectEquivalenceAssertions()
    {
        var dto1 = new TaskDto { Id = Guid.NewGuid(), Title = "Task 1", Status = TaskStatus.Pending };
        var dto2 = new TaskDto { Id = dto1.Id, Title = "Task 1", Status = TaskStatus.Pending };

        // BeEquivalentTo - ตรวจสอบว่า properties เท่ากัน (deep equality)
        dto1.Should().BeEquivalentTo(dto2);

        // ยกเว้น property บางตัว
        var dto3 = new TaskDto { Id = Guid.NewGuid(), Title = "Task 1", Status = TaskStatus.Pending };
        dto1.Should().BeEquivalentTo(dto3, options =>
            options.Excluding(x => x.Id));
    }

    [Fact]
    public void ObjectSatisfyAssertions()
    {
        var task = TaskItem.Create("My Task", priority: Priority.High);

        // Satisfy - ตรวจสอบหลาย conditions พร้อมกัน
        task.Should().Satisfy<TaskItem>(t =>
        {
            t.Title.Should().Be("My Task");
            t.Priority.Should().Be(Priority.High);
            t.Status.Should().Be(TaskStatus.Pending);
        });
    }
}
```

---

## Step 516: Integration Tests ด้วย EF Core InMemory

### ตั้งค่า Integration Test Project

```csharp
// Tests/Integration/Infrastructure/TaskRepositoryIntegrationTests.cs
using Microsoft.EntityFrameworkCore;
using TaskManager.Infrastructure.Data;
using TaskManager.Infrastructure.Repositories;
using Xunit;
using FluentAssertions;

namespace TaskManager.Tests.Integration.Infrastructure;

// IClassFixture - ใช้ DbContext เดิมร่วมกันใน test class
public class TaskRepositoryIntegrationTests : IClassFixture<TaskDbContextFixture>
{
    private readonly TaskDbContextFixture _fixture;

    public TaskRepositoryIntegrationTests(TaskDbContextFixture fixture)
    {
        _fixture = fixture;
        // ล้างข้อมูลก่อนแต่ละ test
        _fixture.ResetDatabase();
    }

    [Fact]
    public async Task AddAsync_ValidTask_PersistsToDatabase()
    {
        // Arrange
        var context = _fixture.CreateContext();
        var repository = new TaskRepository(context);
        var task = TaskItem.Create("Integration Test Task");

        // Act
        await repository.AddAsync(task);
        await context.SaveChangesAsync();

        // Assert - ตรวจสอบด้วย fresh context
        var verifyContext = _fixture.CreateContext();
        var saved = await verifyContext.Tasks.FindAsync(task.Id);

        saved.Should().NotBeNull();
        saved!.Title.Should().Be("Integration Test Task");
        saved.Status.Should().Be(TaskStatus.Pending);
    }

    [Fact]
    public async Task GetByIdAsync_ExistingTask_ReturnsTask()
    {
        // Arrange
        var context = _fixture.CreateContext();
        var task = TaskItem.Create("Find Me Task");
        context.Tasks.Add(task);
        await context.SaveChangesAsync();

        var repository = new TaskRepository(context);

        // Act
        var result = await repository.GetByIdAsync(task.Id);

        // Assert
        result.Should().NotBeNull();
        result!.Id.Should().Be(task.Id);
        result.Title.Should().Be("Find Me Task");
    }

    [Fact]
    public async Task GetByIdAsync_NonExistentId_ReturnsNull()
    {
        // Arrange
        var context = _fixture.CreateContext();
        var repository = new TaskRepository(context);

        // Act
        var result = await repository.GetByIdAsync(Guid.NewGuid());

        // Assert
        result.Should().BeNull();
    }

    [Fact]
    public async Task GetAllAsync_MultipleTasksExist_ReturnsAllTasks()
    {
        // Arrange
        var context = _fixture.CreateContext();
        var tasks = new[]
        {
            TaskItem.Create("Task A"),
            TaskItem.Create("Task B"),
            TaskItem.Create("Task C"),
        };
        context.Tasks.AddRange(tasks);
        await context.SaveChangesAsync();

        var repository = new TaskRepository(context);

        // Act
        var result = await repository.GetAllAsync();

        // Assert
        result.Should().HaveCount(3);
        result.Select(t => t.Title).Should().BeEquivalentTo(new[] { "Task A", "Task B", "Task C" });
    }

    [Fact]
    public async Task UpdateAsync_ModifyTask_PersistsChanges()
    {
        // Arrange
        var context = _fixture.CreateContext();
        var task = TaskItem.Create("Original Title");
        context.Tasks.Add(task);
        await context.SaveChangesAsync();

        // Act - แก้ไข title และ save
        var updateContext = _fixture.CreateContext();
        var existingTask = await updateContext.Tasks.FindAsync(task.Id);
        existingTask!.UpdateTitle("Updated Title");
        await updateContext.SaveChangesAsync();

        // Assert
        var verifyContext = _fixture.CreateContext();
        var updated = await verifyContext.Tasks.FindAsync(task.Id);
        updated!.Title.Should().Be("Updated Title");
    }

    [Fact]
    public async Task DeleteAsync_ExistingTask_RemovesFromDatabase()
    {
        // Arrange
        var context = _fixture.CreateContext();
        var task = TaskItem.Create("Delete Me");
        context.Tasks.Add(task);
        await context.SaveChangesAsync();

        var repository = new TaskRepository(context);

        // Act
        await repository.DeleteAsync(task.Id);
        await context.SaveChangesAsync();

        // Assert
        var verifyContext = _fixture.CreateContext();
        var deleted = await verifyContext.Tasks.FindAsync(task.Id);
        deleted.Should().BeNull();
    }
}
```

### TaskDbContextFixture

```csharp
// Tests/Integration/Fixtures/TaskDbContextFixture.cs
public class TaskDbContextFixture : IDisposable
{
    private readonly string _dbName;
    private readonly DbContextOptions<TaskDbContext> _options;

    public TaskDbContextFixture()
    {
        _dbName = $"TestDb_{Guid.NewGuid():N}";
        _options = new DbContextOptionsBuilder<TaskDbContext>()
            .UseInMemoryDatabase(_dbName)
            .Options;

        // สร้าง schema
        using var context = new TaskDbContext(_options);
        context.Database.EnsureCreated();
    }

    public TaskDbContext CreateContext()
    {
        return new TaskDbContext(_options);
    }

    public void ResetDatabase()
    {
        using var context = new TaskDbContext(_options);
        // ลบข้อมูลทั้งหมดแต่เก็บ schema ไว้
        context.Tasks.RemoveRange(context.Tasks);
        context.SaveChanges();
    }

    public void Dispose()
    {
        using var context = new TaskDbContext(_options);
        context.Database.EnsureDeleted();
    }
}
```

### ใช้ WebApplicationFactory สำหรับ HTTP Integration Tests

```csharp
// Tests/Integration/Api/TasksApiIntegrationTests.cs
using Microsoft.AspNetCore.Mvc.Testing;
using System.Net.Http.Json;

public class TasksApiIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public TasksApiIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // แทน real DB ด้วย InMemory
                var descriptor = services.SingleOrDefault(
                    d => d.ServiceType == typeof(DbContextOptions<TaskDbContext>));
                if (descriptor != null)
                    services.Remove(descriptor);

                services.AddDbContext<TaskDbContext>(options =>
                    options.UseInMemoryDatabase("IntegrationTestDb"));
            });
        }).CreateClient();
    }

    [Fact]
    public async Task POST_Tasks_ValidRequest_Returns201Created()
    {
        // Arrange
        var request = new CreateTaskRequest
        {
            Title = "API Test Task",
            Description = "Testing via HTTP",
            Priority = "High"
        };

        // Act
        var response = await _client.PostAsJsonAsync("/api/tasks", request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        var content = await response.Content.ReadFromJsonAsync<CreateTaskResponse>();
        content.Should().NotBeNull();
        content!.Id.Should().NotBe(Guid.Empty);
    }

    [Fact]
    public async Task GET_Tasks_ReturnsTaskList()
    {
        // Act
        var response = await _client.GetAsync("/api/tasks");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var tasks = await response.Content.ReadFromJsonAsync<List<TaskDto>>();
        tasks.Should().NotBeNull();
    }
}
```

---

## Step 517: Test Coverage - End-to-End Flow

### ทดสอบ Complete Order Flow

```csharp
// Tests/Integration/Flows/TaskLifecycleFlowTests.cs
public class TaskLifecycleFlowTests : IClassFixture<TaskDbContextFixture>
{
    private readonly TaskDbContextFixture _fixture;

    public TaskLifecycleFlowTests(TaskDbContextFixture fixture)
    {
        _fixture = fixture;
        _fixture.ResetDatabase();
    }

    [Fact]
    public async Task CompleteFlow_CreateThenCompleteThenVerify_AllStatesCorrect()
    {
        // === STEP 1: Create Task ===
        var context1 = _fixture.CreateContext();
        var createHandler = new CreateTaskCommandHandler(
            new TaskRepository(context1),
            new UnitOfWork(context1),
            new NullLogger<CreateTaskCommandHandler>());

        var createCommand = new CreateTaskCommand(
            "Complete the project",
            "Finish all pending items",
            DateTime.UtcNow.Date.AddDays(3),
            Priority.High);

        var taskId = await createHandler.Handle(createCommand, CancellationToken.None);
        taskId.Should().NotBe(Guid.Empty);

        // === STEP 2: Verify Task was Created ===
        var context2 = _fixture.CreateContext();
        var getHandler = new GetTaskByIdQueryHandler(
            new TaskRepository(context2),
            new Mapper());

        var getQuery = new GetTaskByIdQuery(taskId);
        var taskDto = await getHandler.Handle(getQuery, CancellationToken.None);

        taskDto.Should().NotBeNull();
        taskDto!.Title.Should().Be("Complete the project");
        taskDto.Status.Should().Be(TaskStatus.Pending);

        // === STEP 3: Complete Task ===
        var context3 = _fixture.CreateContext();
        var completeHandler = new CompleteTaskCommandHandler(
            new TaskRepository(context3),
            new UnitOfWork(context3));

        var completeCommand = new CompleteTaskCommand(taskId);
        await completeHandler.Handle(completeCommand, CancellationToken.None);

        // === STEP 4: Verify Task is Completed ===
        var context4 = _fixture.CreateContext();
        var verifyTask = await context4.Tasks.FindAsync(taskId);

        verifyTask.Should().NotBeNull();
        verifyTask!.Status.Should().Be(TaskStatus.Completed);
        verifyTask.CompletedAt.Should().NotBeNull();
        verifyTask.CompletedAt!.Value.Should().BeCloseTo(DateTime.UtcNow, TimeSpan.FromSeconds(10));
    }
}
```

---

## Step 518: Builder Pattern สำหรับ Test Data

### ปัญหาของการสร้าง Test Data แบบเดิม

```csharp
// ไม่ดี - ต้อง setup ซ้ำๆ ทุก test
[Fact]
public void Test1()
{
    var task = new TaskItem
    {
        Id = Guid.NewGuid(),
        Title = "Task",
        Status = TaskStatus.Pending,
        // ... ต้องกำหนดทุก property
    };
}
```

### TaskItem Builder Pattern

```csharp
// Tests/Helpers/TaskItemBuilder.cs
public class TaskItemBuilder
{
    private string _title = "Default Task Title";
    private string? _description = null;
    private DateTime? _dueDate = null;
    private Priority _priority = Priority.Medium;
    private TaskStatus _status = TaskStatus.Pending;
    private bool _isCompleted = false;

    // Fluent methods
    public TaskItemBuilder WithTitle(string title)
    {
        _title = title;
        return this;
    }

    public TaskItemBuilder WithDescription(string description)
    {
        _description = description;
        return this;
    }

    public TaskItemBuilder WithDueDate(DateTime dueDate)
    {
        _dueDate = dueDate;
        return this;
    }

    public TaskItemBuilder WithPriority(Priority priority)
    {
        _priority = priority;
        return this;
    }

    public TaskItemBuilder AsHighPriority()
    {
        _priority = Priority.High;
        return this;
    }

    public TaskItemBuilder AsLowPriority()
    {
        _priority = Priority.Low;
        return this;
    }

    public TaskItemBuilder AsCompleted()
    {
        _isCompleted = true;
        return this;
    }

    public TaskItemBuilder DueInDays(int days)
    {
        _dueDate = DateTime.UtcNow.Date.AddDays(days);
        return this;
    }

    public TaskItem Build()
    {
        var task = TaskItem.Create(_title, _description, _dueDate, _priority);
        if (_isCompleted) task.Complete();
        return task;
    }

    public List<TaskItem> BuildMany(int count)
    {
        return Enumerable.Range(1, count)
                         .Select(i => WithTitle($"{_title} {i}").Build())
                         .ToList();
    }

    // Static factory methods สำหรับกรณีที่พบบ่อย
    public static TaskItemBuilder ASimpleTask() => new TaskItemBuilder();
    public static TaskItemBuilder AHighPriorityTask() => new TaskItemBuilder().AsHighPriority();
    public static TaskItemBuilder AnOverdueTask() => new TaskItemBuilder().DueInDays(-1);
    public static TaskItemBuilder ACompletedTask() => new TaskItemBuilder().AsCompleted();
}
```

### ObjectMother Pattern

```csharp
// Tests/Helpers/TaskItemMother.cs
public static class TaskItemMother
{
    // สร้าง task สำหรับ scenarios ที่พบบ่อย
    public static TaskItem SimpleTask()
        => TaskItem.Create("Simple Test Task");

    public static TaskItem HighPriorityTask()
        => TaskItem.Create("Urgent Task", priority: Priority.High);

    public static TaskItem CompletedTask()
    {
        var task = TaskItem.Create("Completed Task");
        task.Complete();
        return task;
    }

    public static TaskItem TaskWithAllFields()
        => TaskItem.Create(
            "Full Task",
            "Detailed description",
            DateTime.UtcNow.Date.AddDays(7),
            Priority.High);

    public static List<TaskItem> MultipleTasks(int count = 5)
        => Enumerable.Range(1, count)
                     .Select(i => TaskItem.Create($"Task {i}"))
                     .ToList();

    public static IEnumerable<TaskItem> MixedStatusTasks()
    {
        var pending = TaskItem.Create("Pending Task");
        var completed = TaskItem.Create("Completed Task");
        completed.Complete();
        var cancelled = TaskItem.Create("Cancelled Task");
        cancelled.Cancel("No longer needed");
        return new[] { pending, completed, cancelled };
    }
}
```

### ใช้ Builder ใน Tests

```csharp
public class TaskItemBuilderUsageTests
{
    [Fact]
    public void Builder_CreatesTaskWithCorrectProperties()
    {
        // ใช้ Builder - อ่านง่าย ชัดเจน
        var task = new TaskItemBuilder()
            .WithTitle("My Important Task")
            .AsHighPriority()
            .DueInDays(7)
            .WithDescription("This needs to be done")
            .Build();

        task.Title.Should().Be("My Important Task");
        task.Priority.Should().Be(Priority.High);
        task.DueDate.Should().NotBeNull();
    }

    [Fact]
    public void Builder_CreatesMultipleTasks()
    {
        var tasks = new TaskItemBuilder()
            .WithTitle("Batch Task")
            .AsHighPriority()
            .BuildMany(5);

        tasks.Should().HaveCount(5);
        tasks.Should().AllSatisfy(t =>
        {
            t.Priority.Should().Be(Priority.High);
            t.Title.Should().StartWith("Batch Task");
        });
    }

    [Fact]
    public void ObjectMother_ProvidesCommonScenarios()
    {
        // ใช้ ObjectMother สำหรับ scenarios ที่พบบ่อย
        var tasks = TaskItemMother.MixedStatusTasks().ToList();

        tasks.Should().HaveCount(3);
        tasks.Should().ContainSingle(t => t.Status == TaskStatus.Pending);
        tasks.Should().ContainSingle(t => t.Status == TaskStatus.Completed);
        tasks.Should().ContainSingle(t => t.Status == TaskStatus.Cancelled);
    }
}
```

### Bogus Library สำหรับ Fake Data

```csharp
// Tests/Helpers/FakeDataGenerator.cs
using Bogus;

public class TaskItemFaker : Faker<TaskItem>
{
    public TaskItemFaker()
    {
        // Bogus ช่วยสร้าง random data ที่ realistic
        // Note: ต้องใช้กับ class ที่มี public setters หรือผ่าน factory method
    }

    public static TaskItem GenerateFakeTask()
    {
        var faker = new Faker("th"); // ใช้ภาษาไทย
        var title = faker.Commerce.ProductName().Truncate(100);
        var description = faker.Lorem.Sentence();
        var daysFromNow = faker.Random.Int(1, 30);

        return TaskItem.Create(title, description, DateTime.UtcNow.AddDays(daysFromNow));
    }

    public static List<TaskItem> GenerateFakeTasks(int count)
    {
        return Enumerable.Range(1, count)
                         .Select(_ => GenerateFakeTask())
                         .ToList();
    }
}
```

---

## Step 519: Parameterized Tests ด้วย MemberData และ ClassData

### [InlineData] - ข้อมูลเดียว

```csharp
[Theory]
[InlineData("", "Title cannot be empty")]
[InlineData("ab", "Title must be at least 3 characters")]
[InlineData(null, "Title cannot be empty")]
public void Create_InvalidTitles_ThrowsCorrectMessage(string? title, string expectedMessage)
{
    Action act = () => TaskItem.Create(title!);

    act.Should().Throw<DomainException>()
       .WithMessage(expectedMessage);
}
```

### [MemberData] - ข้อมูลจาก Static Method/Property

```csharp
public class ParameterizedTests
{
    // Static property ที่ return IEnumerable<object[]>
    public static IEnumerable<object[]> ValidTitleData =>
        new List<object[]>
        {
            new object[] { "abc", 3 },           // minimum length
            new object[] { "Hello World", 11 },   // normal
            new object[] { new string('A', 200), 200 } // maximum length
        };

    [Theory]
    [MemberData(nameof(ValidTitleData))]
    public void Create_ValidTitles_CreatesTaskWithCorrectLength(string title, int expectedLength)
    {
        var task = TaskItem.Create(title);

        task.Title.Should().HaveLength(expectedLength);
    }

    // ส่ง data จาก static method
    public static IEnumerable<object[]> GetTaskStatusTransitions()
    {
        yield return new object[] { TaskStatus.Pending, "Complete", TaskStatus.Completed };
        yield return new object[] { TaskStatus.Pending, "Cancel", TaskStatus.Cancelled };
    }

    [Theory]
    [MemberData(nameof(GetTaskStatusTransitions))]
    public void TaskStatusTransition_ValidTransition_ChangesStatus(
        TaskStatus initialStatus,
        string action,
        TaskStatus expectedStatus)
    {
        var task = TaskItem.Create("Test Task");
        // initialStatus จะเป็น Pending อยู่แล้ว

        if (action == "Complete") task.Complete();
        else if (action == "Cancel") task.Cancel("reason");

        task.Status.Should().Be(expectedStatus);
    }
}
```

### [ClassData] - ข้อมูลจาก Class แยกต่างหาก

```csharp
// Tests/Data/InvalidTaskTitleTestData.cs
public class InvalidTaskTitleTestData : IEnumerable<object[]>
{
    public IEnumerator<object[]> GetEnumerator()
    {
        yield return new object[] { "", "Title cannot be empty" };
        yield return new object[] { "  ", "Title cannot be empty" };
        yield return new object[] { "ab", "Title must be at least 3 characters" };
        yield return new object[] { new string('X', 201), "Title cannot exceed 200 characters" };
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

// Tests/Unit/Domain/TaskItemClassDataTests.cs
public class TaskItemClassDataTests
{
    [Theory]
    [ClassData(typeof(InvalidTaskTitleTestData))]
    public void Create_InvalidTitle_ThrowsDomainException(string title, string expectedMessage)
    {
        Action act = () => TaskItem.Create(title);

        act.Should().Throw<DomainException>()
           .WithMessage(expectedMessage);
    }
}
```

### Strongly-Typed Test Data ด้วย TheoryData

```csharp
// TheoryData<T> - type-safe กว่า object[]
public class PriorityTestData : TheoryData<Priority, string, int>
{
    public PriorityTestData()
    {
        Add(Priority.Critical, "Critical", 1);
        Add(Priority.High, "High", 2);
        Add(Priority.Medium, "Medium", 3);
        Add(Priority.Low, "Low", 4);
    }
}

public class PriorityTests
{
    [Theory]
    [ClassData(typeof(PriorityTestData))]
    public void Priority_HasCorrectDisplayNameAndSortOrder(
        Priority priority,
        string expectedName,
        int expectedOrder)
    {
        priority.GetDisplayName().Should().Be(expectedName);
        priority.GetSortOrder().Should().Be(expectedOrder);
    }
}
```

### Cross Product Test Data

```csharp
public class CrossProductTests
{
    public static IEnumerable<object[]> AllPriorityAndStatusCombinations()
    {
        var priorities = Enum.GetValues<Priority>();
        var statuses = Enum.GetValues<TaskStatus>();

        return from priority in priorities
               from status in statuses
               select new object[] { priority, status };
    }

    [Theory]
    [MemberData(nameof(AllPriorityAndStatusCombinations))]
    public void TaskProperties_AllCombinations_AreValidEnumValues(
        Priority priority,
        TaskStatus status)
    {
        // ตรวจสอบว่า enum values ถูกต้องทุก combination
        Enum.IsDefined(priority).Should().BeTrue();
        Enum.IsDefined(status).Should().BeTrue();
    }
}
```

---

## Step 520: สรุปและ Best Practices

### Test Pyramid

```
        /\
       /  \
      / E2E\          <- จำนวนน้อย, ช้า, ค่าใช้จ่ายสูง
     /------\
    /  Integ \        <- ปานกลาง
   /----------\
  /  Unit Tests\      <- มากที่สุด, เร็ว, ราคาถูก
 /--------------\
```

### ตัวอย่าง Test Coverage Report

```bash
# ติดตั้ง coverage tool
dotnet add package coverlet.collector
dotnet add package ReportGenerator

# รัน tests พร้อม coverage
dotnet test --collect:"XPlat Code Coverage"

# สร้าง HTML report
reportgenerator -reports:"**/coverage.cobertura.xml" -targetdir:"coverage-report" -reporttypes:Html
```

### Directory Structure ที่แนะนำ

```
TaskManager.Tests/
├── Unit/
│   ├── Domain/
│   │   ├── TaskItemTests.cs
│   │   └── OrderTests.cs
│   ├── Application/
│   │   ├── CreateTaskCommandHandlerTests.cs
│   │   └── GetTaskListQueryHandlerTests.cs
│   └── Services/
│       └── TaskServiceTests.cs
├── Integration/
│   ├── Infrastructure/
│   │   ├── TaskRepositoryTests.cs
│   │   └── Fixtures/
│   │       └── TaskDbContextFixture.cs
│   ├── Api/
│   │   └── TasksApiIntegrationTests.cs
│   └── Flows/
│       └── TaskLifecycleFlowTests.cs
└── Helpers/
    ├── TaskItemBuilder.cs
    ├── TaskItemMother.cs
    ├── FakeDataGenerator.cs
    └── Data/
        └── InvalidTaskTitleTestData.cs
```

### Best Practices สรุป

```csharp
// 1. ARRANGE-ACT-ASSERT pattern ทุก test
[Fact]
public void MethodName_Scenario_Expected()
{
    // Arrange
    var sut = new SystemUnderTest();  // sut = System Under Test

    // Act
    var result = sut.DoSomething();

    // Assert
    result.Should().Be(expected);
}

// 2. One Assert concept per test (ไม่ใช่ one assert statement)
[Fact]
public void Complete_ValidTask_UpdatesBothStatusAndTimestamp()
{
    var task = TaskItem.Create("Test");
    task.Complete();

    // หลาย assertions แต่เป็น concept เดียวกัน
    task.Status.Should().Be(TaskStatus.Completed);
    task.CompletedAt.Should().NotBeNull();
    task.CompletedAt.Should().BeCloseTo(DateTime.UtcNow, TimeSpan.FromSeconds(5));
}

// 3. Test ต้องเป็น Independent
// แต่ละ test ต้องไม่พึ่งพา state จาก test อื่น

// 4. Test ต้อง Deterministic
// ผลลัพธ์ต้องเหมือนกันทุกครั้งที่รัน
// หลีกเลี่ยง DateTime.Now โดยตรง ใช้ IDateTimeProvider แทน

// 5. Test ต้อง Fast
// Unit tests ควรรันได้ภายใน milliseconds
// Integration tests อาจใช้เวลามากกว่า แต่ควรน้อยกว่า 1 วินาที

// 6. Test ต้อง Readable
// คนอื่นอ่านแล้วต้องเข้าใจ behavior ที่ทดสอบทันที
```

---

## สรุปตาราง

| Step | หัวข้อ | เครื่องมือ | สิ่งที่เรียนรู้ |
|------|--------|------------|----------------|
| 511 | xUnit Basics | xUnit | `[Fact]`, `[Theory]`, Assert methods, AAA pattern |
| 512 | Domain Entity Tests | xUnit + FluentAssertions | Test business rules, exception handling |
| 513 | Mocking | Moq | Mock, Setup, Verify, It.IsAny, Callbacks |
| 514 | Application Layer Tests | Moq + xUnit | Test command/query handlers ด้วย mocked dependencies |
| 515 | FluentAssertions | FluentAssertions | Readable assertions, chaining, collection tests |
| 516 | Integration Tests | EF Core InMemory | Real DbContext testing, IClassFixture |
| 517 | E2E Flow | EF Core InMemory | Complete lifecycle testing |
| 518 | Test Data Builders | Bogus + Builder Pattern | TestDataBuilder, ObjectMother |
| 519 | Parameterized Tests | xUnit | `[MemberData]`, `[ClassData]`, `TheoryData<T>` |
| 520 | Best Practices | All | Test pyramid, coverage, structure |

---

## Packages ที่ใช้ในบทนี้

| Package | Version | วัตถุประสงค์ |
|---------|---------|-------------|
| `xunit` | 2.9.0 | Test framework หลัก |
| `xunit.runner.visualstudio` | 2.8.2 | รัน test ใน Visual Studio / VS Code |
| `Moq` | 4.20.72 | Mock dependencies |
| `FluentAssertions` | 6.12.0 | Readable assertions |
| `Microsoft.EntityFrameworkCore.InMemory` | 8.0.10 | InMemory database สำหรับ integration tests |
| `Bogus` | 35.5.1 | สร้าง fake data ที่ realistic |
| `coverlet.collector` | 6.0.2 | Code coverage collection |
| `Microsoft.AspNetCore.Mvc.Testing` | 8.0.10 | HTTP integration tests |

---

## การนำทาง

| ส่วน | ลิงก์ |
|------|-------|
| ก่อนหน้า | [Part 51: Clean Architecture](./part51-clean-architecture.md) |
| ถัดไป | [Part 53: Performance Optimization](./part53-performance.md) |

---

*หลักสูตร C# Web Programming Enterprise - ส่วนที่ 52*
*Unit Testing: xUnit, Moq, FluentAssertions, EF Core InMemory*
