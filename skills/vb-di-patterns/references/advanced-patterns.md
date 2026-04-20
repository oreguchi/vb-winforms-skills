# Advanced DI Patterns (VB.NET WinForms)

Testing with DI extensions, scope management inside long-lived forms, and advanced registration patterns.

## Contents

- [Testing Benefits](#testing-benefits)
- [Scope Management Inside Long-Lived Forms](#scope-management-inside-long-lived-forms)
- [Common Registration Patterns](#common-registration-patterns)

## Testing Benefits

The main advantage of `Add*` extension methods: **reuse production configuration in tests**.

### Standalone Unit Tests

```vb
Imports Microsoft.Extensions.DependencyInjection
Imports Microsoft.VisualStudio.TestTools.UnitTesting

<TestClass>
Public Class CustomerServiceTests

    Private _provider As ServiceProvider

    <TestInitialize>
    Public Sub Setup()
        Dim services As New ServiceCollection()

        ' Reuse production registrations
        services.AddCustomerServices()

        ' Add test infrastructure
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
        ' ICustomerService is registered as Scoped, so create an explicit
        ' scope rather than resolving from the root provider.
        Using scope = _provider.CreateScope()
            Dim service = scope.ServiceProvider.GetRequiredService(Of ICustomerService)()
            Dim result = Await service.CreateAsync(New CreateCustomerRequest("Alice"))

            Assert.IsTrue(result.IsSuccess)
        End Using
    End Function

End Class
```

### Smoke-Testing the Full Bootstrap

You can verify that every service registered in `Program.vb` can actually be resolved, without opening a Form:

```vb
<TestMethod>
Public Sub AllServices_AreResolvable()
    Dim services As New ServiceCollection()

    services _
        .AddCustomerServices() _
        .AddOrderServices() _
        .AddEmailServices() _
        .AddForms()

    Using sp = services.BuildServiceProvider(validateScopes:=True)
        ' Forcing resolution flushes any missing registration
        Dim mainForm = sp.GetRequiredService(Of MainForm)()
        Assert.IsNotNull(mainForm)
    End Using
End Sub
```

The `validateScopes:=True` flag catches "scoped captured by singleton" mistakes at test time.

## Scope Management Inside Long-Lived Forms

**A WinForms app has no automatic DI scopes.** ASP.NET Core creates one per HTTP request; the Generic Host creates one per `IHostedService` call. In WinForms you must create scopes yourself.

The rule: **scoped services must live no longer than one operation**. Good operation boundaries in a WinForms app:

- One button click (e.g. Save, Refresh)
- One `ShowDialog()` call for a child form
- One worker cycle in a background `BackgroundWorker` / `Task`

### Pattern: Scope Per Operation

```vb
Imports Microsoft.Extensions.DependencyInjection

Public Class MainForm
    Inherits Form

    Private ReadOnly _scopeFactory As IServiceScopeFactory

    Public Sub New(scopeFactory As IServiceScopeFactory)
        _scopeFactory = scopeFactory
        InitializeComponent()
    End Sub

    Private Async Sub btnSave_Click(sender As Object, e As EventArgs) Handles btnSave.Click
        ' Scope starts here...
        Using scope = _scopeFactory.CreateScope()
            Dim service = scope.ServiceProvider.GetRequiredService(Of ICustomerService)()
            Await service.SaveAsync(txtName.Text)
        End Using
        ' ...and ends here. DbContext is disposed, tracked entities released.
    End Sub

End Class
```

### Pattern: Scope Per Child Dialog

When opening a dialog, give it a fresh scope so its `DbContext` / repositories are isolated from the parent form:

```vb
Private Sub btnEdit_Click(sender As Object, e As EventArgs) Handles btnEdit.Click
    Using scope = _scopeFactory.CreateScope()
        Dim dlg = scope.ServiceProvider.GetRequiredService(Of CustomerEditForm)()
        dlg.ShowDialog(Me)
    End Using
End Sub
```

Register `CustomerEditForm` as `Scoped` (or `Transient` if you never need shared per-scope state), and its dependencies resolve from the same scope.

### Why This Pattern Works

1. **Each operation gets a fresh `DbContext`** — no stale entity tracking.
2. **Proper disposal** — connections released after each operation.
3. **Isolation** — one click's errors don't corrupt another click's state.
4. **Testable** — inject a mock `IServiceScopeFactory` or use `ServiceProvider.CreateScope()`.

### Singleton Services Don't Need a Scope

For stateless services, inject directly:

```vb
Public Class MainForm
    Inherits Form

    Private ReadOnly _linkGenerator As IEmailLinkGenerator   ' Singleton — OK
    Private ReadOnly _scopeFactory As IServiceScopeFactory   ' For scoped work

    Public Sub New(linkGenerator As IEmailLinkGenerator, scopeFactory As IServiceScopeFactory)
        _linkGenerator = linkGenerator
        _scopeFactory = scopeFactory
        InitializeComponent()
    End Sub

End Class
```

### Background Work: IHostedService

If you adopt `Host.CreateDefaultBuilder()` (see the main SKILL.md "Generic Host Alternative" section), you can run background workers alongside your UI using `IHostedService`. Each work cycle should create its own scope:

```vb
Imports Microsoft.Extensions.DependencyInjection
Imports Microsoft.Extensions.Hosting

Public Class OrderSyncService
    Inherits BackgroundService

    Private ReadOnly _scopeFactory As IServiceScopeFactory

    Public Sub New(scopeFactory As IServiceScopeFactory)
        _scopeFactory = scopeFactory
    End Sub

    Protected Overrides Async Function ExecuteAsync(stoppingToken As CancellationToken) As Task
        While Not stoppingToken.IsCancellationRequested
            Using scope = _scopeFactory.CreateScope()
                Dim orderService = scope.ServiceProvider.GetRequiredService(Of IOrderService)()
                Await orderService.SyncPendingAsync(stoppingToken)
            End Using

            Await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken)
        End While
    End Function

End Class
```

## Common Registration Patterns

### Conditional Registration

```vb
Imports System.Runtime.CompilerServices
Imports Microsoft.Extensions.DependencyInjection
Imports Microsoft.Extensions.Hosting

Public Module EmailServiceCollectionExtensions

    <Extension()>
    Public Function AddEmailServices(
            services As IServiceCollection,
            environment As IHostEnvironment) As IServiceCollection

        services.AddSingleton(Of IEmailComposer, MjmlEmailComposer)()

        If environment.IsDevelopment() Then
            services.AddSingleton(Of IEmailSender, MailpitEmailSender)()
        Else
            services.AddSingleton(Of IEmailSender, SmtpEmailSender)()
        End If

        Return services
    End Function

End Module
```

### Factory-Based Registration

Use the factory overload when construction requires runtime data from other services:

```vb
Imports System.Runtime.CompilerServices
Imports Microsoft.Extensions.DependencyInjection
Imports Microsoft.Extensions.Logging
Imports Microsoft.Extensions.Options

Public Module PaymentServiceCollectionExtensions

    <Extension()>
    Public Function AddPaymentServices(
            services As IServiceCollection,
            Optional configSection As String = "Stripe") As IServiceCollection

        services.AddOptions(Of StripeOptions)() _
            .BindConfiguration(configSection) _
            .ValidateOnStart()

        services.AddSingleton(Of IPaymentProcessor)(
            Function(sp)
                Dim options = sp.GetRequiredService(Of IOptions(Of StripeOptions))().Value
                Dim logger = sp.GetRequiredService(Of ILogger(Of StripePaymentProcessor))()
                Return New StripePaymentProcessor(options.ApiKey, options.WebhookSecret, logger)
            End Function)

        Return services
    End Function

End Module
```

### Keyed Services (.NET 8+)

Keyed services let you register multiple implementations of the same interface, distinguished by a key:

```vb
Imports System.Runtime.CompilerServices
Imports Microsoft.Extensions.DependencyInjection

Public Module NotificationServiceCollectionExtensions

    <Extension()>
    Public Function AddNotificationServices(services As IServiceCollection) As IServiceCollection
        services.AddKeyedSingleton(Of INotificationSender, EmailNotificationSender)("email")
        services.AddKeyedSingleton(Of INotificationSender, SmsNotificationSender)("sms")
        services.AddKeyedSingleton(Of INotificationSender, PushNotificationSender)("push")

        services.AddScoped(Of INotificationDispatcher, NotificationDispatcher)()

        Return services
    End Function

End Module
```

Resolve keyed services with `<FromKeyedServices("email")>` on the constructor parameter:

```vb
Public Class NotificationDispatcher
    Implements INotificationDispatcher

    Private ReadOnly _emailSender As INotificationSender
    Private ReadOnly _smsSender As INotificationSender

    Public Sub New(
            <FromKeyedServices("email")> emailSender As INotificationSender,
            <FromKeyedServices("sms")> smsSender As INotificationSender)
        _emailSender = emailSender
        _smsSender = smsSender
    End Sub

End Class
```

### Factory Delegates for Child Forms

`Microsoft.Extensions.DependencyInjection` does not automatically provide `Func(Of T)` factories. Register them explicitly when you need to open child dialogs from a parent form:

```vb
<Extension()>
Public Function AddFormFactories(services As IServiceCollection) As IServiceCollection
    services.AddTransient(Of CustomerEditForm)()
    services.AddSingleton(Of Func(Of CustomerEditForm))(
        Function(sp) Function() sp.GetRequiredService(Of CustomerEditForm)())
    Return services
End Function
```

Then inject `Func(Of CustomerEditForm)` into the parent form and call it whenever a fresh dialog is needed.
