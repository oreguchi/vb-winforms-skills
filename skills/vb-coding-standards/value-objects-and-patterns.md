# 値オブジェクトとパターンマッチング

VB.NET における値オブジェクトと条件分岐パターンの完全なコード例。

## 目次

- [Structure + ReadOnly による値オブジェクト](#structure--readonly-による値オブジェクト)
- [制約を強制する値オブジェクト](#制約を強制する値オブジェクト)
- [暗黙的な変換は禁止](#暗黙的な変換は禁止)
- [条件分岐パターン（Select Case と If/ElseIf）](#条件分岐パターンselect-case-と-ifelseif)

## Structure + ReadOnly による値オブジェクト

値オブジェクトにはパフォーマンスと値セマンティクスのために常に `Structure` + `ReadOnly` プロパティ + 手書きの `Equals`/`GetHashCode` を使用する。

（VB.NETには `readonly record struct` に直接対応なし。§3bに基づきStructure+手書きEqualsで代替）

```vbnet
' 単一値オブジェクト
Public Structure OrderId
    Public ReadOnly Property Value As String

    Public Sub New(value As String)
        If String.IsNullOrWhiteSpace(value) Then
            Throw New ArgumentException("OrderId を空にすることはできません。", NameOf(value))
        End If
        Me.Value = value
    End Sub

    Public Overrides Function ToString() As String
        Return Value
    End Function

    Public Overrides Function Equals(obj As Object) As Boolean
        If TypeOf obj Is OrderId Then
            Return DirectCast(obj, OrderId).Value = Value
        End If
        Return False
    End Function

    Public Overrides Function GetHashCode() As Integer
        Return If(Value IsNot Nothing, Value.GetHashCode(), 0)
    End Function

    ' 暗黙的な変換は禁止 — 型安全性を損なう！
    ' 内部値へのアクセスは orderId.Value を使うこと
End Structure

' 複数値オブジェクト
Public Structure Money
    Public ReadOnly Property Amount As Decimal
    Public ReadOnly Property Currency As String

    Public Sub New(amount As Decimal, currency As String)
        If amount < 0 Then
            Throw New ArgumentException("金額は負の値にできません。", NameOf(amount))
        End If
        Me.Amount = amount
        Me.Currency = ValidateCurrency(currency)
    End Sub

    Private Shared Function ValidateCurrency(currency As String) As String
        If String.IsNullOrWhiteSpace(currency) OrElse currency.Length <> 3 Then
            Throw New ArgumentException("通貨は 3 文字のコードでなければなりません。", NameOf(currency))
        End If
        Return currency.ToUpperInvariant()
    End Function

    Public Function Add(other As Money) As Money
        If Currency <> other.Currency Then
            Throw New InvalidOperationException(
                $"{Currency} と {other.Currency} を加算することはできません。")
        End If
        Return New Money(Amount + other.Amount, Currency)
    End Function

    Public Overrides Function ToString() As String
        Return $"{Amount:N2} {Currency}"
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

' 入力正規化を行う値オブジェクト
Public Structure PhoneNumber
    Public ReadOnly Property Value As String

    Public Sub New(input As String)
        If String.IsNullOrWhiteSpace(input) Then
            Throw New ArgumentException("電話番号を空にすることはできません。", NameOf(input))
        End If

        ' 正規化：数字以外を除去
        Dim digits = New String(input.Where(AddressOf Char.IsDigit).ToArray())

        If digits.Length < 10 OrElse digits.Length > 15 Then
            Throw New ArgumentException("電話番号は 10〜15 桁でなければなりません。", NameOf(input))
        End If

        Value = digits
    End Sub

    Public Overrides Function ToString() As String
        Return Value
    End Function

    Public Overrides Function Equals(obj As Object) As Boolean
        If TypeOf obj Is PhoneNumber Then
            Return DirectCast(obj, PhoneNumber).Value = Value
        End If
        Return False
    End Function

    Public Overrides Function GetHashCode() As Integer
        Return If(Value IsNot Nothing, Value.GetHashCode(), 0)
    End Function
End Structure

' 範囲バリデーション付きパーセンテージ値オブジェクト
Public Structure Percentage
    Private ReadOnly _value As Decimal

    Public ReadOnly Property Value As Decimal
        Get
            Return _value
        End Get
    End Property

    Public Sub New(value As Decimal)
        If value < 0 OrElse value > 100 Then
            Throw New ArgumentOutOfRangeException(NameOf(value), "パーセンテージは 0 から 100 の間でなければなりません。")
        End If
        _value = value
    End Sub

    Public Function AsDecimal() As Decimal
        Return _value / 100D
    End Function

    Public Shared Function FromDecimal(decimalValue As Decimal) As Percentage
        If decimalValue < 0 OrElse decimalValue > 1 Then
            Throw New ArgumentOutOfRangeException(NameOf(decimalValue), "小数値は 0 から 1 の間でなければなりません。")
        End If
        Return New Percentage(decimalValue * 100)
    End Function

    Public Overrides Function ToString() As String
        Return $"{_value}%"
    End Function

    Public Overrides Function Equals(obj As Object) As Boolean
        If TypeOf obj Is Percentage Then
            Return DirectCast(obj, Percentage).Value = _value
        End If
        Return False
    End Function

    Public Overrides Function GetHashCode() As Integer
        Return _value.GetHashCode()
    End Function
End Structure

' 強い型付き ID
Public Structure CustomerId
    Public ReadOnly Property Value As Guid

    Public Sub New(value As Guid)
        Me.Value = value
    End Sub

    Public Shared Function NewId() As CustomerId
        Return New CustomerId(Guid.NewGuid())
    End Function

    Public Overrides Function ToString() As String
        Return Value.ToString()
    End Function

    Public Overrides Function Equals(obj As Object) As Boolean
        If TypeOf obj Is CustomerId Then
            Return DirectCast(obj, CustomerId).Value = Value
        End If
        Return False
    End Function

    Public Overrides Function GetHashCode() As Integer
        Return Value.GetHashCode()
    End Function
End Structure

' 単位付き数量
Public Structure Quantity
    Public ReadOnly Property Value As Integer
    Public ReadOnly Property Unit As String

    Public Sub New(value As Integer, unit As String)
        If value < 0 Then
            Throw New ArgumentException("数量は負の値にできません。")
        End If
        If String.IsNullOrWhiteSpace(unit) Then
            Throw New ArgumentException("単位を空にすることはできません。")
        End If
        Me.Value = value
        Me.Unit = unit
    End Sub

    Public Overrides Function ToString() As String
        Return $"{Value} {Unit}"
    End Function

    Public Overrides Function Equals(obj As Object) As Boolean
        If TypeOf obj Is Quantity Then
            Dim other = DirectCast(obj, Quantity)
            Return Value = other.Value AndAlso Unit = other.Unit
        End If
        Return False
    End Function

    Public Overrides Function GetHashCode() As Integer
        Return HashCode.Combine(Value, Unit)
    End Function
End Structure
```

**値オブジェクトに `Structure` + `ReadOnly` を使う理由：**
- **値セマンティクス**：参照ではなく内容に基づく等価性
- **スタックアロケーション**：パフォーマンス向上、GC 圧力なし
- **不変性**：`ReadOnly` が意図しない変更を防ぐ
- **明確な境界**：手書きの `Equals`/`GetHashCode` によって等価性が明示的

## 制約を強制する値オブジェクト

値オブジェクトは識別子だけのためにあるのではない。文字列、数値、URI のドメイン制約を強制するためにも同様に有用であり、型レベルで不正な状態を表現できなくする。

**重要な原則：構築時にバリデーション、その後は信頼。** `AbsoluteUrl` を持っていれば、すべての呼び出し元は再チェックなしに有効性を知っている。

```vbnet
' AbsoluteUrl — HTTP/HTTPS スキームの制約を強制
Public Structure AbsoluteUrl
    Public ReadOnly Property Value As Uri

    Public Sub New(uriString As String)
        Me.New(New Uri(uriString, UriKind.Absolute))
    End Sub

    Public Sub New(value As Uri)
        If Not value.IsAbsoluteUri Then
            Throw New ArgumentException(
                $"値は絶対 URL でなければなりません。代わりに [{value}] が見つかりました。",
                NameOf(value))
        End If
        If value.Scheme <> Uri.UriSchemeHttp AndAlso value.Scheme <> Uri.UriSchemeHttps Then
            Throw New ArgumentException(
                $"値は HTTP または HTTPS URL でなければなりません。代わりに [{value.Scheme}] が見つかりました。",
                NameOf(value))
        End If
        Me.Value = value
    End Sub

    ''' <summary>
    ''' ベース URL に対して相対 URL を絶対 URL に解決する。
    ''' Linux の癖（Uri.TryCreate("/path", UriKind.Absolute) が
    ''' file:///path として成功する）を処理する。
    ''' </summary>
    Public Shared Function FromRelative(url As String, baseUrl As AbsoluteUrl) As AbsoluteUrl
        If String.IsNullOrEmpty(url) Then
            Throw New ArgumentException("URL を null または空にすることはできません。", NameOf(url))
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

    Public Overrides Function Equals(obj As Object) As Boolean
        If TypeOf obj Is AbsoluteUrl Then
            Return DirectCast(obj, AbsoluteUrl).Value = Value
        End If
        Return False
    End Function

    Public Overrides Function GetHashCode() As Integer
        Return If(Value IsNot Nothing, Value.GetHashCode(), 0)
    End Function
End Structure

' NonEmptyString — 空文字列/空白文字列の伝播を防ぐ
Public Structure NonEmptyString
    Public ReadOnly Property Value As String

    Public Sub New(value As String)
        If String.IsNullOrWhiteSpace(value) Then
            Throw New ArgumentException("値を null または空白にすることはできません。", NameOf(value))
        End If
        Me.Value = value
    End Sub

    Public Overrides Function ToString() As String
        Return Value
    End Function

    Public Overrides Function Equals(obj As Object) As Boolean
        If TypeOf obj Is NonEmptyString Then
            Return DirectCast(obj, NonEmptyString).Value = Value
        End If
        Return False
    End Function

    Public Overrides Function GetHashCode() As Integer
        Return If(Value IsNot Nothing, Value.GetHashCode(), 0)
    End Function
End Structure

' EmailAddress — 構築時にフォーマット検証
Public Structure EmailAddress
    Public ReadOnly Property Value As String

    Public Sub New(value As String)
        If String.IsNullOrWhiteSpace(value) Then
            Throw New ArgumentException("メールアドレスを空にすることはできません。", NameOf(value))
        End If
        If Not value.Contains("@") OrElse Not value.Contains(".") Then
            Throw New ArgumentException($"無効なメールフォーマット: {value}", NameOf(value))
        End If
        Me.Value = value.ToLowerInvariant()
    End Sub

    Public Overrides Function ToString() As String
        Return Value
    End Function

    Public Overrides Function Equals(obj As Object) As Boolean
        If TypeOf obj Is EmailAddress Then
            Return DirectCast(obj, EmailAddress).Value = Value
        End If
        Return False
    End Function

    Public Overrides Function GetHashCode() As Integer
        Return If(Value IsNot Nothing, Value.GetHashCode(), 0)
    End Function
End Structure

' PositiveAmount — 数値範囲制約
Public Structure PositiveAmount
    Public ReadOnly Property Value As Decimal

    Public Sub New(value As Decimal)
        If value <= 0 Then
            Throw New ArgumentOutOfRangeException(NameOf(value), "金額は正の値でなければなりません。")
        End If
        Me.Value = value
    End Sub

    Public Overrides Function ToString() As String
        Return Value.ToString("N2")
    End Function

    Public Overrides Function Equals(obj As Object) As Boolean
        If TypeOf obj Is PositiveAmount Then
            Return DirectCast(obj, PositiveAmount).Value = Value
        End If
        Return False
    End Function

    Public Overrides Function GetHashCode() As Integer
        Return Value.GetHashCode()
    End Function
End Structure
```

**この重要性：**
- Slack Block Kit などの API は相対 URL を暗号的なエラーでサイレントに拒否する。トランザクションメールのリンクが相対 URL だと壊れる。`AbsoluteUrl` によってコンパイラがこれを防ぐ。
- プラットフォーム固有の癖は値オブジェクトに閉じ込める。例：Linux での `Uri.TryCreate` が `/path` を `file:///path` として扱う問題は `FromRelative` で一度だけ処理し、すべての呼び出しサイトには持ち込まない。

### 設定バインディングのための TypeConverter サポート

値オブジェクトが `IOptions(Of T)` や設定バインディングと連携できるように `TypeConverter` を追加する。

```vbnet
<TypeConverter(GetType(AbsoluteUrlTypeConverter))>
Public Structure AbsoluteUrl
    ' ... 上記と同じ
End Structure

Public NotInheritable Class AbsoluteUrlTypeConverter
    Inherits TypeConverter

    Public Overrides Function CanConvertFrom(
        context As ITypeDescriptorContext,
        sourceType As Type) As Boolean

        Return sourceType = GetType(String) OrElse MyBase.CanConvertFrom(context, sourceType)
    End Function

    Public Overrides Function ConvertFrom(
        context As ITypeDescriptorContext,
        culture As CultureInfo,
        value As Object) As Object

        If TypeOf value Is String Then
            Return New AbsoluteUrl(DirectCast(value, String))
        End If
        Return MyBase.ConvertFrom(context, culture, value)
    End Function
End Class

' これで appsettings.json バインディングが機能する：
Public NotInheritable Class WebhookOptions
    Public Property CallbackUrl As AbsoluteUrl
    Public Property HealthCheckUrl As AbsoluteUrl
End Class

' appsettings.json:
' { "Webhook": { "CallbackUrl": "https://example.com/callback" } }
services.Configure(Of WebhookOptions)(configuration.GetSection("Webhook"))
```

## 暗黙的な変換は禁止

**重要：暗黙的な変換は禁止。** 暗黙的な演算子は、サイレントな型変換を許すことで値オブジェクトの目的を損なう。

```vbnet
' 誤り — コンパイル時の安全性を損なう：
Public Structure UserId
    Public ReadOnly Property Value As Guid

    ' これらを追加しないこと！
    ' Public Shared Widening Operator CType(value As Guid) As UserId → 禁止
    ' Public Shared Widening Operator CType(value As UserId) As Guid → 禁止
End Structure

' 暗黙的な演算子があると、これがサイレントにコンパイルされてしまう：
' Sub ProcessUser(userId As UserId) ... End Sub
' ProcessUser(Guid.NewGuid())  ' まずい — PostId を渡すつもりだったかも

' 正しい — すべての変換は明示的に：
Public Structure UserId
    Public ReadOnly Property Value As Guid

    Public Sub New(value As Guid)
        Me.Value = value
    End Sub

    Public Shared Function NewId() As UserId
        Return New UserId(Guid.NewGuid())
    End Function

    ' 暗黙的な演算子なし
    ' 作成：New UserId(guid) または UserId.NewId()
    ' 取り出し：userId.Value
End Structure
```

明示的な変換によって、すべての境界横断が可視化される。

```vbnet
' API 境界 — 明示的な変換（入力）
Dim userId = New UserId(request.UserId)  ' 入力時にバリデーション

' データベース境界 — 明示的な変換（出力）
Await _db.ExecuteAsync(sql, New With {.UserId = userId.Value})
```

## 条件分岐パターン（Select Case と If/ElseIf）

VB.NET では C# のパターンマッチング（switch 式、プロパティパターン、リレーショナルパターン、リストパターン）に直接対応する構文はない。`Select Case` 文や `If`/`ElseIf` チェーンで同等のロジックを実現する。

（VB.NETには switch 式・property pattern・relational pattern・list pattern に直接対応なし。§3bに基づきSelect Case文・If/ElseIf・TypeOf...Isで代替）

```vbnet
' 値オブジェクトを使った switch 式の代替
' （VB.NETには switch 式に直接対応なし。§3bに基づきSelect Caseで代替）
Public Function GetPaymentMethodDescription(payment As PaymentMethod) As String
    Select Case payment.Type
        Case PaymentType.CreditCard
            Return $"末尾 {payment.Last4} のクレジットカード"
        Case PaymentType.BankTransfer
            Return $"{payment.AccountNumber} からの銀行振込"
        Case PaymentType.Cash
            Return "現金払い"
        Case Else
            Return "不明な支払い方法"
    End Select
End Function

' プロパティパターンの代替
' （VB.NETには property pattern に直接対応なし。§3bに基づきIf/ElseIfで代替）
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

' リレーショナルパターンの代替
' （VB.NETには relational pattern に直接対応なし。§3bに基づきSelect Caseで代替）
Public Function ClassifyTemperature(temp As Integer) As String
    Select Case temp
        Case Is < 0
            Return "凍結"
        Case 0 To 9
            Return "寒い"
        Case 10 To 19
            Return "涼しい"
        Case 20 To 29
            Return "暖かい"
        Case Else
            Return "暑い"
    End Select
End Function

' リストパターンの代替（C# 11+）
' （VB.NETには list pattern に直接対応なし。§3bに基づきIf/ElseIfで代替）
Public Function IsValidSequence(numbers As Integer()) As Boolean
    If numbers.Length = 0 Then
        Return False                        ' 空
    ElseIf numbers.Length = 1 Then
        Return True                         ' 単一要素
    ElseIf numbers(0) < numbers(numbers.Length - 1) Then
        Return True                         ' 先頭 < 末尾
    Else
        Return False
    End If
End Function

' 型パターンと null チェックの代替
' （VB.NETには type pattern / switch 式に直接対応なし。§3bに基づきTypeOf...Is + If/ElseIfで代替）
Public Function FormatValue(value As Object) As String
    If value Is Nothing Then
        Return "null"
    ElseIf TypeOf value Is String Then
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

' 複合パターンの代替
' （VB.NETには record + property pattern に直接対応なし。§3bに基づきClass+If/ElseIfで代替）
Public NotInheritable Class OrderState
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
    If state.IsCancelled Then
        Return "キャンセル済み"
    ElseIf state.IsPaid AndAlso state.IsShipped Then
        Return "配達済み"
    ElseIf state.IsPaid AndAlso Not state.IsShipped Then
        Return "処理中"
    ElseIf Not state.IsPaid Then
        Return "支払い待ち"
    Else
        Return "不明"
    End If
End Function

' タプルを使った複合パターンの代替
' （VB.NETには tuple switch pattern に直接対応なし。§3bに基づきIf/ElseIfで代替）
Public Function CalculateShipping(total As Money, destination As Country) As Decimal
    ' C# では: (total, destination) switch { ({ Amount: > 100m }, _) => 0m, ... }
    If total.Amount > 100D Then
        Return 0D                               ' 100 ドル超は送料無料
    ElseIf destination.Code = "US" OrElse destination.Code = "CA" Then
        Return 5D                                ' 北米
    ElseIf destination.Code = "GB" OrElse destination.Code = "FR" OrElse destination.Code = "DE" Then
        Return 10D                               ' ヨーロッパ
    Else
        Return 25D                               ' 国際
    End If
End Function
```
