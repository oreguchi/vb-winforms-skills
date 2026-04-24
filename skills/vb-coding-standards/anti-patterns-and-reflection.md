# アンチパターンとリフレクション回避

リフレクションベースのメタプログラミングを避け、一般的な VB.NET アンチパターンを回避するためのガイドライン。

## 目次

- [リフレクションベースのメタプログラミングを避ける](#リフレクションベースのメタプログラミングを避ける)
- [UnsafeAccessorAttribute（.NET 8+）](#unsafeaccessorattributenet-8)
- [避けるべきアンチパターン](#避けるべきアンチパターン)

## リフレクションベースのメタプログラミングを避ける

**リフレクションベースの「魔法の」ライブラリよりも、静的型付けの明示的なコードを優先する。**

AutoMapper のようなリフレクションベースのライブラリは、コンパイル時の安全性を利便性と引き換えにしてしまう。マッピングが壊れてもコンパイル時ではなく、実行時（あるいは最悪の場合は本番環境）に初めて気づくことになる。

### 禁止ライブラリ

| ライブラリ | 問題点 |
|---------|---------|
| **AutoMapper** | リフレクションの魔法、隠れたマッピング、実行時エラー、デバッグが困難 |
| **Mapster** | AutoMapper と同じ問題 |
| **ExpressMapper** | 同上 |

### リフレクションマッピングが失敗する理由

```vbnet
' AutoMapper を使った場合 — コンパイルは通るが実行時に失敗する
' （VB.NETには直接対応なし。§3bに基づきClass+コンストラクタで代替）
Public NotInheritable Class UserDto
    Public ReadOnly Property Id As String
    Public ReadOnly Property Name As String
    Public ReadOnly Property Email As String

    Public Sub New(id As String, name As String, email As String)
        Me.Id = id
        Me.Name = name
        Me.Email = email
    End Sub
End Class

Public NotInheritable Class UserEntity
    Public ReadOnly Property Id As Guid
    Public ReadOnly Property FullName As String
    Public ReadOnly Property EmailAddress As String

    Public Sub New(id As Guid, fullName As String, emailAddress As String)
        Me.Id = id
        Me.FullName = fullName
        Me.EmailAddress = emailAddress
    End Sub
End Class

' このマッピングはサイレントに壊れた結果を生成する：
' - Id: String vs Guid の不一致
' - Name vs FullName: 一致なし、null/デフォルト値
' - Email vs EmailAddress: 一致なし、null/デフォルト値
Dim dto = _mapper.Map(Of UserDto)(entity)  ' コンパイル通過！実行時に壊れる。
```

### 代わりに明示的なマッピングメソッドを使う

```vbnet
' 拡張メソッド — コンパイル時チェック、見つけやすい、デバッグしやすい
Imports System.Runtime.CompilerServices

Public Module UserMappings
    <Extension>
    Public Function ToDto(entity As UserEntity) As UserDto
        Return New UserDto(
            entity.Id.ToString(),
            entity.FullName,
            entity.EmailAddress)
    End Function

    <Extension>
    Public Function ToEntity(request As CreateUserRequest) As UserEntity
        Return New UserEntity(
            Guid.NewGuid(),
            request.Name,
            request.Email)
    End Function
End Module

' 使用例 — 明示的で追跡可能
Dim dto = entity.ToDto()
Dim entity2 = request.ToEntity()
```

### 明示的マッピングのメリット

| 側面 | AutoMapper | 明示的メソッド |
|--------|------------|------------------|
| **コンパイル時の安全性** | なし — 実行時エラー | あり — コンパイラが不一致を検出 |
| **発見しやすさ** | プロファイルに隠れている | 「定義へ移動」が機能する |
| **デバッグ** | ブラックボックス | コードをステップ実行できる |
| **リファクタリング** | 名前変更がサイレントに壊れる | IDE が正しく名前変更する |
| **パフォーマンス** | リフレクションのオーバーヘッド | 直接プロパティアクセス |
| **テスト** | 結合テストが必要 | シンプルな単体テスト |

### 複雑なマッピング

複雑な変換の場合、明示的なコードはさらに価値が高い。

```vbnet
<Extension>
Public Function ToSummary(order As Order) As OrderSummaryDto
    ' C# switch 式の代替 — Select Case + ローカル変数
    ' （VB.NETには switch 式に直接対応なし。§3bに基づきSelect Caseで代替）
    ' OrderStatus は Draft / Submitted / Processing / Completed / Cancelled で統一
    Dim statusText As String
    Select Case order.Status
        Case OrderStatus.Draft
            statusText = "下書き"
        Case OrderStatus.Submitted
            statusText = "受付済み"
        Case OrderStatus.Processing
            statusText = "処理中"
        Case OrderStatus.Completed
            statusText = "完了"
        Case OrderStatus.Cancelled
            statusText = "キャンセル済み"
        Case Else
            statusText = "不明"
    End Select

    Return New OrderSummaryDto(
        order.Id.Value.ToString(),
        order.Customer.FullName,
        order.Items.Count,
        order.Items.Sum(Function(i) i.Quantity * i.UnitPrice),
        statusText,
        order.CreatedAt.ToString("MMMM d, yyyy"))
End Function
```

このコードは：
- **読みやすい**：誰でも変換内容を理解できる
- **デバッグしやすい**：ブレークポイントを設置して値を確認できる
- **テストしやすい**：Order を渡して結果をアサートするだけ
- **リファクタリングしやすい**：プロパティ名を変更するとコンパイラがすべての使用箇所を教えてくれる

### リフレクションが許容される場合

リフレクションには正当な用途があるが、DTO のマッピングはその中に含まれない。

| ユースケース | 許容？ |
|----------|-------------|
| シリアライズ（System.Text.Json、Newtonsoft） | はい — 十分なテストが行われており、ソースジェネレータも利用可能 |
| 依存性注入コンテナ | はい — フレームワークのインフラ |
| ORM エンティティマッピング（EF Core） | はい — データベース抽象化に必要 |
| テストフィクスチャとビルダー | 場合による — テスト専用の利便性のため |
| **DTO/ドメインオブジェクトのマッピング** | **いいえ — 明示的メソッドを使う** |

## UnsafeAccessorAttribute（.NET 8+）

プライベートまたは内部メンバーへのアクセスが本当に必要な場合（シリアライザ、テストヘルパー、フレームワークコード）は、従来のリフレクションではなく `UnsafeAccessorAttribute` を使用する。**ゼロオーバーヘッドで AOT 互換**のメンバーアクセスを提供する。

```vbnet
' 回避：従来のリフレクション — 遅い、アロケーションあり、AOT で動作しない
Dim field = GetType(Order).GetField("_status", BindingFlags.NonPublic Or BindingFlags.Instance)
Dim status = DirectCast(field.GetValue(order), OrderStatus)

' 推奨：UnsafeAccessor — ゼロオーバーヘッド、AOT 互換
' （VB.NET での UnsafeAccessorAttribute 使用例）
<UnsafeAccessor(UnsafeAccessorKind.Field, Name:="_status")>
Private Shared Function GetStatusField(order As Order) As OrderStatus
    ' ランタイムが実装を生成する
    Throw New NotImplementedException()
End Function

Dim status = GetStatusField(order)  ' 直接アクセス、リフレクションなし
```

**サポートされるアクセサの種類：**

```vbnet
' プライベートフィールドアクセス
<UnsafeAccessor(UnsafeAccessorKind.Field, Name:="_items")>
Private Shared Function GetItemsField(order As Order) As List(Of OrderItem)
    Throw New NotImplementedException()
End Function

' プライベートメソッドアクセス
<UnsafeAccessor(UnsafeAccessorKind.Method, Name:="Recalculate")>
Private Shared Sub CallRecalculate(order As Order)
    Throw New NotImplementedException()
End Sub

' プライベート静的フィールド
<UnsafeAccessor(UnsafeAccessorKind.StaticField, Name:="_instanceCount")>
Private Shared Function GetInstanceCount(order As Order) As Integer
    Throw New NotImplementedException()
End Function

' プライベートコンストラクタ
<UnsafeAccessor(UnsafeAccessorKind.Constructor)>
Private Shared Function CreateOrder(id As OrderId, customerId As CustomerId) As Order
    Throw New NotImplementedException()
End Function
```

**リフレクションより UnsafeAccessor を選ぶ理由：**

| 側面 | リフレクション | UnsafeAccessor |
|--------|------------|----------------|
| パフォーマンス | 低速（100〜1000倍） | ゼロオーバーヘッド |
| AOT 互換 | なし | あり |
| アロケーション | あり（ボックス化、配列） | なし |
| コンパイル時チェック | なし | 部分的（シグネチャ） |

**ユースケース：**
- プライベートバッキングフィールドにアクセスするシリアライザ
- 内部状態を検証するテストヘルパー
- 可視性をバイパスする必要があるフレームワークコード

**リソース：**
- [.NET 8 でのリフレクションの新しい方法](https://steven-giesel.com/blogPost/05ecdd16-8dc4-490f-b1cf-780c994346a4)
- [.NET 8.0 でリフレクションなしにプライベートメンバーにアクセスする](https://www.strathweb.com/2023/10/accessing-private-members-without-reflection-in-net-8-0/)
- [UnsafeAccessor を使ったモダンな .NET リフレクション](https://blog.ndepend.com/modern-net-reflection-with-unsafeaccessor/)

## 避けるべきアンチパターン

### してはいけない：ミュータブルな DTO を使う

```vbnet
' 悪い例：ミュータブルな DTO
Public Class CustomerDto
    Public Property Id As String
    Public Property Name As String
End Class

' 良い例：不変クラス（VB.NETには record に直接対応なし。§3bに基づきClass+ReadOnlyで代替）
Public NotInheritable Class CustomerDto
    Public ReadOnly Property Id As String
    Public ReadOnly Property Name As String

    Public Sub New(id As String, name As String)
        Me.Id = id
        Me.Name = name
    End Sub
End Class
```

### してはいけない：値オブジェクトに Class を使う

```vbnet
' 悪い例：Class を使った値オブジェクト
Public Class OrderId
    Public ReadOnly Property Value As String
    Public Sub New(value As String)
        Me.Value = value
    End Sub
End Class

' 良い例：Structure を使った値オブジェクト
' （VB.NETには `readonly record struct` に直接対応なし。§3bに基づきStructureで代替）
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
        Return If(Value IsNot Nothing, Value.GetHashCode(), 0)
    End Function
End Structure
```

### してはいけない：深い継承階層を作る

```vbnet
' 悪い例：深い継承
Public MustInherit Class Entity
End Class

Public MustInherit Class AggregateRoot
    Inherits Entity
End Class

Public MustInherit Class OrderBase
    Inherits AggregateRoot
End Class

Public Class CustomerOrder
    Inherits OrderBase
End Class

' 良い例：コンポジションによるフラットな構造
Public Interface IEntity
    ReadOnly Property Id As Guid
End Interface

' （VB.NETには record に直接対応なし。§3bに基づきClassで代替）
Public NotInheritable Class Order
    Implements IEntity

    Public ReadOnly Property OrderId As OrderId
    Public ReadOnly Property CustomerId As CustomerId
    Public ReadOnly Property Total As Money

    Public ReadOnly Property Id As Guid Implements IEntity.Id
        Get
            Return OrderId.Value
        End Get
    End Property

    Public Sub New(orderId As OrderId, customerId As CustomerId, total As Money)
        Me.OrderId = orderId
        Me.CustomerId = customerId
        Me.Total = total
    End Sub
End Class
```

### してはいけない：IReadOnlyList(Of T) が適切な場面で List(Of T) を返す

```vbnet
' 悪い例：内部リストを変更可能な形で公開する
Public Function GetOrders() As List(Of Order)
    Return _orders
End Function

' 良い例：読み取り専用ビューを返す
Public Function GetOrders() As IReadOnlyList(Of Order)
    Return _orders
End Function
```

### してはいけない：不必要な Byte() 配列アロケーションを繰り返す

```vbnet
' 悪い例：呼び出しのたびに配列を確保する
Public Function GetHeader() As Byte()
    Dim header(63) As Byte
    ' ヘッダーを埋める
    Return header
End Function

' 良い例：呼び出し元のバッファに直接書き込む
' VB.NET では Span(Of Byte) をローカル変数にもシグネチャにも使えない（BC30668）。
' Byte() 配列を受け取り、配列インデックスで直接書き込む。
Public Sub GetHeader(destination As Byte(), ByRef bytesWritten As Integer)
    If destination.Length < 64 Then
        Throw New ArgumentException("バッファが小さすぎます。")
    End If
    ' 呼び出し元のバッファに配列インデックスで直接書き込む
    destination(0) = &H01  ' バージョン
    destination(1) = &H00  ' 予約
    ' ... 以降のフィールドを同様に設定
    bytesWritten = 64
End Sub
```

### してはいけない：Async メソッドで CancellationToken を忘れる

```vbnet
' 悪い例：キャンセルサポートなし
Public Async Function GetOrderAsync(id As OrderId) As Task(Of Order)
    Return Await _repository.GetAsync(id)
End Function

' 良い例：キャンセルサポートあり
Public Async Function GetOrderAsync(
    id As OrderId,
    Optional cancellationToken As CancellationToken = Nothing) As Task(Of Order)

    Return Await _repository.GetAsync(id, cancellationToken)
End Function
```

### してはいけない：Async コードをブロックする

```vbnet
' 悪い例：デッドロックの危険！
Public Function GetOrder(id As OrderId) As Order
    Return GetOrderAsync(id).Result
End Function

' 悪い例：こちらもデッドロックの危険！
Public Function GetOrder(id As OrderId) As Order
    Return GetOrderAsync(id).GetAwaiter().GetResult()
End Function

' 良い例：Async をすべての場所に適用する
Public Async Function GetOrderAsync(
    id As OrderId,
    cancellationToken As CancellationToken) As Task(Of Order)

    Return Await _repository.GetAsync(id, cancellationToken)
End Function
```
