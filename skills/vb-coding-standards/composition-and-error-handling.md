# コンポジションとエラー処理

VB.NET における継承よりコンポジション、Result 型パターン、テストパターン。

## 目次

- [継承よりコンポジション](#継承よりコンポジション)
- [Result 型パターン](#result-型パターン)
- [テストパターン](#テストパターン)

## 継承よりコンポジション

**抽象基底クラスと継承階層を避ける。** 代わりにコンポジションとインターフェイスを使用する。

```vbnet
' 悪い例：抽象基底クラスの階層
Public MustInherit Class PaymentProcessor
    Public MustOverride Function ProcessAsync(amount As Money) As Task(Of PaymentResult)

    Protected Async Function ValidateAsync(amount As Money) As Task(Of Boolean)
        ' 共有バリデーションロジック
        Return amount.Amount > 0
    End Function
End Class

Public Class CreditCardProcessor
    Inherits PaymentProcessor

    Public Overrides Async Function ProcessAsync(amount As Money) As Task(Of PaymentResult)
        Await ValidateAsync(amount)
        ' クレジットカードの処理...
        Return Nothing ' 仮の返値
    End Function
End Class

' 良い例：インターフェイスを使ったコンポジション
Public Interface IPaymentProcessor
    Function ProcessAsync(amount As Money, cancellationToken As CancellationToken) As Task(Of PaymentResult)
End Interface

Public Interface IPaymentValidator
    Function ValidateAsync(amount As Money, cancellationToken As CancellationToken) As Task(Of ValidationResult)
End Interface

' 具体的な実装はバリデータをコンポーズする
Public NotInheritable Class CreditCardProcessor
    Implements IPaymentProcessor

    Private ReadOnly _validator As IPaymentValidator
    Private ReadOnly _gateway As ICreditCardGateway

    Public Sub New(validator As IPaymentValidator, gateway As ICreditCardGateway)
        _validator = validator
        _gateway = gateway
    End Sub

    Public Async Function ProcessAsync(
        amount As Money,
        cancellationToken As CancellationToken) As Task(Of PaymentResult) _
        Implements IPaymentProcessor.ProcessAsync

        Dim validation = Await _validator.ValidateAsync(amount, cancellationToken)
        If Not validation.IsValid Then
            Return PaymentResult.Failed(validation.Error)
        End If

        Return Await _gateway.ChargeAsync(amount, cancellationToken)
    End Function
End Class

' 良い例：共有ロジックには静的ヘルパークラス（継承なし）
Public Module PaymentValidation
    Public Function ValidateAmount(amount As Money) As ValidationResult
        If amount.Amount <= 0 Then
            Return ValidationResult.Invalid("金額は正の値でなければなりません。")
        End If

        If amount.Amount > 10000D Then
            Return ValidationResult.Invalid("金額が上限を超えています。")
        End If

        Return ValidationResult.Valid()
    End Function
End Module

' 良い例：バリアントのモデリングにはファクトリメソッドを持つクラスを使う
' （VB.NETには record に直接対応なし。§3bに基づきClass+ファクトリメソッドで代替）
Public Enum PaymentType
    CreditCard
    BankTransfer
    Cash
End Enum

Public NotInheritable Class PaymentMethod
    Public ReadOnly Property Type As PaymentType
    Public ReadOnly Property Last4 As String        ' クレジットカード用
    Public ReadOnly Property AccountNumber As String ' 銀行振込用
    Public ReadOnly Property CashAmount As Decimal   ' 現金払い用

    ' すべてのパラメータを受け取る Private コンストラクタ
    ' （VB.NETには init-only setter に直接対応なし。§3bに基づき Private Sub New + ReadOnly Property で代替）
    Private Sub New(type As PaymentType,
                    last4 As String,
                    accountNumber As String,
                    cashAmount As Decimal)
        Me.Type = type
        Me.Last4 = last4
        Me.AccountNumber = accountNumber
        Me.CashAmount = cashAmount
    End Sub

    Public Shared Function CreditCard(last4 As String) As PaymentMethod
        Return New PaymentMethod(PaymentType.CreditCard, last4, Nothing, 0D)
    End Function

    Public Shared Function BankTransfer(accountNumber As String) As PaymentMethod
        Return New PaymentMethod(PaymentType.BankTransfer, Nothing, accountNumber, 0D)
    End Function

    Public Shared Function Cash(amount As Decimal) As PaymentMethod
        Return New PaymentMethod(PaymentType.Cash, Nothing, Nothing, amount)
    End Function
End Class
```

**継承が許容される場合：**
- フレームワーク要件（例：ASP.NET Core の `ControllerBase`）
- ライブラリ統合（例：`Exception` を継承するカスタム例外）
- **これらはアプリケーションコードではまれなケースであるべき**

## Result 型パターン

期待されるエラーには、例外ではなく**ドメイン固有の結果型**を使用する。汎用的な `Result(Of T)` を作らず、各操作が成功・失敗の形をドメインで定義し、結果型でそれを表現する。ファクトリメソッドと列挙型エラーコードを持つ Sealed クラスを使う。

```vbnet
' エラー分類用の列挙型 — 型安全で Select Case で分岐可能
Public Enum OrderErrorCode
    ValidationError
    InsufficientInventory
    NotFound
End Enum

' ドメイン固有の結果型 — ファクトリメソッドを持つ Sealed クラス
' （VB.NETには sealed record に直接対応なし。§3bに基づきClass+ファクトリメソッドで代替）
Public NotInheritable Class CreateOrderResult
    Public ReadOnly Property IsSuccess As Boolean
    Public ReadOnly Property Order As Order
    Public ReadOnly Property ErrorCode As Nullable(Of OrderErrorCode)
    Public ReadOnly Property ErrorMessage As String

    ' すべてのパラメータを受け取る Private コンストラクタ（内部使用）
    ' （VB.NETには init-only setter に直接対応なし。§3bに基づき Private Sub New + ReadOnly Property で代替）
    Private Sub New(
        isSuccess As Boolean,
        order As Order,
        errorCode As Nullable(Of OrderErrorCode),
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

' 使用例
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
                OrderErrorCode.ValidationError, "無効な注文リクエストです。")
        End If

        If Not Await HasInventoryAsync(request.Items, cancellationToken) Then
            Return CreateOrderResult.Failed(
                OrderErrorCode.InsufficientInventory, "商品の在庫がありません。")
        End If

        Dim order As New Order(
            OrderId.NewId(),
            New CustomerId(request.CustomerId),
            request.Items)

        Await _repository.SaveAsync(order, cancellationToken)

        Return CreateOrderResult.Success(order)
    End Function

    ' 結果を HTTP レスポンスにマップ — 列挙型エラーコードで分岐
    ' （VB.NETには switch 式に直接対応なし。§3bに基づきSelect Caseで代替）
    Public Function MapToActionResult(result As CreateOrderResult) As IActionResult
        If result.IsSuccess Then
            Return New OkObjectResult(result.Order)
        End If

        Select Case result.ErrorCode
            Case OrderErrorCode.ValidationError
                Return New BadRequestObjectResult(New With {.error = result.ErrorMessage})
            Case OrderErrorCode.InsufficientInventory
                Return New ConflictObjectResult(New With {.error = result.ErrorMessage})
            Case OrderErrorCode.NotFound
                Return New NotFoundObjectResult(New With {.error = result.ErrorMessage})
            Case Else
                Return New ObjectResult(New With {.error = result.ErrorMessage}) With {.StatusCode = 500}
        End Select
    End Function
End Class
```

**Result を使うタイミングと例外を使うタイミング：**
- **Result を使う**：期待されるエラー（バリデーション、ビジネスルール、Not Found）
- **例外を使う**：予期しないエラー（ネットワーク障害、システムエラー、プログラミングバグ）

## テストパターン

```vbnet
' テストデータビルダー用クラス
' （VB.NETには record に直接対応なし。§3bに基づきClassで代替）
' 注意：C# の `with` 式はVB.NETにないため、各プロパティを個別設定するか
'       ヘルパーメソッド WithXxx(...) を手書きする。
Public Class OrderBuilder
    Public Property Id As OrderId = OrderId.NewId()
    Public Property CustomerId As CustomerId = CustomerId.NewId()
    Public Property Total As Money = New Money(100D, "USD")
    Public Property Items As IReadOnlyList(Of OrderItem) = Array.Empty(Of OrderItem)()

    Public Function Build() As Order
        Return New Order(Id, CustomerId, Total, Items)
    End Function

    ' `with` 式の代替 — ヘルパーメソッドを手書きする
    ' （VB.NETには `with` 式に直接対応なし。§3bに基づきWith*メソッドで代替）
    Public Function WithTotal(total As Money) As OrderBuilder
        Dim copy As New OrderBuilder()
        copy.Id = Me.Id
        copy.CustomerId = Me.CustomerId
        copy.Total = total
        copy.Items = Me.Items
        Return copy
    End Function
End Class

' `with` 式を使ったテストバリエーション（C#）の VB.NET 代替
<Fact>
Public Sub CalculateDiscount_LargeOrder_AppliesCorrectDiscount()
    ' Arrange
    Dim baseOrder = New OrderBuilder().Build()
    ' C# では: var largeOrder = baseOrder with { Total = new Money(1500m, "USD") };
    ' VB.NET では WithTotal ヘルパーを使う
    Dim largeOrder = New OrderBuilder().WithTotal(New Money(1500D, "USD")).Build()

    ' Act
    Dim discount = _service.CalculateDiscount(largeOrder)

    ' Assert
    discount.Should().Be(New Money(225D, "USD")) ' 1500 の 15%
End Sub

' 文字列ベースのパーステスト（VB.NET では Span(Of T) をシグネチャ・ローカル変数とも使えないため、
' OrderIdParser.TryParse は String 引数を受け取る形に合わせる）
<Theory>
<InlineData("ORD-12345", True)>
<InlineData("INVALID", False)>
Public Sub TryParseOrderId_VariousInputs_ReturnsExpectedResult(
    input As String,
    expected As Boolean)

    ' Act
    Dim orderId As OrderId
    Dim result = OrderIdParser.TryParse(input, orderId)

    ' Assert
    result.Should().Be(expected)
End Sub

' 値オブジェクトを使ったテスト
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
    Dim act = Sub() usd.Add(eur)
    act.Should().Throw(Of InvalidOperationException)().
        WithMessage("*different currencies*")
End Sub
```
