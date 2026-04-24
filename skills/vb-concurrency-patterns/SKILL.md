---
name: vb-concurrency-patterns
description: VB.NETにおける並行処理抽象の選択指針。I/OバウンドにはAsync/Await、プロデューサー/コンシューマーにはChannel(Of T)、ステートフルなエンティティ管理にはAkka.NETを使う。SyncLockや手動同期は極力避ける。
invocable: false
---

# .NET 並行処理: 適切なツールの選択

## このスキルを使う場面

次の場面で使う。
- VB.NET での並行処理の方式を決定する
- Async/Await、Channel、Akka.NET、その他の抽象のどれを使うか評価する
- SyncLock、セマフォ、その他の同期プリミティブを使おうとしている
- バックプレッシャー、バッチ処理、デバウンスを伴うデータストリームを処理する
- 複数の並行エンティティにまたがる状態を管理する

## 参考ファイル

- [advanced-concurrency.md](advanced-concurrency.md): Akka.NET Streams、Reactive Extensions、Akka.NET アクター（エンティティ単位アクター、ステートマシン、クラスターシャーディング）、非同期ローカル関数パターン

## 基本方針

**シンプルから始め、必要なときだけ上位に移行する。**

ほとんどの並行処理問題は `Async`/`Await` で解決できる。より高度なツールに手を伸ばすのは、`Async`/`Await` では対応できない具体的な要件がある場合だけにする。

**共有可変状態を避けるよう設計する。** 並行処理に最も効果的なのは、設計によって問題を解消することだ。イミュータブルなデータ、メッセージパッシング、アクターによる状態の隔離は、バグのカテゴリーそのものを排除する。

**SyncLock は例外であって、規則ではない。** 共有可変状態をどうしても避けられない場合:
1. **第一選択:** 設計を見直して回避する（イミュータブル化、メッセージパッシング、アクターによる隔離）
2. **第二選択:** `System.Collections.Concurrent`（ConcurrentDictionary など）を使う
3. **第三選択:** `Channel(Of T)` を使ってメッセージパッシングでアクセスをシリアル化する
4. **最終手段:** 単純かつ短命なクリティカルセクションに限り `SyncLock` を使う

---

## 判断ツリー

```
何をしたいか？
│
├─► I/O待機（HTTP、データベース、ファイル）?
│   └─► Async/Await を使う
│
├─► コレクションを並列処理（CPUバウンド）?
│   └─► Parallel.ForEachAsync を使う
│
├─► プロデューサー/コンシューマーパターン（ワークキュー）?
│   └─► System.Threading.Channels を使う
│
├─► UIイベント処理（デバウンス、スロットル、結合）?
│   └─► Reactive Extensions (Rx) を使う
│
├─► サーバーサイドのストリーム処理（バックプレッシャー、バッチ）?
│   └─► Akka.NET Streams を使う
│
├─► 複雑な遷移を持つステートマシン?
│   └─► Akka.NET アクター（Become パターン）を使う
│
├─► 多数の独立したエンティティの状態管理?
│   └─► Akka.NET アクター（エンティティ単位アクター）を使う
│
├─► 複数の非同期操作を協調させる?
│   └─► Task.WhenAll / Task.WhenAny を使う
│
└─► 上記のどれにも当てはまらない?
    └─► 自問する: 「本当に共有可変状態が必要か？」
        ├─► Yes → 設計を見直して回避することを検討する
        └─► 本当に避けられない → Channel またはアクターでアクセスをシリアル化する
```

---

## レベル 1: Async/Await（デフォルトの選択）

**用途:** I/O バウンド処理、ノンブロッキングな待機、日常的な並行処理のほとんど。

```vbnet
' シンプルな非同期 I/O
Public Async Function GetOrderAsync(orderId As String, ct As CancellationToken) As Task(Of Order)
    Dim order = Await _database.GetAsync(orderId, ct)
    Dim customer = Await _customerService.GetAsync(order.CustomerId, ct)
    ' VB.NET に with 式はないため、新しいインスタンスを作成して設定する
    Return New Order With {.Id = order.Id, .Customer = customer}
End Function

' 非同期操作の並列実行（互いに独立している場合）
Public Async Function LoadDashboardAsync(userId As String, ct As CancellationToken) As Task(Of Dashboard)
    Dim ordersTask = _orderService.GetRecentOrdersAsync(userId, ct)
    Dim notificationsTask = _notificationService.GetUnreadAsync(userId, ct)
    Dim statsTask = _statsService.GetUserStatsAsync(userId, ct)

    Await Task.WhenAll(ordersTask, notificationsTask, statsTask)

    Return New Dashboard(
        Await ordersTask,
        Await notificationsTask,
        Await statsTask)
End Function
```

**主要原則:** 常に `CancellationToken` を受け取る。ライブラリコードでは `ConfigureAwait(False)` を使う。非同期コードをブロックしない。

---

## レベル 2: Parallel.ForEachAsync（CPUバウンドの並列処理）

**用途:** CPU バウンド処理、または並行数を制御しながらコレクションを並列処理する場合。

```vbnet
Public Async Function ProcessOrdersAsync(
    orders As IEnumerable(Of Order),
    ct As CancellationToken) As Task

    Await Parallel.ForEachAsync(
        orders,
        New ParallelOptions With {
            .MaxDegreeOfParallelism = Environment.ProcessorCount,
            .CancellationToken = ct
        },
        Async Function(order, token)
            Await ProcessOrderAsync(order, token)
        End Function)
End Function
```

**使ってはいけない場面:** 純粋な I/O 処理、順序が重要な場合、バックプレッシャーが必要な場合。

---

## レベル 3: System.Threading.Channels（プロデューサー/コンシューマー）

**用途:** ワークキュー、プロデューサー/コンシューマーパターン、プロデューサーとコンシューマーの分離。

```vbnet
Public Class OrderProcessor
    Private ReadOnly _channel As Channel(Of Order)

    Public Sub New()
        _channel = Channel.CreateBounded(Of Order)(New BoundedChannelOptions(100) With {
            .FullMode = BoundedChannelFullMode.Wait
        })
    End Sub

    ' プロデューサー
    Public Async Function EnqueueOrderAsync(order As Order, ct As CancellationToken) As Task
        Await _channel.Writer.WriteAsync(order, ct)
    End Function

    ' コンシューマー（バックグラウンドタスクとして実行）
    ' 注意: VB.NET は Await For Each 構文をサポートしないため、
    ' IAsyncEnumerator を手動で操作する
    Public Async Function ProcessOrdersAsync(ct As CancellationToken) As Task
        Dim enumerator = _channel.Reader.ReadAllAsync(ct).GetAsyncEnumerator(ct)
        ' VB.NET は Finally 内に Await を書けない（BC36943）。
        ' 例外時も DisposeAsync を呼ぶために Try/Catch で例外を捕捉し、
        ' Try 外で Await DisposeAsync を呼んだ後に再 throw する 4-block パターンを使う。
        Dim thrown As Exception = Nothing
        Try
            While Await enumerator.MoveNextAsync()
                Await ProcessOrderAsync(enumerator.Current, ct)
            End While
        Catch ex As Exception
            thrown = ex
        End Try
        Await enumerator.DisposeAsync()  ' outside Try/Catch — Await is allowed here
        If thrown IsNot Nothing Then
            Throw thrown
        End If
    End Function

    Public Sub Complete()
        _channel.Writer.Complete()
    End Sub
End Class
```

**Channel が適している場面:** 処理速度の差の吸収、バックプレッシャーを持つバッファリング、ワーカーへのファンアウト、バックグラウンドキュー。

**Channel が適していない場面:** 複雑なストリーム操作（バッチ、ウィンドウ処理）、エンティティ単位のステートフル処理、高度なスーパービジョン。

---

## レベル 4+: Akka.NET Streams、Reactive Extensions、アクター

ストリーム処理、UI イベント合成、ステートフルなエンティティ管理などの高度なシナリオは [advanced-concurrency.md](advanced-concurrency.md) を参照する。

**Akka.NET Streams** はサーバーサイドのバッチ処理、スロットル、バックプレッシャーに優れる。**Reactive Extensions** は UI イベント合成に最適だ。**Akka.NET アクター** はエンティティ単位アクターパターン、`Become()` を使ったステートマシン、Cluster Sharding を用いた分散システムを扱う。

なお、Akka.NET は VB.NET からも利用可能だが、C# と比較すると一部のイディオムは冗長になる場合がある。

---

## アンチパターン: 避けるべきもの

### ビジネスロジックへの SyncLock の使用

```vbnet
' 悪い例: 共有状態を保護するために SyncLock を使う
Private ReadOnly _lock As New Object()
Private _orders As New Dictionary(Of String, Order)()

Public Sub UpdateOrder(id As String, update As Action(Of Order))
    SyncLock _lock
        Dim order As Order = Nothing
        If _orders.TryGetValue(id, order) Then update(order)
    End SyncLock
End Sub

' 良い例: アクターまたは Channel を使ってアクセスをシリアル化する
```

### 手動のスレッド管理

```vbnet
' 悪い例: スレッドを手動で作成する
Dim thread = New Thread(Sub() ProcessOrders())
thread.Start()

' 良い例: Task.Run またはより高レベルな抽象を使う
Dim ignored = Task.Run(Async Function() Await ProcessOrdersAsync(cancellationToken))
```

### 非同期コード内でのブロッキング

```vbnet
' 悪い例: 非同期処理をブロック待機する — デッドロックの危険！
Dim result = GetDataAsync().Result

' 良い例: 非同期を末端まで貫く
Dim result = Await GetDataAsync()
```

### 保護なしの共有可変状態

```vbnet
' 悪い例: 複数のタスクが共有状態を変更する
Dim results As New List(Of Result)()
Await Parallel.ForEachAsync(items, Async Function(item, ct)
    Dim result = Await ProcessAsync(item, ct)
    results.Add(result) ' 競合状態！
End Function)

' 良い例: ConcurrentBag を使う
Dim results As New ConcurrentBag(Of Result)()
```

---

## クイックリファレンス: どのツールをいつ使うか

| 要件 | ツール | 例 |
|------|------|---------|
| I/O 待機 | `Async`/`Await` | HTTP 呼び出し、データベースクエリ |
| CPU バウンドの並列処理 | `Parallel.ForEachAsync` | 画像処理、計算 |
| ワークキュー | `Channel(Of T)` | バックグラウンドジョブ処理 |
| デバウンス/スロットルを伴う UI イベント | Reactive Extensions | 検索候補表示、自動保存 |
| サーバーサイドのバッチ/スロットル | Akka.NET Streams | イベント集約、レート制限 |
| ステートマシン | Akka.NET アクター | 支払いフロー、注文ライフサイクル |
| エンティティの状態管理 | Akka.NET アクター | 注文管理、ユーザーセッション |
| 複数の非同期操作を起動 | `Task.WhenAll` | ダッシュボードデータの読み込み |
| 複数の非同期操作を競争 | `Task.WhenAny` | タイムアウトとフォールバック |
| 定期的な処理 | `PeriodicTimer` | ヘルスチェック、ポーリング |

---

## エスカレーションパス

```
Async/Await（ここから始める）
    │
    ├─► 並列処理が必要? → Parallel.ForEachAsync
    │
    ├─► プロデューサー/コンシューマーが必要? → Channel(Of T)
    │
    ├─► UI イベント合成が必要? → Reactive Extensions
    │
    ├─► サーバーサイドのストリーム処理が必要? → Akka.NET Streams
    │
    └─► ステートマシンまたはエンティティ管理が必要? → Akka.NET アクター
```

**具体的な要件がある場合にのみ上位に移行する。** 「念のため」アクターやストリームに手を伸ばさない。
