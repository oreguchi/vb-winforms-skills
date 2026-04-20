# Anti-Patterns and Reflection Avoidance

Guidelines on avoiding reflection-based metaprogramming and common VB.NET anti-patterns.

## Contents

- [Avoid Reflection-Based Metaprogramming](#avoid-reflection-based-metaprogramming)
- [UnsafeAccessorAttribute (.NET 8+)](#unsafeaccessorattribute-net-8)
- [Anti-Patterns to Avoid](#anti-patterns-to-avoid)

> **VB.NET language notes applied throughout this file**
> - VB.NET has **no `record` / `record struct`**; use `Class` / `Structure` with `ReadOnly` properties.
> - VB.NET has **no `switch` expression**; use `Select Case`.
> - VB.NET can **consume** `ReadOnlySpan(Of Byte)` parameters but cannot **author** new `ref struct` types.
> - All samples assume `Option Strict On`.

## Avoid Reflection-Based Metaprogramming

**Prefer statically-typed, explicit code over reflection-based "magic" libraries.**

Reflection-based libraries like AutoMapper trade compile-time safety for convenience. When mappings break, you find out at runtime (or worse, in production) instead of at compile time.

### Banned Libraries

| Library | Problem |
|---------|---------|
| **AutoMapper** | Reflection magic, hidden mappings, runtime failures, hard to debug |
| **Mapster** | Same issues as AutoMapper |
| **ExpressMapper** | Same issues |

### Why Reflection Mapping Fails

```vb
' With AutoMapper - compiles fine, fails at runtime
Public Class UserDto
    Public ReadOnly Property Id As String
    Public ReadOnly Property Name As String
    Public ReadOnly Property Email As String

    Public Sub New(id As String, name As String, email As String)
        Me.Id = id
        Me.Name = name
        Me.Email = email
    End Sub
End Class

Public Class UserEntity
    Public ReadOnly Property Id As Guid
    Public ReadOnly Property FullName As String
    Public ReadOnly Property EmailAddress As String

    Public Sub New(id As Guid, fullName As String, emailAddress As String)
        Me.Id = id
        Me.FullName = fullName
        Me.EmailAddress = emailAddress
    End Sub
End Class

' This mapping silently produces garbage:
' - Id: String vs Guid mismatch
' - Name vs FullName: no match, Nothing/default
' - Email vs EmailAddress: no match, Nothing/default
Dim dto = _mapper.Map(Of UserDto)(entity)   ' Compiles! Breaks at runtime.
```

### Use Explicit Mapping Methods Instead

```vb
Imports System.Runtime.CompilerServices

' Extension methods go in a Module with <Extension()> attributes - compile-time checked,
' easy to find, easy to debug.
Public Module UserMappings

    <Extension()>
    Public Function ToDto(entity As UserEntity) As UserDto
        Return New UserDto(
            id:=entity.Id.ToString(),
            name:=entity.FullName,
            email:=entity.EmailAddress)
    End Function

    <Extension()>
    Public Function ToEntity(request As CreateUserRequest) As UserEntity
        Return New UserEntity(
            id:=Guid.NewGuid(),
            fullName:=request.Name,
            emailAddress:=request.Email)
    End Function

End Module

' Usage - explicit and traceable
Dim dto = entity.ToDto()
Dim entity = request.ToEntity()
```

### Benefits of Explicit Mappings

| Aspect | AutoMapper | Explicit Methods |
|--------|------------|------------------|
| **Compile-time safety** | No - runtime errors | Yes - compiler catches mismatches |
| **Discoverability** | Hidden in profiles | "Go to Definition" works |
| **Debugging** | Black box | Step through code |
| **Refactoring** | Rename breaks silently | IDE renames correctly |
| **Performance** | Reflection overhead | Direct property access |
| **Testing** | Need integration tests | Simple unit tests |

### Complex Mappings

For complex transformations, explicit code is even more valuable. VB.NET has no `switch` expression, so use `Select Case` inside a helper function:

```vb
Public Module OrderMappings

    <Extension()>
    Public Function ToSummary(order As Order) As OrderSummaryDto
        Return New OrderSummaryDto(
            orderId:=order.Id.Value.ToString(),
            customerName:=order.Customer.FullName,
            itemCount:=order.Items.Count,
            total:=order.Items.Sum(Function(i) i.Quantity * i.UnitPrice),
            status:=FormatStatus(order.Status),
            formattedDate:=order.CreatedAt.ToString("MMMM d, yyyy"))
    End Function

    Private Function FormatStatus(status As OrderStatus) As String
        Select Case status
            Case OrderStatus.Pending
                Return "Awaiting Payment"
            Case OrderStatus.Paid
                Return "Processing"
            Case OrderStatus.Shipped
                Return "On the Way"
            Case OrderStatus.Delivered
                Return "Completed"
            Case Else
                Return "Unknown"
        End Select
    End Function

End Module
```

This is:
- **Readable**: Anyone can understand the transformation
- **Debuggable**: Set a breakpoint, inspect values
- **Testable**: Pass an Order, assert on the result
- **Refactorable**: Change a property name, compiler tells you everywhere it's used

### When Reflection is Acceptable

Reflection has legitimate uses, but mapping DTOs isn't one of them:

| Use Case | Acceptable? |
|----------|-------------|
| Serialization (System.Text.Json, Newtonsoft) | Yes - well-tested, source generators available |
| Dependency injection container | Yes - framework infrastructure |
| ORM entity mapping (EF Core) | Yes - necessary for database abstraction |
| Test fixtures and builders | Sometimes - for convenience in tests only |
| **DTO/domain object mapping** | **No - use explicit methods** |

## UnsafeAccessorAttribute (.NET 8+)

When you genuinely need to access private or internal members (serializers, test helpers, framework code), use `UnsafeAccessorAttribute` instead of traditional reflection. It provides **zero-overhead, AOT-compatible** member access.

> **VB.NET note — `ByRef` returns:** The C# samples return `ref` to a field (e.g. `ref OrderStatus`). VB.NET cannot author `ByRef`-returning functions. You can still apply `UnsafeAccessor` to VB functions that **return by value** (e.g. read a private field) and to `Sub`s that call private methods. For true `ByRef` access you must declare the accessor in a C# project and call it from VB, or fall back to reflection.

```vb
Imports System.Reflection
Imports System.Runtime.CompilerServices

' AVOID: Traditional reflection - slow, allocates, breaks AOT
Dim field = GetType(Order).GetField(
    "_status", BindingFlags.NonPublic Or BindingFlags.Instance)
Dim status = DirectCast(field.GetValue(order), OrderStatus)

' PREFER: UnsafeAccessor - zero overhead, AOT-compatible.
' VB cannot author ByRef returns, so expose the field by value.
<UnsafeAccessor(UnsafeAccessorKind.Field, Name:="_status")>
Private Shared Function GetStatusField(order As Order) As OrderStatus
End Function

Dim status = GetStatusField(order)   ' Direct access, no reflection
```

**Supported accessor kinds:**

```vb
' Private field access - return by value from VB (a C# "ref" return must be authored in C#)
<UnsafeAccessor(UnsafeAccessorKind.Field, Name:="_items")>
Private Shared Function GetItemsField(order As Order) As List(Of OrderItem)
End Function

' Private method access
<UnsafeAccessor(UnsafeAccessorKind.Method, Name:="Recalculate")>
Private Shared Sub CallRecalculate(order As Order)
End Sub

' Private static field
<UnsafeAccessor(UnsafeAccessorKind.StaticField, Name:="_instanceCount")>
Private Shared Function GetInstanceCount(order As Order) As Integer
End Function

' Private constructor
<UnsafeAccessor(UnsafeAccessorKind.Constructor)>
Private Shared Function CreateOrder(id As OrderId, customerId As CustomerId) As Order
End Function
```

**Why UnsafeAccessor over reflection:**

| Aspect | Reflection | UnsafeAccessor |
|--------|------------|----------------|
| Performance | Slow (100-1000x) | Zero overhead |
| AOT compatible | No | Yes |
| Allocations | Yes (boxing, arrays) | None |
| Compile-time checked | No | Partially (signature) |

**Use cases:**
- Serializers accessing private backing fields
- Test helpers verifying internal state
- Framework code that needs to bypass visibility

**Resources:**
- [A new way of doing reflection with .NET 8](https://steven-giesel.com/blogPost/05ecdd16-8dc4-490f-b1cf-780c994346a4)
- [Accessing private members without reflection in .NET 8.0](https://www.strathweb.com/2023/10/accessing-private-members-without-reflection-in-net-8-0/)
- [Modern .NET Reflection with UnsafeAccessor](https://blog.ndepend.com/modern-net-reflection-with-unsafeaccessor/)

## Anti-Patterns to Avoid

### Don't: Use mutable DTOs

```vb
' BAD: Mutable DTO
Public Class CustomerDto
    Public Property Id As String
    Public Property Name As String
End Class

' GOOD: Immutable class with ReadOnly properties + constructor
' (VB.NET equivalent of C# "record CustomerDto(string Id, string Name)")
Public NotInheritable Class CustomerDto
    Public ReadOnly Property Id As String
    Public ReadOnly Property Name As String

    Public Sub New(id As String, name As String)
        Me.Id = id
        Me.Name = name
    End Sub
End Class
```

### Don't: Use Classes for value objects

```vb
' BAD: Value object as Class (reference type, unwanted identity semantics)
Public Class OrderId
    Public ReadOnly Property Value As String

    Public Sub New(value As String)
        Me.Value = value
    End Sub
End Class

' GOOD: Value object as Structure (value type).
' VB.NET has no "readonly record struct"; implement Equals / GetHashCode manually.
Public Structure OrderId
    Public ReadOnly Property Value As String

    Public Sub New(value As String)
        Me.Value = value
    End Sub

    Public Overrides Function Equals(obj As Object) As Boolean
        If TypeOf obj Is OrderId Then
            Return DirectCast(obj, OrderId).Value = Value
        End If
        Return False
    End Function

    Public Overrides Function GetHashCode() As Integer
        Return If(Value, String.Empty).GetHashCode()
    End Function
End Structure
```

### Don't: Create deep inheritance hierarchies

```vb
' BAD: Deep inheritance
Public MustInherit Class Entity
End Class
Public MustInherit Class AggregateRoot
    Inherits Entity
End Class
Public MustInherit Class Order
    Inherits AggregateRoot
End Class
Public Class CustomerOrder
    Inherits Order
End Class

' GOOD: Flat structure with composition
Public Interface IEntity
    ReadOnly Property Id As Guid
End Interface

Public NotInheritable Class Order
    Implements IEntity

    Public ReadOnly Property Id As OrderId
    Public ReadOnly Property CustomerId As CustomerId
    Public ReadOnly Property Total As Money

    Public Sub New(id As OrderId, customerId As CustomerId, total As Money)
        Me.Id = id
        Me.CustomerId = customerId
        Me.Total = total
    End Sub

    Private ReadOnly Property IEntity_Id As Guid Implements IEntity.Id
        Get
            ' OrderId.Value is assumed to be a Guid; return it directly.
            Return Id.Value
        End Get
    End Property
End Class
```

### Don't: Return List(Of T) when you mean IReadOnlyList(Of T)

```vb
' BAD: Exposes internal list for modification
Public Function GetOrders() As List(Of Order)
    Return _orders
End Function

' GOOD: Returns read-only view
Public Function GetOrders() As IReadOnlyList(Of Order)
    Return _orders
End Function
```

### Don't: Use Byte() when ReadOnlySpan(Of Byte) works

```vb
' BAD: Allocates array on every call
Public Function GetHeader() As Byte()
    Dim header(63) As Byte
    ' Fill header
    Return header
End Function

' GOOD: Zero allocation with Span. VB.NET consumes Span APIs even though it
' cannot author new ref struct types.
Public Sub GetHeader(destination As Span(Of Byte))
    If destination.Length < 64 Then
        Throw New ArgumentException("Buffer too small")
    End If
    ' Fill header directly into caller's buffer
End Sub
```

### Don't: Forget CancellationToken in async methods

```vb
' BAD: No cancellation support
Public Async Function GetOrderAsync(id As OrderId) As Task(Of Order)
    Return Await _repository.GetAsync(id)
End Function

' GOOD: Cancellation support
Public Async Function GetOrderAsync(
        id As OrderId,
        Optional cancellationToken As CancellationToken = Nothing) As Task(Of Order)
    Return Await _repository.GetAsync(id, cancellationToken)
End Function
```

### Don't: Block on async code

```vb
' BAD: Deadlock risk!
Public Function GetOrder(id As OrderId) As Order
    Return GetOrderAsync(id).Result
End Function

' BAD: Also deadlock risk!
Public Function GetOrder(id As OrderId) As Order
    Return GetOrderAsync(id).GetAwaiter().GetResult()
End Function

' GOOD: Async all the way
Public Async Function GetOrderAsync(
        id As OrderId,
        cancellationToken As CancellationToken) As Task(Of Order)
    Return Await _repository.GetAsync(id, cancellationToken)
End Function
```
