# vb-winforms-skills

**VB.NET / WinForms で Claude Code を使うときに「最初から知っていてほしいこと」を 6 つのスキルに整理したプラグイン集。**

C# の OSS スキルを VB.NET + 日本語に変換しているため、AI が C# のコードや解説で答えてしまう問題を抑え、最初から VB.NET で答えてくれるようになります。

## なぜこれが必要か

Claude Code（Anthropic の CLI コーディングエージェント）は、デフォルトで .NET を聞かれると C# 中心の回答を返しがちです。VB.NET WinForms 開発者が日常的にぶつかる：

- 「`var x = ...`」「`record`」「`Span<T>`」「`using var`」など、VB.NET にそのまま持ち込めない C# 固有構文
- WinForms 特有の罠（UI スレッドマーシャリング、`Async Sub` 制約、`Activity.Current` の `Control.Invoke` 越境挙動）
- 産業制御や長期稼働向けの観点（Generic Host 統合、`Meter` シングルトン化、SQL Server 自動計装）

これらに **AI が最初から VB.NET 文脈で答える** ためのコンテキストを 6 つのスキルとして提供します。

## 使い始める

### 必要なもの

- **Claude Code** v2.0+（[インストール手順](https://docs.claude.com/claude-code)）
- **.NET SDK** 6+ 推奨（一部スキルは .NET 7+ / Framework 4.8 互換にも言及）
- VB.NET / WinForms プロジェクト

### インストール

Claude Code 起動後、以下を順に実行：

```text
/plugin marketplace add oreguchi/vb-winforms-skills
/plugin install vb-winforms-skills@vb-winforms-skills-marketplace
/reload-plugins
```

### 30 秒で試す

Claude Code に普通に話しかけるだけで、対応するスキルが自動起動します：

```
> VB.NET WinForms で OpenTelemetry の計装を入れたい。
  ActivitySource と Counter で所要時間と件数をメトリクス化したい。

● Skill(opentelemetry-net-instrumentation-vb-ja)
  Loaded skill...

  （VB.NET 構文の完全なサンプルコードと、SDK セットアップ、
   UI スレッド境界、長期稼働ライフサイクルの解説が返ってくる）
```

`Skill(...)` のツール呼び出しが見えれば、スキルが正しく発火しています。

## 収録スキル

`invocable: true` のスキルは自然言語プロンプトで自動起動し、`invocable: false` のスキルは AI が必要に応じて内部参照します（明示呼出しも可）。

| スキル | 起動 | こんなときに |
|---|---|---|
| `vb-winforms` | 自動 | 「Windows Forms アプリを作って」「.NET Framework から .NET 8 に移行したい」「データバインディングのパターンは？」 |
| `vb-project-setup` | 自動 | 「VB.NET ソリューション構成のベストプラクティスは？」「`.vbproj` / `Directory.Build.props` / CPM の設定は？」 |
| `opentelemetry-net-instrumentation-vb-ja` | 自動 | 「OpenTelemetry の計装を入れたい」「`ActivitySource` の使い方は？」「Generic Host で SDK セットアップしたい」 |
| `vb-coding-standards` | 内部参照 | コード生成・レビュー時に AI が参照（値オブジェクト、API 設計、`Memory(Of T)` / `ArrayPool(Of T)` 等） |
| `vb-concurrency-patterns` | 内部参照 | 並行処理の実装時に AI が参照（`Async/Await` / `Channel(Of T)` / `Akka.NET` の選択指針） |
| `vb-di-patterns` | 内部参照 | DI 登録の実装時に AI が参照（`IServiceCollection` 拡張メソッド、`Module` + `<Extension>`） |

> **OpenTelemetry スキルの補足**：本体に加え、`references/winforms-supplement.md`（WinForms 補足）が同梱。AI が必要時に自動参照します。SDK セットアップ（Generic Host / Sdk 直接版）、UI スレッド越境時の `Activity.Current` 不可視対策、`ActivitySource` / `Meter` シングルトン化とリーク防止までカバー。

## VB.NET 固有の制約メモ

C# 向けスキルから変換されているため、VB.NET で対応できない / 扱い方が異なる構文があります。各スキル内で「**§3b 翻訳ノート**（VB.NET に直接対応なし）」として代替パターンを明示しています。

主な落とし穴：

| C# 構文 | VB.NET の代替 |
|---|---|
| `record` / `record struct` / `with` 式 | `Class` / `Structure` + `ReadOnly` プロパティ + 手書き `Equals`/`GetHashCode` + `With*` ヘルパー |
| `init`-only setter | `ReadOnly Property` + コンストラクタ初期化 |
| `switch` 式 / property pattern / list pattern | `Select Case` / `If`/`ElseIf` + `TypeOf ... Is ...` |
| `Span<T>` / `ReadOnlySpan<T>` | **BC30668 により VB.NET では使用不可**。`Memory(Of T)` / 配列 + offset/length / `ArrayPool(Of T)` |
| `await foreach` | `IAsyncEnumerator(Of T)` 手動操作 + 4-block Try/Catch（`Finally` 内 `Await` は BC36943 で不可） |
| メソッド内ローカル関数 | VB.NET 未対応（BC30026/30289）。`Private Async Function` にメソッド抽出 |
| `file-scoped namespace` / `top-level statements` | `Namespace ... End Namespace` ブロック / `Module Program` + `Sub Main` |
| `using var x = expr;`（インライン宣言） | `Using x As T = expr` ～ `End Using` ブロック |
| `var activity = Activity.Current` | `Dim activity As Activity = Activity.Current`（VB は大小区別なしのため、変数名が型名と衝突する場合は型注釈必須） |

## トラブルシュート

**Q. インストールしたのにスキルが auto-fire しない**

`/reload-plugins` を実行してください。それでも fired しない場合は、現在のセッションを閉じて新規セッション（fresh session）で起動：

```
/exit
```
→ ターミナルで `claude` を再実行。

**Q. 既に `dotnet-skills`（C# 公式）プラグインを入れている。衝突しない？**

衝突しません。両者は名前空間が異なり、Claude Code の Skill 解決は分離されています。むしろ C# プロジェクトと VB.NET プロジェクトを行き来する人は両方入れる構成が便利です。

**Q. `invocable: false` のスキルはどうやって使う？**

`vb-coding-standards` などは AI が必要時に内部参照します。明示的に呼びたければ `Skill(vb-coding-standards)` のように Skill ツールを直接呼ぶこともできますが、通常はコード生成・レビュー時に AI が自動で読み込むので意識不要です。

**Q. .NET Framework 4.8 のレガシープロジェクトで使える？**

スキルの中核内容は適用できます。OpenTelemetry スキルは `Stopwatch.GetElapsedTime` 等の .NET 7+ API について .NET 6 / Framework 4.8 互換コードを併記しています。`vb-winforms` も従来の Designer ベース UI に対応しています。

**Q. フィードバックや改善要望を送りたい**

[GitHub Issues](https://github.com/oreguchi/vb-winforms-skills/issues) にお願いします。日本語 / 英語どちらでも可。

## 検証状況

- 全 6 スキルとも **Phase 5 fresh-session auto-fire テスト済**（`opentelemetry-net-instrumentation-vb-ja` は 3/3 PASS、ほか 5 スキルは v2.0.0 / v2.1.0 で同様検証済）
- 各スキルのコードブロックは `dotnet build`（VB.NET、.NET 10 SDK）で構文・API 互換性を確認済
- 変換ドリフトは [`drift-report`](#変換ドリフトの透明性) として承認可能性が監査済み

### 変換ドリフトの透明性

本プラグインは元の C# スキルから加算方向のみで変換しており、削除や歪曲はありません。各バージョンで承認された追加・補足はリリースノート（[`RELEASE_NOTES.md`](RELEASE_NOTES.md)）にカテゴリ（A: Fidelity / B: Essential / C: Differentiating）と影響度（minor / moderate / heavy）付きで記録されています。累計 37 件（v2.2.0 時点）すべてが catalog 駆動 + register 監査済みです。

## 変換元・派生元

- [managedcode/dotnet-skills](https://github.com/managedcode/dotnet-skills) — `dotnet-winforms` / `dotnet-project-setup` の原版
- dotnet-skills 公式プラグイン — `modern-csharp-coding-standards` / `csharp-concurrency-patterns` / `dependency-injection-patterns` / `OpenTelemetry-NET-Instrumentation` の原版

変換には [skill-conversion](https://github.com/oreguchi/skill-conversion) プラグインを使用：

- v1.0.0 / v2.0.0 / v2.1.0 の 5 スキル：`skill-conversion` v0.2.0 で変換
- v2.2.0 の `opentelemetry-net-instrumentation-vb-ja`：`skill-conversion` v1.0.1（high-utility profile + Catalog System）で変換

## ライセンス

[MIT License](LICENSE)

## 作者

[oreguchi](https://github.com/oreguchi)

フィードバック・改善 PR 歓迎。
