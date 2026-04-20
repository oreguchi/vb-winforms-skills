# Value Objects and Pattern Matching

Full code examples for value objects and pattern-based branching in modern VB.NET.

## Contents

- [Value Objects as Structure](#value-objects-as-structure)
- [Constraint-Enforcing Value Objects](#constraint-enforcing-value-objects)
- [No Implicit Conversions](#no-implicit-conversions)
- [Pattern-Style Branching (VB.NET equivalent of C# Pattern Matching)](#pattern-style-branching-vbnet-equivalent-of-c-pattern-matching)

> **VB.NET language notes applied throughout this file**
> - VB.NET has **no `record` / `readonly record struct`**. Use `Public Structure` (for value types) or `Public Class` (for reference types) with `ReadOnly` properties and manual `Equals` / `GetHashCode`.
> - VB.NET has **no `with` expression**. Provide a manual `With*` copy method.
> - VB.NET has **no `switch` expression, property patterns, relational patterns, or list patterns**. Use `Select Case`, `If`/`ElseIf`, `TypeOf`, and `TryCast`.
> - All samples assume `Option Strict On`.

## Value Objects as Structure

Value objects should **always be a `Structure`** (value type) for performance and value semantics. Because VB.NET does not auto-generate structural equality, override `Equals` and `GetHashCode` when value equality matters.

```vb
Imports System.Globalization

' Single-value object
Public Structure OrderId
    Public ReadOnly Property Value As String

    Public Sub New(value As String)
        If String.IsNullOrWhiteSpace(value) Then
            Throw New ArgumentException("OrderId cannot be empty", NameOf(value))
        End If
        Me.Value = value
    End Sub

    Public Overrides Function ToString() As String
        Return Value
    End Function

    ' NO implicit conversions - defeats type safety!
    ' Access inner value explicitly: orderId.Value

    Public Overrides Function Equals(obj As Object) As Boolean
        If TypeOf obj Is OrderId Then
            Return String.Equals(DirectCast(obj, OrderId).Value, Value, StringComparison.Ordinal)
        End If
        Return False
    End Function

    Public Overrides Function GetHashCode() As Integer
        Return If(Value, String.Empty).GetHashCode()
    End Function
End Structure

' Multi-value object
Public Structure Money
    Public ReadOnly Property Amount As Decimal
    Public ReadOnly Property Currency As String

    Public Sub New(amount As Decimal, currency As String)
        If amount < 0D Then
            Throw New ArgumentException("Amount cannot be negative", NameOf(amount))
        End If
        Me.Amount = amount
        Me.Currency = ValidateCurrency(currency)
    End Sub

    Private Shared Function ValidateCurrency(currency As String) As String
        If String.IsNullOrWhiteSpace(currency) OrElse currency.Length <> 3 Then
            Throw New ArgumentException("Currency must be a 3-letter code", NameOf(currency))
        End If
        Return currency.ToUpperInvariant()
    End Function

    Public Function Add(other As Money) As Money
        If Currency <> other.Currency Then
            Throw New InvalidOperationException($"Cannot add {Currency} to {other.Currency}")
        End If
        Return New Money(Amount + other.Amount, Currency)
    End Function

    Public Overrides Function ToString() As String
        Return $"{Amount.ToString("N2", CultureInfo.InvariantCulture)} {Currency}"
    End Function

    Public Overrides Function Equals(obj As Object) As Boolean
        If TypeOf obj Is Money Then
            Dim other = DirectCast(obj, Money)
            Return Amount = other.Amount AndAlso Currency = other.Currency
        End If
        Return False
    End Function

    Public Overrides Function GetHashCode() As Integer
        Return HashCode.Combine(Amount, Currency)
    End Function
End Structure

' Value object with input normalization
Public Structure PhoneNumber
    Public ReadOnly Property Value As String

    Public Sub New(input As String)
        If String.IsNullOrWhiteSpace(input) Then
            Throw New ArgumentException("Phone number cannot be empty", NameOf(input))
        End If

        ' Normalize: remove all non-digits
        Dim digits = New String(input.Where(Function(c) Char.IsDigit(c)).ToArray())

        If digits.Length < 10 OrElse digits.Length > 15 Then
            Throw New ArgumentException("Phone number must be 10-15 digits", NameOf(input))
        End If

        Me.Value = digits
    End Sub

    Public Overrides Function ToString() As String
        Return Value
    End Function
End Structure

' Percentage value object with range validation
Public Structure Percentage
    Private ReadOnly _value As Decimal

    Public ReadOnly Property Value As Decimal
        Get
            Return _value
        End Get
    End Property

    Public Sub New(value As Decimal)
        If value < 0D OrElse value > 100D Then
            Throw New ArgumentOutOfRangeException(NameOf(value), "Percentage must be between 0 and 100")
        End If
        _value = value
    End Sub

    Public Function AsDecimal() As Decimal
        Return _value / 100D
    End Function

    Public Shared Function FromDecimal(decimalValue As Decimal) As Percentage
        If decimalValue < 0D OrElse decimalValue > 1D Then
            Throw New ArgumentOutOfRangeException(NameOf(decimalValue), "Decimal must be between 0 and 1")
        End If
        Return New Percentage(decimalValue * 100D)
    End Function

    Public Overrides Function ToString() As String
        Return $"{_value}%"
    End Function
End Structure

' Strongly-typed ID
Public Structure CustomerId
    Public ReadOnly Property Value As Guid

    Public Sub New(value As Guid)
        Me.Value = value
    End Sub

    Public Shared Function [New]() As CustomerId
        Return New CustomerId(Guid.NewGuid())
    End Function

    Public Overrides Function ToString() As String
        Return Value.ToString()
    End Function
End Structure

' Quantity with units
Public Structure Quantity
    Public ReadOnly Property Value As Integer
    Public ReadOnly Property Unit As String

    Public Sub New(value As Integer, unit As String)
        If value < 0 Then
            Throw New ArgumentException("Quantity cannot be negative")
        End If
        If String.IsNullOrWhiteSpace(unit) Then
            Throw New ArgumentException("Unit cannot be empty")
        End If
        Me.Value = value
        Me.Unit = unit
    End Sub

    Public Overrides Function ToString() As String
        Return $"{Value} {Unit}"
    End Function
End Structure
```

**Why `Structure` for value objects in VB.NET:**
- **Value semantics**: Value-type copy semantics; equality based on content when `Equals` is implemented.
- **Stack allocation**: Better performance, no GC pressure for small value types.
- **Immutability**: `ReadOnly` properties / `ReadOnly` fields prevent accidental mutation.
- **Works with branching**: Can be used directly in `Select Case` and `If`/`ElseIf` predicates.

> **VB.NET note — no `readonly record struct`:** C# `readonly record struct` gives structural equality, `with` support, and a positional constructor automatically. VB.NET does not. Override `Equals` and `GetHashCode` manually (as above) and provide explicit `With*` copy methods when needed.

## Constraint-Enforcing Value Objects

Value objects aren't just for identifiers. They're equally valuable for **enforcing domain constraints** on strings, numbers, and URIs — making illegal states unrepresentable at the type level.

**Key principle: validate at construction, trust everywhere else.** Once you have an `AbsoluteUrl`, every consumer knows it's valid without re-checking.

```vb
' AbsoluteUrl - enforces HTTP/HTTPS scheme constraints
Public Structure AbsoluteUrl
    Public ReadOnly Property Value As Uri

    Public Sub New(uriString As String)
        Me.New(New Uri(uriString, UriKind.Absolute))
    End Sub

    Public Sub New(value As Uri)
        If value Is Nothing Then
            Throw New ArgumentNullException(NameOf(value))
        End If
        If Not value.IsAbsoluteUri Then
            Throw New ArgumentException(
                $"Value must be an absolute URL. Instead found [{value}]", NameOf(value))
        End If
        If value.Scheme <> Uri.UriSchemeHttp AndAlso value.Scheme <> Uri.UriSchemeHttps Then
            Throw New ArgumentException(
                $"Value must be an HTTP or HTTPS URL. Instead found [{value.Scheme}]", NameOf(value))
        End If
        Me.Value = value
    End Sub

    ''' <summary>
    ''' Resolves a potentially relative URL against a base URL.
    ''' Handles Linux quirk where Uri.TryCreate("/path", UriKind.Absolute)
    ''' succeeds as file:///path.
    ''' </summary>
    Public Shared Function FromRelative(url As String, baseUrl As AbsoluteUrl) As AbsoluteUrl
        If String.IsNullOrEmpty(url) Then
            Throw New ArgumentException("URL cannot be null or empty", NameOf(url))
        End If

        Dim absoluteUri As Uri = Nothing
        If Uri.TryCreate(url, UriKind.Absolute, absoluteUri) AndAlso
           (absoluteUri.Scheme = Uri.UriSchemeHttp OrElse absoluteUri.Scheme = Uri.UriSchemeHttps) Then
            Return New AbsoluteUrl(absoluteUri)
        End If

        Return New AbsoluteUrl(New Uri(baseUrl.Value, url))
    End Function

    Public Overrides Function ToString() As String
        Return Value.ToString()
    End Function
End Structure

' NonEmptyString - prevents empty/whitespace strings from propagating
Public Structure NonEmptyString
    Public ReadOnly Property Value As String

    Public Sub New(value As String)
        If String.IsNullOrWhiteSpace(value) Then
            Throw New ArgumentException("Value cannot be null or whitespace", NameOf(value))
        End If
        Me.Value = value
    End Sub

    Public Overrides Function ToString() As String
        Return Value
    End Function
End Structure

' EmailAddress - format validation at construction
Public Structure EmailAddress
    Public ReadOnly Property Value As String

    Public Sub New(value As String)
        If String.IsNullOrWhiteSpace(value) Then
            Throw New ArgumentException("Email cannot be empty", NameOf(value))
        End If
        If Not value.Contains("@"c) OrElse Not value.Contains("."c) Then
            Throw New ArgumentException($"Invalid email format: {value}", NameOf(value))
        End If
        Me.Value = value.ToLowerInvariant()
    End Sub

    Public Overrides Function ToString() As String
        Return Value
    End Function
End Structure

' PositiveAmount - numeric range constraint
Public Structure PositiveAmount
    Public ReadOnly Property Value As Decimal

    Public Sub New(value As Decimal)
        If value <= 0D Then
            Throw New ArgumentOutOfRangeException(NameOf(value), "Amount must be positive")
        End If
        Me.Value = value
    End Sub

    Public Overrides Function ToString() As String
        Return Value.ToString("N2", System.Globalization.CultureInfo.InvariantCulture)
    End Function
End Structure
```

**Why this matters:**
- APIs like Slack Block Kit silently reject relative URLs with cryptic errors. Transactional email links break if they're relative. `AbsoluteUrl` makes the compiler prevent this.
- Platform gotchas belong in the value object — e.g., Linux `Uri.TryCreate` treating `/path` as `file:///path` is handled once in `FromRelative`, not at every call site.

### TypeConverter Support for Configuration Binding

Add a `TypeConverter` so your value objects work with `IOptions(Of T)` and configuration binding:

```vb
Imports System.ComponentModel
Imports System.Globalization

<TypeConverter(GetType(AbsoluteUrlTypeConverter))>
Public Structure AbsoluteUrl
    ' ... same as above
End Structure

Public NotInheritable Class AbsoluteUrlTypeConverter
    Inherits TypeConverter

    Public Overrides Function CanConvertFrom(
            context As ITypeDescriptorContext, sourceType As Type) As Boolean
        Return sourceType Is GetType(String) OrElse MyBase.CanConvertFrom(context, sourceType)
    End Function

    Public Overrides Function ConvertFrom(
            context As ITypeDescriptorContext,
            culture As CultureInfo,
            value As Object) As Object
        Dim s = TryCast(value, String)
        If s IsNot Nothing Then
            Return New AbsoluteUrl(s)
        End If
        Return MyBase.ConvertFrom(context, culture, value)
    End Function
End Class

' Now this works with appsettings.json binding:
Public NotInheritable Class WebhookOptions
    Public Property CallbackUrl As AbsoluteUrl
    Public Property HealthCheckUrl As AbsoluteUrl
End Class

' appsettings.json:
' { "Webhook": { "CallbackUrl": "https://example.com/callback" } }
' services.Configure(Of WebhookOptions)(configuration.GetSection("Webhook"))
```

## No Implicit Conversions

**CRITICAL: NO implicit conversions.** Widening operators defeat the purpose of value objects by allowing silent type coercion. In VB.NET, custom conversion operators are written as `Public Shared Widening Operator CType(...)` (implicit) or `Public Shared Narrowing Operator CType(...)` (explicit). Avoid `Widening` entirely for value objects.

```vb
' WRONG - defeats compile-time safety:
Public Structure UserId
    Public ReadOnly Property Value As Guid

    Public Sub New(value As Guid)
        Me.Value = value
    End Sub

    ' NO! Silent Guid -> UserId conversion:
    Public Shared Widening Operator CType(value As Guid) As UserId
        Return New UserId(value)
    End Operator

    ' NO! Silent UserId -> Guid conversion:
    Public Shared Widening Operator CType(value As UserId) As Guid
        Return value.Value
    End Operator
End Structure

' With widening operators, this compiles silently:
Sub ProcessUser(userId As UserId)
End Sub
ProcessUser(Guid.NewGuid())   ' Oops - meant to pass PostId

' CORRECT - all conversions explicit:
Public Structure UserId
    Public ReadOnly Property Value As Guid

    Public Sub New(value As Guid)
        Me.Value = value
    End Sub

    Public Shared Function [New]() As UserId
        Return New UserId(Guid.NewGuid())
    End Function
    ' No widening operators.
    ' Create: New UserId(guid) or UserId.New()
    ' Extract: userId.Value
End Structure
```

Explicit conversions force every boundary crossing to be visible:

```vb
' API boundary - explicit conversion IN
Dim userId = New UserId(request.UserId)   ' Validates on entry

' Database boundary - explicit conversion OUT
Await _db.ExecuteAsync(sql, New With {.UserId = userId.Value})
```

## Pattern-Style Branching (VB.NET equivalent of C# Pattern Matching)

C# offers `switch` expressions, property patterns (`{ Prop: value }`), relational patterns (`< 10`), and list patterns (`[first, .., last]`). **None of these are available in VB.NET.** Use `Select Case`, `If`/`ElseIf`, `TypeOf`, and `TryCast` to achieve the same outcomes.

```vb
' Multi-branch dispatch with value objects - equivalent to a C# switch expression with property patterns
Public Function GetPaymentMethodDescription(payment As PaymentMethod) As String
    Select Case payment.Type
        Case PaymentType.CreditCard
            Return $"Credit card ending in {payment.Last4}"
        Case PaymentType.BankTransfer
            Return $"Bank transfer from {payment.AccountNumber}"
        Case PaymentType.Cash
            Return "Cash payment"
        Case Else
            Return "Unknown payment method"
    End Select
End Function

' Property-pattern equivalent using If/ElseIf on properties
Public Function CalculateDiscount(order As Order) As Decimal
    If order.Total > 1000D Then
        Return order.Total * 0.15D
    ElseIf order.Total > 500D Then
        Return order.Total * 0.1D
    ElseIf order.Total > 100D Then
        Return order.Total * 0.05D
    Else
        Return 0D
    End If
End Function

' Relational / range patterns - use Select Case with range syntax
Public Function ClassifyTemperature(temp As Integer) As String
    Select Case temp
        Case Is < 0
            Return "Freezing"
        Case 0 To 9
            Return "Cold"
        Case 10 To 19
            Return "Cool"
        Case 20 To 29
            Return "Warm"
        Case Is >= 30
            Return "Hot"
        Case Else
            Throw New ArgumentOutOfRangeException(NameOf(temp))
    End Select
End Function

' "List patterns" - VB.NET has no list pattern syntax. Inspect Length / indexes directly.
Public Function IsValidSequence(numbers As Integer()) As Boolean
    If numbers Is Nothing OrElse numbers.Length = 0 Then
        Return False                                        ' Empty
    End If
    If numbers.Length = 1 Then
        Return True                                         ' Single element
    End If
    Return numbers(0) < numbers(numbers.Length - 1)         ' First < last
End Function

' Type patterns with null checks - use TypeOf / DirectCast
Public Function FormatValue(value As Object) As String
    If value Is Nothing Then Return "null"
    If TypeOf value Is String Then
        Return $"""{DirectCast(value, String)}"""
    ElseIf TypeOf value Is Integer Then
        Return DirectCast(value, Integer).ToString()
    ElseIf TypeOf value Is Double Then
        Return DirectCast(value, Double).ToString("F2")
    ElseIf TypeOf value Is DateTime Then
        Return DirectCast(value, DateTime).ToString("yyyy-MM-dd")
    ElseIf TypeOf value Is Money Then
        Return DirectCast(value, Money).ToString()
    ElseIf TypeOf value Is IEnumerable(Of Object) Then
        Return $"[{String.Join(", ", DirectCast(value, IEnumerable(Of Object)))}]"
    Else
        Return If(value.ToString(), "unknown")
    End If
End Function

' Combining patterns for complex logic
Public Class OrderState
    Public ReadOnly Property IsPaid As Boolean
    Public ReadOnly Property IsShipped As Boolean
    Public ReadOnly Property IsCancelled As Boolean

    Public Sub New(isPaid As Boolean, isShipped As Boolean, isCancelled As Boolean)
        Me.IsPaid = isPaid
        Me.IsShipped = isShipped
        Me.IsCancelled = isCancelled
    End Sub
End Class

Public Function GetOrderStatus(state As OrderState) As String
    If state.IsCancelled Then Return "Cancelled"
    If state.IsPaid AndAlso state.IsShipped Then Return "Delivered"
    If state.IsPaid AndAlso Not state.IsShipped Then Return "Processing"
    If Not state.IsPaid Then Return "Awaiting Payment"
    Return "Unknown"
End Function

' Tuple-style "switch" on multiple values - use nested If/Select Case
Public Function CalculateShipping(total As Money, destination As Country) As Decimal
    If total.Amount > 100D Then Return 0D              ' Free shipping over $100

    Select Case destination.Code
        Case "US", "CA"
            Return 5D                                   ' North America
        Case "GB", "FR", "DE"
            Return 10D                                  ' Europe
        Case Else
            Return 25D                                  ' International
    End Select
End Function
```

> **VB.NET note — `switch` expression equivalence:** The `If(cond, a, b)` ternary expression is a closer analogue of a two-arm `switch` expression when you only have a single predicate. For multi-arm expressions, assign the result inside a `Select Case` or `If`/`ElseIf` chain to a local variable.
