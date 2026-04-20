---
name: vb-concurrency-patterns
description: "Choosing the right concurrency abstraction in a VB.NET Windows Forms application - from Async/Await for I/O to Channels for producer/consumer, with UI thread marshalling, IProgress(Of T), and CancellationTokenSource wired to a Cancel button. Avoid SyncLock and manual synchronization unless absolutely necessary."
compatibility: "Applies to VB.NET projects on .NET or .NET Framework with async/await support."
---

# VB.NET WinForms Concurrency: Choosing the Right Tool

## When to Use This Skill

Use this skill when:
- Deciding how to handle concurrent operations in a VB.NET Windows Forms application
- Evaluating whether to use Async/Await, Channels, or other abstractions
- Tempted to use `SyncLock`, semaphores, or other synchronization primitives
- Need to process streams of data with backpressure, batching, or debouncing
- Reporting progress from a background operation to a Form
- Wiring a Cancel button to cancel a running operation
- Managing state across multiple concurrent entities

## Reference Files

- [advanced-concurrency.md](references/advanced-concurrency.md): Akka.NET Streams, Reactive Extensions, Akka.NET Actors (entity-per-actor, state machines), and async local function patterns translated to VB.NET

## The Philosophy

**Start simple, escalate only when needed.**

Most concurrency problems in a WinForms LOB app can be solved with `Async/Await` plus `IProgress(Of T)` and a `CancellationTokenSource`. Only reach for more sophisticated tools when you have a specific need those cannot address cleanly.

**Try to avoid shared mutable state.** The best way to handle concurrency is to design it away. Immutable data, message passing, and isolated state (like actors) eliminate entire categories of bugs.

**Locks should be the exception, not the rule.** When you can't avoid shared mutable state:
1. **First choice:** Redesign to avoid it (immutability, message passing, actor isolation)
2. **Second choice:** Use `System.Collections.Concurrent` (`ConcurrentDictionary`, etc.)
3. **Third choice:** Use `Channel(Of T)` to serialize access through message passing
4. **Last resort:** Use `SyncLock` for simple, short-lived critical sections

---

## Decision Tree

```
What are you trying to do?
│
├─► Wait for I/O (HTTP, database, file, serial port, SPEL+ command)?
│   └─► Use Async/Await
│
├─► Process a collection in parallel (CPU-bound)?
│   └─► Use Parallel.ForEachAsync
│
├─► Producer/consumer pattern (work queue)?
│   └─► Use System.Threading.Channels
│
├─► Report progress from background work to a Form?
│   └─► Use IProgress(Of T) + Progress(Of T)
│
├─► Let the user cancel a long-running operation with a button?
│   └─► Use CancellationTokenSource tied to the Cancel button
│
├─► Update UI from non-UI thread (timer, worker thread, serial callback)?
│   └─► Use Control.Invoke / Control.BeginInvoke (or SynchronizationContext)
│
├─► UI event composition (debounce, throttle, combine)?
│   └─► Use Reactive Extensions (Rx)
│
├─► Server-side stream processing (backpressure, batching)?
│   └─► Use Akka.NET Streams
│
├─► State machines with complex transitions?
│   └─► Use Akka.NET Actors (Become pattern)
│
├─► Coordinate multiple async operations?
│   └─► Use Task.WhenAll / Task.WhenAny
│
└─► None of the above fits?
    └─► Ask yourself: "Do I really need shared mutable state?"
        ├─► Yes → Consider redesigning to avoid it
        └─► Truly unavoidable → Use Channels or Actors to serialize access
```

---

## Level 1: Async/Await (Default Choice)

**Use for:** I/O-bound operations, non-blocking waits, most everyday concurrency.

```vb
' Simple async I/O
Public Async Function GetOrderAsync(orderId As String, ct As CancellationToken) As Task(Of Order)
    Dim order = Await _database.GetAsync(orderId, ct)
    Dim customer = Await _customerService.GetAsync(order.CustomerId, ct)
    order.Customer = customer
    Return order
End Function

' Parallel async operations (when independent)
Public Async Function LoadDashboardAsync(userId As String, ct As CancellationToken) As Task(Of Dashboard)
    Dim ordersTask = _orderService.GetRecentOrdersAsync(userId, ct)
    Dim notificationsTask = _notificationService.GetUnreadAsync(userId, ct)
    Dim statsTask = _statsService.GetUserStatsAsync(userId, ct)

    Await Task.WhenAll(ordersTask, notificationsTask, statsTask)

    Return New Dashboard(
        orders:=Await ordersTask,
        notifications:=Await notificationsTask,
        stats:=Await statsTask)
End Function
```

**Key principles:** Always accept `CancellationToken`. Use `ConfigureAwait(False)` in library code. Don't block on async code (no `.Result` or `.Wait()`).

---

## Level 2: Parallel.ForEachAsync (CPU-Bound Parallelism)

**Use for:** Processing collections in parallel when work is CPU-bound or you need controlled concurrency.

```vb
Public Async Function ProcessOrdersAsync(
    orders As IEnumerable(Of Order),
    ct As CancellationToken) As Task

    Await Parallel.ForEachAsync(
        orders,
        New ParallelOptions With {
            .MaxDegreeOfParallelism = Environment.ProcessorCount,
            .CancellationToken = ct
        },
        Async Function(order, token)
            Await ProcessOrderAsync(order, token)
        End Function)
End Function
```

**When NOT to use:** Pure I/O operations, when order matters, when you need backpressure.

---

## Level 3: System.Threading.Channels (Producer/Consumer)

**Use for:** Work queues, producer/consumer patterns, decoupling producers from consumers (e.g. a robot controller pushing data points while a background worker writes them to disk).

```vb
Imports System.Threading.Channels

Public Class OrderProcessor
    Private ReadOnly _channel As Channel(Of Order)

    Public Sub New()
        _channel = Channel.CreateBounded(Of Order)(
            New BoundedChannelOptions(100) With {
                .FullMode = BoundedChannelFullMode.Wait
            })
    End Sub

    ' Producer
    Public Async Function EnqueueOrderAsync(order As Order, ct As CancellationToken) As Task
        Await _channel.Writer.WriteAsync(order, ct)
    End Function

    ' Consumer (run as background task)
    ' NOTE: VB.NET has no "Await For Each". Use GetAsyncEnumerator + MoveNextAsync.
    Public Async Function ProcessOrdersAsync(ct As CancellationToken) As Task
        Dim reader = _channel.Reader
        While Await reader.WaitToReadAsync(ct)
            Dim order As Order = Nothing
            While reader.TryRead(order)
                Await ProcessOrderAsync(order, ct)
            End While
        End While
    End Function

    Public Sub Complete()
        _channel.Writer.Complete()
    End Sub
End Class
```

**Channels are good for:** Decoupling producer/consumer speeds, buffering with backpressure, fan-out to workers, background queues.

**Channels are NOT good for:** Complex stream operations (batching, windowing), stateful per-entity processing, sophisticated supervision.

**VB.NET note:** C# consumers typically use `Await foreach (var x in reader.ReadAllAsync(ct))`. VB.NET has no `Await For Each`, so the idiomatic consumer pattern is the `WaitToReadAsync` + `TryRead` loop shown above. Authoring async iterators (`Yield Return` in an `IAsyncEnumerable`) is C#-only; consume them in VB by calling `GetAsyncEnumerator` and driving `MoveNextAsync` in a `While` loop.

---

## WinForms Specifics

### UI Thread Marshalling

Windows Forms has a single UI thread. Only the UI thread may touch UI controls. Violating this raises a `InvalidOperationException` ("Cross-thread operation not valid").

Good news: `Async/Await` in WinForms captures the current `SynchronizationContext` and returns to the UI thread automatically. The code below is safe:

```vb
' Inside a Form — runs on the UI thread
Private Async Sub btnLoad_Click(sender As Object, e As EventArgs) Handles btnLoad.Click
    btnLoad.Enabled = False
    Try
        ' Await captures the UI SynchronizationContext;
        ' when the Task completes, execution resumes on the UI thread.
        Dim data = Await _service.LoadAsync()
        lblResult.Text = data.Summary   ' Safe: back on the UI thread.
    Finally
        btnLoad.Enabled = True
    End Try
End Sub
```

When the work is **not** driven by `Async/Await` (e.g. a `System.Threading.Timer` callback, a serial-port `DataReceived` event, or a SPEL+ event callback on a worker thread), you must marshal explicitly:

```vb
' Called from a non-UI thread (timer, serial port, etc.)
Private Sub OnSensorReading(value As Double)
    If lblSensor.InvokeRequired Then
        lblSensor.BeginInvoke(Sub() lblSensor.Text = value.ToString("F3"))
    Else
        lblSensor.Text = value.ToString("F3")
    End If
End Sub
```

- `Control.Invoke` — synchronous; blocks the caller until the UI processes the delegate.
- `Control.BeginInvoke` — fire-and-forget; preferred for high-frequency updates (sensor streams, log messages).
- `SynchronizationContext.Current` — captured on the UI thread; use `.Post` (async) or `.Send` (sync) from worker code that doesn't hold a `Control` reference.

### Library code: ConfigureAwait(False)

In a **Form**, you want to return to the UI thread after `Await` — do *not* use `ConfigureAwait(False)`.

In a **class library** called from a Form, prefer `ConfigureAwait(False)` on every `Await` so the library doesn't depend on a particular synchronization context:

```vb
' Library code
Public Async Function LoadAsync() As Task(Of Data)
    Dim json = Await _http.GetStringAsync(_url).ConfigureAwait(False)
    Return JsonSerializer.Deserialize(Of Data)(json)
End Function
```

### IProgress(Of T) for Progress Reporting

`IProgress(Of T)` is the standard way to report progress from background work to a Form. `Progress(Of T)` captures the UI `SynchronizationContext` when constructed, so the callback fires on the UI thread — no manual `Invoke` needed.

```vb
' Service (library-style): takes an IProgress(Of T).
Public Async Function ProcessFilesAsync(
    files As IReadOnlyList(Of String),
    progress As IProgress(Of Integer),
    ct As CancellationToken) As Task

    For i = 0 To files.Count - 1
        ct.ThrowIfCancellationRequested()
        Await ProcessOneAsync(files(i), ct).ConfigureAwait(False)
        progress?.Report(CInt((i + 1) / files.Count * 100))
    Next
End Function

' Form: constructs Progress(Of T) on the UI thread.
Private Async Sub btnStart_Click(sender As Object, e As EventArgs) Handles btnStart.Click
    Dim reporter As IProgress(Of Integer) = New Progress(Of Integer)(
        Sub(pct)
            ' Runs on the UI thread. Safe to touch controls.
            progressBar1.Value = pct
            lblPercent.Text = $"{pct}%"
        End Sub)

    Await _service.ProcessFilesAsync(_files, reporter, CancellationToken.None)
End Sub
```

**Report complex state** (percent + message + ETA) with a small `Structure` or `Class` parameter instead of scalar `Integer`.

### Cancellation from a Cancel Button

Wire a `CancellationTokenSource` to the Cancel button so the user can abort long-running work. Dispose it when the Form closes or after the operation completes.

```vb
Public Class ImportForm
    Private _cts As CancellationTokenSource

    Private Async Sub btnStart_Click(sender As Object, e As EventArgs) Handles btnStart.Click
        _cts = New CancellationTokenSource()
        btnStart.Enabled = False
        btnCancel.Enabled = True
        Try
            Await _service.ImportAsync(_cts.Token)
            lblStatus.Text = "Done."
        Catch ex As OperationCanceledException
            lblStatus.Text = "Cancelled."
        Catch ex As Exception
            lblStatus.Text = $"Error: {ex.Message}"
        Finally
            btnStart.Enabled = True
            btnCancel.Enabled = False
            _cts?.Dispose()
            _cts = Nothing
        End Try
    End Sub

    Private Sub btnCancel_Click(sender As Object, e As EventArgs) Handles btnCancel.Click
        _cts?.Cancel()
    End Sub

    Private Sub ImportForm_FormClosing(sender As Object, e As FormClosingEventArgs) Handles MyBase.FormClosing
        ' Cancel and dispose if the user closes the form mid-import.
        _cts?.Cancel()
        _cts?.Dispose()
    End Sub
End Class
```

**Inside the service**, honour the token:

```vb
Public Async Function ImportAsync(ct As CancellationToken) As Task
    For Each row In _rows
        ct.ThrowIfCancellationRequested()
        Await _db.InsertAsync(row, ct).ConfigureAwait(False)
    Next
End Function
```

### Application.DoEvents is a Footgun

Legacy WinForms code often uses `Application.DoEvents()` inside a tight loop to "keep the UI responsive." Don't.

```vb
' BAD: re-entrant message pump, partially-initialized state, hard-to-debug behaviour
For i = 0 To 10000
    Process(i)
    Application.DoEvents()    ' pumps messages — user can click buttons mid-loop
Next
```

```vb
' GOOD: move the work off the UI thread with Async/Await (or Task.Run for CPU-bound),
' and report progress + support cancellation.
Private Async Sub btnProcess_Click(sender As Object, e As EventArgs) Handles btnProcess.Click
    Dim progress As IProgress(Of Integer) = New Progress(Of Integer)(
        Sub(pct) progressBar1.Value = pct)

    Await Task.Run(
        Sub()
            For i = 0 To 10000
                Process(i)
                progress.Report(CInt(i / 10000.0 * 100))
            Next
        End Sub)
End Sub
```

`Application.DoEvents` pumps the message queue re-entrantly — users can click buttons, close the form, or trigger timers while your loop is half-done. Replace with proper `Async/Await` + `IProgress(Of T)` + `CancellationToken`.

### Async Sub — Use Only for Event Handlers

`Async Sub` (the VB equivalent of C# `async void`) does not return a `Task`, so the caller cannot `Await` it and unhandled exceptions crash the process. **Use `Async Sub` only for WinForms event handlers.** Everywhere else, return `Task` or `Task(Of T)`.

```vb
' OK: event handler
Private Async Sub btnSave_Click(sender As Object, e As EventArgs) Handles btnSave.Click
    Try
        Await SaveAsync()
    Catch ex As Exception
        MessageBox.Show(ex.Message)
    End Try
End Sub

' BAD: fire-and-forget Async Sub as a general method
Public Async Sub StartBackgroundWork()    ' exceptions here will crash the app
    Await DoWorkAsync()
End Sub

' GOOD: return Task; let the caller decide how to handle it
Public Async Function StartBackgroundWorkAsync() As Task
    Await DoWorkAsync()
End Function
```

---

## Level 4+: Akka.NET Streams, Reactive Extensions, Actors

For advanced scenarios requiring stream processing, UI event composition, or stateful entity management, see [advanced-concurrency.md](references/advanced-concurrency.md).

**Akka.NET Streams** excel at server-side batching, throttling, and backpressure. **Reactive Extensions** are ideal for UI event composition in WinForms (search-as-you-type, double-click detection, auto-save). **Akka.NET Actors** handle entity-per-actor patterns, state machines with `Become()`, and distributed systems via Cluster Sharding.

---

## Anti-Patterns: What to Avoid

### SyncLock for Business Logic

```vb
' BAD: using SyncLock to protect shared state
Private ReadOnly _lock As New Object()
Private _orders As New Dictionary(Of String, Order)()

Public Sub UpdateOrder(id As String, update As Action(Of Order))
    SyncLock _lock
        Dim order As Order = Nothing
        If _orders.TryGetValue(id, order) Then update(order)
    End SyncLock
End Sub

' GOOD: use an actor or Channel to serialize access,
' or ConcurrentDictionary for simple key/value state.
```

### Manual Thread Management

```vb
' BAD: creating threads manually
Dim t As New Thread(AddressOf ProcessOrders)
t.Start()

' GOOD: Task.Run or better abstractions
' Keep a reference so exceptions are observed and cancellation can be coordinated.
' Note: Task.Run is unnecessary for already-async methods — just invoke it directly.
' (Wrapping an async method in Task.Run(Function() ...) returns Task(Of Task) and swallows exceptions.)
Dim backgroundTask As Task = ProcessOrdersAsync(cancellationToken)

' Elsewhere (form close or cancel):
Try
    Await backgroundTask
Catch ex As OperationCanceledException
    ' expected on cancel
Catch ex As Exception
    ' log / surface to user
End Try
```

Unobserved Task exceptions raise `TaskScheduler.UnobservedTaskException` and may crash on older runtimes. Always keep a reference and `Await` (or observe via `ContinueWith`) to handle failures.

### Blocking on Async Code (Deadlock Risk in WinForms!)

```vb
' BAD: blocking on async — classic WinForms deadlock
Dim result = GetDataAsync().Result        ' UI thread blocked;
                                          ' awaited continuation also wants the UI thread → deadlock
GetDataAsync().Wait()                     ' same problem

' GOOD: async all the way
Dim result = Await GetDataAsync()
```

### Shared Mutable State Without Protection

```vb
' BAD: multiple tasks mutating a plain List
Dim results As New List(Of Result)()
Await Parallel.ForEachAsync(items,
    Async Function(item, ct)
        Dim r = Await ProcessAsync(item, ct)
        results.Add(r)        ' Race condition!
    End Function)

' GOOD: ConcurrentBag (or collect Task results and combine at the end)
Dim results As New ConcurrentBag(Of Result)()
```

### Application.DoEvents in a Loop

See "Application.DoEvents is a Footgun" above.

---

## Quick Reference: Which Tool When?

| Need | Tool | Example |
|------|------|---------|
| Wait for I/O | `Async/Await` | HTTP calls, database queries, serial port reads |
| Parallel CPU work | `Parallel.ForEachAsync` | Image processing, calculations |
| Work queue | `Channel(Of T)` | Background job processing |
| Progress from background to UI | `IProgress(Of T)` + `Progress(Of T)` | Import/export progress bar |
| User cancels running operation | `CancellationTokenSource` + Cancel button | Long imports, network calls |
| Marshal to UI thread from worker | `Control.Invoke` / `BeginInvoke` | Serial-port `DataReceived`, timer callbacks |
| UI events with debounce/throttle | Reactive Extensions | Search-as-you-type, auto-save |
| Server-side batching/throttling | Akka.NET Streams | Event aggregation, rate limiting |
| State machines | Akka.NET Actors | Payment flows, order lifecycles |
| Entity state management | Akka.NET Actors | Order management, user sessions |
| Fire multiple async ops | `Task.WhenAll` | Loading dashboard data |
| Race multiple async ops | `Task.WhenAny` | Timeout with fallback |
| Periodic work | `PeriodicTimer` | Health checks, polling |

---

## The Escalation Path

```
Async/Await (start here)
    │
    ├─► Need progress reporting? → IProgress(Of T)
    │
    ├─► Need cancellation? → CancellationTokenSource + Cancel button
    │
    ├─► Need parallelism? → Parallel.ForEachAsync
    │
    ├─► Need producer/consumer? → Channel(Of T)
    │
    ├─► Need UI event composition? → Reactive Extensions
    │
    ├─► Need server-side stream processing? → Akka.NET Streams
    │
    └─► Need state machines or entity management? → Akka.NET Actors
```

**Only escalate when you have a concrete need.** Don't reach for actors or streams "just in case."

---

## VB.NET vs C# — Concurrency Cheat Sheet

| C# | VB.NET |
|---|---|
| `async Task DoAsync()` | `Async Function DoAsync() As Task` |
| `async Task<T> GetAsync()` | `Async Function GetAsync() As Task(Of T)` |
| `async void OnClick(...)` (event handler only) | `Async Sub OnClick(...)` (event handler only) |
| `await foo` | `Await foo` |
| `await Task.WhenAll(a, b)` | `Await Task.WhenAll(a, b)` |
| `CancellationToken ct` | `ct As CancellationToken` |
| `new CancellationTokenSource()` | `New CancellationTokenSource()` |
| `lock (_sync) { ... }` | `SyncLock _sync ... End SyncLock` |
| `Task.Run(() => ...)` | `Task.Run(Function() ...)` or `Task.Run(Sub() ... End Sub)` |
| `await foreach (var x in reader.ReadAllAsync(ct))` | **No `Await For Each`.** Use `WaitToReadAsync` + `TryRead` loop, or `GetAsyncEnumerator` + `MoveNextAsync`. |
| `yield return` in an async iterator | **C#-only.** Consume `IAsyncEnumerable(Of T)` in VB, don't author it. |
| `await using var x = ...` | **No `Await Using`.** Use `Using ... End Using` for sync `IDisposable`; for `IAsyncDisposable` wrap in `Try/Finally` and `Await x.DisposeAsync()`. |
| `ConfigureAwait(false)` | `.ConfigureAwait(False)` — same semantics |
| `ValueTask` / `ValueTask<T>` | `ValueTask` / `ValueTask(Of T)` — same semantics |
