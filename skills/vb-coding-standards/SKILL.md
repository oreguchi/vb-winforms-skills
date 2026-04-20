---
name: vb-coding-standards
description: "Write modern VB.NET code using idiomatic language features (classes, structures, async/await, extension methods, LINQ). Covers VB.NET equivalents or alternatives where C# has features VB does not (records, pattern matching, init-only setters, with expressions, primary constructors, ref structs)."
compatibility: "Applies to VB.NET projects on .NET or .NET Framework. A few examples (Span(Of Char) overloads of Integer.Parse, UnsafeAccessorAttribute) require .NET 6+ / .NET 8+ and are called out inline."
invocable: false
---

# Modern VB.NET Coding Standards

## When to Use This Skill

Use this skill when:
- Writing new VB.NET code or refactoring existing code
- Designing public APIs for libraries or services
- Implementing domain models with strong typing
- Building async/await-heavy applications
- Working with binary data, buffers, or high-throughput scenarios

## Reference Files

- [references/value-objects-and-patterns.md](references/value-objects-and-patterns.md): Full value object examples and alternative patterns
- [references/performance-and-api-design.md](references/performance-and-api-design.md): Memory/buffer examples and API design principles
- [references/composition-and-error-handling.md](references/composition-and-error-handling.md): Composition over inheritance, Result type, testing patterns
- [references/anti-patterns-and-reflection.md](references/anti-patterns-and-reflection.md): Reflection avoidance and common anti-patterns

## Core Principles

1. **Immutability by Default** — Use `ReadOnly Property` and constructor-initialized fields; VB.NET does not have `record` types or `init`-only setters, but the same intent is achievable with `Public ReadOnly Property` set through a constructor.
2. **Type Safety** — Declare explicit types; use `Nullable(Of T)` / `T?` and `If(x, y)` null-coalescing to handle nulls explicitly.
3. **Conditional Branching** — Use `Select Case` for multi-branch dispatch; `If TypeOf x Is T Then` / `TryCast` for type-based branching. VB.NET does not support C#-style `switch` expressions or property patterns.
4. **Async Everywhere** — Prefer `Async Function` with `CancellationToken` for all I/O.
5. **Performance Awareness** — Consume `Span(Of T)` / `Memory(Of T)` APIs where available; VB.NET cannot *author* `ByRef`-struct types.
6. **API Design** — Accept abstractions (`IEnumerable(Of T)`, `IReadOnlyList(Of T)`), return appropriately specific types.
7. **Composition Over Inheritance** — Avoid deep abstract base class chains; prefer interfaces and composition.
8. **Value Objects as Structures** — Use `Public Structure` with `ReadOnly` fields for small value types; implement `Equals` / `GetHashCode` manually.
9. **Strict Compilation** — Set `Option Strict On` and `Option Explicit On` at the file or project level to enforce compile-time type checking and catch implicit conversions early.

---

## Language Patterns

### Immutable Data Objects (VB.NET equivalent of C# records)

C# has `record` and `record struct` types that provide value equality, `with` expressions, and positional constructors automatically. **VB.NET does not have `record` types.** Use a `Class` with `ReadOnly Property` members and a constructor to achieve the same intent.

```vb
' Simple immutable DTO — equivalent to C# "record CustomerDto(string Id, string Name, string Email)"
Public Class CustomerDto
    Public ReadOnly Property Id As String
    Public ReadOnly Property Name As String
    Public ReadOnly Property Email As String

    Public Sub New(id As String, name As String, email As String)
        Me.Id = id
        Me.Name = name
        Me.Email = email
    End Sub
End Class

' Immutable class with validation in constructor
Public Class EmailAddress
    Public ReadOnly Property Value As String

    Public Sub New(value As String)
        If String.IsNullOrWhiteSpace(value) OrElse Not value.Contains("@"c) Then
            Throw New ArgumentException("Invalid email address", NameOf(value))
        End If
        Me.Value = value
    End Sub
End Class

' Immutable class with a computed property
Public Class ShoppingCart
    Public ReadOnly Property CartId As String
    Public ReadOnly Property CustomerId As String
    Public ReadOnly Property Items As IReadOnlyList(Of CartItem)

    Public Sub New(cartId As String, customerId As String, items As IReadOnlyList(Of CartItem))
        Me.CartId = cartId
        Me.CustomerId = customerId
        Me.Items = items
    End Sub

    Public ReadOnly Property Total As Decimal
        Get
            Return Items.Sum(Function(item) item.Price * item.Quantity)
        End Get
    End Property
End Class
```

> **C#-only feature — `with` expressions:** C# records support `with { Property = newValue }` to create a shallow copy with one or more fields changed. VB.NET has no equivalent syntax. Implement a copy helper method manually:

```vb
' Manual "with"-style copy helper
Public Function WithEmail(newEmail As String) As CustomerDto
    Return New CustomerDto(Me.Id, Me.Name, newEmail)
End Function
```

> **C#-only feature — `init`-only setters:** C# supports `{ get; init; }` for properties that may only be set during object initialization. VB.NET has no `init` accessor. Use constructor-assigned `ReadOnly Property` instead (shown above).

---

### Value Objects as Structures

For small value types, use `Public Structure` with `ReadOnly` fields. Value equality must be implemented manually by overriding `Equals` and `GetHashCode` — there is no automatic structural equality as with C# `readonly record struct`.

```vb
Public Structure OrderId
    Public ReadOnly Property Value As String

    Public Sub New(value As String)
        If String.IsNullOrWhiteSpace(value) Then
            Throw New ArgumentException("OrderId cannot be empty", NameOf(value))
        End If
        Me.Value = value
    End Sub

    Public Shared Function [New]() As OrderId
        Return New OrderId(Guid.NewGuid().ToString())
    End Function

    Public Overrides Function ToString() As String
        Return Value
    End Function

    Public Overrides Function Equals(obj As Object) As Boolean
        If TypeOf obj Is OrderId Then
            Return DirectCast(obj, OrderId).Value = Me.Value
        End If
        Return False
    End Function

    Public Overrides Function GetHashCode() As Integer
        Return If(Value, String.Empty).GetHashCode()
    End Function
End Structure

Public Structure Money
    Public ReadOnly Property Amount As Decimal
    Public ReadOnly Property Currency As String

    Public Sub New(amount As Decimal, currency As String)
        Me.Amount = amount
        Me.Currency = currency
    End Sub
End Structure

Public Structure CustomerId
    Public ReadOnly Property Value As Guid

    Public Sub New(value As Guid)
        Me.Value = value
    End Sub

    Public Shared Function [New]() As CustomerId
        Return New CustomerId(Guid.NewGuid())
    End Function
End Structure
```

> **C#-only feature — `readonly record struct`:** C# `readonly record struct` gives structural equality, `with` support, and a positional constructor for free. VB.NET `Structure` provides value-type semantics but requires manual `Equals`/`GetHashCode` for proper value equality.

---

### Conditional Branching (VB.NET equivalent of C# pattern matching)

C# offers `switch` expressions, property patterns (`{ Prop: value }`), type patterns, and relational patterns. **VB.NET does not have these constructs.** Use `Select Case`, `If TypeOf x Is T Then`, and `TryCast` instead.

```vb
' Multi-branch dispatch — equivalent to C# switch expression on a value
Public Function CalculateDiscount(order As Order) As Decimal
    If order.Total > 1000D Then
        Return order.Total * 0.15D
    ElseIf order.Total > 500D Then
        Return order.Total * 0.10D
    ElseIf order.Total > 100D Then
        Return order.Total * 0.05D
    Else
        Return 0D
    End If
End Function

' Select Case for simple value-based dispatch
Public Function DescribeStatus(status As OrderStatus) As String
    Select Case status
        Case OrderStatus.Draft
            Return "Draft order"
        Case OrderStatus.Submitted
            Return "Submitted, awaiting processing"
        Case OrderStatus.Completed
            Return "Order completed"
        Case Else
            Return "Unknown status"
    End Select
End Function

' Type-based dispatch — equivalent to C# type pattern in switch
Public Function GetLabel(obj As Object) As String
    If TypeOf obj Is Order Then
        Dim order = DirectCast(obj, Order)
        Return $"Order {order.Id}"
    ElseIf TypeOf obj Is Customer Then
        Dim customer = DirectCast(obj, Customer)
        Return $"Customer {customer.Name}"
    Else
        Return "Unknown"
    End If
End Function

' Null-aware dispatch using TryCast
Public Function GetDiscount(customer As Customer) As Decimal
    If customer Is Nothing Then Return 0D
    If customer.IsVip Then Return 0.2D
    If customer.OrderCount > 10 Then Return 0.1D
    Return 0.05D
End Function
```

> **C#-only features:** C# property patterns (`{ IsVip: true }`), list patterns (`[first, ..]`), and `switch` expressions with `=>` arms are not available in VB.NET. The `If`/`ElseIf` and `Select Case` patterns above are the idiomatic VB.NET replacements.

---

### Null Handling

```vb
' Null-conditional — same syntax as C#
Dim name = user?.Name

' Null-coalescing — use two-argument If() instead of ??
Dim displayName = If(user?.Name, "Anonymous")

' ArgumentNullException guard
Public Sub ProcessOrder(order As Order)
    If order Is Nothing Then Throw New ArgumentNullException(NameOf(order))
    Console.WriteLine(order.Id)
End Sub

' Nullable value type
Dim count As Integer? = Nothing
Dim actual = If(count, 0)   ' returns 0 when count is Nothing
```

---

### Collections and Collection Initialization

```vb
' Array literal — VB equivalent of C# collection expression [1, 2, 3]
Dim numbers = {1, 2, 3}

' List initialization — equivalent to C# new List<int> { 1, 2, 3 }
Dim list As New List(Of Integer) From {1, 2, 3}

' Dictionary initialization
Dim map As New Dictionary(Of String, Integer) From {
    {"one", 1},
    {"two", 2}
}

' IReadOnlyList accepted from caller — prefer abstractions
Public Function ProcessItems(items As IReadOnlyList(Of OrderItem)) As Decimal
    Return items.Sum(Function(i) i.Price)
End Function
```

> **C#-only feature — collection expressions:** C# 12 introduced `[1, 2, 3]` collection expressions that can target arrays, lists, and spans via spread syntax. VB.NET uses `{1, 2, 3}` for array literals and `New List(Of T) From {…}` for lists.

---

### Async/Await

Async/await is fully supported in VB.NET.

```vb
' Async all the way — always accept CancellationToken
Public Async Function GetOrderAsync(orderId As String, cancellationToken As CancellationToken) As Task(Of Order)
    Dim order = Await _repository.GetAsync(orderId, cancellationToken)
    Return order
End Function

' ValueTask for frequently-called, often-synchronous methods
Public Function GetCachedOrderAsync(orderId As String, cancellationToken As CancellationToken) As ValueTask(Of Order)
    Dim cached As Order = Nothing
    If _cache.TryGetValue(orderId, cached) Then
        Return New ValueTask(Of Order)(cached)
    End If
    Return New ValueTask(Of Order)(GetFromDatabaseAsync(orderId, cancellationToken))
End Function

' Consuming IAsyncEnumerable — VB.NET has NO "Await For Each" statement.
' Use a manual IAsyncEnumerator(Of T) loop with MoveNextAsync / DisposeAsync.
Public Async Function PrintOrdersAsync(customerId As String, cancellationToken As CancellationToken) As Task
    Dim source = _repository.StreamAllAsync(cancellationToken)
    Dim enumerator = source.GetAsyncEnumerator()
    Try
        While Await enumerator.MoveNextAsync()
            Dim order = enumerator.Current
            If order.CustomerId = customerId Then
                Console.WriteLine(order.Id)
            End If
        End While
    Finally
        Await enumerator.DisposeAsync()
    End Try
End Function
```

> **VB.NET limitation — consuming and authoring async streams:** VB.NET does **not** have an `Await For Each` statement (that is C# `await foreach` syntax). To consume an `IAsyncEnumerable(Of T)` in VB.NET, obtain an `IAsyncEnumerator(Of T)` via `GetAsyncEnumerator()` and loop with `While Await enumerator.MoveNextAsync()`, ensuring `Await enumerator.DisposeAsync()` runs in a `Finally` block (as shown above). Authoring an `IAsyncEnumerable(Of T)` — i.e., an `Async` iterator using `yield return` — is a **C#-only** feature; VB.NET does not support combining `Yield` with `Async`. If you need to *produce* an async stream, implement the iterator in a C# class library and consume it from VB using the manual enumerator pattern above.

**Key async rules:**
- Always accept `CancellationToken` with `Optional cancellationToken As CancellationToken = Nothing`
- Use `ConfigureAwait(False)` in library code
- Never block on async code (no `.Result` or `.Wait()`)

```vb
' ConfigureAwait(False) example - REQUIRED in library/class-library code to avoid
' capturing a SynchronizationContext (e.g. the WinForms UI thread) on resumption.
' Do NOT use ConfigureAwait(False) in WinForms UI event handlers that must
' touch controls after the await - those need the captured UI context.
Public Async Function ReadConfigAsync(path As String,
                                      cancellationToken As CancellationToken) As Task(Of String)
    Using reader As New StreamReader(path)
        Return Await reader.ReadToEndAsync().ConfigureAwait(False)
    End Using
End Function
```

---

### Span(Of T) and Memory(Of T)

Use `Span(Of T)` for synchronous zero-allocation operations and `Memory(Of T)` for async scenarios. Use `ArrayPool(Of T)` for large temporary buffers.

```vb
' Consuming a Span-based API (fully supported in VB.NET).
' NOTE: Integer.Parse(ReadOnlySpan(Of Char)) is a .NET Core 2.1+ / .NET 5+ overload
' and is NOT available on .NET Framework. On .NET Framework, materialize a String
' first (e.g. Integer.Parse(segment.ToString())).
Public Function ParseFirstInt(text As String) As Integer
    Dim span As ReadOnlySpan(Of Char) = text.AsSpan()
    Dim commaIndex = span.IndexOf(","c)
    Dim segment = If(commaIndex >= 0, span.Slice(0, commaIndex), span)
    Return Integer.Parse(segment)
End Function

' ArrayPool for large buffers
Public Function ProcessData(data As Byte()) As Integer
    Dim buffer = ArrayPool(Of Byte).Shared.Rent(4096)
    Try
        ' ... process using buffer ...
        Return 0
    Finally
        ArrayPool(Of Byte).Shared.Return(buffer)
    End Try
End Function
```

> **C#-only feature — authoring `ref struct` / `Span`-backed types:** C# can declare `ref struct` types (like `Span(Of T)` itself), functions that return `ref`, and parameters marked `in` / `ref readonly`. VB.NET fully supports `ByRef` parameters (the equivalent of C# `ref`), but **does not support `ByRef`-returning functions, `ref struct` authoring, or `In` / `ref readonly` parameter authoring**. VB.NET is a *consumer* of Span-based APIs, not an author of new `ref struct` types. If you need to author high-performance buffer types, implement them in a C# project and reference that assembly from VB.

---

### LINQ

Both method syntax and query syntax are supported in VB.NET.

```vb
' Method syntax — identical to C# except lambda uses Function keyword
Dim results = list.Where(Function(x) x > 0).Select(Function(x) x * 2).ToList()

' Query syntax — natural in VB.NET
Dim queryResults = From x In list
                   Where x > 0
                   Select x * 2

' Aggregates
Dim total = orders.Sum(Function(o) o.Total)
Dim count = orders.Count(Function(o) o.Status = OrderStatus.Completed)

' Grouping
Dim byStatus = From o In orders
               Group o By o.Status Into Group
               Select Status, Items = Group.ToList()
```

---

### Extension Methods

VB.NET requires the `<Extension()>` attribute on the method, and the method must be in a `Module` (not a `Class`).

```vb
Imports System.Runtime.CompilerServices

Public Module StringExtensions
    <Extension()>
    Public Function IsNullOrBlank(value As String) As Boolean
        Return String.IsNullOrWhiteSpace(value)
    End Function

    <Extension()>
    Public Function Truncate(value As String, maxLength As Integer) As String
        If value Is Nothing Then Return Nothing
        Return If(value.Length <= maxLength, value, value.Substring(0, maxLength))
    End Function
End Module

' Usage
If someString.IsNullOrBlank() Then ...
```

---

### Tuples

```vb
' Declare a tuple return type
Public Function GetCoordinates() As (X As Integer, Y As Integer)
    Return (10, 20)
End Function

' Access tuple components — VB.NET does NOT support C#-style deconstruction
' declarations ("var (x, y) = ...").  Access fields by name (for named tuples)
' or by Item1/Item2/... for unnamed tuples.
Dim coord = GetCoordinates()
Dim x = coord.X
Dim y = coord.Y

' Named tuple fields
Dim point = (X:=5, Y:=10)
Console.WriteLine(point.X)

' Unnamed tuple — access via Item1 / Item2
Dim pair = (10, 20)
Dim first = pair.Item1
Dim second = pair.Item2
```

> **VB.NET vs. C# tuples:** VB.NET supports tuple types and named fields, but has no deconstruction-declaration syntax (`Dim (x, y) = expr` is **not** valid VB.NET). Access tuple components by their name (e.g. `coord.X`) or by positional `ItemN` members (e.g. `coord.Item1`).

---

### String Interpolation and String Utilities

```vb
' String interpolation — same $ syntax as C# (VB 14+)
Dim greeting = $"Hello, {name}! You have {count} messages."

' NameOf operator
Public Sub SetName(value As String)
    If value Is Nothing Then Throw New ArgumentNullException(NameOf(value))
    _name = value
End Sub

' Multi-line strings — VB.NET does not have raw string literals (C# """ syntax)
' Use string concatenation or StringBuilder for multi-line content
Dim multiLine = "Line one" & Environment.NewLine &
                "Line two" & Environment.NewLine &
                "Line three"

' Or use an XML literal for structured text
Dim xml = <root>
              <item>value</item>
          </root>
```

> **C#-only feature — raw string literals:** C# 11 introduced `"""…"""` raw string literals that avoid the need for escape sequences. VB.NET does not have raw string literals. Use string concatenation, `StringBuilder`, or XML literals for multi-line or special-character content.

---

### Primary Constructors (C# 12)

> **C#-only feature:** C# 12 introduced primary constructors, which allow constructor parameters to be declared directly on the class/struct declaration. VB.NET does not have primary constructors. Use a standard `Public Sub New(…)` constructor.

```vb
' VB.NET equivalent of a C# primary constructor class
Public Class OrderProcessor
    Private ReadOnly _repository As IOrderRepository
    Private ReadOnly _logger As ILogger

    Public Sub New(repository As IOrderRepository, logger As ILogger)
        _repository = repository
        _logger = logger
    End Sub

    Public Async Function ProcessAsync(orderId As String) As Task
        Dim order = Await _repository.GetAsync(orderId, CancellationToken.None)
        _logger.LogInformation("Processing order {OrderId}", orderId)
    End Function
End Class
```

---

## Composition Over Inheritance

Avoid abstract base classes. Use interfaces and composition. Use shared helper methods in a `Module` for utility logic shared across multiple classes.

```vb
' Prefer interface + composition
Public Interface IOrderValidator
    Function Validate(order As Order) As ValidationResult
End Interface

Public Class OrderService
    Private ReadOnly _repository As IOrderRepository
    Private ReadOnly _validator As IOrderValidator

    Public Sub New(repository As IOrderRepository, validator As IOrderValidator)
        _repository = repository
        _validator = validator
    End Sub
End Class
```

---

## Error Handling: Result Type

For expected business errors, use a `Result(Of T, TError)` type instead of exceptions. Use exceptions only for unexpected or system-level errors.

```vb
' Simple Result type
Public Class Result(Of TValue, TError)
    Public ReadOnly Property IsSuccess As Boolean
    Public ReadOnly Property Value As TValue
    Public ReadOnly Property [Error] As TError

    Private Sub New(isSuccess As Boolean, value As TValue, [error] As TError)
        Me.IsSuccess = isSuccess
        Me.Value = value
        Me.[Error] = [error]
    End Sub

    Public Shared Function Success(value As TValue) As Result(Of TValue, TError)
        Return New Result(Of TValue, TError)(True, value, Nothing)
    End Function

    Public Shared Function Failure([error] As TError) As Result(Of TValue, TError)
        Return New Result(Of TValue, TError)(False, Nothing, [error])
    End Function
End Class
```

---

## Avoid Reflection-Based Metaprogramming

**Avoid:** AutoMapper, Mapster, and similar convention-based mapping libraries. Use explicit mapping methods instead. When private member access is genuinely needed on .NET 8+, use `UnsafeAccessorAttribute`.

```vb
' Explicit mapping extension method — no reflection
Imports System.Runtime.CompilerServices

Public Module CustomerMappingExtensions
    <Extension()>
    Public Function ToDto(customer As Customer) As CustomerDto
        Return New CustomerDto(
            customer.Id.ToString(),
            customer.Name,
            customer.Email.Value
        )
    End Function
End Module
```

---

## Code Organization

```vb
' File: Domain/Orders/Order.vb

Namespace MyApp.Domain.Orders

    ' 1. Primary domain type
    Public Class Order
        Public ReadOnly Property Id As OrderId
        Public ReadOnly Property CustomerId As CustomerId
        Public ReadOnly Property Total As Money
        Public ReadOnly Property Status As OrderStatus
        Public ReadOnly Property Items As IReadOnlyList(Of OrderItem)

        Public Sub New(id As OrderId, customerId As CustomerId, total As Money,
                       status As OrderStatus, items As IReadOnlyList(Of OrderItem))
            Me.Id = id
            Me.CustomerId = customerId
            Me.Total = total
            Me.Status = status
            Me.Items = items
        End Sub

        Public ReadOnly Property IsCompleted As Boolean
            Get
                Return Status = OrderStatus.Completed
            End Get
        End Property

        Public Function AddItem(item As OrderItem) As Result(Of Order, OrderError)
            If Status <> OrderStatus.Draft Then
                Return Result(Of Order, OrderError).Failure(
                    New OrderError("ORDER_NOT_DRAFT", "Can only add items to draft orders"))
            End If

            Dim newItems As New List(Of OrderItem)(Items)
            newItems.Add(item)

            Dim newAmount = Items.Sum(Function(i) i.Total.Amount) + item.Total.Amount
            Dim newTotal As New Money(newAmount, Total.Currency)

            Return Result(Of Order, OrderError).Success(
                New Order(Id, CustomerId, newTotal, Status, newItems.AsReadOnly()))
        End Function
    End Class

    ' 2. Enum for state
    Public Enum OrderStatus
        Draft
        Submitted
        Processing
        Completed
        Cancelled
    End Enum

    ' 3. Related type
    Public Class OrderItem
        Public ReadOnly Property ProductId As ProductId
        Public ReadOnly Property Quantity As Quantity
        Public ReadOnly Property UnitPrice As Money

        Public Sub New(productId As ProductId, quantity As Quantity, unitPrice As Money)
            Me.ProductId = productId
            Me.Quantity = quantity
            Me.UnitPrice = unitPrice
        End Sub

        Public ReadOnly Property Total As Money
            Get
                Return New Money(UnitPrice.Amount * Quantity.Value, UnitPrice.Currency)
            End Get
        End Property
    End Class

    ' 4. Value objects
    Public Structure OrderId
        Public ReadOnly Property Value As Guid

        Public Sub New(value As Guid)
            Me.Value = value
        End Sub

        Public Shared Function [New]() As OrderId
            Return New OrderId(Guid.NewGuid())
        End Function
    End Structure

    ' 5. Error type
    Public Structure OrderError
        Public ReadOnly Property Code As String
        Public ReadOnly Property Message As String

        Public Sub New(code As String, message As String)
            Me.Code = code
            Me.Message = message
        End Sub
    End Structure

End Namespace
```

---

## Best Practices Summary

### DO's
- Use `Class` with `ReadOnly Property` and a constructor for immutable DTOs and domain entities (VB.NET equivalent of C# `record`)
- Use `Structure` with `ReadOnly` fields for value objects; implement `Equals`/`GetHashCode` manually
- Use `Select Case` and `If`/`ElseIf` chains for multi-branch logic (VB.NET equivalent of C# pattern matching)
- Use `If(x, y)` for null-coalescing (equivalent of C# `??`)
- Use `x?.Prop` for null-conditional member access
- Use async/await (`Async Function … As Task(Of T)`) for all I/O operations
- Accept `CancellationToken` in all async methods
- Use `Span(Of T)` and `Memory(Of T)` APIs for high-performance scenarios (consumer role)
- Accept abstractions (`IEnumerable(Of T)`, `IReadOnlyList(Of T)`)
- Use `Result(Of T, TError)` for expected business errors
- Pool buffers with `ArrayPool(Of T)` for large allocations
- Prefer composition over inheritance
- Define extension methods in a `Module` with `<Extension()>` attribute

### DON'Ts
- Don't use mutable public fields; use `Property` or `ReadOnly Property`
- Don't create deep inheritance hierarchies
- Don't block on async code (`.Result`, `.Wait()`)
- Don't use `byte()` buffers for every operation when `Span(Of Byte)` APIs are available
- Don't forget `CancellationToken` parameters in async methods
- Don't return mutable collections from public APIs; return `IReadOnlyList(Of T)` or similar
- Don't throw exceptions for expected business errors; use `Result`
- Don't allocate large arrays repeatedly; use `ArrayPool`
- Don't use reflection-based mapping libraries; write explicit mapping methods

---

## C#-Only Features — Quick Reference

The following C# features have **no direct VB.NET equivalent**. The VB.NET approach for each is described above in the relevant section.

| C# Feature | VB.NET Approach |
|---|---|
| `record` / `record class` | `Class` with `ReadOnly Property` + constructor |
| `readonly record struct` | `Structure` with `ReadOnly` fields; manual `Equals`/`GetHashCode` |
| `init`-only setters (`{ get; init; }`) | Constructor-assigned `ReadOnly Property` |
| `with` expressions | Manual copy/clone method |
| Primary constructors (C# 12) | Standard `Public Sub New(…)` |
| `switch` expressions and property patterns | `If`/`ElseIf` chains, `Select Case`, `TypeOf`/`DirectCast` |
| Collection expressions `[1, 2, 3]` | `{1, 2, 3}` (array), `New List(Of T) From {…}` |
| Raw string literals `"""…"""` (C# 11) | String concatenation, `StringBuilder`, XML literals |
| `ref struct` / `ref` fields authorship | Not supported; use C# project for authoring |
| `IAsyncEnumerable(Of T)` authorship (async `yield return`) | Not supported in VB.NET; author in C#. Consume from VB via a manual `GetAsyncEnumerator` + `While Await MoveNextAsync()` loop (no `Await For Each` statement exists in VB.NET) |

---

## Additional Resources

- **VB.NET Language Reference**: https://learn.microsoft.com/en-us/dotnet/visual-basic/
- **VB.NET Programming Guide**: https://learn.microsoft.com/en-us/dotnet/visual-basic/programming-guide/
- **Async/Await in VB.NET**: https://learn.microsoft.com/en-us/dotnet/visual-basic/programming-guide/concepts/async/
- **LINQ in VB.NET**: https://learn.microsoft.com/en-us/dotnet/visual-basic/programming-guide/concepts/linq/
- **Memory and Spans**: https://learn.microsoft.com/en-us/dotnet/standard/memory-and-spans/
- **Async Best Practices**: https://learn.microsoft.com/en-us/archive/msdn-magazine/2013/march/async-await-best-practices-in-asynchronous-programming
