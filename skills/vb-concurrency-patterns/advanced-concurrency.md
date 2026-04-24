# 高度な並行処理パターン

Akka.NET Streams、Reactive Extensions、Akka.NET アクター、非同期ローカル関数パターンによる高度な並行処理シナリオ。

## 目次

- [Akka.NET Streams（複雑なストリーム処理）](#akkane-streams複雑なストリーム処理)
- [Reactive Extensions（UI とイベント合成）](#reactive-extensionsui-とイベント合成)
- [Akka.NET アクター（ステートフルな並行処理）](#akkane-アクターステートフルな並行処理)
- [非同期ローカル関数を優先する](#非同期ローカル関数を優先する)

## Akka.NET Streams（複雑なストリーム処理）

**用途:** バックプレッシャー、バッチ処理、デバウンス、スロットル、ストリームのマージ、複雑な変換。

```vbnet
Imports Akka.Streams
Imports Akka.Streams.Dsl

' タイムアウト付きバッチ処理
Public Function BatchEvents(events As Source(Of [Event], NotUsed)) As Source(Of IReadOnlyList(Of [Event]), NotUsed)
    Return events _
        .GroupedWithin(100, TimeSpan.FromSeconds(1)) ' 最大 100 件または 1 秒でバッチ化
        .Select(Function(batch) CType(batch.ToList(), IReadOnlyList(Of [Event])))
End Function

' スロットル処理
Public Function ThrottleRequests(requests As Source(Of Request, NotUsed)) As Source(Of Request, NotUsed)
    Return requests _
        .Throttle(10, TimeSpan.FromSeconds(1), 5, ThrottleMode.Shaping)
End Function

' 並列処理と順序付き結果
Public Function ProcessWithParallelism(items As Source(Of Item, NotUsed)) As Source(Of ProcessedItem, NotUsed)
    Return items _
        .SelectAsync(4, Async Function(item) Await ProcessAsync(item)) ' 4 並列
End Function

' 複合パイプライン
Public Function CreatePipeline(
    events As Source(Of RawEvent, NotUsed),
    sink As Sink(Of ProcessedEvent, Task(Of Done))) As IRunnableGraph(Of Task(Of Done))

    Return events _
        .Where(Function(e) e.IsValid) _
        .GroupedWithin(50, TimeSpan.FromMilliseconds(500)) _
        .SelectAsync(4, Async Function(batch) Await ProcessBatchAsync(batch)) _
        .SelectMany(Function(results) results) _
        .ToMaterialized(sink, Keep.Right)
End Function
```

**Akka.NET Streams が優れている場面:**
- サイズと時間の両方を条件にしたバッチ処理
- スロットルとレート制限
- パイプライン全体に伝播するバックプレッシャー
- ストリームのマージ/分割
- 順序保証付きの並列処理
- スーパービジョンによるエラー処理

## Reactive Extensions（UI とイベント合成）

**用途:** UI イベント処理、イベントストリームの合成、クライアントアプリケーションでの時間ベース処理。

Rx は、デバウンス、スロットル、複数のイベントソースの結合が必要な UI シナリオで威力を発揮する。

```vbnet
Imports System.Reactive.Linq

' デバウンス付き検索候補表示
Public Class SearchViewModel
    Public Sub New(searchService As ISearchService)
        SearchResults = SearchText _
            .Throttle(TimeSpan.FromMilliseconds(300)) ' 入力の停止を待つ
            .DistinctUntilChanged()                   ' 同じテキストは無視
            .Where(Function(text) text.Length >= 3)  ' 最低文字数
            .SelectMany(Function(text) searchService.SearchAsync(text).ToObservable()) _
            .ObserveOn(RxApp.MainThreadScheduler)     ' UI スレッドに戻す
    End Sub

    Public Property SearchText As IObservable(Of String)
    Public Property SearchResults As IObservable(Of IList(Of SearchResult))
End Class

' 複数の UI イベントの結合
Public ReadOnly Property CanSubmit As IObservable(Of Boolean)
    Get
        Return Observable.CombineLatest(
            UsernameValid,
            PasswordValid,
            EmailValid,
            Function(user, pass, email) user AndAlso pass AndAlso email)
    End Get
End Property

' ダブルクリックの検出
Public ReadOnly Property DoubleClicks As IObservable(Of Point)
    Get
        Return MouseClicks _
            .Buffer(TimeSpan.FromMilliseconds(300)) _
            .Where(Function(clicks) clicks.Count >= 2) _
            .Select(Function(clicks) clicks.Last())
    End Get
End Property

' デバウンス付き自動保存
Public ReadOnly Property AutoSave As IDisposable
    Get
        Return DocumentChanges _
            .Throttle(TimeSpan.FromSeconds(2)) _
            .Subscribe(Async Sub(doc) Await SaveAsync(doc))
    End Get
End Property
```

**Rx が最適な場面:**
- UI イベント合成（WPF、WinForms、MAUI、Blazor）
- デバウンス付き検索候補表示
- 複数のイベントソースの結合
- UI でのタイムウィンドウ処理
- ドラッグ&ドロップジェスチャーの検出
- リアルタイムデータの可視化

**Rx vs Akka.NET Streams:**

| シナリオ | Rx | Akka.NET Streams |
|----------|----|--------------------|
| UI イベント | 最適 | 過剰 |
| クライアントサイドの合成 | 最適 | 過剰 |
| サーバーサイドのパイプライン | 動作するが制限あり | バックプレッシャーが優れる |
| 分散処理 | 設計対象外 | 設計対象 |
| ホット Observable | ネイティブサポート | セットアップが複雑 |

**目安:** UI/クライアントには Rx、サーバーサイドのパイプラインには Akka.NET Streams。

## Akka.NET アクター（ステートフルな並行処理）

**用途:** 多数のエンティティの状態管理、ステートマシン、プッシュベースの更新、複雑な協調処理、スーパービジョンとフォールトトレランス。

### エンティティ単位アクターパターン

```vbnet
' エンティティ単位アクター — 各注文が隔離された状態を持つ
Public Class OrderActor
    Inherits ReceiveActor

    Private _state As OrderState

    Public Sub New(orderId As String)
        _state = New OrderState(orderId)

        Receive(Of AddItem)(Sub(msg)
            _state = _state.AddItem(msg.Item)
            Sender.Tell(New ItemAdded(msg.Item))
        End Sub)

        Receive(Of Checkout)(Sub(msg)
            If _state.CanCheckout Then
                _state = _state.Checkout()
                Sender.Tell(New CheckoutSucceeded(_state.Total))
            Else
                Sender.Tell(New CheckoutFailed("Cart is empty"))
            End If
        End Sub)

        Receive(Of GetState)(Sub(msg) Sender.Tell(_state))
    End Sub
End Class
```

### Become を使ったステートマシン

`Become()` を使ってメッセージハンドラを切り替えることで、アクターはステートマシンの実装に優れている。

```vbnet
Public Class PaymentActor
    Inherits ReceiveActor

    Private _payment As PaymentData

    Public Sub New(paymentId As String)
        _payment = New PaymentData(paymentId)
        Pending()
    End Sub

    Private Sub Pending()
        Receive(Of AuthorizePayment)(Sub(msg)
            ' VB.NET に with 式はないため、新しい PaymentData インスタンスを作成する
            _payment = New PaymentData(_payment.Id) With {.Amount = msg.Amount}
            Become(AddressOf Authorizing)
            Self.Tell(New ProcessAuthorization())
        End Sub)

        Receive(Of CancelPayment)(Sub(msg)
            Become(AddressOf Cancelled)
            Sender.Tell(New PaymentCancelled(_payment.Id))
        End Sub)
    End Sub

    Private Sub Authorizing()
        Receive(Of ProcessAuthorization)(Async Sub(msg)
            Dim result = Await _gateway.AuthorizeAsync(_payment)
            If result.Success Then
                _payment = New PaymentData(_payment.Id) With {
                    .Amount = _payment.Amount,
                    .AuthCode = result.AuthCode
                }
                Become(AddressOf Authorized)
            Else
                Become(AddressOf Failed)
            End If
        End Sub)

        Receive(Of CancelPayment)(Sub(msg)
            Sender.Tell(New PaymentError("Cannot cancel during authorization"))
        End Sub)
    End Sub

    Private Sub Authorized()
        Receive(Of CapturePayment)(Sub(msg)
            Become(AddressOf Capturing)
            Self.Tell(New ProcessCapture())
        End Sub)

        Receive(Of VoidPayment)(Sub(msg)
            Become(AddressOf Voiding)
            Self.Tell(New ProcessVoid())
        End Sub)
    End Sub

    Private Sub Capturing() ' ...
    End Sub
    Private Sub Voiding() ' ...
    End Sub
    Private Sub Cancelled() ' GetState のみに応答
    End Sub
    Private Sub Failed() ' GetState と Retry のみに応答
    End Sub
End Class
```

### Cluster Sharding による分散エンティティ

```vbnet
builder.WithShardRegion(Of OrderActor)(
    typeName:="orders",
    entityPropsFactory:=Function(_, _, resolver)
        Return Function(orderId) Props.Create(Function() New OrderActor(orderId))
    End Function,
    messageExtractor:=New OrderMessageExtractor(),
    shardOptions:=New ShardOptions())

Dim orderRegion = registry.Get(Of OrderActor)()
orderRegion.Tell(New ShardingEnvelope("order-123", New AddItem(item)))
```

### Akka.NET を使う場面

**次の要件がある場合に Akka.NET アクターを使う:**

| シナリオ | なぜアクターか？ |
|----------|-------------|
| 独立した状態を持つ多数のエンティティ | 各エンティティが専用のアクターを持つ — SyncLock 不要 |
| ステートマシン | `Become()` が状態遷移をエレガントにモデル化する |
| プッシュベース/リアクティブな更新 | アクターは自然に Tell-don't-ask をサポート |
| スーパービジョン要件 | 親アクターが子アクターを監視し自動再起動 |
| 分散システム | Cluster Sharding がノード間で分散 |
| 長期実行ワークフロー | アクター + 永続化 = 耐久性のあるワークフロー |
| リアルタイムシステム | メッセージ駆動、ノンブロッキングで設計 |
| IoT / デバイス管理 | デバイス 1 台 = アクター 1 つ、数百万台までスケール |

**次の場合は Akka.NET を使わない:**

| シナリオ | より良い代替 |
|----------|-------------------|
| 単純なワークキュー | `Channel(Of T)` |
| リクエスト/レスポンス API | `Async`/`Await` |
| バッチ処理 | `Parallel.ForEachAsync` または Akka.NET Streams |
| UI イベント処理 | Reactive Extensions |
| CRUD 操作 | 標準的な非同期サービス |

### アクターの考え方

問題が次のような形をしているときにアクターを検討する。
- 「**何千もの**[注文/ユーザー/デバイス]に独立した状態を持たせる必要がある」
- 「各エンティティが各段階で異なる振る舞いをする**ライフサイクル**を経る」
- 「何かが変化したときに関係者に**更新をプッシュ**する必要がある」
- 「処理が失敗した場合、**そのエンティティだけを再起動**したい」
- 「**複数のサーバー**にまたがって動作する必要がある」

## 非同期ローカル関数を優先する

> **VB.NET 制約**: VB.NET は**メソッド内でのローカル関数定義をサポートしない**（BC30026/BC30289）。C# で使われる「`async` メソッド内に `async` ローカル関数を入れてカプセル化する」パターンは VB.NET では実装できない。
>
> **代替**: 同一クラス内の `Private Async Function` にメソッド抽出することで同等の責務分割を実現できる。ただし「呼び出し元メソッドのローカル変数をキャプチャする」というローカル関数特有の利便性は得られず、パラメータ明示が必要になる。
>
> 本セクションは scope-strip（`00-master-plan.md` §4）により VB.NET 版では削除扱いとする。
