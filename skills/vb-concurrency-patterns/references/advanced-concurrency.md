# Advanced Concurrency Patterns (VB.NET)

Akka.NET Streams, Reactive Extensions, Akka.NET Actors, and async local function patterns for advanced concurrency scenarios — translated to VB.NET. Most line-of-business WinForms apps will not need anything in this file; it is here for completeness when a scenario genuinely calls for it.

## Contents

- [Akka.NET Streams (Complex Stream Processing)](#akkanet-streams-complex-stream-processing)
- [Reactive Extensions (UI and Event Composition)](#reactive-extensions-ui-and-event-composition)
- [Akka.NET Actors (Stateful Concurrency)](#akkanet-actors-stateful-concurrency)
- [Prefer Async Local Functions](#prefer-async-local-functions)

## Akka.NET Streams (Complex Stream Processing)

**Use for:** Backpressure, batching, debouncing, throttling, merging streams, complex transformations.

```vb
Imports Akka.Streams
Imports Akka.Streams.Dsl

' Batching with timeout
Public Function BatchEvents(
    events As Source(Of [Event], NotUsed)) As Source(Of IReadOnlyList(Of [Event]), NotUsed)

    Return events _
        .GroupedWithin(100, TimeSpan.FromSeconds(1)) _
        .Select(Function(batch) CType(batch.ToList(), IReadOnlyList(Of [Event])))
End Function

' Throttling
Public Function ThrottleRequests(
    requests As Source(Of Request, NotUsed)) As Source(Of Request, NotUsed)

    Return requests.Throttle(10, TimeSpan.FromSeconds(1), 5, ThrottleMode.Shaping)
End Function

' Parallel processing with ordered results
Public Function ProcessWithParallelism(
    items As Source(Of Item, NotUsed)) As Source(Of ProcessedItem, NotUsed)

    Return items.SelectAsync(4,
        Async Function(item) As Task(Of ProcessedItem)
            Return Await ProcessAsync(item)
        End Function)
End Function

' Complex pipeline
Public Function CreatePipeline(
    events As Source(Of RawEvent, NotUsed),
    sink As Sink(Of ProcessedEvent, Task(Of Done))) As IRunnableGraph(Of Task(Of Done))

    Return events _
        .Where(Function(e) e.IsValid) _
        .GroupedWithin(50, TimeSpan.FromMilliseconds(500)) _
        .SelectAsync(4, Function(batch) ProcessBatchAsync(batch)) _
        .SelectMany(Function(results) results) _
        .ToMaterialized(sink, Keep.Right)
End Function
```

**Akka.NET Streams excel at:**
- Batching with size AND time limits
- Throttling and rate limiting
- Backpressure that propagates through the entire pipeline
- Merging/splitting streams
- Parallel processing with ordering guarantees
- Error handling with supervision

## Reactive Extensions (UI and Event Composition)

**Use for:** UI event handling, composing event streams, time-based operations in client applications. This is the advanced-concurrency tool most relevant to WinForms.

Rx shines in UI scenarios where you need to react to user events with debouncing, throttling, or combining multiple event sources. `Observable.FromEventPattern` turns any `Control` event (`TextChanged`, `MouseClick`, etc.) into an `IObservable`.

```vb
Imports System.Reactive.Linq
Imports System.Threading

' Search-as-you-type with debouncing (wire up in the Form's Load)
Private _searchSubscription As IDisposable

Private Sub MainForm_Load(sender As Object, e As EventArgs) Handles MyBase.Load
    Dim textChanged = Observable.FromEventPattern(
        Sub(h) AddHandler txtSearch.TextChanged, h,
        Sub(h) RemoveHandler txtSearch.TextChanged, h)

    _searchSubscription = textChanged _
        .Select(Function(_u) txtSearch.Text) _
        .Throttle(TimeSpan.FromMilliseconds(300)) _
        .DistinctUntilChanged() _
        .Where(Function(text) text.Length >= 3) _
        .SelectMany(Function(text) _searchService.SearchAsync(text).ToObservable()) _
        .ObserveOn(SynchronizationContext.Current) _
        .Subscribe(Sub(results) ShowResults(results))
End Sub

Private Sub MainForm_FormClosed(sender As Object, e As FormClosedEventArgs) Handles MyBase.FormClosed
    _searchSubscription?.Dispose()
End Sub

' Double-click detection
Public ReadOnly Property DoubleClicks As IObservable(Of Point)
    Get
        Return MouseClicks _
            .Buffer(TimeSpan.FromMilliseconds(300)) _
            .Where(Function(clicks) clicks.Count >= 2) _
            .Select(Function(clicks) clicks.Last())
    End Get
End Property

' Auto-save with debouncing
Public ReadOnly Property AutoSave As IDisposable
    Get
        Return DocumentChanges _
            .Throttle(TimeSpan.FromSeconds(2)) _
            .SelectMany(Function(doc) Observable.FromAsync(Function() SaveAsync(doc))) _
            .Subscribe(
                Sub(__) LogInfo("Saved"),
                Sub(ex) LogError(ex))
    End Get
End Property
```

To compose async work inside an observable, use `Observable.FromAsync` with `SelectMany` — passing an `Async Function` directly to `Subscribe` is the same footgun as `Async Sub`: exceptions are swallowed.

**Rx is ideal for:**
- UI event composition (WinForms, WPF, MAUI, Blazor)
- Search-as-you-type with debouncing
- Combining multiple event sources
- Time-windowed operations in UI
- Drag-and-drop gesture detection
- Real-time data visualization

**Rx vs Akka.NET Streams:**

| Scenario | Rx | Akka.NET Streams |
|----------|----|--------------------|
| UI events | Best choice | Overkill |
| Client-side composition | Best choice | Overkill |
| Server-side pipelines | Works but limited | Better backpressure |
| Distributed processing | Not designed for | Built for this |
| Hot observables | Native support | Requires more setup |

**Rule of thumb:** Rx for UI/client (WinForms), Akka.NET Streams for server-side pipelines.

## Akka.NET Actors (Stateful Concurrency)

**Use for:** Managing state for multiple entities, state machines, push-based updates, complex coordination, supervision and fault tolerance. Unusual in a WinForms LOB app but not impossible — e.g. a plant-floor controller that tracks dozens of robots each with their own lifecycle.

### Entity-Per-Actor Pattern

```vb
' One actor per entity — each order has isolated state.
Public Class OrderActor
    Inherits ReceiveActor

    Private _state As OrderState

    Public Sub New(orderId As String)
        _state = New OrderState(orderId)

        Receive(Of AddItem)(
            Sub(msg)
                _state = _state.AddItem(msg.Item)
                Sender.Tell(New ItemAdded(msg.Item))
            End Sub)

        Receive(Of Checkout)(
            Sub(msg)
                If _state.CanCheckout Then
                    _state = _state.Checkout()
                    Sender.Tell(New CheckoutSucceeded(_state.Total))
                Else
                    Sender.Tell(New CheckoutFailed("Cart is empty"))
                End If
            End Sub)

        Receive(Of GetState)(Sub(__) Sender.Tell(_state))
    End Sub
End Class
```

### State Machines with Become

Actors excel at implementing state machines using `Become()` to switch message handlers:

```vb
Public Class PaymentActor
    Inherits ReceiveActor

    Private _payment As PaymentData

    Public Sub New(paymentId As String)
        _payment = New PaymentData(paymentId)
        Pending()
    End Sub

    Private Sub Pending()
        Receive(Of AuthorizePayment)(
            Sub(msg)
                _payment = _payment.WithAmount(msg.Amount)   ' VB: manual copy; no "with" expression
                Become(AddressOf Authorizing)
                Self.Tell(New ProcessAuthorization())
            End Sub)

        Receive(Of CancelPayment)(
            Sub(__)
                Become(AddressOf Cancelled)
                Sender.Tell(New PaymentCancelled(_payment.Id))
            End Sub)
    End Sub

    Private Sub Authorizing()
        ' Use ReceiveAsync for handlers that Await — plain Receive takes Action(Of T)
        ' and would make async work fire-and-forget on the actor dispatcher.
        ReceiveAsync(Of ProcessAuthorization)(
            Async Function(__) As Task
                Dim result = Await _gateway.AuthorizeAsync(_payment)
                If result.Success Then
                    _payment = _payment.WithAuthCode(result.AuthCode)
                    Become(AddressOf Authorized)
                Else
                    Become(AddressOf Failed)
                End If
            End Function)

        Receive(Of CancelPayment)(
            Sub(__)
                Sender.Tell(New PaymentError("Cannot cancel during authorization"))
            End Sub)
    End Sub

    Private Sub Authorized()
        Receive(Of CapturePayment)(
            Sub(__)
                Become(AddressOf Capturing)
                Self.Tell(New ProcessCapture())
            End Sub)

        Receive(Of VoidPayment)(
            Sub(__)
                Become(AddressOf Voiding)
                Self.Tell(New ProcessVoid())
            End Sub)
    End Sub

    Private Sub Capturing()
        ' ...
    End Sub

    Private Sub Voiding()
        ' ...
    End Sub

    Private Sub Cancelled()
        ' Only responds to GetState
    End Sub

    Private Sub Failed()
        ' Only responds to GetState, Retry
    End Sub
End Class
```

**VB.NET note:** C# uses records + `with` expressions to copy state immutably (`_payment with { Amount = msg.Amount }`). VB.NET has no `with` expression and no `record` type. Replicate the effect with an immutable `Class` or `Structure` and explicit `WithXxx` methods that return a new instance, or with a regular constructor.

### Distributed Entities with Cluster Sharding

```vb
builder.WithShardRegion(Of OrderActor)(
    typeName:="orders",
    entityPropsFactory:=Function(__1, __2, resolver)
                            Return Function(orderId) Props.Create(Function() New OrderActor(orderId))
                        End Function,
    messageExtractor:=New OrderMessageExtractor(),
    shardOptions:=New ShardOptions())

Dim orderRegion = registry.Get(Of OrderActor)()
orderRegion.Tell(New ShardingEnvelope("order-123", New AddItem(item)))
```

### When to Use Akka.NET

**Use Akka.NET Actors when you have:**

| Scenario | Why Actors? |
|----------|-------------|
| Many entities with independent state | Each entity gets its own actor — no locks |
| State machines | `Become()` elegantly models state transitions |
| Push-based/reactive updates | Actors naturally support tell-don't-ask |
| Supervision requirements | Parent actors supervise children, auto restart |
| Distributed systems | Cluster Sharding distributes across nodes |
| Long-running workflows | Actors + persistence = durable workflows |
| Real-time systems | Message-driven, non-blocking by design |
| IoT / device management | Each device = one actor, scales to millions |

**Don't use Akka.NET when:**

| Scenario | Better Alternative |
|----------|-------------------|
| Simple work queue | `Channel(Of T)` |
| Request/response API | `Async/Await` |
| Batch processing | `Parallel.ForEachAsync` or Akka.NET Streams |
| UI event handling | Reactive Extensions |
| CRUD operations | Standard async services |

### The Actor Mindset

Think of actors when your problem looks like:
- "I have **thousands** of [orders/users/devices] that need independent state"
- "Each entity goes through a **lifecycle** with different behaviors at each stage"
- "I need to **push updates** to interested parties when something changes"
- "If processing fails, I want to **restart** just that entity"
- "This needs to work across **multiple servers**"

## Prefer Async Local Functions

VB.NET does not have C# **local functions** — functions declared inside another function. The closest idiom is a `private` helper method on the containing class. Use a named helper instead of `Task.Run(Async Function() ...)` or `ContinueWith()`:

### Don't: Anonymous Async Lambda

```vb
Private Sub HandleCommand(cmd As MyCommand)
    Dim self = Self

    Dim _ = Task.Run(
        Async Function()
            Dim result = Await DoWorkAsync()
            Return New WorkCompleted(result)
        End Function).PipeTo(self)
End Sub
```

### Do: Named Private Async Helper

```vb
Private Sub HandleCommand(cmd As MyCommand)
    ExecuteAsync().PipeTo(Self)
End Sub

Private Async Function ExecuteAsync() As Task(Of WorkCompleted)
    Dim result = Await DoWorkAsync()
    Return New WorkCompleted(result)
End Function
```

### Avoid ContinueWith for Sequencing

**Don't:**
```vb
someTask _
    .ContinueWith(Function(t) ProcessResult(t.Result)) _
    .ContinueWith(Function(t) SendNotification(t.Result))
```

**Do:**
```vb
Private Async Function ProcessAndNotifyAsync() As Task
    Dim result = Await someTask
    Dim processed = Await ProcessResult(result)
    Await SendNotification(processed)
End Function

' Caller — Await the Task so exceptions surface instead of being lost.
Await ProcessAndNotifyAsync()
```

### Akka.NET Example

When using `PipeTo` in actors, a named helper keeps the pattern clean:

```vb
Private Sub HandleSync(cmd As StartSync)
    PerformSyncAsync(cmd).PipeTo(Self)
End Sub

Private Async Function PerformSyncAsync(cmd As StartSync) As Task(Of SyncResult)
    Dim scope = _scopeFactory.CreateAsyncScope()
    Try
        Dim service = scope.ServiceProvider.GetRequiredService(Of ISyncService)()
        Dim count = Await service.SyncAsync(cmd.EntityId)
        Return New SyncResult(cmd.EntityId, count)
    Finally
        Await scope.DisposeAsync()    ' VB has no "Await Using"; emulate with Try/Finally.
    End Try
End Function
```

| Benefit | Description |
|---------|-------------|
| **Readability** | Named functions are self-documenting |
| **Debugging** | Stack traces show meaningful function names |
| **Exception handling** | Cleaner Try/Catch without `AggregateException` |
| **Scope clarity** | Explicit parameters make captured state obvious |
| **Testability** | Easier to extract and unit test the async logic |
