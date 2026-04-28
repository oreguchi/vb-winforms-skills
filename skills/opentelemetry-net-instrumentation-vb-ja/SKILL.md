---
name: opentelemetry-net-instrumentation-vb-ja
description: VB.NET コードベースで OpenTelemetry の計装を実装するための指針を提供する。トレース（Activities/Spans）、メトリクス、命名規則、エラー処理、性能、API 設計のベストプラクティスをカバーする。WinForms 環境特有の補足（UI スレッド境界、Generic Host 統合、長期稼働時ライフサイクル）も別ファイルで提供する。
version: 1.0.0
tags:
  - opentelemetry
  - dotnet
  - vbnet
  - winforms
  - observability
  - tracing
  - metrics
  - performance
---

# VB.NET 向け OpenTelemetry 計装スキル

## 概要

VB.NET コードベースで OpenTelemetry の計装を実装するための指針を提供する。トレース（Activities/Spans）、メトリクス、命名規則、エラー処理、性能、API 設計のベストプラクティスをカバーする。

## いつ使うか

- VB.NET コードに OpenTelemetry 計装を追加するとき
- `ActivitySource` やメトリクスを作成・修正するとき
- 既存のテレメトリ実装を準拠性の観点でレビューするとき
- 計装の性能を最適化するとき
- 公開 API 表面の一部となるテレメトリ API を設計するとき

## 前提条件

- OpenTelemetry SDK を利用する VB.NET アプリケーション
- `System.Diagnostics.Metrics` および `ActivitySource` API の理解
- オブザーバビリティのバックエンド（Jaeger、Prometheus、Grafana など）へのアクセス

> 以下のサンプルコードは `Imports System.Diagnostics`、`Imports System.Diagnostics.Metrics`、`Imports System.Collections.Generic`、`Imports System.Linq` を前提とする。

## コア原則

### 回復性ファースト

**重要**：診断／トレース／メトリクスのロジックで発生した例外が、アプリケーション処理に影響を **絶対に与えてはならない**。

- `Activity` 拡張メソッド内を除き、常に null の `Activity` 参照に対して保護する（`activity?.ExtensionMethod()` を使う）
- `Activity` インスタンスは null になり得ると想定する（リスナーが購読したときのみ作成される）
- すべての計装コードを適切な null チェックで保護する

### API 表面の認識

- 出力（emit）されたあらゆるテレメトリは公開 API 表面の一部となる
- 変更は破壊的変更ガイドラインの対象となる
- テレメトリは既定で出力すべき（収集側は OpenTelemetry SDK で利用者がオプトインする）
- 例外：高カーディナリティのメトリクスディメンションは明示的なオプトインを要する場合がある

### 標準準拠

- マイクロソフトのベストプラクティス [distributed tracing instrumentation](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/distributed-tracing-instrumentation-walkthroughs) に従う
- [OpenTelemetry semantic conventions](https://opentelemetry.io/docs/concepts/semantic-conventions/) に従う
- すべての属性値は非 null かつ非空の文字列でなければならない

## トレース / スパン（Activity）

### `ActivitySource` のセットアップ

```vbnet
' ✅ 正しい：DiagnosticSource ではなく ActivitySource を使う
Public Class MyFeature
    ' プライマリ ActivitySource — 名前は通常コンポーネント名または NuGet パッケージ名と一致させる
    Private Shared ReadOnly ActivitySource As New ActivitySource("MyApp.MyComponent", "1.0.0")

    ' オプトイン用の追加 ActivitySource（用途を絞った特殊用途向け）
    Private Shared ReadOnly DetailedActivitySource As New ActivitySource("MyApp.MyComponent.Detailed", "1.0.0")
End Class
```

**ルール**：
- すべてのコンポーネントは主要なアクティビティ用にプライマリ `ActivitySource` を定義する
- 名前は通常コンポーネントまたは NuGet パッケージ名と一致させる（例：`"MyCompany.MyLibrary"`）
- `ActivitySource` は SemVer でバージョニングする
- 特殊用途やオプトイン専用のシナリオには別の `ActivitySource` を作成する

### `Activity` の作成

```vbnet
' ✅ 正しい：作成前に HasListeners をチェックする
If ActivitySource.HasListeners() Then
    Using activity As Activity = ActivitySource.StartActivity("ProcessItem", ActivityKind.Internal)
        If activity IsNot Nothing Then
            activity.DisplayName = "Processing order #12345"

            ' 要求されたときのみ重いタグ計算を行う
            If activity.IsAllDataRequested Then
                activity.SetTag("app.item_id", itemId)
                activity.SetTag("app.item_type", itemType)
            End If
        End If
    End Using
End If

' ❌ 誤り：非同期ヘルパーメソッド内で Activity を開始しない（呼び出し元から見て親子関係が壊れる）
Private Async Function HelperAsync() As Task
    Using activity As Activity = ActivitySource.StartActivity("Helper") ' ❌ よくない
        Await DoWorkAsync()
    End Using
End Function
```

**ルール**：
- 作成前に `ActivitySource.HasListeners()` をチェックする（ゼロアロケーションの高速パス）
- 作成後は必ず `activity` が `Nothing` でないか確認する
- 非同期ヘルパーメソッド内で Activity を開始しない（`Activity.Current` は `AsyncLocal(Of T)` ベース。ヘルパー内で Activity を `Dispose` すると `Activity.Current` が呼び出し側で本来の値に戻らない場合があり、呼び出し側のトレース親子関係が崩れる）
- 高コストな計算の前に `activity.IsAllDataRequested` をチェックする
- 常に W3C ID 形式を使う（親が階層形式の場合は形式変更を強制する）

### `Activity` の命名

```vbnet
' ✅ 正しい：一意の操作名 + 親しみやすい表示名
Using activity As Activity = ActivitySource.StartActivity(
    name:="ProcessItem",              ' 一意、スパンのクラスを識別
    kind:=ActivityKind.Internal
)
    activity.DisplayName = "Processing order #12345" ' 人間に読みやすい、具体的でよい
End Using

' ❌ 誤り：操作名に実行時データを含めない
Using activity As Activity = ActivitySource.StartActivity($"Process_{itemId}") ' ❌ よくない
End Using
```

**ルール**：
- 各スパン種別は一意の `OperationName` を持つ（統計的に意味のあるスパンのクラスを識別）
- 操作名に実行時データを含めない（コンパイル時／設定時の情報のみ）
- 具体情報は人間に読みやすい `DisplayName` で表現する
- [OpenTelemetry スパン命名規約](https://opentelemetry.io/docs/specs/otel/trace/api/#span) に従う

### スパン属性（タグ）

```vbnet
' ✅ 正しい：名前空間付き、小文字、アンダースコア区切り
activity?.SetTag("myapp.order_id", orderId)
activity?.SetTag("myapp.order_type", orderType)
activity?.SetTag("myapp.db.table_name", tableName)

' 該当する場合は標準 semantic conventions を使う
activity?.SetTag("db.system", "postgresql")
activity?.SetTag("http.method", "GET")

' ❌ 誤り：さまざまな命名規則違反
activity?.SetTag("MyApp.OrderId", orderId)         ' ❌ 大小文字違反
activity?.SetTag("myapp.order-id", orderId)        ' ❌ 区切り文字違反
activity?.SetTag("myapp.orders", count)            ' ❌ 複数形
activity?.SetTag("unrelated.ip_address", ip)       ' ❌ このアクティビティと無関係
```

**命名規則**：
- コンポーネントに合わせた名前空間 prefix を使う：`myapp.*`、`myapp.db.*`
- 全て小文字
- 複数語の属性はアンダースコア（`_`）区切り
- 単数形
- このアクティビティに直接関連するタグのみ設定する
- 該当する標準 [OpenTelemetry semantic conventions](https://opentelemetry.io/docs/specs/semconv/) を独自属性より優先する
- 標準 semantic convention は、下流のライブラリが同じ規約を設定しないと確信できる場合のみ自前で設定する（重複設定を避けるため）

### `Activity` のステータスとエラー

```vbnet
' ✅ 正しい：ステータスを設定し例外を記録する
Try
    Await ProcessItemAsync()
    activity?.SetStatus(ActivityStatusCode.Ok)
Catch ex As Exception
    If activity IsNot Nothing Then
        activity.SetStatus(ActivityStatusCode.Error)
        activity.SetTag("otel.status_code", "error")
        activity.SetTag("otel.status_description", ex.Message)

        ' OTel 仕様に従って exception イベントを記録
        activity.AddEvent(New ActivityEvent(
            "exception",
            tags:=New ActivityTagsCollection From {
                {"exception.type", ex.GetType().FullName},
                {"exception.message", ex.Message},
                {"exception.stacktrace", ex.ToString()}
            }
        ))
    End If
    Throw
End Try
```

**ルール**：
- 成功時は `ActivityStatusCode.Ok` を設定する
- 例外時は `ActivityStatusCode.Error` を設定する
- 必ず `otel.status_code` と `otel.status_description` タグを付与する
- [OTel 例外規約](https://opentelemetry.io/docs/reference/specification/trace/semantic_conventions/exceptions/) に従って exception イベントを記録する

### `Activity` のイベント

```vbnet
' ✅ 正しい：追加コンテキストにイベントを使う（控えめに）
activity?.AddEvent(New ActivityEvent("ItemRetried", tags:=New ActivityTagsCollection From {
    {"retry_attempt", retryCount},
    {"next_retry_delay", delayMs}
}))

' ❌ 誤り：詳細ログにイベントを使わない
activity?.AddEvent(New ActivityEvent($"Step {i} completed")) ' ❌ ログ機能を使うべき
```

**ルール**：
- イベントは送信までインメモリに保持される（控えめに使う）
- 追加コンテキスト用のみ。複数イベントが必要ならネストしたスパンを検討する
- 詳細情報にはログ機能を使う

### `Activity` へのアクセス

```vbnet
' ❌ 誤り：特定のスパンが必要な場合、Activity.Current に依存しない
Public Async Function HandleAsync(context As Context) As Task
    Dim activity As Activity = Activity.Current ' ❌ ユーザー作成のスパンかもしれず、自分が開始したものとは限らない
    activity?.SetTag("custom", "value")
End Function

' ✅ 正しい：Activity を明示的に渡す、または専用 context オブジェクトに格納する
Public Async Function HandleAsync(context As Context) As Task
    Dim activity As Activity = Nothing
    If context.TryGetActivity(activity) Then
        activity?.SetTag("custom", "value")
    End If
End Function
```

## メトリクス

### `Meter` とメトリクスクラスのセットアップ

```vbnet
' ✅ 正しい：機能／コンポーネント単位でメトリクスをグルーピング
Public NotInheritable Class OrderProcessingMetrics
    Implements IDisposable

    Private ReadOnly meter As Meter
    Private ReadOnly processingDuration As Histogram(Of Double)
    Private ReadOnly itemsProcessed As Counter(Of Long)

    Public Sub New()
        meter = New Meter("MyApp.OrderProcessing", "1.0.0")

        ' 単数形の名前、適切な単位、ネスト階層
        processingDuration = meter.CreateHistogram(Of Double)(
            "myapp.order.processing.duration",
            unit:="s",
            description:="Duration of order processing"
        )

        itemsProcessed = meter.CreateCounter(Of Long)(
            "myapp.order.processing.count",
            unit:="{order}",
            description:="Number of orders processed"
        )
    End Sub

    Public Sub Dispose() Implements IDisposable.Dispose
        meter.Dispose()
    End Sub
End Class
```

**命名規則**（[OTel semantic conventions](https://opentelemetry.io/docs/specs/semconv/general/metrics/) に従う）：
- 単数形の名前を使う（数量を示したいときは複数形ではなく `_count` 接尾辞を付ける）
- ネスト階層：`myapp.order.processing.duration`
- 単位を定義する（s、ms、{item}、{connection}）
- 技術的接尾辞を避ける（`_counter`、`_histogram`）
- 採用が証明されるまでは pre-1.0.0 バージョンから始める

### メトリクス記録メソッドの命名

```vbnet
' ✅ 正しい：アクション／結果ベースの命名、結果ごとに別メソッド
Public NotInheritable Class OrderProcessingMetrics
    ' 発生したイベント：何が起こったかを記述
    Public Sub OrderProcessingSucceeded(orderType As String, duration As TimeSpan)
        processingDuration.Record(duration.TotalSeconds,
            New KeyValuePair(Of String, Object)("myapp.order_type", orderType),
            New KeyValuePair(Of String, Object)("outcome", "success")
        )
    End Sub

    Public Sub OrderProcessingFailed(orderType As String, exception As Exception, duration As TimeSpan)
        processingDuration.Record(duration.TotalSeconds,
            New KeyValuePair(Of String, Object)("myapp.order_type", orderType),
            New KeyValuePair(Of String, Object)("outcome", "failure"),
            New KeyValuePair(Of String, Object)("exception.type", exception.GetType().Name)
        )
    End Sub

    Public Sub ConnectionOpened()
        connectionsOpen.Add(1)
    End Sub

    Public Sub ConnectionClosed()
        connectionsOpen.Add(-1)
    End Sub
End Class

' ❌ 誤り：さまざまな命名アンチパターン
Public Sub RecordOrderProcessingDuration(...) ' ❌ メトリクス名を関数名にしない
Public Sub RecordError(succeeded As Boolean, ex As Exception) ' ❌ 紛らわしいシグネチャ
```

**ルール**（ASP.NET Core パターンを参考）：
- アクション／結果で命名する：`OrderProcessingSucceeded`、`RetryAttempted`、`ConnectionFailed`
- メトリクス名で命名しない：`RecordXxx`、`IncrementXxx` を避ける
- 結果ごとに別メソッドにする（boolean フラグ＋オプション例外の組み合わせを避ける）
- 状態変化はイベントベース命名：`ConnectionOpened()`、`ItemQueued()`

### メトリクスのディメンション

```vbnet
' ✅ 正しい：低カーディナリティ、事前定義のディメンション
Public Sub OrderProcessingSucceeded(orderType As String, duration As TimeSpan)
    processingDuration.Record(duration.TotalSeconds,
        New KeyValuePair(Of String, Object)("myapp.order_type", orderType),
        New KeyValuePair(Of String, Object)("myapp.region", region),
        New KeyValuePair(Of String, Object)("outcome", "success")
    )
End Sub

' ❌ 誤り：高カーディナリティのディメンション（無制限な値はカーディナリティ爆発を引き起こす）
Public Sub OrderFailed(orderId As String, exceptionMessage As String)
    failureCount.Add(1,
        New KeyValuePair(Of String, Object)("order_id", orderId),               ' ❌ 無制限
        New KeyValuePair(Of String, Object)("exception_message", exceptionMessage) ' ❌ 無制限
    )
End Sub
```

**ルール**：
- ディメンションは Instrument（`Counter` / `Histogram` 等）の作成時に事前定義しなければならない
- 動的／無制限な値を避ける（カーディナリティ爆発：一意の値ごとに新しい時系列行が作られる）
- 高カーディナリティのディメンションはオプトイン設定にしなければならない
- 低カーディナリティの識別子を使う：item type、queue name、outcome
- ディメンション名はコンポーネント間で一貫させる：`myapp.region` はどこでも同じ意味
- 機微情報を避ける
- [メトリクスエンリッチメントの代替手段](https://github.com/open-telemetry/opentelemetry-dotnet/tree/main/docs/metrics#metrics-enrichment) を検討する
- 利用者は相関のために [メトリクス exemplar](https://github.com/open-telemetry/opentelemetry-dotnet/tree/main/docs/metrics#metrics-correlation) を有効化できる（ディメンション経由ではない）

## 性能要件

計装は既定で軽量でなければならない。オーバーヘッドを最小化するために以下のルールに従う。

### ゼロアロケーションの高速パス

```vbnet
' ✅ 正しい：軽いチェックでガードする
If ActivitySource.HasListeners() Then
    Using activity As Activity = ActivitySource.StartActivity("Operation")
        ' ... 重い処理
    End Using
End If

' ✅ 正しい：メトリクスには TagList（structure）を使う
Dim tags As New TagList()
tags.Add("myapp.order_type", orderType)
tags.Add("outcome", "success")
counter.Add(1, tags)
```

### タイミング計測

.NET 7 以降では `Stopwatch.GetElapsedTime` で簡潔に計測できる。古いランタイム（.NET 6 / .NET Framework 4.8）には同 API が無いため、`GetTimestamp` の差分を `Stopwatch.Frequency` で割る互換実装を使う。

```vbnet
' ✅ 正しい：.NET 7+ — タイムスタンプ計算（割り当てなし）
Dim startTime = Stopwatch.GetTimestamp()
Try
    Await ProcessAsync()
Finally
    Dim duration = Stopwatch.GetElapsedTime(startTime)
    metrics.OrderProcessingSucceeded(orderType, duration)
End Try

' ✅ 正しい：.NET 6 / .NET Framework 4.8 互換版（GetElapsedTime が無い環境）
Dim startTime2 = Stopwatch.GetTimestamp()
Try
    Await ProcessAsync()
Finally
    ' GetElapsedTime が無い環境向けの計算（ticks → 秒 → TimeSpan）
    Dim elapsedTicks = Stopwatch.GetTimestamp() - startTime2
    Dim duration = TimeSpan.FromSeconds(elapsedTicks / CDbl(Stopwatch.Frequency))
    metrics.OrderProcessingSucceeded(orderType, duration)
End Try

' ❌ 誤り：Stopwatch オブジェクトを割り当てる
Dim stopwatchObj = Stopwatch.StartNew() ' ❌ 割り当てが発生

' ❌ 誤り：IDisposable のタイミングクラス（使用ごとに割り当て）
Using New MetricScope(metrics, "ProcessOrder") ' ❌ よくない
    ProcessOrder()
End Using
```

### 隠れた割り当てを避ける

```vbnet
' ❌ 誤り：文字列補間で割り当てが発生
activity?.SetTag("item", $"Processing {itemId}") ' ❌ 割り当てが発生

' ✅ 正しい：先に IsAllDataRequested を確認
If activity IsNot Nothing AndAlso activity.IsAllDataRequested Then
    activity.SetTag("item", $"Processing {itemId}")
End If

' ❌ 誤り：LINQ で列挙子が割り当てられる
activity?.SetTag("handlers", handlers.Select(Function(h) h.Name).ToArray()) ' ❌ よくない

' ✅ 正しい：手動構築または事前確認
If activity IsNot Nothing AndAlso activity.IsAllDataRequested Then
    activity.SetTag("handlers", String.Join(",", handlers.Select(Function(h) h.Name)))
End If
```

**ルール**：
- `Stopwatch.StartNew()` を使わない（タイムスタンプ計算を使う）
- `IDisposable` のタイミングラッパークラスを使わない
- 配列／辞書よりも `TagList`（structure）を優先する
- 隠れた処理を避ける：ホットパスでは LINQ、文字列補間、async ステートマシンを避ける

## テスト要件

### スパンのテスト

```vbnet
<Test>
Public Async Function Should_create_processing_span_with_correct_parent() As Task
    ' Arrange
    Using parent As Activity = New Activity("Parent").Start()
        ' Act
        Await handler.Handle(item)

        ' Assert
        Dim processingSpan = recordedActivities.Single(Function(a) a.OperationName = "ProcessItem")
        Assert.AreEqual(parent.Id, processingSpan.ParentId)
        Assert.AreEqual("myapp.item_type", processingSpan.Tags.First().Key)
    End Using
End Function

<Test>
Public Sub Should_not_introduce_breaking_changes_to_span_names()
    ' スパン名の文字列値がテスト対象であることを保証
    Assert.AreEqual("ProcessItem", MyFeature.SpanName)
End Sub
```

**ルール**：
- アクティビティがどのスパンに接続するかをテストする
- 文字列値（スパン名、タグ名）をテストして破壊的変更を防ぐ
- テレメトリは公開 API の一部であることを忘れない

## バージョニング

- テレメトリのバージョニングはパッケージバージョンと切り離す
- SemVer のセマンティクスを使う
- トレースとメトリクスは別バージョンを使う（独立に進化させる）
- 採用度／有用性が証明されるまで pre-1.0.0 バージョンから始める

```vbnet
Private Shared ReadOnly ActivitySource As New ActivitySource("MyApp.MyComponent", "0.9.0")
Private ReadOnly meter As New Meter("MyApp.MyComponent", "0.8.0")
```

## VB.NET WinForms 特有の補足

本スキル中核は VB.NET 全般向け。WinForms 環境では以下の追加考慮が必要となるため、要点を以下に示す。**詳細パターン・コードサンプル全文は `references/winforms-supplement.md` を参照**。

### SDK セットアップ概要

WinForms アプリで OpenTelemetry SDK を有効化する標準パターン：

- **Generic Host 経由（推奨）**：`services.AddOpenTelemetry().WithTracing(...).WithMetrics(...)` を `Microsoft.Extensions.Hosting` の DI と統合。`MainForm` も DI 解決し `Application.Run` に渡す
- **`Sdk.CreateTracerProviderBuilder` 直接版**：Hosting なしの最小構成

`AddSqlClientInstrumentation()` で SQL Server 接続を **手書き計装ゼロでスパン化** できる。`AddHttpClientInstrumentation()` も同様に外部 API 呼び出しをカバー。詳細は `references/winforms-supplement.md` §SDK セットアップ。

### UI スレッドと Activity の境界

- **`Async Sub` は UI ハンドラー（`Button_Click` 等）専用**。ビジネスロジックは `Async Function ... As Task` を使う（`Async Sub` 内例外は呼び出し側で捕捉できず、計装の `SetStatus(Error)` も呼ばれない／BC42356）
- UI ハンドラー内では `Try/Catch` 必須、例外時 `activity?.SetStatus(ActivityStatusCode.Error)` を呼ぶ
- `Activity.Current` は `AsyncLocal(Of T)` で `Await` continuation には伝搬するが、**`Control.Invoke` / `BeginInvoke` 越境では見えない場合がある**
- UI 越境でタグ付けする場合は **Activity 参照を明示的にクロージャーでキャプチャ** する。新規スパンを張るなら `parentContext` で親を明示
- ワーカースレッドから UI コントロールに触ると `InvalidOperationException`（クロススレッド）。UI 値は `Task.Run` 突入前にローカル変数へスナップショット

詳細とコードサンプルは `references/winforms-supplement.md` §UI スレッドと Activity の境界。

### 長期稼働時のライフサイクル管理

- `ActivitySource` / `Meter` は **プロセス単一インスタンス**（シングルトン）にする。フォーム単位で `New` するとリスナーがコールバック保持しメモリリーク
- アプリ終了時は **`IHostedService.StopAsync` または `Application.ApplicationExit` でフラッシュ + Dispose** を呼ぶ。`ForceFlush` を `Dispose` の前に置くこと
- 長期稼働ほどカーディナリティ累積が深刻。「動的／無制限な値をディメンションにしない」原則の遵守が必須

詳細は `references/winforms-supplement.md` §長期稼働時のライフサイクル管理。

## 参考資料

- [OpenTelemetry .NET Trace ドキュメント](https://github.com/open-telemetry/opentelemetry-dotnet/tree/main/docs/trace)
- [OpenTelemetry .NET Metrics ドキュメント](https://github.com/open-telemetry/opentelemetry-dotnet/tree/main/docs/metrics)
- [OpenTelemetry Semantic Conventions](https://opentelemetry.io/docs/concepts/semantic-conventions/)
- [Microsoft 分散トレース計装](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/distributed-tracing-instrumentation-walkthroughs)
- [ASP.NET Core メトリクスの例](https://github.com/search?q=repo%3Adotnet%2Faspnetcore+Metrics&type=code)
- [OpenTelemetry Trace API スパン定義](https://opentelemetry.io/docs/specs/otel/trace/api/#span)
- [OpenTelemetry 例外規約](https://opentelemetry.io/docs/reference/specification/trace/semantic_conventions/exceptions/)
- [OpenTelemetry 属性仕様](https://github.com/open-telemetry/opentelemetry-specification/tree/main/specification/common#attribute)
- [OpenTelemetry カーディナリティ制限](https://github.com/open-telemetry/opentelemetry-dotnet/blob/main/docs/metrics/README.md#cardinality-limits)
