---
name: vb-winforms
description: "VB.NET で Windows Forms アプリケーションを構築、保守、またはモダナイズする。デザイナー駆動 UI、イベント処理、データバインディング、MVP 分離、モダン .NET への移行を実践的に扱う。WinForms プロジェクトの開発または .NET Framework からの移行時に使用する。"
compatibility: ".NET または .NET Framework 上の Windows Forms プロジェクトが必要。"
---

# Windows Forms

## Trigger On

- Windows Forms UI、イベント駆動ワークフロー、またはクラシック LOB アプリケーションの開発
- WinForms を .NET Framework からモダン .NET へ移行する
- 肥大化したフォームコードやデザイナー結合の整理
- データバインディング、入力検証、またはコントロールカスタマイズの実装

## Workflow

1. **デザイナー境界を尊重する** — `.Designer.vb` を直接編集しない。再生成時に変更が失われる。
2. **ビジネスロジックをフォームから分離する** — MVP（Model-View-Presenter）パターンを使用する。フォームは UI を統括し、プレゼンターがロジックを保持し、サービスがデータアクセスを担う。
   ```vbnet
   ' View インターフェース — フォームがこれを実装する
   Public Interface ICustomerView
       Property CustomerName As String
       Event SaveRequested As EventHandler
       Sub ShowError(message As String)
   End Interface

   ' Presenter — UI なしでテスト可能
   Public Class CustomerPresenter
       Private ReadOnly _view As ICustomerView
       Private ReadOnly _service As ICustomerService

       Public Sub New(view As ICustomerView, service As ICustomerService)
           _view = view
           _service = service
           AddHandler _view.SaveRequested, Async Sub(s, e)
               Try
                   Await _service.SaveAsync(_view.CustomerName)
               Catch ex As Exception
                   _view.ShowError(ex.Message)
               End Try
           End Sub
       End Sub
   End Class
   ```
3. **Program.vb から DI を使用する**（.NET 6+、コピペで実行可能な完全版）:
   ```vbnet
   Imports Microsoft.Extensions.DependencyInjection
   Imports System.Windows.Forms

   Module Program
       <STAThread>
       Sub Main()
           ' .NET 6+ の WinForms ソースジェネレータが生成する初期化ヘルパー
           ' （<OutputType>WinExe</OutputType> + <UseWindowsForms>true</UseWindowsForms> の
           '  プロジェクトで自動生成される。Library ビルドでは使用不可）
           ApplicationConfiguration.Initialize()

           ' 旧来の個別呼び出し形式（.NET 5 以前や手動制御したい場合）:
           ' Application.EnableVisualStyles()
           ' Application.SetCompatibleTextRenderingDefault(False)
           ' Application.SetHighDpiMode(HighDpiMode.SystemAware)

           Dim services As New ServiceCollection()
           services.AddSingleton(Of ICustomerService, CustomerService)()
           services.AddTransient(Of MainForm)()

           Using sp = services.BuildServiceProvider()
               Application.Run(sp.GetRequiredService(Of MainForm)())
           End Using
       End Sub
   End Module
   ```
   必須事項: `<STAThread>` 属性（WinForms は STA 必須）、`Module Program` + `Sub Main`（VB.NET は top-level statements を持たない）、`ApplicationConfiguration.Initialize()` または従来 3 API 呼び出し（ビジュアルスタイル + DPI 対応）。これらを省略すると正しく起動しない。
4. **`BindingSource` と `INotifyPropertyChanged` を使ったデータバインディング**を利用し、手動によるコントロール設定を避ける。完全なバインディングパターンは references/patterns.md を参照。
5. **I/O 操作には async/await を使用する** — 読み込み中はコントロールを無効化し、進捗報告には `Progress(Of T)` を使用する。UI スレッドをブロックしない。
6. **`ErrorProvider` と `Validating` イベントで入力検証を行う**。保存操作の前に `ValidateChildren()` を呼び出す。
7. **段階的にモダナイズする** — ビッグバン的な書き換えより、良い構造への漸進的改善を優先する。利用可能であれば .NET 8+ の機能（ボタンコマンド、ストックアイコン）を活用する。

```mermaid
flowchart LR
  A["Form event"] --> B["Presenter handles logic"]
  B --> C["Service layer / data access"]
  C --> D["Update view via interface"]
  D --> E["Validate and display results"]
```

## Key Decisions

| 決定事項 | ガイダンス |
|----------|----------|
| MVP vs MVVM | WinForms では MVP を推奨 — イベント駆動モデルとの親和性が高い |
| BindingSource vs 手動 | リスト/詳細バインディングでは常に BindingSource を優先する |
| 同期 vs 非同期 I/O | 常に非同期 — `Async Sub` はイベントハンドラにのみ使用する |
| カスタムコントロール | フォームが約 300 行を超えたら再利用可能な `UserControl` に抽出する |
| .NET Framework → .NET | 公式移行ガイドを使用し、まずデザイナーの互換性を検証する |

## Deliver

- 明確な UI/ロジック分離による堅牢なフォームコード
- テスト可能なプレゼンターを持つ MVP パターン
- WinForms 重視のアプリに対する実践的なモダナイズガイダンス
- 手動配線を削減するデータバインディングと入力検証のパターン

## Validate

- デザイナーファイルが安定しており、手動編集されていない
- フォームがアプリケーションサービス層として機能していない
- 非同期処理が UI スレッドをブロックしていない
- 入力検証が ErrorProvider で一貫して実装されている
- Windows 専用のランタイム動作がターゲット環境でテストされている

## References

- references/patterns.md - WinForms アーキテクチャパターン（MVP、MVVM、Passive View）、データバインディング、入力検証、フォーム間通信、スレッド処理、DI セットアップ、および .NET 8+ の機能
- references/migration.md - .NET Framework からモダン .NET へのステップバイステップの移行手順、よくある問題、デプロイオプション、段階的移行戦略
