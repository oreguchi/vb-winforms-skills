---
name: vb-coding-standards
description: VB.NETで堅牢・高品質なコードを記述するための標準。Class/Structure ベースの値オブジェクト（record 非対応のため手書き Equals/GetHashCode）、Option Strict On、Async/Await、関数型スタイルの LINQ、Memory(Of T)/ArrayPool(Of T) による低アロケーションパターン、ベストプラクティスな API 設計パターンを中心に解説する。なお Span(Of T)/ReadOnlySpan(Of T) は BC30668 のため VB.NET では使用不可（ローカル変数・シグネチャともに不可）。
invocable: false
---

# VB.NET コーディング標準

## このスキルを使う場面

次の場面で使用する。
- 新規 VB.NET コードを書く、または既存コードをリファクタリングする
- ライブラリやサービスの公開 API を設計する
- パフォーマンスクリティカルなコードパスを最適化する
- 強い型付けでドメインモデルを実装する
- Async/Await を多用するアプリケーションを構築する
- バイナリデータ、バッファ、高スループットシナリオを扱う

## リファレンスファイル

- [value-objects-and-patterns.md](value-objects-and-patterns.md)：値オブジェクトの完全な実装例とパターンマッチング相当のコード
- [performance-and-api-design.md](performance-and-api-design.md)：Memory(Of T)/ArrayPool(Of T) による低アロケーションパターンと API 設計原則（Span(Of T) は BC30668 のため VB.NET では使用不可）
- [composition-and-error-handling.md](composition-and-error-handling.md)：継承よりコンポジション、Result 型、テストパターン
- [anti-patterns-and-reflection.md](anti-patterns-and-reflection.md)：リフレクション回避と一般的なアンチパターン

## コア原則

1. **デフォルトは不変（イミュータブル）** — `ReadOnly` プロパティとコンストラクタによる初期化を使用する（VB.NET に `record` / `init`-only は直接対応なし。§3b に基づき Class + ReadOnly Property で代替）
2. **型の安全性** — `Option Strict On` を有効化し、値オブジェクトを活用する
3. **条件分岐の明示化** — `Select Case` 文や `If`/`ElseIf` チェーンで条件ロジックを明快に書く（VB.NET に switch 式は直接対応なし。§3b に基づき Select Case 文で代替）
4. **Async はあらゆる場所に** — すべての I/O 操作で Async/Await API を優先する
5. **低アロケーションパターン** — パフォーマンスクリティカルなコードには `Memory(Of T)` と `ArrayPool(Of T)` を使用する（`Span(Of T)` / `ReadOnlySpan(Of T)` は BC30668 のため VB.NET では使用不可）
6. **API 設計** — 抽象型を受け取り、適切に具体的な型を返す
7. **継承よりコンポジション** — 抽象基底クラスを避け、コンポジションを優先する
8. **値オブジェクトは Structure で** — 値オブジェクトには `Structure` + `ReadOnly` プロパティ + 手書き `Equals`/`GetHashCode` を使用する（VB.NET に `readonly record struct` は直接対応なし。§3b に基づき Structure で代替）

---

## 言語パターン

### 不変データの表現（VB.NETに直接対応なし — §3b に基づき Class + ReadOnly Property で代替）

C# では `record` 型を使って DTO、メッセージ、イベント、ドメインエンティティを表現する。VB.NET には `record` 糖衣構文が存在しないため、Class + ReadOnly Property + コンストラクタで同等の不変性を実現する。

```vbnet
' シンプルな不変 DTO
' （VB.NETには直接対応なし。§3bに基づきClass+コンストラクタで代替）
Public NotInheritable Class CustomerDto
    Public ReadOnly Property Id As String
    Public ReadOnly Property Name As String
    Public ReadOnly Property Email As String

    Public Sub New(id As String, name As String, email As String)
        Me.Id = id
        Me.Name = name
        Me.Email = email
    End Sub
End Class

' バリデーション付き値オブジェクト（Class 版）
' （VB.NETには直接対応なし。§3bに基づきClass+コンストラクタで代替）
Public NotInheritable Class EmailAddress
    Public ReadOnly Property Value As String

    Public Sub New(value As String)
        If String.IsNullOrWhiteSpace(value) OrElse Not value.Contains("@") Then
            Throw New ArgumentException("無効なメールアドレスです。", NameOf(value))
        End If
        Me.Value = value
    End Sub
End Class

' コレクションを持つ不変クラス — IReadOnlyList(Of T) を使用
' （VB.NETには直接対応なし。§3bに基づきClass+コンストラクタで代替）
Public NotInheritable Class ShoppingCart
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

**`Class` と `Structure` の使い分け：**
- `Class`（参照型）：エンティティ、集約、多くのプロパティを持つ DTO
- `Structure`（値型）：値オブジェクト（次のセクション参照）

### 値オブジェクトは Structure + ReadOnly で（VB.NETに直接対応なし）

値オブジェクトには常に `Structure` + `ReadOnly` プロパティ + 手書き `Equals`/`GetHashCode` を使用する（C# の `readonly record struct` に相当）。暗黙的な変換演算子は使わず、明示的な変換のみを使う。

（VB.NETには `readonly record struct` に直接対応なし。§3bに基づきStructure+手書きEqualsで代替）

```vbnet
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
            Dim other As OrderId = DirectCast(obj, OrderId)
            Return Value = other.Value
        End If
        Return False
    End Function

    Public Overrides Function GetHashCode() As Integer
        Return If(Value IsNot Nothing, Value.GetHashCode(), 0)
    End Function
    ' 暗黙的な変換は禁止 — 型安全性を損なう！
    ' 内部値へのアクセスは orderId.Value を使うこと
End Structure

Public Structure Money
    Public ReadOnly Property Amount As Decimal
    Public ReadOnly Property Currency As String

    Public Sub New(amount As Decimal, currency As String)
        If amount < 0 Then
            Throw New ArgumentException("金額は負の値にできません。", NameOf(amount))
        End If
        Me.Amount = amount
        Me.Currency = currency
    End Sub
End Structure

Public Structure CustomerId
    Public ReadOnly Property Value As Guid

    Public Sub New(value As Guid)
        Me.Value = value
    End Sub

    Public Shared Function NewId() As CustomerId
        Return New CustomerId(Guid.NewGuid())
    End Function
End Structure
```

完全な例（複数値オブジェクト、ファクトリパターン、暗黙的変換禁止ルール）は [value-objects-and-patterns.md](value-objects-and-patterns.md) を参照。

### 条件分岐（Select Case と If/ElseIf）

VB.NET に C# の switch 式はない。`Select Case` 文、または条件ごとの `If`/`ElseIf` で同等のロジックを記述する。

（VB.NETには switch 式「x switch { ... }」に直接対応なし。§3bに基づきSelect Case文で代替）

```vbnet
Public Function CalculateDiscount(order As Order) As Decimal
    ' C# switch 式の代替 — Select Case + 一時変数
    Dim discount As Decimal

    If order.Total > 1000D Then
        discount = order.Total * 0.15D
    ElseIf order.Total > 500D Then
        discount = order.Total * 0.10D
    ElseIf order.Total > 100D Then
        discount = order.Total * 0.05D
    Else
        discount = 0D
    End If

    Return discount
End Function
```

完全なパターンマッチング相当例は [value-objects-and-patterns.md](value-objects-and-patterns.md) を参照。

---

### Null 許容型と Nothing チェック

VB.NET では、値型の null 許容は `Nullable(Of T)` / `T?` で表現する。参照型は `Nothing` を許容するかどうかをコメントや命名で明示する。C# の `#nullable enable` に相当するコンパイラ警告の強化は `Option Strict On` と `#Enable Warning` で補完する。

```vbnet
' プロジェクト設定（.vbproj / ソースファイル先頭）
Option Strict On
Option Infer On

' Null を返す可能性がある関数
Public Function FindUserName(userId As String) As String
    Dim user = _repository.Find(userId)
    Return If(user IsNot Nothing, user.Name, Nothing)
End Function

' null 条件演算子（VB.NET でも使用可能）
Public Function GetDiscountRate(customer As Customer) As Decimal
    ' C# の switch 式 + null パターン の代替
    ' （VB.NETには property pattern 等なし。§3bに基づきIf/ElseIfで代替）
    If customer Is Nothing Then
        Return 0D
    ElseIf customer.IsVip Then
        Return 0.20D
    ElseIf customer.OrderCount > 10 Then
        Return 0.10D
    Else
        Return 0.05D
    End If
End Function

' ガード節（ArgumentNullException）
Public Sub ProcessOrder(order As Order)
    If order Is Nothing Then
        Throw New ArgumentNullException(NameOf(order))
    End If
    ' ここでは order は Nothing ではない
    Console.WriteLine(order.Id)
End Sub
```

---

## 継承よりコンポジション

**抽象基底クラスを避ける。** インターフェイス + コンポジションを使う。共有ロジックには静的ヘルパーを使う。バリアントのモデリングにはファクトリメソッドを持つクラスを使う。

完全な例は [composition-and-error-handling.md](composition-and-error-handling.md) を参照。

---

## パフォーマンスパターン

### Async/Await のベストプラクティス

```vbnet
' すべて Async — 常に CancellationToken を受け取る
Public Async Function GetOrderAsync(
    orderId As String,
    cancellationToken As CancellationToken) As Task(Of Order)

    Dim order = Await _repository.GetAsync(orderId, cancellationToken)
    Return order
End Function

' ValueTask(Of T) — 頻繁に呼ばれ、多くの場合同期的に完了するメソッドに使用
Public Function GetCachedOrderAsync(
    orderId As String,
    cancellationToken As CancellationToken) As ValueTask(Of Order)

    Dim order As Order = Nothing
    If _cache.TryGetValue(orderId, order) Then
        Return New ValueTask(Of Order)(order)
    End If
    Return New ValueTask(Of Order)(GetFromDatabaseAsync(orderId, cancellationToken))
End Function

' IAsyncEnumerable(Of T) — ストリーミング
' **VB.NET 制約**: VB.NET は同一メソッドに `Iterator` と `Async` を同時指定できないため、
' `IAsyncEnumerable(Of T)` を `Async Iterator Function` で実装することは**コンパイル不可**。
' また、VB.NET は Await For Each / Finally 内 Await のいずれも未対応。以下は IAsyncEnumerator を手動操作する代替パターン。
' 実装時は `AsyncEnumerable.Create(...)`（System.Linq.Async 等のヘルパー）または
' `IAsyncEnumerator(Of T)` の手動実装が必要となる。
' ※ 擬似コード（コンパイル不可） ↓
Public Async Iterator Function StreamOrdersAsync(
    customerId As String,
    Optional cancellationToken As CancellationToken = Nothing) As IAsyncEnumerable(Of Order)

    ' [擬似コードのため省略 — 実際は下記 IAsyncEnumerator パターンを使用]
    ' Await For Each は VB.NET 非対応。代替は IAsyncEnumerator(Of T) の手動操作：
    Dim enumerator = _repository.StreamAllAsync(cancellationToken).GetAsyncEnumerator(cancellationToken)
    Dim thrown As Exception = Nothing
    Try
        While Await enumerator.MoveNextAsync()
            Dim order = enumerator.Current
            If order.CustomerId = customerId Then
                Yield order
            End If
        End While
    Catch ex As Exception
        thrown = ex
    End Try
    Await enumerator.DisposeAsync()  ' Finally 内の Await は BC36943 のため不可 — Try/Catch 後に呼ぶ
    If thrown IsNot Nothing Then
        Throw thrown
    End If
End Function
```

**重要なルール：**
- `CancellationToken` は常に `= Nothing`（デフォルト）付きで受け取る
- ライブラリコードでは `ConfigureAwait(False)` を使用する
- Async コードをブロックしない（`.Result` や `.Wait()` は使わない）
- タイムアウトにはリンクされた CancellationTokenSource を使用する

### Memory(Of T) と ArrayPool(Of T)

**VB.NET の制約**：`Span(Of T)` / `ReadOnlySpan(Of T)` は ref struct セマンティクスを持つため、VB.NET では**ローカル変数にもメソッドシグネチャにも使用できない**（BC30668）。

VB.NET で使用可能な低アロケーションパターン：
- `Memory(Of T)` — Async I/O バッファ（`Stream.ReadAsync(Memory(Of Byte))` 等）
- `ArrayPool(Of T)` — 大きな一時バッファのプーリングで GC 圧力を削減
- 文字列解析は `String.IndexOf` + `String.Substring` で代替（Span のゼロアロケーション効果は得られないが正確に動作する）

完全な例と API 設計セクションは [performance-and-api-design.md](performance-and-api-design.md) を参照。

---

## エラー処理：Result 型

期待されるエラーには例外ではなく `Result` 型（ドメイン固有の結果型）を使用する。例外は予期しない / システムエラーにのみ使う。

完全な Result 型の実装と使用例は [composition-and-error-handling.md](composition-and-error-handling.md) を参照。

---

## リフレクションベースのメタプログラミングを避ける

**禁止：** AutoMapper、Mapster、ExpressMapper。明示的なマッピング拡張メソッドを代わりに使用する。プライベートメンバーへのアクセスが本当に必要な場合は `UnsafeAccessorAttribute`（.NET 8+）を使用する。

詳細なガイダンスは [anti-patterns-and-reflection.md](anti-patterns-and-reflection.md) を参照。

---

## コードの整理

```vbnet
' ファイル: Domain/Orders/Order.vb

Namespace MyApp.Domain.Orders

    ' 1. 主要ドメイン型
    ' （VB.NETには直接対応なし。§3bに基づきClass+コンストラクタで代替）
    Public NotInheritable Class Order
        Public ReadOnly Property Id As OrderId
        Public ReadOnly Property CustomerId As CustomerId
        Public ReadOnly Property Total As Money
        Public ReadOnly Property Status As OrderStatus
        Public ReadOnly Property Items As IReadOnlyList(Of OrderItem)

        Public Sub New(
            id As OrderId,
            customerId As CustomerId,
            total As Money,
            status As OrderStatus,
            items As IReadOnlyList(Of OrderItem))

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

        Public Function AddItem(item As OrderItem) As CreateOrderResult
            If Status <> OrderStatus.Draft Then
                Return CreateOrderResult.Failed(
                    OrderErrorCode.ValidationError,
                    "下書き状態の注文にのみ商品を追加できます。")
            End If

            Dim newItems As New List(Of OrderItem)(Items)
            newItems.Add(item)

            Dim newTotal As New Money(
                Items.Sum(Function(i) i.Total.Amount) + item.Total.Amount,
                Total.Currency)

            ' C# の `with` 式の代替 — 新しいオブジェクトを手動で構築する
            ' （VB.NETには `with` 式に直接対応なし。§3bに基づきコンストラクタで代替）
            Dim newOrder As New Order(Id, CustomerId, newTotal, Status, newItems.AsReadOnly())
            Return CreateOrderResult.Success(newOrder)
        End Function
    End Class

    ' 2. 状態を表す列挙型
    Public Enum OrderStatus
        Draft
        Submitted
        Processing
        Completed
        Cancelled
    End Enum

    ' 3. 関連型
    ' （VB.NETには直接対応なし。§3bに基づきClass+コンストラクタで代替）
    Public NotInheritable Class OrderItem
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

    ' 4. 値オブジェクト
    ' （VB.NETには `readonly record struct` に直接対応なし。§3bに基づきStructureで代替）
    Public Structure OrderId
        Public ReadOnly Property Value As Guid

        Public Sub New(value As Guid)
            Me.Value = value
        End Sub

        Public Shared Function NewId() As OrderId
            Return New OrderId(Guid.NewGuid())
        End Function

        Public Overrides Function Equals(obj As Object) As Boolean
            If TypeOf obj Is OrderId Then
                Return DirectCast(obj, OrderId).Value = Value
            End If
            Return False
        End Function

        Public Overrides Function GetHashCode() As Integer
            Return Value.GetHashCode()
        End Function
    End Structure

    ' 5. エラー型
    ' （VB.NETには `readonly record struct` に直接対応なし。§3bに基づきStructureで代替）
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

## ベストプラクティスまとめ

### すること（DO）
- DTO、メッセージ、ドメインエンティティには Class + ReadOnly プロパティ + コンストラクタを使う
- 値オブジェクトには Structure + ReadOnly + 手書き Equals/GetHashCode を使う
- `Select Case` 文や `If`/`ElseIf` チェーンで条件ロジックを明快に書く
- `Option Strict On` を有効にし、Null 許容性を明示的に扱う
- すべての I/O 操作で Async/Await を使う
- すべての Async メソッドで `CancellationToken` を受け取る
- パフォーマンスクリティカルなシナリオには `Memory(Of T)` と `ArrayPool(Of T)` を使う（`Span(Of T)` は VB.NET 未対応 — BC30668）
- 抽象型（`IEnumerable(Of T)`、`IReadOnlyList(Of T)`）を受け取る
- 期待されるエラーには `Result` 型（ドメイン固有の結果型）を使う
- 大きなアロケーションのバッファプーリングには `ArrayPool(Of T)` を使う
- 継承よりコンポジションを優先する

### してはいけないこと（DON'T）
- Class が使えるときにミュータブルな DTO を作らない
- 値オブジェクトに Class を使わない（Structure を使う）
- 深い継承階層を作らない
- Null 許容の警告を無視しない
- Async コードをブロックしない（`.Result`、`.Wait()`）
- 大きな一時バッファをメソッド呼び出しのたびに `New Byte(n) {}` で確保しない — `ArrayPool(Of T)` を使う（`Span(Of T)` は VB.NET でローカル変数にも使えないため BC30668。代替として配列インデックスアクセスを使う）
- `CancellationToken` パラメータを忘れない
- API からミュータブルなコレクションを返さない
- 期待されるビジネスエラーに例外を使わない
- 繰り返し大きな配列を確保しない（`ArrayPool` を使う）

詳細なアンチパターン例は [anti-patterns-and-reflection.md](anti-patterns-and-reflection.md) を参照。

---

## 追加リソース

- **VB.NET 言語リファレンス**：https://learn.microsoft.com/ja-jp/dotnet/visual-basic/
- **Memory(Of T) と Span(Of T)（C# 参考）**：https://learn.microsoft.com/ja-jp/dotnet/standard/memory-and-spans/（Span は VB.NET 未対応 BC30668。Memory と ArrayPool は VB.NET で使用可）
- **非同期プログラミングのベストプラクティス**：https://learn.microsoft.com/ja-jp/archive/msdn-magazine/2013/march/async-await-best-practices-in-asynchronous-programming
- **.NET パフォーマンスのヒント**：https://learn.microsoft.com/ja-jp/dotnet/framework/performance/
