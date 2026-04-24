# パフォーマンスと API 設計パターン

`Memory(Of T)` / `ArrayPool(Of T)` を使った低アロケーションパターンと、適切な型を受け取り・返すための API 設計原則。

## 目次

- [Span(Of T) と Memory(Of T) — VB.NET の現実的制約](#spanof-t-と-memoryof-t--vbnet-の現実的制約)
- [Memory(Of T) と ArrayPool(Of T) による低アロケーションパターン](#memoryof-t-と-arraypoolof-t-による低アロケーションパターン)
- [String / 配列ベースの代替パターン](#string--配列ベースの代替パターン)
- [API 設計原則](#api-設計原則)
- [メソッドシグネチャのベストプラクティス](#メソッドシグネチャのベストプラクティス)

## Span(Of T) と Memory(Of T) — VB.NET の現実的制約

**VB.NET の現実的制約**：`Span(Of T)` / `ReadOnlySpan(Of T)` は ref struct セマンティクスを持つため、VB.NET では**メソッドシグネチャにもローカル変数宣言にも使用できない**（BC30668）。実質的に VB.NET からこれらの型を直接使うことはできない。

```vbnet
' これは BC30668 でコンパイルエラー — ローカル変数宣言もシグネチャも不可
Dim span As ReadOnlySpan(Of Char) = input.AsSpan()  ' BC30668
' → VB.NET では Span(Of T) / ReadOnlySpan(Of T) の直接使用は不可
```

`Memory(Of T)` / `ReadOnlyMemory(Of T)` は ref struct ではないため VB.NET でも使用可能。非同期 I/O（`Stream.ReadAsync(Memory(Of Byte))` 等）や `ArrayPool(Of T)` によるバッファ再利用では活用できる。

`stackalloc` は VB.NET に存在しない。代わりにヒープ確保（`New Byte(n - 1) {}`）または `ArrayPool(Of T)` を使う。

**本セクションは VB.NET で使用可能な範囲に絞る：**
- `Memory(Of T)` + `ArrayPool(Of T)` による低アロケーションパターン
- `String` / 配列ベースの代替パターン（文字列解析等）
- `Span(Of T)` は C# コードとの相互参照情報として最後に記述する

> **C# との比較**：C# では `Span(Of T)` / `ReadOnlySpan(Of T)` をローカル変数・パラメータ・戻り値に自由に使えるため、ゼロアロケーション文字列解析等が可能。VB.NET からそれらのメソッドを呼ぶことはできない（引数として渡せないため）。C# ライブラリを経由する設計も選択肢のひとつ。

---

## Memory(Of T) と ArrayPool(Of T) による低アロケーションパターン

### Memory(Of T) — Async I/O

`Memory(Of T)` は ref struct ではないため、VB.NET のローカル変数・パラメータ・戻り値すべてに使用できる。非同期 I/O のバッファとして活用する。

```vbnet
' Memory(Of Byte) を Async メソッドのバッファに使用 — VB.NET でコンパイル可
Public Async Function ReadDataAsync(
    buffer As Memory(Of Byte),
    cancellationToken As CancellationToken) As Task(Of Integer)

    Return Await _stream.ReadAsync(buffer, cancellationToken)
End Function
```

### ArrayPool(Of T) — 大きな一時バッファのプーリング

`ArrayPool(Of T)` は通常のクラスであり VB.NET でも使用可能。1KB 超の一時バッファには必ず使う。

```vbnet
' 大きな一時バッファには ArrayPool を使用 — Rent/Return で GC 圧力を回避
Public Async Function ProcessLargeFileAsync(
    stream As Stream,
    cancellationToken As CancellationToken) As Task

    Dim buffer = ArrayPool(Of Byte).Shared.Rent(8192)
    Try
        Dim bytesRead As Integer
        ' Stream.ReadAsync は Memory(Of Byte) オーバーロードを受け取れる
        bytesRead = Await stream.ReadAsync(buffer.AsMemory(0, buffer.Length), cancellationToken)
        Do While bytesRead > 0
            ' buffer(0 .. bytesRead-1) を処理する
            ' VB.NET 版では Span 未対応のため配列インデックスアクセスで代替
            ProcessChunkFromArray(buffer, bytesRead)
            bytesRead = Await stream.ReadAsync(buffer.AsMemory(0, buffer.Length), cancellationToken)
        Loop
    Finally
        ArrayPool(Of Byte).Shared.Return(buffer)
    End Try
End Function

Private Sub ProcessChunkFromArray(buffer As Byte(), length As Integer)
    ' 配列の先頭 length バイトを処理する
    For i As Integer = 0 To length - 1
        ' buffer(i) を処理
    Next
End Sub
```

### ハッシュコード計算 — ArrayPool + 配列インデックスアクセス

UTF-8 エンコード後のバイト列を処理する例。`ReadOnlySpan(Of Byte)` が使えないため、バイト配列を直接渡す。

```vbnet
' GenerateHashCode — VB.NET 版では Span 未対応のため Byte() 配列で代替
' （C# 版では ReadOnlySpan(Of Byte) で stackalloc / Span を直接扱えるが、
'   VB.NET では BC30668 のため不可。配列 + ArrayPool パターンで対応する）
Private Shared Function GenerateHashCode(key As String) As Short
    If key Is Nothing Then Return 0

    Const StackLimit As Integer = 256

    Dim enc = Encoding.UTF8
    Dim maxBytes = enc.GetMaxByteCount(key.Length)

    Dim rented As Byte() = Nothing
    Dim buf As Byte()
    If maxBytes <= StackLimit Then
        buf = New Byte(StackLimit - 1) {}  ' ヒープ確保（stackalloc はVB.NET未対応）
    Else
        rented = ArrayPool(Of Byte).Shared.Rent(maxBytes)
        buf = rented
    End If

    Try
        Dim written = enc.GetBytes(key, 0, key.Length, buf, 0)
        ' ComputeHash は Byte() 配列と有効バイト長を受け取るよう設計する
        ' （C# 版の ReadOnlySpan(Of Byte) オーバーロードは VB.NET から呼べないため）
        Dim h1 As Integer, h2 As Integer
        ComputeHash(buf, written, h1, h2)
        Return CShort(h1 Xor h2)
    Finally
        If rented IsNot Nothing Then
            ArrayPool(Of Byte).Shared.Return(rented)
        End If
    End Try
End Function

' ComputeHash は Byte() 配列 + 長さで受け取るように定義する
Private Shared Sub ComputeHash(data As Byte(), length As Integer,
                                ByRef h1 As Integer, ByRef h2 As Integer)
    ' 実装（MurmurHash 等）
    h1 = 0
    h2 = 0
    For i As Integer = 0 To length - 1
        h1 = h1 Xor CInt(data(i))
    Next
End Sub
```

---

## String / 配列ベースの代替パターン

VB.NET では Span による文字列解析（`span.Slice` / `span.IndexOf` 等）ができないため、`String.IndexOf` + `String.Substring` を使う。これは若干のアロケーションを伴うが、VB.NET で実際にコンパイル・動作するコードを優先する。

### ParseOrderId — "ORD-" プレフィックスの数値抽出

```vbnet
' VB.NET 版では Span 未対応のため Substring / IndexOf で代替
' （C# 版では ReadOnlySpan(Of Char) + span.Slice(4) でゼロアロケーション実現可能だが、
'   VB.NET では BC30668 のため不可。Substring は String を確保するがシンプルで正確）
Public Shared Function ParseOrderId(input As String) As Integer
    If Not input.StartsWith("ORD-") Then
        Throw New FormatException("無効な注文 ID フォーマットです。")
    End If
    Return Integer.Parse(input.Substring(4))
End Function
```

### TryParseKeyValue — "key:value" 行の解析

```vbnet
' VB.NET 版では Span 未対応のため String.IndexOf + Substring で代替
' （C# 版では ReadOnlySpan(Of Char).Slice でゼロアロケーション可能）
Public Function TryParseKeyValue(
    line As String,
    ByRef key As String,
    ByRef value As String) As Boolean

    key = String.Empty
    value = String.Empty

    Dim colonIndex = line.IndexOf(":"c)
    If colonIndex = -1 Then
        Return False
    End If

    key = line.Substring(0, colonIndex).Trim()
    value = line.Substring(colonIndex + 1).Trim()
    Return True
End Function
```

### ParseUrl — プロトコル・ホスト・ポートの抽出

```vbnet
' VB.NET 版では Span 未対応のため String.IndexOf + Substring で代替
' 入力例: "https://example.com:8080"
Public Shared Function ParseUrl(url As String) As (Protocol As String, Host As String, Port As Integer)
    Dim protocolEnd = url.IndexOf("://")
    If protocolEnd < 0 Then
        Throw New FormatException("無効な URL フォーマットです。")
    End If
    Dim protocol = url.Substring(0, protocolEnd)

    Dim afterProtocol = url.Substring(protocolEnd + 3)
    Dim portStart = afterProtocol.IndexOf(":"c)
    If portStart < 0 Then
        Throw New FormatException("ポート番号が見つかりません。")
    End If

    Dim host = afterProtocol.Substring(0, portStart)
    Dim port = Integer.Parse(afterProtocol.Substring(portStart + 1))

    Return (protocol, host, port)
End Function
```

### TryFormatOrderId — "ORD-XXXXXXXXXX" 形式への変換

```vbnet
' VB.NET 版では Span 未対応のため String.Concat + Integer.ToString で代替
' （C# 版では Span(Of Char) + TryFormat でゼロアロケーション可能だが、
'   VB.NET では BC30668 のため不可。String.Concat はアロケーションを伴うが正確）
Public Function FormatOrderId(orderId As Integer) As String
    Return "ORD-" & orderId.ToString("D10")
End Function

' 呼び出し元が既存バッファに書き込みたい場合は Char() で受け取る
Public Function TryFormatOrderIdToBuffer(
    orderId As Integer,
    destination As Char(),
    ByRef charsWritten As Integer) As Boolean

    Const Prefix As String = "ORD-"
    Const NumberWidth As Integer = 10

    If destination.Length < Prefix.Length + NumberWidth Then
        charsWritten = 0
        Return False
    End If

    Dim formatted = Prefix & orderId.ToString("D" & NumberWidth)
    formatted.CopyTo(0, destination, 0, formatted.Length)
    charsWritten = formatted.Length
    Return True
End Function
```

### Sum — 配列要素の合計

```vbnet
' VB.NET 版では Span 未対応のため For Each / LINQ で代替
' （C# 版では ReadOnlySpan(Of Integer) でゼロコピー反復可能）
Public Function Sum(numbers As Integer()) As Integer
    Dim total As Integer = 0
    For Each num In numbers
        total += num
    Next
    Return total
End Function
```

---

## 何をいつ使うか

| 型 | VB.NET での使用可否 | ユースケース |
|------|:---:|----------|
| `Span(Of T)` | **不可**（BC30668） | C# のみ。同期操作のゼロアロケーションスライス |
| `ReadOnlySpan(Of T)` | **不可**（BC30668） | C# のみ。読み取り専用ビュー、文字列解析 |
| `Memory(Of T)` | 可 | Async 操作のバッファ（`Stream.ReadAsync` 等） |
| `ReadOnlyMemory(Of T)` | 可 | 読み取り専用の Async 操作 |
| `Byte()` / `Char()` | 可 | データを長期保存する場合、または配列を要求する API |
| `ArrayPool(Of T)` | 可 | 大きな一時バッファ（1KB 超）で GC 圧力を避ける場合 |
| `String.Substring` / `IndexOf` | 可 | VB.NET での文字列解析（Span の代替、若干アロケーションあり） |

---

## API 設計原則

### 抽象型を受け取り、適切に具体的な型を返す

**パラメータ（受け取る側）：**

```vbnet
' 一度だけ反復する場合は IEnumerable(Of T) を受け取る
Public Function CalculateTotal(items As IEnumerable(Of OrderItem)) As Decimal
    Return items.Sum(Function(item) item.Price * item.Quantity)
End Function

' Count が必要な場合は IReadOnlyCollection(Of T) を受け取る
Public Function HasMinimumItems(items As IReadOnlyCollection(Of OrderItem), minimum As Integer) As Boolean
    Return items.Count >= minimum
End Function

' インデックスアクセスが必要な場合は IReadOnlyList(Of T) を受け取る
Public Function GetMiddleItem(items As IReadOnlyList(Of OrderItem)) As OrderItem
    If items.Count = 0 Then
        Throw New ArgumentException("リストは空にできません。")
    End If

    Return items(items.Count \ 2)  ' インデックスアクセス
End Function

' 高パフォーマンス API には Integer() 配列で受け取る
' （VB.NET は ReadOnlySpan(Of T) をシグネチャに使えないため BC30668 — 配列で代替）
Public Function Sum(numbers As Integer()) As Integer
    Dim total As Integer = 0
    For Each num In numbers
        total += num
    Next
    Return total
End Function

' Async ストリーミングには IAsyncEnumerable(Of T) を受け取る
' VB.NET は Await For Each / Finally 内 Await のいずれも未対応。以下は IAsyncEnumerator を手動操作する代替パターン。
Public Async Function CountItemsAsync(
    orders As IAsyncEnumerable(Of Order),
    cancellationToken As CancellationToken) As Task(Of Integer)

    Dim count As Integer = 0
    Dim enumerator = orders.GetAsyncEnumerator(cancellationToken)
    Dim thrown As Exception = Nothing
    Try
        While Await enumerator.MoveNextAsync()
            count += 1
        End While
    Catch ex As Exception
        thrown = ex
    End Try
    Await enumerator.DisposeAsync()  ' Finally 内の Await は BC36943 のため不可 — Try/Catch 後に呼ぶ
    If thrown IsNot Nothing Then
        Throw thrown
    End If
    Return count
End Function
```

**戻り値の型：**

```vbnet
' 遅延/遅延評価には IEnumerable(Of T) を返す
Public Iterator Function GetOrdersLazy(customerId As String) As IEnumerable(Of Order)
    For Each order In _repository.Query()
        If order.CustomerId = customerId Then
            Yield order  ' 遅延評価
        End If
    Next
End Function

' 実体化された不変コレクションには IReadOnlyList(Of T) を返す
Public Function GetOrders(customerId As String) As IReadOnlyList(Of Order)
    Return _repository.
        Query().
        Where(Function(o) o.CustomerId = customerId).
        ToList()  ' 実体化
End Function

' 呼び出し元が変更を必要とする場合は具体的な型を返す
Public Function GetMutableOrders(customerId As String) As List(Of Order)
    ' 明示的に変更を許可する形で List(Of T) を返す
    Return _repository.
        Query().
        Where(Function(o) o.CustomerId = customerId).
        ToList()
End Function

' Async ストリーミングには IAsyncEnumerable(Of T) を返す
' **VB.NET 制約**: VB.NET は同一メソッドに `Iterator` と `Async` を同時指定できないため、
' `IAsyncEnumerable(Of T)` を `Async Iterator Function` で実装することは**コンパイル不可**。
' また、VB.NET は Await For Each / Finally 内 Await のいずれも未対応。以下は IAsyncEnumerator を手動操作する代替パターン。
' 実装時は `AsyncEnumerable.Create(...)`（System.Linq.Async 等のヘルパー）または
' `IAsyncEnumerator(Of T)` の手動実装が必要となる。
' ※ 擬似コード（コンパイル不可） ↓
Public Async Iterator Function StreamOrdersAsync(
    customerId As String,
    Optional cancellationToken As CancellationToken = Nothing) As IAsyncEnumerable(Of Order)

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

' 相互運用または呼び出し元が配列を期待する場合は配列を返す
Public Function SerializeOrder(order As Order) As Byte()
    ' バイナリシリアライズ — Byte() が適切
    Return MessagePackSerializer.Serialize(order)
End Function
```

**まとめ表：**

| シナリオ | 受け取る | 返す |
|----------|--------|--------|
| 一度だけ反復 | `IEnumerable(Of T)` | `IEnumerable(Of T)`（遅延の場合） |
| Count が必要 | `IReadOnlyCollection(Of T)` | `IReadOnlyCollection(Of T)` |
| インデックスが必要 | `IReadOnlyList(Of T)` | `IReadOnlyList(Of T)` |
| 高パフォーマンス・同期（VB.NET） | `T()` 配列 | `T()` 配列（Span 未対応のため） |
| Async ストリーミング | `IAsyncEnumerable(Of T)` | `IAsyncEnumerable(Of T)` |
| 呼び出し元が変更を必要 | — | `List(Of T)`、`T()` |

## メソッドシグネチャのベストプラクティス

```vbnet
' 完全な Async メソッドシグネチャ
Public Async Function CreateOrderAsync(
    request As CreateOrderRequest,
    Optional cancellationToken As CancellationToken = Nothing) As Task(Of CreateOrderResult)

    ' 実装
End Function

' オプションパラメータは末尾に配置
Public Async Function GetOrdersAsync(
    customerId As String,
    Optional startDate As Nullable(Of DateTime) = Nothing,
    Optional endDate As Nullable(Of DateTime) = Nothing,
    Optional cancellationToken As CancellationToken = Nothing) As Task(Of List(Of Order))

    ' 実装
End Function

' 複数の関連パラメータにはクラスを使用
' （VB.NETには record に直接対応なし。§3bに基づきClassで代替）
' 注意：以下は ASP.NET Core のモデルバインディング（クエリ文字列 → オブジェクト）向けの可変プロパティ。
' バインディングが不要な場合は ReadOnly Property + コンストラクタで不変化できる。
Public NotInheritable Class SearchOrdersRequest
    Public Property CustomerId As String
    Public Property StartDate As Nullable(Of DateTime)
    Public Property EndDate As Nullable(Of DateTime)
    Public Property Status As Nullable(Of OrderStatus)
    Public Property PageSize As Integer = 20
    Public Property PageNumber As Integer = 1
End Class

Public Async Function SearchOrdersAsync(
    request As SearchOrdersRequest,
    Optional cancellationToken As CancellationToken = Nothing) As Task(Of PagedResult(Of Order))

    ' 実装
End Function

' プライマリコンストラクタ（C# 12+）の VB.NET 代替
' （VB.NETには primary constructor に直接対応なし。§3bに基づき通常のコンストラクタで代替）
Public NotInheritable Class OrderService
    Private ReadOnly _repository As IOrderRepository
    Private ReadOnly _logger As ILogger(Of OrderService)

    Public Sub New(repository As IOrderRepository, logger As ILogger(Of OrderService))
        _repository = repository
        _logger = logger
    End Sub

    Public Async Function GetOrderAsync(
        orderId As OrderId,
        cancellationToken As CancellationToken) As Task(Of Order)

        _logger.LogInformation("注文を取得中 {OrderId}", orderId)
        Return Await _repository.GetAsync(orderId, cancellationToken)
    End Function
End Class

' 複雑な設定にはオプションパターンを使用
' （init-only setter に直接対応なし。§3bに基づきProperty + コンストラクタで代替）
Public NotInheritable Class EmailServiceOptions
    Public Property SmtpHost As String  ' required の代替 — コンストラクタで必須化
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
