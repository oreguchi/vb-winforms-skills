# Performance and API Design Patterns

Zero-allocation patterns with `Span(Of T)` / `Memory(Of T)` and API design principles for accepting and returning the right types, adapted for VB.NET.

## Contents

- [Span(Of T) and Memory(Of T) for Zero-Allocation Code](#spanof-t-and-memoryof-t-for-zero-allocation-code)
- [API Design Principles](#api-design-principles)
- [Method Signatures Best Practices](#method-signatures-best-practices)

> **VB.NET language notes applied throughout this file**
> - VB.NET **consumes** `Span(Of T)` / `Memory(Of T)` APIs but **cannot author `ref struct` types** or functions with `ByRef` returns / `In` / `ref readonly` parameters. If you need to define a new `ref struct`, do so in a C# project and reference it.
> - VB.NET has **no `stackalloc` expression.** Zero-allocation stack buffers must come from Span-based APIs the BCL exposes, or be allocated as arrays.
> - VB.NET has **no `[SkipLocalsInit]` expression syntax for locals**; the attribute can be applied at method level but the `stackalloc` scenario it optimizes does not exist here.
> - VB.NET **cannot author `IAsyncEnumerable(Of T)` iterators** (no `Async` + `Yield`). Consume them with a manual `GetAsyncEnumerator` + `While Await MoveNextAsync()` loop.
> - VB.NET has **no `Await For Each` / `Await Using` statement**. Use `Try` / `Finally` with `Await enumerator.DisposeAsync()` / `Await resource.DisposeAsync()`.
> - All samples assume `Option Strict On`.

## Span(Of T) and Memory(Of T) for Zero-Allocation Code

Use `Span(Of T)` and `Memory(Of T)` instead of `Byte()` or `String` for performance-critical code.

```vb
Imports System.Buffers
Imports System.IO
Imports System.Runtime.CompilerServices
Imports System.Text
Imports System.Threading
Imports System.Threading.Tasks

' Span(Of T) for synchronous, zero-allocation operations
' NOTE: Integer.Parse(ReadOnlySpan(Of Char)) requires .NET Core 2.1+ / .NET 5+.
' It is NOT available on .NET Framework.
Public Function ParseOrderId(input As ReadOnlySpan(Of Char)) As Integer
    ' Work with data without allocations
    If Not input.StartsWith("ORD-".AsSpan()) Then
        Throw New FormatException("Invalid order ID format")
    End If

    Dim numberPart = input.Slice(4)
    Return Integer.Parse(numberPart)
End Function

' VB.NET note: VB has no "stackalloc". The closest approach is calling Span-based
' BCL methods and letting them allocate a short buffer, or using ArrayPool (shown below).
Public Sub FormatMessage()
    Dim buffer(255) As Char
    Dim span = buffer.AsSpan()
    Dim written = FormatInto(span)
    Dim message = New String(span.Slice(0, written))
End Sub

' Memory(Of T) for async operations (Span cannot cross Await)
Public Async Function ReadDataAsync(
        buffer As Memory(Of Byte),
        cancellationToken As CancellationToken) As Task(Of Integer)
    Return Await _stream.ReadAsync(buffer, cancellationToken)
End Function

' String manipulation with Span to avoid allocations
Public Function TryParseKeyValue(line As ReadOnlySpan(Of Char),
                                 ByRef key As String,
                                 ByRef value As String) As Boolean
    key = String.Empty
    value = String.Empty

    Dim colonIndex = line.IndexOf(":"c)
    If colonIndex = -1 Then Return False

    ' Only allocate strings once we know the format is valid
    key = New String(line.Slice(0, colonIndex).Trim())
    value = New String(line.Slice(colonIndex + 1).Trim())
    Return True
End Function

' ArrayPool for temporary large buffers
Public Async Function ProcessLargeFileAsync(
        stream As Stream,
        cancellationToken As CancellationToken) As Task
    Dim buffer = ArrayPool(Of Byte).Shared.Rent(8192)
    Try
        Dim bytesRead As Integer
        bytesRead = Await stream.ReadAsync(buffer.AsMemory(), cancellationToken)
        While bytesRead > 0
            ProcessChunk(buffer.AsSpan(0, bytesRead))
            bytesRead = Await stream.ReadAsync(buffer.AsMemory(), cancellationToken)
        End While
    Finally
        ArrayPool(Of Byte).Shared.Return(buffer)
    End Try
End Function

' Hybrid buffer pattern for transient UTF-8 work using ArrayPool.
' VB.NET cannot use stackalloc, so the "small-buffer" path is also pool-based.
Public Shared Function GenerateHashCode(key As String) As Short
    If key Is Nothing Then Return 0

    Const StackLimit As Integer = 256

    Dim enc = Encoding.UTF8
    Dim max = enc.GetMaxByteCount(key.Length)

    Dim rented As Byte() = ArrayPool(Of Byte).Shared.Rent(Math.Max(max, StackLimit))
    Try
        Dim buf = rented.AsSpan(0, max)
        Dim written = enc.GetBytes(key.AsSpan(), buf)
        Dim h1, h2 As UInteger
        ComputeHash(buf.Slice(0, written), h1, h2)
        Return CShort(CInt(h1 Xor h2) And &HFFFF)
    Finally
        ArrayPool(Of Byte).Shared.Return(rented)
    End Try
End Function

' Span-based parsing without substring allocations
Public Shared Function ParseUrl(url As ReadOnlySpan(Of Char)) _
        As (Protocol As String, Host As String, Port As Integer)
    Dim protocolEnd = url.IndexOf("://".AsSpan())
    Dim protocol = New String(url.Slice(0, protocolEnd))

    Dim afterProtocol = url.Slice(protocolEnd + 3)
    Dim portStart = afterProtocol.IndexOf(":"c)

    Dim host = New String(afterProtocol.Slice(0, portStart))
    Dim portSpan = afterProtocol.Slice(portStart + 1)
    Dim port = Integer.Parse(portSpan)

    Return (protocol, host, port)
End Function

' Writing data to Span
Public Function TryFormatOrderId(orderId As Integer,
                                 destination As Span(Of Char),
                                 ByRef charsWritten As Integer) As Boolean
    Const prefix As String = "ORD-"

    If destination.Length < prefix.Length + 10 Then
        charsWritten = 0
        Return False
    End If

    prefix.AsSpan().CopyTo(destination)
    Dim numberChars As Integer
    Dim numberWritten = orderId.TryFormat(destination.Slice(prefix.Length), numberChars)

    charsWritten = prefix.Length + numberChars
    Return numberWritten
End Function
```

> **VB.NET note — authoring Span-backed types:** C# can declare new `ref struct` types, methods with `ref readonly` / `in` parameters, and functions that return `ref`. VB.NET cannot **author** any of these. VB.NET can **consume** `Span(Of T)`, `ReadOnlySpan(Of T)`, and `Memory(Of T)` parameters and return values freely. If you need to build a new zero-allocation primitive, write it in C# and reference the assembly.

**When to use what:**

| Type | Use Case |
|------|----------|
| `Span(Of T)` | Synchronous operations, slicing without allocation. VB consumes existing APIs; no `stackalloc` available. |
| `ReadOnlySpan(Of T)` | Read-only views, method parameters for data you won't modify |
| `Memory(Of T)` | Async operations (Span cannot cross `Await` boundaries) |
| `ReadOnlyMemory(Of T)` | Read-only async operations |
| `Byte()` | When you need to store data long-term or pass to APIs requiring arrays |
| `ArrayPool(Of T)` | Large temporary buffers (>1KB) to avoid GC pressure |

## `SkipLocalsInit` and `stackalloc` — Not Applicable to VB.NET

C# uses `[SkipLocalsInit]` to skip the default zero-initialization of locals (the `.locals init` flag) when combined with `stackalloc`, trading safety for a measurable speed-up in tight loops that write a stack buffer before reading it.

VB.NET cannot author any of that:

- **No `stackalloc`.** VB.NET has no `stackalloc` expression and no way to allocate an uninitialized stack buffer from source. Every VB local that looks like a buffer — e.g. `Dim b(255) As Byte` or an `AsSpan()` over an array — is a managed heap allocation (or an `ArrayPool` rental) that is already zero-initialized by the runtime.
- **`<SkipLocalsInitAttribute>` is a no-op for VB-authored code.** The attribute compiles when applied at method or module level (`<Module: SkipLocalsInit>`), but because VB never emits the uninitialized stack-allocation patterns that benefit from it, it has no practical effect on VB-authored methods.
- **If you need uninitialized buffers, author the helper in C#.** Build the hot path as a `static` method on a C# class marked `[SkipLocalsInit]` with an internal `stackalloc`, expose it as a normal method taking / returning `Span(Of T)` / `ReadOnlySpan(Of T)`, and consume it from VB. The VB caller gets the performance without needing language features VB does not have.

```vb
' VB side - consume a C#-authored helper that uses [SkipLocalsInit] + stackalloc internally.
' The helper's signature is a plain Span(Of Char) API that VB can call directly.
Public Function FormatOrderIdFast(orderId As Integer) As String
    Dim buffer(63) As Char                          ' heap-allocated, zero-initialized
    Dim written = NativeFormatters.FormatOrderId(   ' defined in a C# library
        orderId,
        buffer.AsSpan())
    Return New String(buffer, 0, written)
End Function
```

The bottom line: treat `SkipLocalsInit` as a C#-side optimization. Do not add `<SkipLocalsInitAttribute>` to VB methods expecting a perf win — there is none.

## API Design Principles

### Accept Abstractions, Return Appropriately Specific

**For Parameters (Accept):**

```vb
' Accept IEnumerable(Of T) if you only iterate once
Public Function CalculateTotal(items As IEnumerable(Of OrderItem)) As Decimal
    Return items.Sum(Function(item) item.Price * item.Quantity)
End Function

' Accept IReadOnlyCollection(Of T) if you need Count
Public Function HasMinimumItems(items As IReadOnlyCollection(Of OrderItem), minimum As Integer) As Boolean
    Return items.Count >= minimum
End Function

' Accept IReadOnlyList(Of T) if you need indexing
Public Function GetMiddleItem(items As IReadOnlyList(Of OrderItem)) As OrderItem
    If items.Count = 0 Then
        Throw New ArgumentException("List cannot be empty")
    End If
    Return items(items.Count \ 2)   ' Indexed access (integer division)
End Function

' Accept ReadOnlySpan(Of T) for high-performance, zero-allocation APIs
Public Function Sum(numbers As ReadOnlySpan(Of Integer)) As Integer
    Dim total As Integer = 0
    For Each num In numbers
        total += num
    Next
    Return total
End Function

' Accept IAsyncEnumerable(Of T) for async streaming.
' VB.NET has no "Await For Each"; use the manual enumerator pattern.
Public Async Function CountItemsAsync(
        orders As IAsyncEnumerable(Of Order),
        cancellationToken As CancellationToken) As Task(Of Integer)
    Dim count As Integer = 0
    Dim enumerator = orders.WithCancellation(cancellationToken).GetAsyncEnumerator()
    Try
        While Await enumerator.MoveNextAsync()
            count += 1
        End While
    Finally
        Await enumerator.DisposeAsync()
    End Try
    Return count
End Function
```

**For Return Types:**

```vb
' Lazy/deferred execution via Iterator Function (synchronous only in VB.NET)
Public Iterator Function GetOrdersLazy(customerId As String) As IEnumerable(Of Order)
    For Each order In _repository.Query()
        If order.CustomerId = customerId Then
            Yield order    ' Lazy evaluation
        End If
    Next
End Function

' Return IReadOnlyList(Of T) for materialized, immutable collections
Public Function GetOrders(customerId As String) As IReadOnlyList(Of Order)
    Return _repository _
        .Query() _
        .Where(Function(o) o.CustomerId = customerId) _
        .ToList()                 ' Materialized
End Function

' Return concrete types when callers need mutation
Public Function GetMutableOrders(customerId As String) As List(Of Order)
    ' Explicitly allow mutation by returning List(Of T)
    Return _repository _
        .Query() _
        .Where(Function(o) o.CustomerId = customerId) _
        .ToList()
End Function

' Returning IAsyncEnumerable(Of T) - VB.NET CANNOT author async iterators
' (no Async + Yield). If you need to return an IAsyncEnumerable, implement the
' iterator in a C# class library and expose it to VB, or return a Task-of-list
' instead.
Public Function StreamOrdersAsync(customerId As String) As IAsyncEnumerable(Of Order)
    ' Pseudo-code: in a real project this iterator would be authored in C#
    ' (VB lacks Async Iterator Functions, so the filter "WhereCustomerAsync"
    ' below must live in a C# helper assembly that this VB project references).
    Throw New NotImplementedException(
        "Async iterator filter must be authored in C# — VB cannot author async iterators.")
End Function

' Return arrays for interop or when caller expects array
Public Function SerializeOrder(order As Order) As Byte()
    ' Binary serialization - Byte() is appropriate here
    Return MessagePackSerializer.Serialize(order)
End Function
```

> **VB.NET note — async iterators:** `IAsyncEnumerable(Of T)` *authorship* requires combining `Async` with `Yield`, which VB.NET does not support. If you cannot move authoring to C#, return `Task(Of IReadOnlyList(Of T))` (materialized) or `ChannelReader(Of T)` (streaming via `System.Threading.Channels`).

**Summary Table:**

| Scenario | Accept | Return |
|----------|--------|--------|
| Only iterate once | `IEnumerable(Of T)` | `IEnumerable(Of T)` (via `Iterator Function`) |
| Need count | `IReadOnlyCollection(Of T)` | `IReadOnlyCollection(Of T)` |
| Need indexing | `IReadOnlyList(Of T)` | `IReadOnlyList(Of T)` |
| High-performance, sync | `ReadOnlySpan(Of T)` | `Span(Of T)` (rarely) |
| Async streaming | `IAsyncEnumerable(Of T)` | `IAsyncEnumerable(Of T)` (author in C#) / `ChannelReader(Of T)` |
| Caller needs mutation | - | `List(Of T)`, `T()` |

## Method Signatures Best Practices

```vb
' Complete async method signature
Public Async Function CreateOrderAsync(
        request As CreateOrderRequest,
        Optional cancellationToken As CancellationToken = Nothing) As Task(Of Result(Of Order, OrderError))
    ' Implementation
End Function

' Optional parameters at the end
Public Async Function GetOrdersAsync(
        customerId As String,
        Optional startDate As Date? = Nothing,
        Optional endDate As Date? = Nothing,
        Optional cancellationToken As CancellationToken = Nothing) As Task(Of List(Of Order))
    ' Implementation
End Function

' Use a small immutable Class for multiple related parameters
' (VB.NET equivalent of a C# record: ReadOnly properties assigned via constructor).
Public Class SearchOrdersRequest
    Public ReadOnly Property CustomerId As String
    Public ReadOnly Property StartDate As Date?
    Public ReadOnly Property EndDate As Date?
    Public ReadOnly Property Status As OrderStatus?
    Public ReadOnly Property PageSize As Integer
    Public ReadOnly Property PageNumber As Integer

    Public Sub New(customerId As String,
                   startDate As Date?,
                   endDate As Date?,
                   status As OrderStatus?,
                   Optional pageSize As Integer = 20,
                   Optional pageNumber As Integer = 1)
        Me.CustomerId = customerId
        Me.StartDate = startDate
        Me.EndDate = endDate
        Me.Status = status
        Me.PageSize = pageSize
        Me.PageNumber = pageNumber
    End Sub
End Class

Public Async Function SearchOrdersAsync(
        request As SearchOrdersRequest,
        Optional cancellationToken As CancellationToken = Nothing) As Task(Of PagedResult(Of Order))
    ' Implementation
End Function

' Primary constructors - VB.NET has no primary constructor syntax.
' Declare the fields and a regular "Sub New" instead.
Public NotInheritable Class OrderService
    Private ReadOnly _repository As IOrderRepository
    Private ReadOnly _logger As ILogger(Of OrderService)

    Public Sub New(repository As IOrderRepository, logger As ILogger(Of OrderService))
        _repository = repository
        _logger = logger
    End Sub

    Public Async Function GetOrderAsync(orderId As OrderId,
                                        cancellationToken As CancellationToken) As Task(Of Order)
        _logger.LogInformation("Fetching order {OrderId}", orderId)
        Return Await _repository.GetAsync(orderId, cancellationToken)
    End Function
End Class

' Options pattern for complex configuration.
' VB.NET has no "required" keyword and no "init"-only setters. Use a constructor
' for required values and normal "Property" setters for the rest, or validate
' required values at startup via IValidateOptions.
Public NotInheritable Class EmailServiceOptions
    Public Property SmtpHost As String
    Public Property SmtpPort As Integer = 587
    Public Property UseSsl As Boolean = True
    Public Property Timeout As TimeSpan = TimeSpan.FromSeconds(30)
End Class

Public NotInheritable Class EmailService
    Private ReadOnly _options As EmailServiceOptions

    Public Sub New(options As IOptions(Of EmailServiceOptions))
        _options = options.Value
    End Sub
End Class
```
