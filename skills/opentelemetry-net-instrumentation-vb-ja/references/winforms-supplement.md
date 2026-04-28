# VB.NET WinForms 補足

本スキル（OpenTelemetry .NET Instrumentation の VB.NET 版）の中核は VB.NET 全般向けに書かれている。WinForms 特有の状況（UI スレッド境界、長期稼働、SDK セットアップ）について、本ファイルで詳細パターンと具体例を提供する。

> 以下のサンプルコードは `Imports System.Diagnostics`、`Imports System.Diagnostics.Metrics`、`Imports System.Threading`、`Imports System.Threading.Tasks`、`Imports System.Windows.Forms` を前提とする。

---

## SDK セットアップ

WinForms アプリで OpenTelemetry SDK を有効化するパターンは 2 系統ある。

### パターン A：`OpenTelemetry.Extensions.Hosting` 経由（Generic Host、推奨）

`Microsoft.Extensions.Hosting` で DI / 設定 / ロギング / OpenTelemetry を統合する。長期稼働 WinForms アプリで最も保守性が高い。

```vbnet
Imports Microsoft.Extensions.DependencyInjection
Imports Microsoft.Extensions.Hosting
Imports OpenTelemetry
Imports OpenTelemetry.Metrics
Imports OpenTelemetry.Resources
Imports OpenTelemetry.Trace

Friend Module Program

    <STAThread>
    Public Sub Main(args As String())
        Application.SetHighDpiMode(HighDpiMode.SystemAware)
        Application.EnableVisualStyles()
        Application.SetCompatibleTextRenderingDefault(False)

        Dim appHost As IHost = Microsoft.Extensions.Hosting.Host.CreateDefaultBuilder(args) _
            .ConfigureServices(Sub(context, services)
                                   ' --- WinForms 依存サービス登録 ---
                                   services.AddSingleton(Of MainForm)()
                                   services.AddSingleton(Of OrderProcessingMetrics)()

                                   ' --- OpenTelemetry セットアップ ---
                                   services.AddOpenTelemetry() _
                                       .ConfigureResource(Sub(r) r.AddService(
                                           serviceName:="MyApp.WinForms",
                                           serviceVersion:="1.0.0")) _
                                       .WithTracing(Sub(tracing)
                                                        tracing.AddSource("MyApp.WinForms")
                                                        tracing.AddSqlClientInstrumentation()
                                                        tracing.AddHttpClientInstrumentation()
                                                        tracing.AddOtlpExporter()
                                                    End Sub) _
                                       .WithMetrics(Sub(metrics)
                                                        metrics.AddMeter("MyApp.WinForms")
                                                        metrics.AddOtlpExporter()
                                                    End Sub)
                               End Sub) _
            .Build()

        appHost.Start()

        Try
            ' MainForm を DI で解決し、WinForms メッセージループに渡す
            Dim mainForm = appHost.Services.GetRequiredService(Of MainForm)()
            Application.Run(mainForm)
        Finally
            ' フォーム終了 → Host を停止 → エクスポーターの最終フラッシュ
            appHost.StopAsync(TimeSpan.FromSeconds(10)).GetAwaiter().GetResult()
            appHost.Dispose()
        End Try
    End Sub

End Module
```

> **注**：変数名 `host` は VB.NET の大小区別なしルール下で型名 `Host` と衝突するため `appHost` を使用。あるいは `Microsoft.Extensions.Hosting.Host.CreateDefaultBuilder(...)` のように完全修飾でも回避可能。`ApplicationConfiguration.Initialize()` は C# 専用で VB.NET には存在しないため、`Application.SetHighDpiMode` + `Application.EnableVisualStyles` + `Application.SetCompatibleTextRenderingDefault` の 3 行で代替する。

**ポイント**：
- `services.AddOpenTelemetry().WithTracing(...).WithMetrics(...)` のチェーンが **公式推奨パターン**（Microsoft Learn / opentelemetry.io）
- `AddSqlClientInstrumentation()` は SQL Server / Azure SQL の全クエリにスパンを自動付与（手書き計装不要、強力）
- `AddHttpClientInstrumentation()` は外部 API 呼び出しを自動トレース化
- `appHost.StopAsync()` がエクスポーターのフラッシュを駆動。`Application.Run` 終了後の `Finally` で確実に呼ぶ

### パターン B：`Sdk.CreateTracerProviderBuilder` 直接版（Generic Host なし、軽量）

Generic Host を入れたくない小規模アプリ向け。

```vbnet
Imports OpenTelemetry
Imports OpenTelemetry.Metrics
Imports OpenTelemetry.Trace

Module Program

    Private tracerProvider As TracerProvider
    Private meterProvider As MeterProvider

    <STAThread>
    Public Sub Main()
        tracerProvider = Sdk.CreateTracerProviderBuilder() _
            .AddSource("MyApp.WinForms") _
            .AddSqlClientInstrumentation() _
            .AddOtlpExporter() _
            .Build()

        meterProvider = Sdk.CreateMeterProviderBuilder() _
            .AddMeter("MyApp.WinForms") _
            .AddOtlpExporter() _
            .Build()

        AddHandler Application.ApplicationExit,
            Sub(s, e)
                tracerProvider?.ForceFlush()
                meterProvider?.ForceFlush()
                tracerProvider?.Dispose()
                meterProvider?.Dispose()
            End Sub

        Application.EnableVisualStyles()
        Application.SetCompatibleTextRenderingDefault(False)
        Application.Run(New MainForm())
    End Sub

End Module
```

**選択指針**：DI / 設定 / ロギング統合が必要 → A、最小構成で済ませたい → B。

---

## UI スレッドと Activity の境界

### `Async Sub` vs `Async Function ... As Task`

- **`Async Sub` はイベントハンドラー（`Button_Click` 等）専用**
- ビジネスロジックは **`Async Function ... As Task`** で書く
- 理由：`Async Sub` 内の例外は呼び出し側で捕捉できず、UnhandledException に飛ぶ（BC42356 警告）。計装した `Activity` の `SetStatus(Error)` も呼ばれない

```vbnet
' ✅ 正しい：UI ハンドラーは Async Sub、Try/Catch 必須
Private Async Sub btnProcess_Click(sender As Object, e As EventArgs) Handles btnProcess.Click
    btnProcess.Enabled = False
    Try
        Dim result = Await ProcessOrderAsync(orderType:="standard")
        lblResult.Text = $"完了：{result}"
    Catch ex As Exception
        lblResult.Text = $"失敗：{ex.Message}"
        MessageBox.Show(ex.ToString(), "エラー")
    Finally
        btnProcess.Enabled = True
    End Try
End Sub

' ✅ ビジネスロジックは Async Function ... As Task
Private Async Function ProcessOrderAsync(orderType As String) As Task(Of String)
    ' ... Activity と Try/Catch を内部で実装
    Return "OK"
End Function
```

### `Activity.Current` の `AsyncLocal` 伝搬挙動

`Activity.Current` は内部的に `AsyncLocal(Of Activity)` で実装されている。

| 境界 | 伝搬するか |
|---|---|
| `Await` を介した continuation | **伝搬する**（ExecutionContext がフロー） |
| `Task.Run(...)` 内 | **伝搬する**（同上） |
| `Control.Invoke(...)` 越境 | **伝搬しない場合がある**（UI スレッドの既存 ExecutionContext で実行されるため） |
| `Control.BeginInvoke(...)` 越境 | **伝搬しない場合がある**（同上） |

### `Control.Invoke` 越境時のスパン継続

```vbnet
' ❌ 誤り：Control.Invoke 内で Activity.Current に依存する
Private Async Function BadPatternAsync() As Task
    Using activity As Activity = ActivitySource.StartActivity("Render")
        Await Task.Run(
            Sub()
                Me.Invoke(
                    Sub()
                        ' ❌ ここでの Activity.Current は Nothing になり得る
                        Activity.Current?.SetTag("ui.label_text", lblResult.Text)
                    End Sub)
            End Sub)
    End Using
End Function

' ✅ 正しい：activity 参照を明示的にクロージャーでキャプチャ
Private Async Function GoodPatternAsync() As Task
    If Not ActivitySource.HasListeners() Then
        Await Task.Run(Sub() DoRender())
        Return
    End If

    Using activity As Activity = ActivitySource.StartActivity("Render")
        Dim capturedActivity As Activity = activity ' クロージャー用に明示確保

        Await Task.Run(
            Sub()
                Me.Invoke(
                    Sub()
                        ' ✅ UI スレッド上でも capturedActivity は確実に参照可能
                        ' Activity.SetTag はスレッドセーフ（内部 ConcurrentDictionary）
                        If capturedActivity IsNot Nothing AndAlso capturedActivity.IsAllDataRequested Then
                            capturedActivity.SetTag("ui.label_text", lblResult.Text)
                        End If
                    End Sub)
            End Sub)
    End Using
End Function
```

新しいスパンを UI スレッド側で開始したい場合は `parentContext` を明示渡し：

```vbnet
Dim parentContext = capturedActivity.Context
Me.Invoke(
    Sub()
        Using childActivity As Activity = ActivitySource.StartActivity(
            "UpdateUI", ActivityKind.Internal, parentContext:=parentContext)
            ' 親子関係を明示的に保つ
        End Using
    End Sub)
```

### `InvalidOperationException`（クロススレッド）の罠

`Task.Run` 等のワーカースレッドから直接 UI コントロールに触ると `InvalidOperationException: Cross-thread operation not valid` が発生する。

```vbnet
' ❌ 誤り：ワーカースレッドから UI を読む
Await Task.Run(
    Sub()
        Dim value = txtInput.Text ' ❌ Cross-thread 例外
        activity?.SetTag("user.input", value)
    End Sub)

' ✅ 正しい：UI 値を Task.Run 突入前にスナップショット
Dim inputValue As String = txtInput.Text
Await Task.Run(
    Sub()
        activity?.SetTag("user.input", inputValue) ' ローカル変数なので OK
    End Sub)
```

---

## 長期稼働時のライフサイクル管理

### `ActivitySource` / `Meter` のシングルトン化

WinForms アプリで `ActivitySource` / `Meter` をフォーム単位で `New` すると、内部 `ActivityListener` / `MeterListener` がコールバックを保持し続け、フォームを Close しても GC されない（実質的なメモリリーク）。

**プロセス全体で 1 つに集約する**：

```vbnet
Imports System.Diagnostics
Imports System.Diagnostics.Metrics

Public Module Telemetry
    ' プロセス単一インスタンス
    Public ReadOnly ActivitySource As New ActivitySource("MyApp.WinForms", "1.0.0")
    Public ReadOnly Meter As New Meter("MyApp.WinForms", "1.0.0")
End Module
```

各フォーム / クラスからは `Telemetry.ActivitySource` / `Telemetry.Meter` を参照する。

### アプリ終了時のフラッシュと Dispose

未送信テレメトリの消失を防ぐため、アプリ終了時に **必ず flush + dispose** する。

#### Generic Host を使う場合（`IHostedService` で）

```vbnet
Friend NotInheritable Class TelemetryLifetimeService
    Implements IHostedService

    Public Function StartAsync(cancellationToken As CancellationToken) As Task _
        Implements IHostedService.StartAsync
        Return Task.CompletedTask
    End Function

    Public Function StopAsync(cancellationToken As CancellationToken) As Task _
        Implements IHostedService.StopAsync
        ' Meter / ActivitySource を明示的に Dispose（リーク防止）
        Telemetry.Meter.Dispose()
        Telemetry.ActivitySource.Dispose()
        Return Task.CompletedTask
    End Function
End Class
```

DI 登録：
```vbnet
services.AddHostedService(Of TelemetryLifetimeService)()
```

`appHost.StopAsync()` が呼ばれると `IHostedService.StopAsync` が走り、その後 OTel SDK のエクスポーターも自動でフラッシュされる。

#### Generic Host なしの場合（`Application.ApplicationExit` で）

```vbnet
AddHandler Application.ApplicationExit,
    Sub(s, e)
        tracerProvider?.ForceFlush(timeoutMilliseconds:=5000)
        meterProvider?.ForceFlush(timeoutMilliseconds:=5000)
        tracerProvider?.Dispose()
        meterProvider?.Dispose()
        Telemetry.Meter.Dispose()
        Telemetry.ActivitySource.Dispose()
    End Sub
```

`ForceFlush` は同期的にエクスポーターをフラッシュする（最大 5 秒待つ）。`Dispose` の前に呼ぶことが重要。

### カーディナリティ管理

長期稼働ほど時系列が累積するため、本スキルの「メトリクスのディメンション」節（特に「動的／無制限な値を避ける」）の遵守が一層重要。`order_id` / `user_email` / `exception_message` などをディメンションにすると、稼働時間に比例してメモリと送信コストが線形増加する。
