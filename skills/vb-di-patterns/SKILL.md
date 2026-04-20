---
name: vb-di-patterns
description: "Organize dependency injection registrations in VB.NET Windows Forms applications using IServiceCollection extension methods. Covers lifetimes, form constructor injection, scoped services inside long-lived forms, factories, keyed services, and the ServiceProvider + Application.Run bootstrap pattern."
compatibility: "Applies to VB.NET projects on .NET 6+ using Microsoft.Extensions.DependencyInjection."
---

# VB.NET WinForms Dependency Injection Patterns

## When to Use This Skill

Use this skill when:
- Organizing service registrations in a VB.NET Windows Forms application
- Avoiding a massive `Program.vb` with hundreds of `AddXxx` calls
- Injecting services (loggers, repositories, business services) into Forms
- Managing scoped services (like `DbContext`) inside long-lived forms
- Making service configuration reusable between production and tests

## Reference Files

- [advanced-patterns.md](references/advanced-patterns.md): Testing, scope management inside long-lived Forms, conditional/factory/keyed registration patterns

---

## The Problem

Without organization, `Program.vb` becomes unmanageable:

```vb
' BAD: 200+ lines of unorganized registrations in Program.vb
Imports Microsoft.Extensions.DependencyInjection

Module Program
    <STAThread>
    Sub Main()
        Application.EnableVisualStyles()
        Application.SetCompatibleTextRenderingDefault(False)

        Dim services As New ServiceCollection()

        services.AddScoped(Of ICustomerRepository, CustomerRepository)()
        services.AddScoped(Of IOrderRepository, OrderRepository)()
        services.AddScoped(Of IProductRepository, ProductRepository)()
        services.AddScoped(Of ICustomerService, CustomerService)()
        ' ... 150 more lines ...

        services.AddTransient(Of MainForm)()

        Using sp = services.BuildServiceProvider()
            Application.Run(sp.GetRequiredService(Of MainForm)())
        End Using
    End Sub
End Module
```

Problems: hard to find related registrations, no clear boundaries, can't reuse in tests, merge conflicts.

---

## The Solution: Extension Method Composition

VB.NET supports extension methods on any interface (including `IServiceCollection`) when declared inside a `Module` with the `<Extension()>` attribute. Group related registrations into extension methods, then compose them in `Program.vb`:

```vb
' GOOD: Clean, composable Program.vb
Imports Microsoft.Extensions.DependencyInjection

Module Program
    <STAThread>
    Sub Main()
        Application.EnableVisualStyles()
        Application.SetCompatibleTextRenderingDefault(False)

        Dim services As New ServiceCollection()

        services _
            .AddCustomerServices() _
            .AddOrderServices() _
            .AddEmailServices() _
            .AddPaymentServices() _
            .AddValidators() _
            .AddForms()

        ' validateScopes:=True catches root-provider + Scoped bugs at resolve time
        Using sp = services.BuildServiceProvider(validateScopes:=True)
            Application.Run(sp.GetRequiredService(Of MainForm)())
        End Using
    End Sub
End Module
```

---

## Extension Method Pattern

### Basic Structure

Extension methods on `IServiceCollection` live in a `Module`, marked with `<Extension()>`:

```vb
Imports System.Runtime.CompilerServices
Imports Microsoft.Extensions.DependencyInjection

Namespace MyApp.Customers

    Public Module CustomerServiceCollectionExtensions

        <Extension()>
        Public Function AddCustomerServices(services As IServiceCollection) As IServiceCollection
            services.AddScoped(Of ICustomerRepository, CustomerRepository)()
            services.AddScoped(Of ICustomerReadStore, CustomerReadStore)()
            services.AddScoped(Of ICustomerWriteStore, CustomerWriteStore)()
            services.AddScoped(Of ICustomerService, CustomerService)()
            services.AddScoped(Of ICustomerValidationService, CustomerValidationService)()

            Return services
        End Function

    End Module

End Namespace
```

### With Configuration

```vb
Imports System.Runtime.CompilerServices
Imports Microsoft.Extensions.DependencyInjection
Imports Microsoft.Extensions.Options

Namespace MyApp.Email

    Public Module EmailServiceCollectionExtensions

        <Extension()>
        Public Function AddEmailServices(
                services As IServiceCollection,
                Optional configSectionName As String = "EmailSettings") As IServiceCollection

            services.AddOptions(Of EmailOptions)() _
                .BindConfiguration(configSectionName) _
                .ValidateDataAnnotations() _
                .ValidateOnStart()

            services.AddSingleton(Of IMjmlTemplateRenderer, MjmlTemplateRenderer)()
            services.AddSingleton(Of IEmailLinkGenerator, EmailLinkGenerator)()
            services.AddScoped(Of ICustomerEmailComposer, CustomerEmailComposer)()
            services.AddScoped(Of IEmailSender, SmtpEmailSender)()

            Return services
        End Function

    End Module

End Namespace
```

---

## File Organization

Place extension methods near the services they register:

```
src/
  MyApp.App/
    Program.vb                            ' Composes all Add* methods
    MainForm.vb                           ' Forms and presenters
  MyApp.Customers/
    Services/
      CustomerService.vb
    CustomerServiceCollectionExtensions.vb    ' AddCustomerServices()
  MyApp.Orders/
    OrderServiceCollectionExtensions.vb       ' AddOrderServices()
  MyApp.Email/
    EmailServiceCollectionExtensions.vb       ' AddEmailServices()
```

**Convention**: `{Feature}ServiceCollectionExtensions.vb` next to the feature's services.

---

## Naming Conventions

| Pattern | Use For |
|---------|---------|
| `Add{Feature}Services()` | General feature registration |
| `Add{Feature}()` | Short form when unambiguous |
| `Configure{Feature}()` | When primarily setting options |
| `Add{Feature}Forms()` | When registering forms as DI services |

---

## Injecting Services into Forms

Forms participate in DI like any other service. Three steps:

1. Register the form itself (usually `Transient` so each `Show()`/`ShowDialog()` gets a fresh instance).
2. Have the form accept its dependencies via constructor.
3. Resolve the root form (`MainForm`) from the provider and pass it to `Application.Run`.

```vb
' Step 1 — register the form
Imports System.Runtime.CompilerServices
Imports Microsoft.Extensions.DependencyInjection

Public Module FormServiceCollectionExtensions

    <Extension()>
    Public Function AddForms(services As IServiceCollection) As IServiceCollection
        services.AddTransient(Of MainForm)()
        services.AddTransient(Of CustomerEditForm)()
        Return services
    End Function

End Module
```

```vb
' Step 2 — constructor injection on the Form
Public Class MainForm
    Inherits Form

    Private ReadOnly _customerService As ICustomerService
    Private ReadOnly _logger As ILogger(Of MainForm)

    Public Sub New(customerService As ICustomerService, logger As ILogger(Of MainForm))
        _customerService = customerService
        _logger = logger
        InitializeComponent()
    End Sub

End Class
```

```vb
' Step 3 — bootstrap in Program.vb
Imports Microsoft.Extensions.DependencyInjection

Module Program
    <STAThread>
    Sub Main()
        Application.EnableVisualStyles()
        Application.SetCompatibleTextRenderingDefault(False)

        Dim services As New ServiceCollection()
        services.AddCustomerServices()
        services.AddForms()   ' registers MainForm as Transient (see Step 1)

        Using sp = services.BuildServiceProvider(validateScopes:=True)
            Application.Run(sp.GetRequiredService(Of MainForm)())
        End Using
    End Sub
End Module
```

> **Warning — root provider + `Scoped` services:** Resolving a `Scoped` service from the root provider behaves as a de facto Singleton and fails when `validateScopes:=True` is set, so register long-lived services like `MainForm` as `Transient` or `Singleton` and use `IServiceScopeFactory` for per-operation scoped work (see below).

### Opening child forms with DI

Don't call `New CustomerEditForm(...)` directly — that bypasses DI. Inject a factory delegate (`Func(Of CustomerEditForm)`) or resolve through the provider:

```vb
Public Class MainForm
    Inherits Form

    Private ReadOnly _editFormFactory As Func(Of CustomerEditForm)

    Public Sub New(editFormFactory As Func(Of CustomerEditForm))
        _editFormFactory = editFormFactory
        InitializeComponent()
    End Sub

    Private Sub btnEdit_Click(sender As Object, e As EventArgs) Handles btnEdit.Click
        Using dlg = _editFormFactory()
            dlg.ShowDialog(Me)
        End Using
    End Sub

End Class
```

`Microsoft.Extensions.DependencyInjection` does NOT auto-provide `Func(Of T)` factories (unlike Autofac). Register the factory explicitly:

```vb
services.AddTransient(Of CustomerEditForm)()
services.AddSingleton(Of Func(Of CustomerEditForm))(
    Function(sp) Function() sp.GetRequiredService(Of CustomerEditForm)())
```

Then inject `Func(Of CustomerEditForm)` and call it to get a new instance from DI.

---

## Lifetime Management

| Lifetime | Use When | Examples (WinForms) |
|----------|----------|---------------------|
| **Singleton** | Stateless, thread-safe, expensive to create | Configuration, HTTP client factories, caches, `ILogger(Of T)` |
| **Scoped** | Stateful per "unit of work" | `DbContext`, repositories tied to a single edit operation |
| **Transient** | Lightweight, stateful, cheap to create | Validators, per-dialog Forms, short-lived helpers |

```vb
' SINGLETON: Stateless services, shared safely
services.AddSingleton(Of IMjmlTemplateRenderer, MjmlTemplateRenderer)()

' SCOPED: Database access, per-operation state
services.AddScoped(Of ICustomerRepository, CustomerRepository)()

' TRANSIENT: Cheap, short-lived, per-dialog
services.AddTransient(Of CreateCustomerRequestValidator)()
services.AddTransient(Of CustomerEditForm)()
```

**Scoped services require a scope.** ASP.NET Core creates one per HTTP request automatically; a WinForms app has no such automatic boundary. You must create scopes yourself (typically one per "operation" like opening a dialog or pressing Save).

See [advanced-patterns.md](references/advanced-patterns.md) for the full scope-per-operation pattern.

---

## Testing Benefits

The `Add*` pattern lets you **reuse production configuration in tests** and only override what's different:

```vb
<TestClass>
Public Class CustomerServiceTests

    Private _provider As ServiceProvider

    <TestInitialize>
    Public Sub Setup()
        Dim services As New ServiceCollection()

        ' Reuse production registrations
        services.AddCustomerServices()

        ' Replace infrastructure with test doubles
        services.RemoveAll(Of ICustomerRepository)()
        services.AddSingleton(Of ICustomerRepository, InMemoryCustomerRepository)()

        _provider = services.BuildServiceProvider(validateScopes:=True)
    End Sub

    <TestCleanup>
    Public Sub Cleanup()
        _provider.Dispose()
    End Sub

    <TestMethod>
    Public Async Function CreateCustomer_ValidData_Succeeds() As Task
        ' Create an explicit scope — ICustomerService is Scoped, so it
        ' must not be resolved directly from the root provider.
        Using scope = _provider.CreateScope()
            Dim service = scope.ServiceProvider.GetRequiredService(Of ICustomerService)()
            Dim result = Await service.CreateAsync(New CreateCustomerRequest("Alice"))

            Assert.IsTrue(result.IsSuccess)
        End Using
    End Function

End Class
```

See [advanced-patterns.md](references/advanced-patterns.md) for complete testing examples.

---

## Layered Extensions

For larger applications, compose extensions hierarchically:

```vb
Imports System.Runtime.CompilerServices
Imports Microsoft.Extensions.DependencyInjection

Public Module AppServiceCollectionExtensions

    <Extension()>
    Public Function AddAppServices(services As IServiceCollection) As IServiceCollection
        Return services _
            .AddDomainServices() _
            .AddInfrastructureServices() _
            .AddUiServices()
    End Function

End Module

Public Module DomainServiceCollectionExtensions

    <Extension()>
    Public Function AddDomainServices(services As IServiceCollection) As IServiceCollection
        Return services _
            .AddCustomerServices() _
            .AddOrderServices() _
            .AddProductServices()
    End Function

End Module
```

---

## Generic Host Alternative

For apps that also need configuration loading, logging, or hosted services, use `Host.CreateDefaultBuilder()` to bootstrap instead of building the `ServiceProvider` by hand. This is the WinForms equivalent of ASP.NET Core's `WebApplication.CreateBuilder()`:

```vb
Imports Microsoft.Extensions.DependencyInjection
Imports Microsoft.Extensions.Hosting

Module Program
    <STAThread>
    Sub Main(args As String())
        Application.EnableVisualStyles()
        Application.SetCompatibleTextRenderingDefault(False)

        Dim host = Host.CreateDefaultBuilder(args) _
            .ConfigureServices(Sub(ctx, services)
                                   services.AddCustomerServices()
                                   services.AddForms()
                               End Sub) _
            .Build()

        Using host
            host.Start()
            Application.Run(host.Services.GetRequiredService(Of MainForm)())
            host.StopAsync().GetAwaiter().GetResult()
        End Using
    End Sub
End Module
```

This gives you `appsettings.json` binding, `ILogger(Of T)` with proper providers, and `IHostedService` support for long-running background work — all consistent with the rest of the .NET ecosystem.

---

## Anti-Patterns

### Don't: Register Everything in Program.vb

```vb
' BAD: Massive Program.vb with 200+ lines of registrations
```

### Don't: Create Overly Generic Extensions

```vb
' BAD: Too vague, doesn't communicate what's registered
<Extension()>
Public Function AddServices(services As IServiceCollection) As IServiceCollection
    ' ...
    Return services
End Function
```

### Don't: Hide Important Configuration

```vb
' BAD: Buried settings
<Extension()>
Public Function AddDatabase(services As IServiceCollection) As IServiceCollection
    services.AddDbContext(Of AppDbContext)(
        Sub(options) options.UseSqlServer("hardcoded-connection-string"))  ' Hidden!
    Return services
End Function

' GOOD: Accept configuration explicitly
<Extension()>
Public Function AddDatabase(services As IServiceCollection, connectionString As String) As IServiceCollection
    services.AddDbContext(Of AppDbContext)(
        Sub(options) options.UseSqlServer(connectionString))
    Return services
End Function
```

### Don't: `New MyForm()` When MyForm Is Registered

```vb
' BAD: Bypasses DI — dependencies won't be resolved
Dim dlg As New CustomerEditForm()
dlg.ShowDialog()

' GOOD: Resolve through a factory or the provider
Using dlg = _editFormFactory()
    dlg.ShowDialog(Me)
End Using
```

---

## Best Practices Summary

| Practice | Benefit |
|----------|---------|
| Group related services into `Add*` methods | Clean Program.vb, clear boundaries |
| Place extensions near the services they register | Easy to find and maintain |
| Return `IServiceCollection` for chaining | Fluent API |
| Accept configuration parameters | Flexibility |
| Use consistent naming (`Add{Feature}Services`) | Discoverability |
| Inject forms via constructor, register them in DI | Testable, swappable |
| Use `Func(Of TForm)` factories for child dialogs | Respect DI lifetimes |
| Test by reusing production extensions | Confidence, less duplication |

---

## Common Mistakes

### Injecting Scoped into Singleton

```vb
' BAD: Singleton captures scoped service — stale DbContext!
Public Class CacheService   ' Registered as Singleton
    Private ReadOnly _repo As ICustomerRepository  ' Scoped — captured at startup!

    Public Sub New(repo As ICustomerRepository)
        _repo = repo
    End Sub
End Class

' GOOD: Inject IServiceScopeFactory, create a scope per operation
Public Class CacheService
    Private ReadOnly _scopeFactory As IServiceScopeFactory

    Public Sub New(scopeFactory As IServiceScopeFactory)
        _scopeFactory = scopeFactory
    End Sub

    Public Async Function GetCustomerAsync(id As String) As Task(Of Customer)
        Using scope = _scopeFactory.CreateScope()
            Dim repo = scope.ServiceProvider.GetRequiredService(Of ICustomerRepository)()
            Return Await repo.GetByIdAsync(id)
        End Using
    End Function
End Class
```

### Holding a Scoped Service for the Life of a Form

A `MainForm` that lives for the whole session should **not** inject `DbContext` (scoped) directly — the context will be kept alive forever, accumulate tracked entities, and eventually throw. Instead, inject `IServiceScopeFactory` and create a fresh scope for each operation:

```vb
Public Class MainForm
    Inherits Form

    Private ReadOnly _scopeFactory As IServiceScopeFactory

    Public Sub New(scopeFactory As IServiceScopeFactory)
        _scopeFactory = scopeFactory
        InitializeComponent()
    End Sub

    Private Async Sub btnSave_Click(sender As Object, e As EventArgs) Handles btnSave.Click
        Using scope = _scopeFactory.CreateScope()
            Dim service = scope.ServiceProvider.GetRequiredService(Of ICustomerService)()
            Await service.SaveAsync(txtName.Text)
        End Using
    End Sub

End Class
```

See [advanced-patterns.md](references/advanced-patterns.md) for a deeper treatment of scope-per-operation in WinForms.

---

## Resources

- **Microsoft.Extensions.DependencyInjection**: https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection
- **Generic Host for desktop apps**: https://learn.microsoft.com/en-us/dotnet/core/extensions/generic-host
- **Options Pattern**: See related options/configuration skills
