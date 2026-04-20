# Composition and Error Handling

Composition over inheritance, Result type pattern, and testing patterns for modern VB.NET.

## Contents

- [Composition Over Inheritance](#composition-over-inheritance)
- [Result Type Pattern](#result-type-pattern)
- [Testing Patterns](#testing-patterns)

> **VB.NET language notes applied throughout this file**
> - VB.NET has **no `record` / `record struct`**. Use `Class` / `Structure` with `ReadOnly` properties assigned via a constructor.
> - VB.NET has **no `with` expression**. Provide an explicit `With*` copy method on the immutable type.
> - VB.NET has **no `switch` expression**. Use `Select Case` / `If`/`ElseIf`.
> - All samples assume `Option Strict On`.

## Composition Over Inheritance

**Avoid abstract base classes and inheritance hierarchies.** Use composition and interfaces instead.

```vb
Imports System.Threading
Imports System.Threading.Tasks

' BAD: Abstract base class hierarchy
Public MustInherit Class PaymentProcessor
    Public MustOverride Function ProcessAsync(amount As Money) As Task(Of PaymentResult)

    Protected Async Function ValidateAsync(amount As Money) As Task(Of Boolean)
        ' Shared validation logic
        Return amount.Amount > 0
    End Function
End Class

Public Class CreditCardProcessor
    Inherits PaymentProcessor

    Public Overrides Async Function ProcessAsync(amount As Money) As Task(Of PaymentResult)
        Await ValidateAsync(amount)
        ' Process credit card...
        Return Nothing
    End Function
End Class

' GOOD: Composition with interfaces
Public Interface IPaymentProcessor
    Function ProcessAsync(amount As Money, cancellationToken As CancellationToken) As Task(Of PaymentResult)
End Interface

Public Interface IPaymentValidator
    Function ValidateAsync(amount As Money, cancellationToken As CancellationToken) As Task(Of ValidationResult)
End Interface

' Concrete implementations compose validators
Public NotInheritable Class CreditCardProcessorImpl
    Implements IPaymentProcessor

    Private ReadOnly _validator As IPaymentValidator
    Private ReadOnly _gateway As ICreditCardGateway

    Public Sub New(validator As IPaymentValidator, gateway As ICreditCardGateway)
        _validator = validator
        _gateway = gateway
    End Sub

    Public Async Function ProcessAsync(amount As Money,
                                       cancellationToken As CancellationToken) As Task(Of PaymentResult) _
            Implements IPaymentProcessor.ProcessAsync
        Dim validation = Await _validator.ValidateAsync(amount, cancellationToken)
        If Not validation.IsValid Then
            Return PaymentResult.Failed(validation.Error)
        End If
        Return Await _gateway.ChargeAsync(amount, cancellationToken)
    End Function
End Class

' GOOD: Static helper in a Module for shared logic (no inheritance)
Public Module PaymentValidation
    Public Function ValidateAmount(amount As Money) As ValidationResult
        If amount.Amount <= 0D Then
            Return ValidationResult.Invalid("Amount must be positive")
        End If
        If amount.Amount > 10000D Then
            Return ValidationResult.Invalid("Amount exceeds maximum")
        End If
        Return ValidationResult.Valid()
    End Function
End Module

' GOOD: Immutable class + factory methods for modeling variants (not inheritance).
' VB.NET has no "record" and no "init" setters; use ReadOnly properties + Sub New.
Public Enum PaymentType
    CreditCard
    BankTransfer
    Cash
End Enum

Public Class PaymentMethod
    Public ReadOnly Property Type As PaymentType
    Public ReadOnly Property Last4 As String            ' For credit cards
    Public ReadOnly Property AccountNumber As String    ' For bank transfers

    Private Sub New(type As PaymentType, last4 As String, accountNumber As String)
        Me.Type = type
        Me.Last4 = last4
        Me.AccountNumber = accountNumber
    End Sub

    Public Shared Function CreditCard(last4 As String) As PaymentMethod
        Return New PaymentMethod(PaymentType.CreditCard, last4, Nothing)
    End Function

    Public Shared Function BankTransfer(accountNumber As String) As PaymentMethod
        Return New PaymentMethod(PaymentType.BankTransfer, Nothing, accountNumber)
    End Function

    Public Shared Function Cash() As PaymentMethod
        Return New PaymentMethod(PaymentType.Cash, Nothing, Nothing)
    End Function
End Class
```

**When inheritance is acceptable:**
- Framework requirements (e.g., inheriting `Form` for WinForms, `ControllerBase` in ASP.NET Core)
- Library integration (e.g., custom exceptions inheriting from `Exception`)
- **These should be rare cases in your application code**

## Result Type Pattern

For expected errors, use a **domain-specific result type** instead of exceptions. Don't build a generic `Result(Of T)` — each operation knows what success and failure look like, so let the result type reflect that. Use `NotInheritable Class` (equivalent of a `sealed record`) with factory methods and enum error codes.

```vb
' Enum for error classification - type-safe and switchable
Public Enum OrderErrorCode
    ValidationError
    InsufficientInventory
    NotFound
End Enum

' Domain-specific result type - sealed class with factory methods.
' VB.NET has no "record"; use Public ReadOnly Property + private constructor.
Public NotInheritable Class CreateOrderResult
    Public ReadOnly Property IsSuccess As Boolean
    Public ReadOnly Property Order As Order
    Public ReadOnly Property ErrorCode As OrderErrorCode?
    Public ReadOnly Property ErrorMessage As String

    Private Sub New(isSuccess As Boolean,
                    order As Order,
                    errorCode As OrderErrorCode?,
                    errorMessage As String)
        Me.IsSuccess = isSuccess
        Me.Order = order
        Me.ErrorCode = errorCode
        Me.ErrorMessage = errorMessage
    End Sub

    Public Shared Function Success(order As Order) As CreateOrderResult
        Return New CreateOrderResult(True, order, Nothing, Nothing)
    End Function

    Public Shared Function Failed(code As OrderErrorCode, message As String) As CreateOrderResult
        Return New CreateOrderResult(False, Nothing, code, message)
    End Function
End Class

' Usage example
Public NotInheritable Class OrderService
    Private ReadOnly _repository As IOrderRepository

    Public Sub New(repository As IOrderRepository)
        _repository = repository
    End Sub

    Public Async Function CreateOrderAsync(
            request As CreateOrderRequest,
            cancellationToken As CancellationToken) As Task(Of CreateOrderResult)

        If Not IsValid(request) Then
            Return CreateOrderResult.Failed(
                OrderErrorCode.ValidationError, "Invalid order request")
        End If

        If Not Await HasInventoryAsync(request.Items, cancellationToken) Then
            Return CreateOrderResult.Failed(
                OrderErrorCode.InsufficientInventory, "Items out of stock")
        End If

        Dim order As New Order(
            OrderId.[New](),
            New CustomerId(request.CustomerId),
            request.Items)

        Await _repository.SaveAsync(order, cancellationToken)

        Return CreateOrderResult.Success(order)
    End Function

    ' Map result to HTTP response - Select Case on enum error codes
    Public Function MapToActionResult(result As CreateOrderResult) As IActionResult
        If result.IsSuccess Then
            Return New OkObjectResult(result.Order)
        End If

        Select Case result.ErrorCode
            Case OrderErrorCode.ValidationError
                Return New BadRequestObjectResult(New With {.[error] = result.ErrorMessage})
            Case OrderErrorCode.InsufficientInventory
                Return New ConflictObjectResult(New With {.[error] = result.ErrorMessage})
            Case OrderErrorCode.NotFound
                Return New NotFoundObjectResult(New With {.[error] = result.ErrorMessage})
            Case Else
                Return New ObjectResult(New With {.[error] = result.ErrorMessage}) With {.StatusCode = 500}
        End Select
    End Function
End Class
```

**When to use Result vs Exceptions:**
- **Use Result**: Expected errors (validation, business rules, not found)
- **Use Exceptions**: Unexpected errors (network failures, system errors, programming bugs)

## Testing Patterns

```vb
Imports Xunit
Imports FluentAssertions

' Use an immutable builder class for test data.
' VB.NET has no "record" and no "with" expression. Provide "With*" copy methods
' so tests can create variations without mutating shared state.
Public NotInheritable Class OrderBuilder
    Public ReadOnly Property Id As OrderId
    Public ReadOnly Property CustomerId As CustomerId
    Public ReadOnly Property Total As Money
    Public ReadOnly Property Items As IReadOnlyList(Of OrderItem)

    Public Sub New(Optional id As OrderId? = Nothing,
                   Optional customerId As CustomerId? = Nothing,
                   Optional total As Money? = Nothing,
                   Optional items As IReadOnlyList(Of OrderItem) = Nothing)
        ' Under Option Strict On, two-arg If() with a Nullable(Of T) first arg
        ' returns Nullable(Of T), not T. Use GetValueOrDefault to unwrap.
        Me.Id = id.GetValueOrDefault(OrderId.[New]())
        Me.CustomerId = customerId.GetValueOrDefault(CustomerId.[New]())
        Me.Total = total.GetValueOrDefault(New Money(100D, "USD"))
        Me.Items = If(items, Array.Empty(Of OrderItem)())
    End Sub

    Public Function WithTotal(newTotal As Money) As OrderBuilder
        Return New OrderBuilder(Me.Id, Me.CustomerId, newTotal, Me.Items)
    End Function

    Public Function Build() As Order
        Return New Order(Id, CustomerId, Total, Items)
    End Function
End Class

' Replacement for C# "baseOrder with { Total = ... }": call a "With*" helper.
<Fact>
Public Sub CalculateDiscount_LargeOrder_AppliesCorrectDiscount()
    ' Arrange
    Dim baseOrder = New OrderBuilder().Build()
    Dim largeOrder = New OrderBuilder().WithTotal(New Money(1500D, "USD")).Build()

    ' Act
    Dim discount = _service.CalculateDiscount(largeOrder)

    ' Assert
    discount.Should().Be(New Money(225D, "USD"))   ' 15% of 1500
End Sub

' Span-based testing
<Theory>
<InlineData("ORD-12345", True)>
<InlineData("INVALID", False)>
Public Sub TryParseOrderId_VariousInputs_ReturnsExpectedResult(input As String, expected As Boolean)
    ' Act
    Dim orderId As OrderId = Nothing
    Dim result = OrderIdParser.TryParse(input.AsSpan(), orderId)

    ' Assert
    result.Should().Be(expected)
End Sub

' Testing with value objects
<Fact>
Public Sub Money_Add_SameCurrency_ReturnsSum()
    ' Arrange
    Dim money1 = New Money(100D, "USD")
    Dim money2 = New Money(50D, "USD")

    ' Act
    Dim result = money1.Add(money2)

    ' Assert
    result.Should().Be(New Money(150D, "USD"))
End Sub

<Fact>
Public Sub Money_Add_DifferentCurrency_ThrowsException()
    ' Arrange
    Dim usd = New Money(100D, "USD")
    Dim eur = New Money(50D, "EUR")

    ' Act & Assert
    Dim act As Action = Sub() usd.Add(eur)
    act.Should().Throw(Of InvalidOperationException)() _
       .WithMessage("*different currencies*")
End Sub
```
