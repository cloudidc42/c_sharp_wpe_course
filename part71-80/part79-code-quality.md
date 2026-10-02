# Part 79: Code Quality & Static Analysis (Steps 781-790)

บทนี้ครอบคลุมเครื่องมือและแนวทางปฏิบัติในการรักษาคุณภาพโค้ด C# ตั้งแต่การวัดค่า metrics ไปจนถึงการตรวจสอบโค้ดด้วย analyzers และการจัดการ technical debt

---

## Step 781: Code Quality Metrics — Cyclomatic Complexity, Coupling, Cohesion

### แนวคิดหลัก

คุณภาพของโค้ดวัดได้หลายมิติ โดยค่า metrics สำคัญที่ใช้บ่อยได้แก่:

- **Cyclomatic Complexity** — ความซับซ้อนของ control flow ยิ่งสูงยิ่งทดสอบยาก
- **Coupling** — การพึ่งพากันระหว่าง class หรือ module ยิ่งน้อยยิ่งดี
- **Cohesion** — ความสัมพันธ์ระหว่างสมาชิกใน class เดียวกัน ยิ่งสูงยิ่งดี
- **Lines of Code (LOC)** — จำนวนบรรทัด
- **Depth of Inheritance Tree (DIT)** — ความลึกของ inheritance

### Cyclomatic Complexity

ค่า Cyclomatic Complexity คำนวณจากจำนวน independent paths ใน control flow graph สูตรคือ `M = E - N + 2P` แต่ในทางปฏิบัติให้นับจำนวน branch (`if`, `else`, `for`, `while`, `case`, `&&`, `||`) บวก 1

```csharp
// ❌ Cyclomatic Complexity สูง (CC = 10+) — ยากต่อการทดสอบ
public string ProcessOrder(Order order)
{
    if (order == null)
        return "Invalid";
    
    if (order.Items == null || !order.Items.Any())
        return "Empty order";
    
    if (order.Customer == null)
        return "No customer";
    
    if (order.Customer.IsBlocked)
        return "Customer blocked";
    
    decimal total = 0;
    foreach (var item in order.Items)
    {
        if (item.Quantity <= 0)
            continue;
        
        if (item.IsDiscounted)
            total += item.Price * item.Quantity * 0.9m;
        else if (item.IsPremium)
            total += item.Price * item.Quantity * 1.1m;
        else
            total += item.Price * item.Quantity;
    }
    
    if (total > 10000)
        return $"Large order: {total}";
    else if (total > 5000)
        return $"Medium order: {total}";
    else
        return $"Small order: {total}";
}

// ✅ Cyclomatic Complexity ต่ำ — แยก concern ออกจากกัน
public string ProcessOrder(Order order)
{
    var validation = ValidateOrder(order);
    if (!validation.IsValid)
        return validation.ErrorMessage;
    
    var total = CalculateOrderTotal(order.Items);
    return ClassifyOrder(total);
}

private ValidationResult ValidateOrder(Order order)
{
    if (order == null) return ValidationResult.Fail("Invalid");
    if (order.Items == null || !order.Items.Any()) return ValidationResult.Fail("Empty order");
    if (order.Customer == null) return ValidationResult.Fail("No customer");
    if (order.Customer.IsBlocked) return ValidationResult.Fail("Customer blocked");
    return ValidationResult.Ok();
}

private decimal CalculateOrderTotal(IEnumerable<OrderItem> items)
    => items
        .Where(i => i.Quantity > 0)
        .Sum(i => i.Price * i.Quantity * GetPriceMultiplier(i));

private decimal GetPriceMultiplier(OrderItem item) =>
    item.IsDiscounted ? 0.9m :
    item.IsPremium ? 1.1m : 1.0m;

private string ClassifyOrder(decimal total) =>
    total > 10000 ? $"Large order: {total}" :
    total > 5000  ? $"Medium order: {total}" :
                    $"Small order: {total}";
```

### ตัวอย่างการวัด Coupling และ Cohesion

```csharp
// ❌ High Coupling, Low Cohesion
public class UserManager
{
    private readonly SqlConnection _dbConnection;        // infrastructure concern
    private readonly SmtpClient _emailClient;            // infrastructure concern
    private readonly ILogger _logger;
    private readonly PaymentProcessor _paymentProcessor; // unrelated concern
    
    public void CreateUser(string email, string password) { /* ... */ }
    public void ProcessPayment(decimal amount) { /* ... */ }  // ไม่ควรอยู่ที่นี่
    public void SendNewsletter(string content) { /* ... */ }  // ไม่ควรอยู่ที่นี่
    public void GenerateReport() { /* ... */ }                 // ไม่ควรอยู่ที่นี่
}

// ✅ Low Coupling, High Cohesion
public class UserService
{
    private readonly IUserRepository _userRepository;
    private readonly IPasswordHasher _passwordHasher;
    private readonly IEventPublisher _eventPublisher;
    
    public UserService(
        IUserRepository userRepository,
        IPasswordHasher passwordHasher,
        IEventPublisher eventPublisher)
    {
        _userRepository = userRepository;
        _passwordHasher = passwordHasher;
        _eventPublisher = eventPublisher;
    }
    
    public async Task<User> CreateUserAsync(string email, string password)
    {
        var hashedPassword = _passwordHasher.Hash(password);
        var user = new User(email, hashedPassword);
        await _userRepository.AddAsync(user);
        await _eventPublisher.PublishAsync(new UserCreatedEvent(user.Id, email));
        return user;
    }
    
    public async Task<bool> ValidateCredentialsAsync(string email, string password)
    {
        var user = await _userRepository.FindByEmailAsync(email);
        return user != null && _passwordHasher.Verify(password, user.PasswordHash);
    }
}
```

### คำสั่งวัด metrics ด้วย dotnet

```bash
# ติดตั้ง tool สำหรับวัด code metrics
dotnet tool install --global dotnet-code-metrics

# หรือใช้ Microsoft Code Metrics ผ่าน Visual Studio
# Analyze → Calculate Code Metrics → For Solution

# ใช้ CSharpGuidelinesAnalyzer ผ่าน NuGet
dotnet add package CSharpGuidelinesAnalyzer

# ดู complexity ด้วย VisualStudio Metrics PowerTool
msbuild /t:Metrics MyProject.csproj
```

---

## Step 782: Roslyn Analyzers — Microsoft.CodeAnalysis.NetAnalyzers

### ภาพรวม

Roslyn analyzers ทำงานระหว่าง compile time เพื่อตรวจจับปัญหาใน code โดยไม่ต้องรัน .NET SDK 5+ มาพร้อมกับ analyzers หลักในตัว และสามารถเพิ่ม package เพิ่มเติมได้

### การตั้งค่า Directory.Build.props

สร้างไฟล์ `Directory.Build.props` ที่ root ของ solution เพื่อให้ทุก project ใช้ analyzer ร่วมกัน:

```xml
<!-- Directory.Build.props -->
<Project>
  <PropertyGroup>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <WarningsAsErrors />
    <AnalysisLevel>latest</AnalysisLevel>
    <AnalysisMode>AllEnabledByDefault</AnalysisMode>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    <EnableNETAnalyzers>true</EnableNETAnalyzers>
    <!-- กำหนด version ให้ reproducible builds -->
    <LangVersion>latest</LangVersion>
  </PropertyGroup>

  <ItemGroup>
    <!-- Microsoft built-in analyzers (included in SDK 5+) -->
    <PackageReference Include="Microsoft.CodeAnalysis.NetAnalyzers" Version="8.0.0">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
    </PackageReference>

    <!-- Roslynator — analyzers เพิ่มเติม 500+ rules -->
    <PackageReference Include="Roslynator.Analyzers" Version="4.12.0">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
    </PackageReference>
  </ItemGroup>
</Project>
```

### การตั้งค่า global.json

```json
// global.json — ที่ root ของ solution
{
  "sdk": {
    "version": "8.0.400",
    "rollForward": "latestPatch"
  },
  "msbuild-sdks": {
    "Microsoft.Build.NoTargets": "3.7.0"
  }
}
```

### การ suppress analyzer warnings อย่างถูกต้อง

```csharp
// วิธีที่ 1: suppress ด้วย pragma (เฉพาะบรรทัด)
#pragma warning disable CA1062 // Validate arguments of public methods
public void ProcessData(string input)
{
    // หากมีการ validate ก่อนหน้าแล้ว
    DoWork(input);
}
#pragma warning restore CA1062

// วิธีที่ 2: suppress ด้วย attribute (ทั้ง method)
[System.Diagnostics.CodeAnalysis.SuppressMessage(
    "Reliability",
    "CA2007:Consider calling ConfigureAwait on the awaited task",
    Justification = "Console application, no SynchronizationContext")]
public async Task RunAsync()
{
    await Task.Delay(1000);
}

// วิธีที่ 3: suppress ผ่าน .editorconfig (แนะนำสำหรับ project-wide)
// dotnet_diagnostic.CA1062.severity = none
```

### ตัวอย่าง rules สำคัญ

```csharp
// CA1031: Do not catch general exception types
// ❌
try { DoWork(); }
catch (Exception ex) { Log(ex); } // CA1031 warning

// ✅
try { DoWork(); }
catch (InvalidOperationException ex) { Log(ex); }
catch (HttpRequestException ex) { Log(ex); }

// CA2016: Forward CancellationToken
// ❌
public async Task ProcessAsync(CancellationToken ct)
{
    await _service.DoWorkAsync(); // CA2016: ควรส่ง ct ไปด้วย
}

// ✅
public async Task ProcessAsync(CancellationToken ct)
{
    await _service.DoWorkAsync(ct);
}

// CA1852: Seal internal types
// ❌
internal class InternalHelper { } // CA1852: ควรเป็น sealed

// ✅
internal sealed class InternalHelper { }

// CA1805: Do not initialize unnecessarily
// ❌
private bool _isReady = false; // CA1805: default value ซ้ำซ้อน

// ✅
private bool _isReady;
```

---

## Step 783: StyleCop และ .editorconfig

### .editorconfig ครบถ้วนสำหรับ C#

```ini
# .editorconfig — วางที่ root ของ solution
root = true

# ตั้งค่าทั่วไปสำหรับทุกไฟล์
[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 4
insert_final_newline = true
trim_trailing_whitespace = true

# C# files
[*.cs]
indent_size = 4
tab_width = 4

#### Core EditorConfig Options ####

# ตั้งค่า C# style
csharp_style_var_for_built_in_types = false:suggestion
csharp_style_var_when_type_is_apparent = true:suggestion
csharp_style_var_elsewhere = false:suggestion

# Expression-bodied members
csharp_style_expression_bodied_methods = false:silent
csharp_style_expression_bodied_constructors = false:silent
csharp_style_expression_bodied_operators = false:silent
csharp_style_expression_bodied_properties = true:suggestion
csharp_style_expression_bodied_indexers = true:suggestion
csharp_style_expression_bodied_accessors = true:suggestion
csharp_style_expression_bodied_lambdas = true:suggestion
csharp_style_expression_bodied_local_functions = false:silent

# Pattern matching
csharp_style_pattern_matching_over_is_with_cast_check = true:suggestion
csharp_style_pattern_matching_over_as_with_null_check = true:suggestion
csharp_style_prefer_switch_expression = true:suggestion
csharp_style_prefer_pattern_matching = true:silent
csharp_style_prefer_not_pattern = true:suggestion

# Null-checking preferences
csharp_style_prefer_null_check_over_type_check = true:suggestion
csharp_style_conditional_delegate_call = true:suggestion

# Modifier preferences
csharp_prefer_static_local_function = true:suggestion
csharp_preferred_modifier_order = public,private,protected,internal,static,extern,new,virtual,abstract,sealed,override,readonly,unsafe,volatile,async:suggestion

# Code block preferences
csharp_prefer_braces = true:silent
csharp_prefer_simple_using_statement = true:suggestion
csharp_style_namespace_declarations = file_scoped:suggestion

# Expression level preferences
csharp_prefer_simple_default_expression = true:suggestion
csharp_style_deconstructed_variable_declaration = true:suggestion
csharp_style_inlined_variable_declaration = true:suggestion
csharp_style_prefer_index_operator = true:suggestion
csharp_style_prefer_range_operator = true:suggestion
csharp_style_throw_expression = true:suggestion
csharp_style_unused_value_assignment_preference = discard_variable:suggestion
csharp_style_unused_value_expression_statement_preference = discard_variable:silent

#### Formatting Rules ####

# Newlines
csharp_new_line_before_open_brace = all
csharp_new_line_before_else = true
csharp_new_line_before_catch = true
csharp_new_line_before_finally = true
csharp_new_line_before_members_in_object_initializers = true
csharp_new_line_before_members_in_anonymous_types = true
csharp_new_line_between_query_expression_clauses = true

# Indentation
csharp_indent_case_contents = true
csharp_indent_switch_labels = true
csharp_indent_labels = flush_left
csharp_indent_block_contents = true
csharp_indent_braces = false
csharp_indent_case_contents_when_block = true

# Spacing
csharp_space_after_cast = false
csharp_space_after_keywords_in_control_flow_statements = true
csharp_space_between_parentheses = false
csharp_space_before_colon_in_inheritance_clause = true
csharp_space_after_colon_in_inheritance_clause = true
csharp_space_around_binary_operators = before_and_after
csharp_space_between_method_declaration_parameter_list_parentheses = false
csharp_space_between_method_declaration_empty_parameter_list_parentheses = false
csharp_space_between_method_declaration_name_and_open_parenthesis = false
csharp_space_between_method_call_parameter_list_parentheses = false
csharp_space_between_method_call_empty_parameter_list_parentheses = false
csharp_space_between_method_call_name_and_opening_parenthesis = false
csharp_space_after_comma = true
csharp_space_after_dot = false
csharp_space_after_semicolon_in_for_statement = true
csharp_space_around_declaration_statements = false
csharp_space_before_open_square_brackets = false
csharp_space_between_empty_square_brackets = false
csharp_space_between_square_brackets = false

# Wrapping
csharp_preserve_single_line_statements = true
csharp_preserve_single_line_blocks = true

#### .NET Naming Conventions ####

# Interfaces ขึ้นต้นด้วย I
dotnet_naming_rule.interface_should_be_begins_with_i.severity = error
dotnet_naming_rule.interface_should_be_begins_with_i.symbols = interface
dotnet_naming_rule.interface_should_be_begins_with_i.style = begins_with_i
dotnet_naming_symbols.interface.applicable_kinds = interface
dotnet_naming_symbols.interface.applicable_accessibilities = public, internal, private, protected, protected_internal, private_protected
dotnet_naming_style.begins_with_i.required_prefix = I
dotnet_naming_style.begins_with_i.capitalization = pascal_case

# Type parameters ขึ้นต้นด้วย T
dotnet_naming_rule.type_parameters_should_begin_with_t.severity = suggestion
dotnet_naming_rule.type_parameters_should_begin_with_t.symbols = type_parameter
dotnet_naming_rule.type_parameters_should_begin_with_t.style = begins_with_t
dotnet_naming_symbols.type_parameter.applicable_kinds = type_parameter
dotnet_naming_style.begins_with_t.required_prefix = T
dotnet_naming_style.begins_with_t.capitalization = pascal_case

# Private fields ขึ้นต้นด้วย _
dotnet_naming_rule.private_members_with_underscore.severity = suggestion
dotnet_naming_rule.private_members_with_underscore.symbols = private_fields
dotnet_naming_rule.private_members_with_underscore.style = prefix_underscore
dotnet_naming_symbols.private_fields.applicable_kinds = field
dotnet_naming_symbols.private_fields.applicable_accessibilities = private
dotnet_naming_style.prefix_underscore.required_prefix = _
dotnet_naming_style.prefix_underscore.capitalization = camel_case

# Constants ใช้ PascalCase
dotnet_naming_rule.constants_should_be_pascal_case.severity = suggestion
dotnet_naming_rule.constants_should_be_pascal_case.symbols = constants
dotnet_naming_rule.constants_should_be_pascal_case.style = pascal_case
dotnet_naming_symbols.constants.applicable_kinds = field, local
dotnet_naming_symbols.constants.required_modifiers = const

#### .NET Code Style Settings ####

dotnet_sort_system_directives_first = true
dotnet_separate_import_directive_groups = false

# this. preferences
dotnet_style_qualification_for_field = false:silent
dotnet_style_qualification_for_property = false:silent
dotnet_style_qualification_for_method = false:silent
dotnet_style_qualification_for_event = false:silent

# Language keywords vs BCL types
dotnet_style_predefined_type_for_locals_parameters_members = true:silent
dotnet_style_predefined_type_for_member_access = true:silent

# Parentheses preferences
dotnet_style_parentheses_in_arithmetic_binary_operators = always_for_clarity:silent
dotnet_style_parentheses_in_relational_binary_operators = always_for_clarity:silent
dotnet_style_parentheses_in_other_binary_operators = always_for_clarity:silent
dotnet_style_parentheses_in_other_operators = never_if_unnecessary:silent

# Modifier preferences
dotnet_style_require_accessibility_modifiers = for_non_interface_members:silent
dotnet_style_readonly_field = true:suggestion

# Expression-level preferences
dotnet_style_object_initializer = true:suggestion
dotnet_style_collection_initializer = true:suggestion
dotnet_style_explicit_tuple_names = true:suggestion
dotnet_style_null_propagation = true:suggestion
dotnet_style_coalesce_expression = true:suggestion
dotnet_style_prefer_is_null_check_over_reference_equality_method = true:suggestion
dotnet_style_prefer_inferred_tuple_names = true:suggestion
dotnet_style_prefer_inferred_anonymous_type_member_names = true:suggestion
dotnet_style_prefer_auto_properties = true:silent
dotnet_style_prefer_conditional_expression_over_assignment = true:silent
dotnet_style_prefer_conditional_expression_over_return = true:silent

# Diagnostic severities
dotnet_diagnostic.CA1062.severity = warning
dotnet_diagnostic.CA1031.severity = warning
dotnet_diagnostic.CA2007.severity = none      # ปิดสำหรับ console apps
dotnet_diagnostic.IDE0058.severity = none      # ปิด expression value unused

# JSON/YAML/XML/Markdown
[*.{json,yml,yaml}]
indent_size = 2

[*.xml]
indent_size = 2

[*.md]
trim_trailing_whitespace = false
```

### การใช้ dotnet format

```bash
# ตรวจสอบ formatting (ไม่แก้ไขไฟล์)
dotnet format --verify-no-changes

# แก้ไข formatting ทั้งหมด
dotnet format

# แก้เฉพาะ style issues
dotnet format style

# แก้เฉพาะ whitespace
dotnet format whitespace

# แก้เฉพาะ analyzer diagnostics
dotnet format analyzers

# ระบุ severity ขั้นต่ำ
dotnet format --severity warn

# exclude บาง path
dotnet format --exclude "**/*.g.cs" --exclude "**/Migrations/**"

# ดู report
dotnet format --report format-report.json
```

### StyleCop.Analyzers

```xml
<!-- เพิ่มใน .csproj หรือ Directory.Build.props -->
<ItemGroup>
  <PackageReference Include="StyleCop.Analyzers" Version="1.2.0-beta.556">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
  </PackageReference>
</ItemGroup>
```

```json
// stylecop.json — configuration file สำหรับ StyleCop
{
  "$schema": "https://raw.githubusercontent.com/DotNetAnalyzers/StyleCopAnalyzers/master/StyleCop.Analyzers/StyleCop.Analyzers/Settings/stylecop.schema.json",
  "settings": {
    "documentationRules": {
      "companyName": "MyCompany",
      "copyrightText": "Copyright (c) {year} {companyName}. All rights reserved.",
      "xmlHeader": false,
      "documentInternalElements": false,
      "documentPrivateElements": false,
      "documentPrivateFields": false
    },
    "orderingRules": {
      "usingDirectivesPlacement": "outsideNamespace",
      "systemUsingDirectivesFirst": true
    },
    "namingRules": {
      "tupleElementNameCasing": "camelCase"
    },
    "layoutRules": {
      "newlineAtEndOfFile": "require"
    }
  }
}
```

---

## Step 784: Custom Roslyn Analyzer

### ภาพรวมสถาปัตยกรรม

Custom analyzer ประกอบด้วยสองส่วนหลัก:
1. **DiagnosticAnalyzer** — ตรวจจับ code smell และรายงาน diagnostic
2. **CodeFixProvider** — เสนอการแก้ไขอัตโนมัติ

### สร้าง Analyzer Project

```bash
# สร้าง analyzer project
dotnet new classlib -n MyAnalyzer -f netstandard2.0
cd MyAnalyzer
dotnet add package Microsoft.CodeAnalysis.CSharp
dotnet add package Microsoft.CodeAnalysis.Analyzers

# สร้าง test project
dotnet new xunit -n MyAnalyzer.Tests
dotnet add MyAnalyzer.Tests/MyAnalyzer.Tests.csproj package Microsoft.CodeAnalysis.CSharp.Testing.XUnit
```

### DiagnosticAnalyzer — ตรวจจับ string.Format ที่ควรเป็น interpolation

```csharp
// MyAnalyzer/StringFormatAnalyzer.cs
using System.Collections.Immutable;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.CSharp.Syntax;
using Microsoft.CodeAnalysis.Diagnostics;

namespace MyAnalyzer;

[DiagnosticAnalyzer(LanguageNames.CSharp)]
public class StringFormatAnalyzer : DiagnosticAnalyzer
{
    public const string DiagnosticId = "MY001";
    
    private static readonly LocalizableString Title =
        new LocalizableResourceString(
            nameof(Resources.MY001Title),
            Resources.ResourceManager,
            typeof(Resources));
    
    private static readonly LocalizableString MessageFormat =
        new LocalizableResourceString(
            nameof(Resources.MY001MessageFormat),
            Resources.ResourceManager,
            typeof(Resources));
    
    private static readonly LocalizableString Description =
        new LocalizableResourceString(
            nameof(Resources.MY001Description),
            Resources.ResourceManager,
            typeof(Resources));
    
    // หรือใช้ string โดยตรงก็ได้ในตัวอย่างนี้
    private static readonly DiagnosticDescriptor Rule = new DiagnosticDescriptor(
        id: DiagnosticId,
        title: "Use string interpolation instead of string.Format",
        messageFormat: "Use string interpolation instead of 'string.Format'",
        category: "Style",
        defaultSeverity: DiagnosticSeverity.Info,
        isEnabledByDefault: true,
        description: "String interpolation is more readable than string.Format.");
    
    public override ImmutableArray<DiagnosticDescriptor> SupportedDiagnostics
        => ImmutableArray.Create(Rule);
    
    public override void Initialize(AnalysisContext context)
    {
        // ป้องกัน re-entrant calls และเปิด concurrent execution
        context.ConfigureGeneratedCodeAnalysis(GeneratedCodeAnalysisFlags.None);
        context.EnableConcurrentExecution();
        
        // ลงทะเบียน action สำหรับตรวจสอบ invocation expression
        context.RegisterSyntaxNodeAction(
            AnalyzeInvocation,
            SyntaxKind.InvocationExpression);
    }
    
    private static void AnalyzeInvocation(SyntaxNodeAnalysisContext context)
    {
        var invocation = (InvocationExpressionSyntax)context.Node;
        
        // ตรวจสอบว่าเป็น string.Format หรือไม่
        if (invocation.Expression is not MemberAccessExpressionSyntax memberAccess)
            return;
        
        if (memberAccess.Name.Identifier.Text != "Format")
            return;
        
        // ตรวจสอบ symbol ว่าเป็น string.Format จริงๆ
        var methodSymbol = context.SemanticModel.GetSymbolInfo(invocation).Symbol as IMethodSymbol;
        if (methodSymbol is null)
            return;
        
        if (methodSymbol.ContainingType.SpecialType != SpecialType.System_String)
            return;
        
        if (methodSymbol.Name != "Format")
            return;
        
        // ตรวจสอบว่า argument แรกเป็น string literal
        var args = invocation.ArgumentList.Arguments;
        if (args.Count < 2)
            return;
        
        if (args[0].Expression is not LiteralExpressionSyntax literal ||
            !literal.IsKind(SyntaxKind.StringLiteralExpression))
            return;
        
        // รายงาน diagnostic
        var diagnostic = Diagnostic.Create(Rule, invocation.GetLocation());
        context.ReportDiagnostic(diagnostic);
    }
}
```

### CodeFixProvider — แก้ไขอัตโนมัติ

```csharp
// MyAnalyzer/StringFormatCodeFixProvider.cs
using System.Collections.Immutable;
using System.Composition;
using System.Text;
using System.Text.RegularExpressions;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CodeActions;
using Microsoft.CodeAnalysis.CodeFixes;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.CSharp.Syntax;
using Microsoft.CodeAnalysis.Formatting;

namespace MyAnalyzer;

[ExportCodeFixProvider(LanguageNames.CSharp, Name = nameof(StringFormatCodeFixProvider)), Shared]
public class StringFormatCodeFixProvider : CodeFixProvider
{
    public override ImmutableArray<string> FixableDiagnosticIds
        => ImmutableArray.Create(StringFormatAnalyzer.DiagnosticId);
    
    // บอกว่า fix นี้สามารถใช้แก้ทั้ง solution ได้
    public override FixAllProvider GetFixAllProvider()
        => WellKnownFixAllProviders.BatchFixer;
    
    public override async Task RegisterCodeFixesAsync(CodeFixContext context)
    {
        var root = await context.Document
            .GetSyntaxRootAsync(context.CancellationToken)
            .ConfigureAwait(false);
        
        var diagnostic = context.Diagnostics[0];
        var diagnosticSpan = diagnostic.Location.SourceSpan;
        
        // หา node ที่มีปัญหา
        var node = root?.FindNode(diagnosticSpan) as InvocationExpressionSyntax;
        if (node is null) return;
        
        // ลงทะเบียน code fix
        context.RegisterCodeFix(
            CodeAction.Create(
                title: "Convert to string interpolation",
                createChangedDocument: ct => ConvertToInterpolationAsync(
                    context.Document, node, ct),
                equivalenceKey: "ConvertToInterpolation"),
            diagnostic);
    }
    
    private static async Task<Document> ConvertToInterpolationAsync(
        Document document,
        InvocationExpressionSyntax invocation,
        CancellationToken cancellationToken)
    {
        var args = invocation.ArgumentList.Arguments;
        var formatString = ((LiteralExpressionSyntax)args[0].Expression).Token.ValueText;
        
        // แปลง {0}, {1}, ... เป็น interpolation expressions
        var interpolatedParts = new List<InterpolatedStringContentSyntax>();
        var segments = Regex.Split(formatString, @"(\{[^}]+\})");
        
        foreach (var segment in segments)
        {
            var placeholderMatch = Regex.Match(segment, @"^\{(\d+)(?::([^}]+))?\}$");
            
            if (placeholderMatch.Success)
            {
                var index = int.Parse(placeholderMatch.Groups[1].Value);
                var format = placeholderMatch.Groups[2].Value;
                
                if (index + 1 >= args.Count) continue;
                
                var argExpression = args[index + 1].Expression;
                
                // สร้าง interpolation expression
                InterpolationSyntax interpolation;
                if (!string.IsNullOrEmpty(format))
                {
                    var formatClause = SyntaxFactory.InterpolationFormatClause(
                        SyntaxFactory.Token(SyntaxKind.ColonToken),
                        SyntaxFactory.Token(
                            SyntaxTriviaList.Empty,
                            SyntaxKind.InterpolatedStringTextToken,
                            format,
                            format,
                            SyntaxTriviaList.Empty));
                    interpolation = SyntaxFactory.Interpolation(argExpression, formatClause, null);
                }
                else
                {
                    interpolation = SyntaxFactory.Interpolation(argExpression);
                }
                
                interpolatedParts.Add(interpolation);
            }
            else if (!string.IsNullOrEmpty(segment))
            {
                // Escape วงเล็บปีกกาในส่วนที่เป็น literal text
                var escapedText = segment.Replace("{", "{{").Replace("}", "}}");
                var textToken = SyntaxFactory.Token(
                    SyntaxTriviaList.Empty,
                    SyntaxKind.InterpolatedStringTextToken,
                    escapedText,
                    escapedText,
                    SyntaxTriviaList.Empty);
                interpolatedParts.Add(SyntaxFactory.InterpolatedStringText(textToken));
            }
        }
        
        // สร้าง interpolated string
        var interpolatedString = SyntaxFactory.InterpolatedStringExpression(
            SyntaxFactory.Token(SyntaxKind.InterpolatedStringStartToken),
            SyntaxFactory.List(interpolatedParts),
            SyntaxFactory.Token(SyntaxKind.InterpolatedStringEndToken));
        
        // แทนที่ node เดิม
        var root = await document.GetSyntaxRootAsync(cancellationToken);
        var newRoot = root!.ReplaceNode(invocation, interpolatedString
            .WithAdditionalAnnotations(Formatter.Annotation));
        
        return document.WithSyntaxRoot(newRoot);
    }
}
```

### ทดสอบ Analyzer

```csharp
// MyAnalyzer.Tests/StringFormatAnalyzerTests.cs
using Microsoft.CodeAnalysis.CSharp.Testing;
using Microsoft.CodeAnalysis.Testing;
using Xunit;

namespace MyAnalyzer.Tests;

public class StringFormatAnalyzerTests
{
    [Fact]
    public async Task StringFormat_ShouldReportDiagnostic()
    {
        var testCode = """
            using System;
            
            class Program
            {
                static void Main()
                {
                    var name = "World";
                    var msg = {|MY001:string.Format("Hello, {0}!", name)|};
                }
            }
            """;
        
        await new CSharpAnalyzerTest<StringFormatAnalyzer, DefaultVerifier>
        {
            TestCode = testCode,
        }.RunAsync();
    }
    
    [Fact]
    public async Task InterpolatedString_ShouldNotReportDiagnostic()
    {
        var testCode = """
            class Program
            {
                static void Main()
                {
                    var name = "World";
                    var msg = $"Hello, {name}!";
                }
            }
            """;
        
        await new CSharpAnalyzerTest<StringFormatAnalyzer, DefaultVerifier>
        {
            TestCode = testCode,
            ExpectedDiagnostics = { /* ไม่มี diagnostic */ }
        }.RunAsync();
    }
    
    [Fact]
    public async Task CodeFix_ShouldConvertToInterpolation()
    {
        var testCode = """
            using System;
            
            class Program
            {
                static void Main()
                {
                    var name = "World";
                    var msg = {|MY001:string.Format("Hello, {0}!", name)|};
                }
            }
            """;
        
        var fixedCode = """
            using System;
            
            class Program
            {
                static void Main()
                {
                    var name = "World";
                    var msg = $"Hello, {name}!";
                }
            }
            """;
        
        await new CSharpCodeFixTest<StringFormatAnalyzer, StringFormatCodeFixProvider, DefaultVerifier>
        {
            TestCode = testCode,
            FixedCode = fixedCode,
        }.RunAsync();
    }
}
```

### การ package และใช้งาน analyzer

```xml
<!-- MyAnalyzer/MyAnalyzer.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <PackageId>MyCompany.Analyzers</PackageId>
    <Version>1.0.0</Version>
    
    <!-- บอกว่า DLL นี้คือ analyzer ไม่ใช่ runtime library -->
    <IncludeBuildOutput>false</IncludeBuildOutput>
    <GeneratePackageOnBuild>true</GeneratePackageOnBuild>
    <DevelopmentDependency>true</DevelopmentDependency>
  </PropertyGroup>
  
  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="4.8.0" PrivateAssets="all" />
    <PackageReference Include="Microsoft.CodeAnalysis.Analyzers" Version="3.3.4" PrivateAssets="all" />
  </ItemGroup>
  
  <!-- วาง DLL ในตำแหน่งที่ถูกต้องสำหรับ NuGet analyzer package -->
  <ItemGroup>
    <None Include="$(OutputPath)\$(AssemblyName).dll"
          Pack="true"
          PackagePath="analyzers/dotnet/cs"
          Visible="false" />
  </ItemGroup>
</Project>
```

---

## Step 785: SonarQube/SonarCloud Integration

### ภาพรวม SonarQube

SonarQube วิเคราะห์ code quality ในหลายมิติ:
- **Bugs** — code ที่มีแนวโน้มจะผิดพลาดที่ runtime
- **Vulnerabilities** — ช่องโหว่ด้านความปลอดภัย
- **Code Smells** — code ที่ maintainability ต่ำ
- **Security Hotspots** — code ที่ต้องตรวจสอบด้านความปลอดภัย
- **Duplications** — code ที่ซ้ำซ้อน
- **Coverage** — code coverage จาก tests

### GitHub Actions พร้อม SonarCloud

```yaml
# .github/workflows/sonarcloud.yml
name: SonarCloud Analysis

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

env:
  DOTNET_VERSION: '8.0.x'
  SONAR_PROJECT_KEY: 'myorg_myproject'
  SONAR_ORGANIZATION: 'myorg'

jobs:
  sonarcloud:
    name: SonarCloud
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # SonarCloud ต้องการ full history
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}
      
      - name: Setup Java (required for SonarScanner)
        uses: actions/setup-java@v4
        with:
          distribution: 'microsoft'
          java-version: '17'
      
      - name: Cache SonarCloud packages
        uses: actions/cache@v4
        with:
          path: ~/.sonar/cache
          key: ${{ runner.os }}-sonar
          restore-keys: ${{ runner.os }}-sonar
      
      - name: Cache NuGet packages
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
          restore-keys: ${{ runner.os }}-nuget-
      
      - name: Install SonarCloud scanner
        run: |
          dotnet tool install --global dotnet-sonarscanner
      
      - name: Install coverage tool
        run: |
          dotnet tool install --global dotnet-coverage
      
      - name: Restore dependencies
        run: dotnet restore
      
      - name: Begin SonarCloud scan
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: |
          dotnet sonarscanner begin \
            /k:"${{ env.SONAR_PROJECT_KEY }}" \
            /o:"${{ env.SONAR_ORGANIZATION }}" \
            /d:sonar.token="${{ secrets.SONAR_TOKEN }}" \
            /d:sonar.host.url="https://sonarcloud.io" \
            /d:sonar.cs.vscoveragexml.reportsPaths=coverage.xml \
            /d:sonar.cs.opencover.reportsPaths=coverage.opencover.xml \
            /d:sonar.exclusions="**/Migrations/**,**/*.Designer.cs" \
            /d:sonar.coverage.exclusions="**/*Tests*/**,**/Program.cs"
      
      - name: Build
        run: dotnet build --no-restore --configuration Release
      
      - name: Run tests with coverage
        run: |
          dotnet-coverage collect \
            "dotnet test --no-build --configuration Release" \
            -f xml -o coverage.xml
      
      - name: End SonarCloud scan
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: |
          dotnet sonarscanner end /d:sonar.token="${{ secrets.SONAR_TOKEN }}"
```

### sonar-project.properties

```properties
# sonar-project.properties — วางที่ root ของ project
sonar.projectKey=myorg_myproject
sonar.organization=myorg

# Project information
sonar.projectName=My C# Project
sonar.projectVersion=1.0

# Source encoding
sonar.sourceEncoding=UTF-8

# Source paths
sonar.sources=src
sonar.tests=tests

# Exclusions
sonar.exclusions=**/obj/**,**/bin/**,**/*.Designer.cs,**/Migrations/**

# Test exclusions from coverage
sonar.coverage.exclusions=**/*Tests.cs,**/*Spec.cs,**/Program.cs

# C# specific settings
sonar.cs.opencover.reportsPaths=**/coverage.opencover.xml

# Quality gate
sonar.qualitygate.wait=true
```

### Quality Gate Configuration

```json
// sonar-quality-gate.json — กำหนดเกณฑ์คุณภาพ
{
  "name": "My Quality Gate",
  "conditions": [
    {
      "metric": "new_reliability_rating",
      "op": "GT",
      "error": "1"
    },
    {
      "metric": "new_security_rating",
      "op": "GT",
      "error": "1"
    },
    {
      "metric": "new_maintainability_rating",
      "op": "GT",
      "error": "2"
    },
    {
      "metric": "new_coverage",
      "op": "LT",
      "error": "80"
    },
    {
      "metric": "new_duplicated_lines_density",
      "op": "GT",
      "error": "3"
    }
  ]
}
```

---

## Step 786: NuGet Package Security Scanning

### การตรวจสอบ vulnerabilities ด้วย dotnet

```bash
# ตรวจสอบ NuGet packages ที่มี known vulnerabilities
dotnet list package --vulnerable

# ตรวจสอบ packages ที่ outdated
dotnet list package --outdated

# ตรวจสอบทั้ง vulnerable และ deprecated
dotnet list package --vulnerable --include-transitive

# ตรวจสอบผ่าน dotnet audit (SDK 8+)
dotnet nuget audit
dotnet nuget audit --audit-level moderate
```

### GitHub Actions สำหรับ Security Scanning

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  push:
    branches: [ main ]
  schedule:
    - cron: '0 6 * * 1'  # ทุกวันจันทร์ 6am UTC

jobs:
  dependency-scan:
    name: Dependency Security Scan
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'
      
      - name: Restore dependencies
        run: dotnet restore
      
      - name: Check for vulnerable packages
        run: |
          dotnet list package --vulnerable --include-transitive 2>&1 | tee vuln-report.txt
          if grep -q "has the following vulnerable packages" vuln-report.txt; then
            echo "::error::Found vulnerable NuGet packages!"
            cat vuln-report.txt
            exit 1
          fi
      
      - name: Upload vulnerability report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: vulnerability-report
          path: vuln-report.txt
  
  codeql:
    name: CodeQL Analysis
    runs-on: ubuntu-latest
    permissions:
      actions: read
      contents: read
      security-events: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: csharp
          queries: security-and-quality
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'
      
      - name: Build
        run: dotnet build --configuration Release
      
      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
        with:
          category: "/language:csharp"
```

### การจัดการ Package Versions ด้วย Central Package Management

```xml
<!-- Directory.Packages.props — ที่ root ของ solution -->
<Project>
  <PropertyGroup>
    <!-- เปิดใช้ Central Package Management -->
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    <!-- ป้องกัน override version ใน project files -->
    <CentralPackageTransitivePinningEnabled>true</CentralPackageTransitivePinningEnabled>
  </PropertyGroup>

  <ItemGroup>
    <!-- กำหนด version กลางสำหรับทุก project -->
    <PackageVersion Include="Microsoft.EntityFrameworkCore" Version="8.0.7" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore.SqlServer" Version="8.0.7" />
    <PackageVersion Include="Newtonsoft.Json" Version="13.0.3" />
    <PackageVersion Include="Serilog" Version="3.1.1" />
    <PackageVersion Include="Serilog.Sinks.Console" Version="5.0.1" />
    <PackageVersion Include="xunit" Version="2.9.0" />
    <PackageVersion Include="xunit.runner.visualstudio" Version="2.8.2" />
    <PackageVersion Include="Moq" Version="4.20.72" />
    <PackageVersion Include="FluentAssertions" Version="6.12.0" />
  </ItemGroup>
</Project>
```

---

## Step 787: Code Coverage Goals and Tools

### การตั้งค่า Coverage ด้วย Coverlet

```xml
<!-- เพิ่มใน Test project -->
<ItemGroup>
  <PackageReference Include="coverlet.collector" Version="6.0.2">
    <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    <PrivateAssets>all</PrivateAssets>
  </PackageReference>
  <PackageReference Include="coverlet.msbuild" Version="6.0.2">
    <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    <PrivateAssets>all</PrivateAssets>
  </PackageReference>
</ItemGroup>
```

```bash
# รัน tests พร้อม coverage
dotnet test --collect:"XPlat Code Coverage"

# กำหนด format
dotnet test \
  --collect:"XPlat Code Coverage" \
  --results-directory ./coverage \
  -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=opencover

# กำหนด threshold — fail ถ้า coverage ต่ำกว่า 80%
dotnet test \
  /p:CollectCoverage=true \
  /p:CoverletOutputFormat=opencover \
  /p:Threshold=80 \
  /p:ThresholdType=line \
  /p:ThresholdStat=average

# สร้าง HTML report ด้วย ReportGenerator
dotnet tool install --global dotnet-reportgenerator-globaltool
reportgenerator \
  -reports:"./coverage/*/coverage.opencover.xml" \
  -targetdir:"./coverage/report" \
  -reporttypes:"Html;Cobertura;TextSummary"
```

### GitHub Actions พร้อม Coverage Report

```yaml
# .github/workflows/coverage.yml
name: Tests and Coverage

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'
      
      - name: Restore
        run: dotnet restore
      
      - name: Build
        run: dotnet build --no-restore --configuration Release
      
      - name: Test with coverage
        run: |
          dotnet test \
            --no-build \
            --configuration Release \
            /p:CollectCoverage=true \
            /p:CoverletOutputFormat="opencover%2ccobertura" \
            /p:CoverletOutput=./coverage/ \
            /p:Threshold=80 \
            /p:ThresholdType=line \
            /p:ExcludeByFile="**/Migrations/**/*,**/Program.cs"
      
      - name: Install ReportGenerator
        run: dotnet tool install --global dotnet-reportgenerator-globaltool
      
      - name: Generate coverage report
        run: |
          reportgenerator \
            -reports:"**/coverage/coverage.opencover.xml" \
            -targetdir:"coverage-report" \
            -reporttypes:"Html;Cobertura"
      
      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage-report/
      
      - name: Code Coverage Summary
        uses: irongut/CodeCoverageSummary@v1.3.0
        with:
          filename: '**/coverage/coverage.cobertura.xml'
          badge: true
          fail_below_min: true
          format: markdown
          hide_branch_rate: false
          hide_complexity: true
          indicators: true
          output: both
          thresholds: '60 80'
      
      - name: Add Coverage PR Comment
        uses: marocchino/sticky-pull-request-comment@v2
        if: github.event_name == 'pull_request'
        with:
          recreate: true
          path: code-coverage-results.md
```

### การวาง Coverage Attribute

```csharp
// ยกเว้น code จาก coverage ด้วย attribute
using System.Diagnostics.CodeAnalysis;

[ExcludeFromCodeCoverage]
public class Program
{
    public static void Main(string[] args)
    {
        // entry point — ไม่จำเป็นต้อง test
        var host = CreateHostBuilder(args).Build();
        host.Run();
    }
}

// ยกเว้นเฉพาะ method
public class UserRepository
{
    [ExcludeFromCodeCoverage(Justification = "Infrastructure code tested by integration tests")]
    public async Task<User?> GetByIdAsync(int id)
    {
        // database call
    }
    
    // method นี้ต้องมี unit tests
    public User? FindActiveUser(IEnumerable<User> users, int id)
    {
        return users.FirstOrDefault(u => u.Id == id && u.IsActive);
    }
}
```

---

## Step 788: Technical Debt Tracking

### การวัด Technical Debt

```csharp
// ตัวอย่าง code ที่มี technical debt — พร้อม annotation
public class LegacyOrderService
{
    // TODO: [TECH-DEBT-001] Refactor this method — Cyclomatic Complexity = 15
    // Estimated effort: 3 days
    // Impact: High (affects order processing performance)
    // Created: 2024-01-15 by @developer
    public decimal CalculateDiscount(Order order, Customer customer)
    {
        // ... complex nested conditions ...
        return 0;
    }
    
    // FIXME: [TECH-DEBT-002] Thread safety issue — not protected with lock
    // Estimated effort: 1 day  
    // Impact: Medium
    private static int _orderCount = 0;
    
    // HACK: [TECH-DEBT-003] Workaround for legacy API limitation
    // Can be removed after upgrading to API v3
    // Estimated effort: 2 days
    private void LegacyWorkaround()
    {
        // temporary workaround
    }
}
```

### Tech Debt Tracker ในรูปแบบ YAML

```yaml
# tech-debt.yml — ที่ root ของ project
version: "1.0"
last_updated: "2024-10-01"

items:
  - id: TECH-DEBT-001
    title: "Refactor LegacyOrderService.CalculateDiscount"
    category: complexity
    severity: high
    estimated_effort_days: 3
    file: "src/Services/LegacyOrderService.cs"
    line: 45
    created_date: "2024-01-15"
    created_by: "@developer1"
    tags: [refactoring, performance]
    description: |
      Cyclomatic Complexity = 15, เกินค่า threshold ที่กำหนด (10)
      ควร extract เป็น strategy pattern
    acceptance_criteria:
      - CC ≤ 8
      - มี unit tests ครอบคลุม 90%+
      - ไม่กระทบ external behavior

  - id: TECH-DEBT-002
    title: "Fix thread safety in order counter"
    category: correctness
    severity: critical
    estimated_effort_days: 1
    file: "src/Services/LegacyOrderService.cs"
    line: 78
    created_date: "2024-03-10"
    created_by: "@developer2"
    tags: [thread-safety, concurrency]
    description: |
      _orderCount ไม่ได้ใช้ Interlocked หรือ lock
      อาจเกิด race condition ใน concurrent requests

summary:
  total_items: 2
  total_estimated_days: 4
  critical: 1
  high: 1
  medium: 0
  low: 0
```

### script ตรวจนับ TODO/FIXME/HACK

```bash
#!/bin/bash
# count-tech-debt.sh

echo "=== Technical Debt Summary ==="
echo ""

echo "TODOs:"
grep -rn "// TODO:" src/ --include="*.cs" | wc -l

echo "FIXMEs:"
grep -rn "// FIXME:" src/ --include="*.cs" | wc -l

echo "HACKs:"
grep -rn "// HACK:" src/ --include="*.cs" | wc -l

echo ""
echo "=== Top Files with Most TODOs ==="
grep -rn "// TODO:" src/ --include="*.cs" | \
  awk -F: '{print $1}' | \
  sort | uniq -c | sort -rn | head -10

echo ""
echo "=== Detail ==="
grep -rn "// TODO:\|// FIXME:\|// HACK:" src/ --include="*.cs"
```

---

## Step 789: Refactoring Patterns

### Pattern 1: Extract Method

```csharp
// ❌ Method ยาวเกินไป — ควร extract
public async Task<OrderResult> PlaceOrderAsync(OrderRequest request)
{
    // Validate
    if (request == null)
        throw new ArgumentNullException(nameof(request));
    if (request.Items == null || !request.Items.Any())
        throw new InvalidOperationException("Order must have items");
    if (request.CustomerId <= 0)
        throw new ArgumentException("Invalid customer ID");
    
    var customer = await _customerRepository.GetByIdAsync(request.CustomerId);
    if (customer == null)
        throw new NotFoundException($"Customer {request.CustomerId} not found");
    if (customer.CreditLimit < 0)
        throw new InvalidOperationException("Customer has exceeded credit limit");
    
    // Calculate pricing
    decimal subtotal = 0;
    foreach (var item in request.Items)
    {
        var product = await _productRepository.GetByIdAsync(item.ProductId);
        if (product == null) continue;
        subtotal += product.Price * item.Quantity;
    }
    
    decimal tax = subtotal * 0.07m;
    decimal shipping = subtotal > 1000 ? 0 : 50;
    decimal total = subtotal + tax + shipping;
    
    // Create and save order
    var order = new Order
    {
        CustomerId = request.CustomerId,
        Items = request.Items.Select(i => new OrderItem { ... }).ToList(),
        Subtotal = subtotal,
        Tax = tax,
        Shipping = shipping,
        Total = total,
        Status = OrderStatus.Pending,
        CreatedAt = DateTime.UtcNow
    };
    
    await _orderRepository.AddAsync(order);
    await _emailService.SendOrderConfirmationAsync(customer.Email, order);
    
    return new OrderResult { OrderId = order.Id, Total = total };
}

// ✅ หลัง Extract Method
public async Task<OrderResult> PlaceOrderAsync(OrderRequest request)
{
    await ValidateOrderRequestAsync(request);
    var pricing = await CalculatePricingAsync(request.Items);
    var order = await CreateAndSaveOrderAsync(request, pricing);
    await NotifyCustomerAsync(order);
    return OrderResult.From(order);
}

private async Task ValidateOrderRequestAsync(OrderRequest request)
{
    ArgumentNullException.ThrowIfNull(request);
    
    if (request.Items == null || !request.Items.Any())
        throw new InvalidOperationException("Order must have items");
    
    if (request.CustomerId <= 0)
        throw new ArgumentException("Invalid customer ID", nameof(request));
    
    var customer = await _customerRepository.GetByIdAsync(request.CustomerId)
        ?? throw new NotFoundException($"Customer {request.CustomerId} not found");
    
    if (customer.CreditLimit < 0)
        throw new InvalidOperationException("Customer has exceeded credit limit");
}

private async Task<OrderPricing> CalculatePricingAsync(IEnumerable<OrderRequestItem> items)
{
    decimal subtotal = 0;
    foreach (var item in items)
    {
        var product = await _productRepository.GetByIdAsync(item.ProductId);
        if (product != null)
            subtotal += product.Price * item.Quantity;
    }
    
    return new OrderPricing(
        Subtotal: subtotal,
        Tax: subtotal * 0.07m,
        Shipping: subtotal > 1000 ? 0 : 50);
}

private async Task<Order> CreateAndSaveOrderAsync(OrderRequest request, OrderPricing pricing)
{
    var order = Order.Create(request, pricing);
    await _orderRepository.AddAsync(order);
    return order;
}

private async Task NotifyCustomerAsync(Order order)
{
    var customer = await _customerRepository.GetByIdAsync(order.CustomerId);
    if (customer != null)
        await _emailService.SendOrderConfirmationAsync(customer.Email, order);
}
```

### Pattern 2: Replace Conditional with Polymorphism

```csharp
// ❌ ใช้ switch/if ที่ต้องเพิ่มทุกครั้งที่มี type ใหม่
public class ShippingCalculator
{
    public decimal Calculate(Order order, string shippingMethod)
    {
        return shippingMethod switch
        {
            "Standard"   => order.Total > 500 ? 0 : 50,
            "Express"    => 150,
            "Overnight"  => 300,
            "SameDay"    => 500,
            "PickUp"     => 0,
            _ => throw new ArgumentException($"Unknown shipping method: {shippingMethod}")
        };
    }
    
    public string GetEstimatedDelivery(string shippingMethod) =>
        shippingMethod switch
        {
            "Standard"   => "3-5 business days",
            "Express"    => "1-2 business days",
            "Overnight"  => "Next business day",
            "SameDay"    => "Same day",
            "PickUp"     => "Ready for pickup",
            _ => throw new ArgumentException($"Unknown shipping method: {shippingMethod}")
        };
}

// ✅ Replace Conditional with Polymorphism
public abstract class ShippingMethod
{
    public abstract string Name { get; }
    public abstract string EstimatedDelivery { get; }
    public abstract decimal CalculateCost(Order order);
    
    // Factory method
    public static ShippingMethod Create(string methodName) => methodName switch
    {
        "Standard"  => new StandardShipping(),
        "Express"   => new ExpressShipping(),
        "Overnight" => new OvernightShipping(),
        "SameDay"   => new SameDayShipping(),
        "PickUp"    => new PickUpShipping(),
        _ => throw new ArgumentException($"Unknown shipping method: {methodName}")
    };
}

public sealed class StandardShipping : ShippingMethod
{
    public override string Name => "Standard";
    public override string EstimatedDelivery => "3-5 business days";
    public override decimal CalculateCost(Order order) =>
        order.Total > 500 ? 0 : 50;
}

public sealed class ExpressShipping : ShippingMethod
{
    public override string Name => "Express";
    public override string EstimatedDelivery => "1-2 business days";
    public override decimal CalculateCost(Order order) => 150;
}

public sealed class OvernightShipping : ShippingMethod
{
    public override string Name => "Overnight";
    public override string EstimatedDelivery => "Next business day";
    public override decimal CalculateCost(Order order) => 300;
}

public sealed class SameDayShipping : ShippingMethod
{
    public override string Name => "SameDay";
    public override string EstimatedDelivery => "Same day";
    
    public override decimal CalculateCost(Order order)
    {
        // same day มีค่าใช้จ่ายพิเศษตาม distance
        var distanceFee = order.DeliveryDistanceKm * 5;
        return 500 + distanceFee;
    }
}

public sealed class PickUpShipping : ShippingMethod
{
    public override string Name => "PickUp";
    public override string EstimatedDelivery => "Ready for pickup";
    public override decimal CalculateCost(Order order) => 0;
}

// การใช้งาน — ไม่ต้องแก้ ShippingCalculator เมื่อเพิ่ม method ใหม่
public class ShippingCalculator
{
    public decimal Calculate(Order order, ShippingMethod method) =>
        method.CalculateCost(order);
}
```

### Pattern 3: Replace Primitive Obsession with Value Object

```csharp
// ❌ Primitive Obsession
public class Order
{
    public string CustomerEmail { get; set; }   // ไม่ validate
    public string PhoneNumber { get; set; }      // ไม่ validate
    public decimal Amount { get; set; }           // ไม่ชัดเจนว่า currency อะไร
    public string Currency { get; set; }
}

// ✅ Value Objects
public sealed class Email : IEquatable<Email>
{
    public string Value { get; }
    
    private Email(string value) => Value = value;
    
    public static Email Create(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new ArgumentException("Email cannot be empty");
        
        if (!value.Contains('@') || !value.Contains('.'))
            throw new ArgumentException($"'{value}' is not a valid email address");
        
        return new Email(value.ToLowerInvariant().Trim());
    }
    
    public static implicit operator string(Email email) => email.Value;
    
    public bool Equals(Email? other) => other?.Value == Value;
    public override bool Equals(object? obj) => obj is Email email && Equals(email);
    public override int GetHashCode() => Value.GetHashCode(StringComparison.OrdinalIgnoreCase);
    public override string ToString() => Value;
}

public sealed class Money : IEquatable<Money>, IComparable<Money>
{
    public decimal Amount { get; }
    public string Currency { get; }
    
    private Money(decimal amount, string currency)
    {
        if (amount < 0)
            throw new ArgumentException("Amount cannot be negative");
        if (string.IsNullOrEmpty(currency) || currency.Length != 3)
            throw new ArgumentException("Currency must be a 3-letter ISO code");
        
        Amount = amount;
        Currency = currency.ToUpperInvariant();
    }
    
    public static Money Of(decimal amount, string currency) =>
        new Money(amount, currency);
    
    public static Money Baht(decimal amount) => new Money(amount, "THB");
    public static Money Usd(decimal amount) => new Money(amount, "USD");
    
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException(
                $"Cannot add {Currency} and {other.Currency}");
        return new Money(Amount + other.Amount, Currency);
    }
    
    public Money Multiply(decimal factor) =>
        new Money(Amount * factor, Currency);
    
    public bool Equals(Money? other) =>
        other?.Amount == Amount && other?.Currency == Currency;
    
    public override bool Equals(object? obj) =>
        obj is Money money && Equals(money);
    
    public override int GetHashCode() =>
        HashCode.Combine(Amount, Currency);
    
    public int CompareTo(Money? other)
    {
        if (other is null) return 1;
        if (Currency != other.Currency)
            throw new InvalidOperationException("Cannot compare different currencies");
        return Amount.CompareTo(other.Amount);
    }
    
    public override string ToString() => $"{Amount:N2} {Currency}";
    
    public static bool operator >(Money left, Money right) =>
        left.CompareTo(right) > 0;
    
    public static bool operator <(Money left, Money right) =>
        left.CompareTo(right) < 0;
    
    public static Money operator +(Money left, Money right) =>
        left.Add(right);
}

// การใช้งาน
public class Order
{
    public Email CustomerEmail { get; private set; }
    public Money TotalAmount { get; private set; }
    
    public Order(Email email, Money total)
    {
        CustomerEmail = email;
        TotalAmount = total;
    }
    
    public void AddShippingFee(Money fee)
    {
        TotalAmount = TotalAmount + fee; // compile error ถ้า currency ไม่ตรงกัน
    }
}

// สร้าง Order
var order = new Order(
    Email.Create("user@example.com"),
    Money.Baht(1500));
```

---

## Step 790: Code Review Checklist and PR Guidelines

### Code Review Checklist

```markdown
# Code Review Checklist

## ✅ Correctness
- [ ] Logic ถูกต้องตาม requirements
- [ ] Edge cases ได้รับการจัดการ (null, empty, boundary values)
- [ ] Error handling เหมาะสม — ไม่ swallow exceptions
- [ ] Thread safety (concurrent access, shared state)
- [ ] Resource disposal ถูกต้อง (IDisposable, async/await)
- [ ] ไม่มี race conditions หรือ deadlocks

## ✅ Security
- [ ] ไม่มี hardcoded credentials หรือ secrets
- [ ] Input validation ครบถ้วน
- [ ] SQL Injection prevention (parameterized queries)
- [ ] XSS prevention (ถ้ามี web output)
- [ ] Sensitive data ไม่ถูก log
- [ ] Authentication/Authorization ถูกต้อง

## ✅ Performance
- [ ] ไม่มี N+1 query problems
- [ ] Async/await ใช้อย่างถูกต้อง (ConfigureAwait)
- [ ] Caching ใช้เหมาะสม
- [ ] LINQ queries ไม่ trigger multiple enumerations
- [ ] Large collections ใช้ pagination

## ✅ Code Quality
- [ ] Method ไม่ยาวเกิน 40 บรรทัด
- [ ] Cyclomatic Complexity ≤ 10
- [ ] ตั้งชื่อ variables/methods/classes ชัดเจน
- [ ] ไม่มี magic numbers/strings — ใช้ constants
- [ ] DRY — ไม่มี code ซ้ำซ้อน
- [ ] SOLID principles ถูกนำไปใช้

## ✅ Tests
- [ ] มี unit tests สำหรับ business logic ใหม่
- [ ] Test names อธิบาย behavior ชัดเจน (Given_When_Then)
- [ ] Tests ไม่ขึ้นกัน external state
- [ ] Coverage ≥ 80% สำหรับ new code
- [ ] Integration tests สำหรับ API endpoints

## ✅ Documentation
- [ ] Public API มี XML doc comments
- [ ] Complex algorithms มี comments อธิบาย
- [ ] Breaking changes documented
- [ ] README อัปเดตถ้าจำเป็น

## ✅ Maintainability
- [ ] Code อ่านเข้าใจได้โดยไม่ต้องอ่าน comments
- [ ] ไม่มี TODO/FIXME ที่ไม่มี tracking issue
- [ ] Dependencies เหมาะสม (ไม่มี circular dependencies)
- [ ] Feature flags/toggles ถ้า deploy ก่อน release
```

### PR Template

```markdown
<!-- .github/PULL_REQUEST_TEMPLATE.md -->
## สรุปการเปลี่ยนแปลง (Summary of Changes)

<!-- อธิบายสิ้นๆ ว่าทำอะไร และทำไม -->

## ประเภท PR (Type of Change)

- [ ] 🐛 Bug fix (แก้ปัญหาที่มีอยู่)
- [ ] ✨ New feature (เพิ่ม functionality ใหม่)
- [ ] ♻️ Refactoring (ไม่กระทบ functionality)
- [ ] 📝 Documentation
- [ ] 🔧 Configuration/Infrastructure
- [ ] ⚡ Performance improvement
- [ ] 🔒 Security fix

## Issues ที่เกี่ยวข้อง

Closes #<!-- issue number -->

## วิธีทดสอบ (How to Test)

1. ...
2. ...
3. ผลที่คาดหวัง: ...

## Screenshots (ถ้ามี UI changes)

<!-- แนบ screenshot ก่อน/หลัง -->

## Checklist

- [ ] Code ผ่าน linting และ formatting
- [ ] เพิ่ม/อัปเดต tests
- [ ] Tests ผ่านทั้งหมด
- [ ] Documentation อัปเดตแล้ว
- [ ] ไม่มี breaking changes (หรือ documented แล้ว)
- [ ] Self-reviewed แล้ว

## Notes สำหรับ Reviewer

<!-- สิ่งที่ควรให้ reviewer สนใจเป็นพิเศษ -->
```

### GitHub Actions สำหรับ PR Validation

```yaml
# .github/workflows/pr-checks.yml
name: PR Checks

on:
  pull_request:
    branches: [ main, develop ]
    types: [ opened, synchronize, reopened ]

jobs:
  code-quality:
    name: Code Quality Checks
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'
      
      - name: Restore dependencies
        run: dotnet restore
      
      - name: Check formatting
        run: dotnet format --verify-no-changes --severity warn
      
      - name: Build with warnings as errors
        run: dotnet build --no-restore --configuration Release -warnaserror
      
      - name: Run tests
        run: |
          dotnet test \
            --no-build \
            --configuration Release \
            /p:CollectCoverage=true \
            /p:Threshold=80 \
            /p:ThresholdType=line \
            --logger "trx;LogFileName=test-results.trx"
      
      - name: Publish test results
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Test Results
          path: '**/*.trx'
          reporter: dotnet-trx
      
      - name: Check for secrets
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
      
      - name: Validate PR title
        uses: amannn/action-semantic-pull-request@v5
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          types: |
            feat
            fix
            docs
            style
            refactor
            perf
            test
            build
            ci
            chore
            revert
          requireScope: false
```

### Automated Code Review ด้วย Danger

```javascript
// Dangerfile.js — automated PR checks
import { danger, warn, fail, message } from "danger";

// ตรวจสอบ PR description
if (!danger.github.pr.body || danger.github.pr.body.length < 100) {
  fail("PR description ควรอธิบายการเปลี่ยนแปลงอย่างละเอียด (อย่างน้อย 100 ตัวอักษร)");
}

// ตรวจสอบว่ามี tests
const hasNewTests = danger.git.created_files
  .filter(f => f.includes("Tests") || f.includes(".Test."))
  .length > 0;

const hasChangedSource = danger.git.modified_files
  .filter(f => f.endsWith(".cs") && !f.includes("Tests"))
  .length > 0;

if (hasChangedSource && !hasNewTests) {
  warn("การเปลี่ยนแปลง source code ควรมี tests ด้วย");
}

// ตรวจสอบขนาด PR
const bigPRThreshold = 500;
const totalChanges = danger.github.pr.additions + danger.github.pr.deletions;
if (totalChanges > bigPRThreshold) {
  warn(`PR นี้มีการเปลี่ยนแปลง ${totalChanges} บรรทัด ควรพิจารณาแบ่งเป็น PR เล็กลง`);
}

// ตรวจสอบ TODO ใหม่
const newTodos = danger.git.created_files
  .concat(danger.git.modified_files)
  .filter(f => f.endsWith(".cs"));
  
// ตรวจสอบว่ามี CHANGELOG update
const hasChangelog = danger.git.modified_files.includes("CHANGELOG.md");
if (!hasChangelog && totalChanges > 50) {
  message("พิจารณาอัปเดต CHANGELOG.md ด้วย");
}
```

### ตัวอย่าง Code Smell และการแก้ไข

```csharp
// ❌ Code Smell: Long Parameter List
public User CreateUser(
    string firstName,
    string lastName,
    string email,
    string phone,
    string address,
    string city,
    string country,
    DateTime birthDate,
    bool isAdmin,
    bool isActive)
{
    // ...
}

// ✅ แก้ด้วย Parameter Object
public record CreateUserRequest(
    string FirstName,
    string LastName,
    Email Email,
    PhoneNumber Phone,
    Address Address,
    DateOnly BirthDate,
    bool IsAdmin = false,
    bool IsActive = true);

public User CreateUser(CreateUserRequest request)
{
    // ...
}

// ❌ Code Smell: Divergent Change — class เปลี่ยนด้วยเหตุผลหลายอย่าง
public class ReportService
{
    // เปลี่ยนเมื่อ data format เปลี่ยน
    public List<SalesData> FetchSalesData(DateTime from, DateTime to) { /* ... */ }
    
    // เปลี่ยนเมื่อ calculation logic เปลี่ยน
    public decimal CalculateTotalRevenue(List<SalesData> data) { /* ... */ }
    
    // เปลี่ยนเมื่อ report format เปลี่ยน
    public string GeneratePdfReport(List<SalesData> data) { /* ... */ }
    
    // เปลี่ยนเมื่อ email template เปลี่ยน
    public void SendReportByEmail(string report, string recipient) { /* ... */ }
}

// ✅ แก้ด้วยการแยก responsibilities
public class SalesDataRepository
{
    public Task<IEnumerable<SalesData>> GetAsync(DateRange dateRange) { /* ... */ }
}

public class SalesCalculationService
{
    public decimal CalculateTotalRevenue(IEnumerable<SalesData> data) { /* ... */ }
    public Dictionary<string, decimal> CalculateRevenueByProduct(IEnumerable<SalesData> data) { /* ... */ }
}

public class ReportGenerator
{
    public Stream GeneratePdf(IEnumerable<SalesData> data, ReportOptions options) { /* ... */ }
    public string GenerateCsv(IEnumerable<SalesData> data) { /* ... */ }
}

public class ReportEmailService
{
    public Task SendAsync(string recipientEmail, Stream reportAttachment) { /* ... */ }
}

// Orchestration ผ่าน use case / application service
public class GenerateSalesReportUseCase
{
    private readonly SalesDataRepository _repository;
    private readonly SalesCalculationService _calculator;
    private readonly ReportGenerator _generator;
    private readonly ReportEmailService _emailService;
    
    public async Task ExecuteAsync(GenerateSalesReportCommand command)
    {
        var data = await _repository.GetAsync(command.DateRange);
        var report = _generator.GeneratePdf(data, command.Options);
        await _emailService.SendAsync(command.RecipientEmail, report);
    }
}
```

### CODEOWNERS สำหรับ Code Review

```
# .github/CODEOWNERS
# กำหนดผู้รับผิดชอบ review code ในแต่ละส่วน

# Default owners สำหรับทุกไฟล์
* @team-leads

# Backend services
/src/Services/**  @backend-team
/src/Domain/**    @domain-experts @backend-team

# Infrastructure
/src/Infrastructure/**  @platform-team @backend-team

# Tests
/tests/**  @qa-team @backend-team

# CI/CD configurations
/.github/**  @platform-team
/docker/**   @platform-team

# Database migrations — ต้องให้ DBA approve
/src/Infrastructure/Migrations/**  @dba-team @backend-team

# Security-sensitive code
/src/Security/**  @security-team @backend-team
```

---

## สรุป (Summary)

ในส่วน Step 781-790 นี้เราได้เรียนรู้:

| Step | หัวข้อ | เครื่องมือ/Pattern |
|------|--------|-------------------|
| 781 | Code Metrics | Cyclomatic Complexity, Coupling, Cohesion |
| 782 | Roslyn Analyzers | NetAnalyzers, Roslynator, Directory.Build.props |
| 783 | StyleCop & .editorconfig | dotnet format, StyleCop.Analyzers |
| 784 | Custom Analyzer | DiagnosticAnalyzer, CodeFixProvider |
| 785 | SonarQube/SonarCloud | GitHub Actions, Quality Gate |
| 786 | Security Scanning | dotnet audit, CodeQL, Central Package Management |
| 787 | Code Coverage | Coverlet, ReportGenerator, GitHub Actions |
| 788 | Technical Debt | YAML tracking, TODO/FIXME conventions |
| 789 | Refactoring Patterns | Extract Method, Replace Conditional with Polymorphism |
| 790 | Code Review | PR Template, CODEOWNERS, Danger.js |

### Key Takeaways

1. **Quality ไม่ใช่เรื่องของ style** — มันเกี่ยวกับ correctness, maintainability และ security
2. **Automate ทุกที่ที่ทำได้** — analyzers, formatters, CI/CD checks ช่วยลด human error
3. **Metrics เป็นเครื่องมือ ไม่ใช่เป้าหมาย** — ใช้เพื่อหาจุดที่ควรปรับปรุง
4. **Refactoring ต้องมี tests รองรับ** — อย่า refactor โดยไม่มี safety net
5. **Code review เป็น collaboration** — เป้าหมายคือคุณภาพ ไม่ใช่การ criticize

---

**ก่อนหน้า → [Part 78: WinForms Advanced](part78-winforms-advanced.md)**
**ต่อไป → [Part 80: Architecture Review](part80-architecture-review.md)**
