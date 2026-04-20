---
name: vb-winforms
description: "Use when building, maintaining, or modernizing VB.NET Windows Forms apps on .NET or .NET Framework, covering designer-driven UI, MVP separation, data binding, async UI, validation, and .NET Framework-to-modern migration."
---

# VB.NET Windows Forms

## Trigger On

- working on VB.NET Windows Forms UI, event-driven workflows, or classic LOB applications
- migrating VB.NET WinForms from .NET Framework to modern .NET
- cleaning up oversized form code or designer coupling
- implementing data binding, validation, or control customization

## Workflow

1. **Respect designer boundaries** — never edit `.Designer.vb` directly; changes are lost on regeneration.
2. **Separate business logic from forms** — use MVP (Model-View-Presenter) pattern. Forms orchestrate UI; presenters contain logic; services handle data access.
   ```vb
   ' View interface — forms implement this
   Public Interface ICustomerView
       Property CustomerName As String
       Event SaveRequested As EventHandler
       Sub ShowError(message As String)
   End Interface

   ' Presenter — testable without UI
   Public Class CustomerPresenter
       Private ReadOnly _view As ICustomerView
       Private ReadOnly _service As ICustomerService

       Public Sub New(view As ICustomerView, service As ICustomerService)
           _view = view
           _service = service
           AddHandler _view.SaveRequested, AddressOf OnSaveRequested
       End Sub

       Private Async Sub OnSaveRequested(sender As Object, e As EventArgs)
           Try
               Await _service.SaveAsync(_view.CustomerName)
           Catch ex As Exception
               _view.ShowError(ex.Message)
           End Try
       End Sub
   End Class
   ```
3. **Use DI from Program.vb** (.NET 6+):
   ```vb
   Imports Microsoft.Extensions.DependencyInjection

   Module Program
       <STAThread>
       Sub Main()
           ApplicationConfiguration.Initialize()

           Dim services As New ServiceCollection()
           services.AddSingleton(Of ICustomerService, CustomerService)()
           services.AddTransient(Of MainForm)()

           Using sp = services.BuildServiceProvider()
               Application.Run(sp.GetRequiredService(Of MainForm)())
           End Using
       End Sub
   End Module
   ```
4. **Use data binding** via `BindingSource` and `INotifyPropertyChanged` instead of manual control population. See references/patterns.md for complete binding patterns.
5. **Use async/await** for I/O operations — disable controls during loading, use `Progress(Of T)` for progress reporting. Never block the UI thread.
6. **Validate with `ErrorProvider`** and the `Validating` event. Call `ValidateChildren()` before save operations.
7. **Modernize incrementally** — prefer better structure over big-bang rewrites. Use modern .NET features where available (stock icons in .NET 7+, button commands in .NET 9+).

```mermaid
flowchart LR
  A["Form event"] --> B["Presenter handles logic"]
  B --> C["Service layer / data access"]
  C --> D["Update view via interface"]
  D --> E["Validate and display results"]
```

## Key Decisions

| Decision | Guidance |
|----------|----------|
| MVP vs MVVM | Prefer MVP for WinForms — simpler with event-driven model |
| BindingSource vs manual | Always prefer BindingSource for list/detail binding |
| Sync vs async I/O | Always async — use `Async Sub` only for event handlers |
| Custom controls | Extract reusable `UserControl` when form grows beyond ~300 lines |
| .NET Framework → .NET | Use the official migration guide; validate designer compatibility first |

## Deliver

- less brittle form code with clear UI/logic separation
- MVP pattern with testable presenters
- pragmatic modernization guidance for VB.NET WinForms-heavy apps
- data binding and validation patterns that reduce manual wiring

## Validate

- designer files stay stable and are not hand-edited
- forms are not acting as the application service layer
- async operations do not block the UI thread
- validation is implemented consistently with ErrorProvider
- Windows-only runtime behavior is tested on target

## References

- references/patterns.md - VB.NET WinForms architectural patterns (MVP, MVVM, Passive View), data binding, validation, form communication, threading, DI setup, and .NET 8+ features
- references/migration.md - step-by-step migration from .NET Framework to modern .NET, common issues, deployment options, and gradual migration strategies
