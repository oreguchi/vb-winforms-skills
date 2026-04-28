# vb-winforms-skills

VB.NET / .NET 向け Claude Code スキル集。

## 概要

本プラグインは managedcode-dotnet-skills（コミュニティ）と dotnet-skills（公式）の C# 向けスキルを VB.NET + 日本語に変換した 6 つのスキルを提供する。VB.NET プロジェクトで Claude Code を使う際の実用ガイドとして設計されている。

## 収録スキル

| スキル | `invocable` | 用途 |
|---|---|---|
| `vb-winforms` | true | Windows Forms アプリの設計・実装・.NET Framework からの移行 |
| `vb-project-setup` | true | .NET ソリューション構成、`.vbproj`、`Directory.Build.props`、CPM、`global.json` |
| `vb-coding-standards` | false | モダン VB.NET コーディング標準（値オブジェクト、API 設計、Memory/ArrayPool） |
| `vb-concurrency-patterns` | false | 並行処理抽象の選択（Async/Await、Channel、Akka.NET） |
| `vb-di-patterns` | false | `IServiceCollection` 拡張メソッドによる DI 登録整理 |
| `opentelemetry-net-instrumentation-vb-ja` | true | OpenTelemetry 計装（トレース・メトリクス・命名規則・性能・API 設計）+ WinForms 補足（UI スレッド境界、Generic Host 統合、長期稼働ライフサイクル） |

`invocable: true` のスキルは自然言語プロンプトで自動起動する。`invocable: false` のスキルは `Skill(<name>)` で明示的に呼び出す。

## インストール

```text
/plugin marketplace add <path-or-url>
/plugin install vb-winforms-skills
```

## VB.NET 固有の制約メモ

本プラグインは C# 向けスキルから変換されており、VB.NET で対応できない / 扱い方が異なる構文については **§3b 翻訳ノート**（「VB.NET に直接対応なし」）を付して代替パターンを明示している。主なものは以下。

- `record` / `record struct` / `with` 式 → `Class` / `Structure` + `ReadOnly` プロパティ + 手書き `Equals`/`GetHashCode` + `With*` ヘルパー
- `init`-only setter → `ReadOnly Property` + コンストラクタ初期化
- `switch` 式 / property pattern / list pattern → `Select Case` / `If`/`ElseIf` + `TypeOf ... Is ...`
- `Span(Of T)` / `ReadOnlySpan(Of T)` → **BC30668 により VB.NET では使用不可**。`String`/配列 + `Memory(Of T)` / `ArrayPool(Of T)` で代替
- `await foreach` → `IAsyncEnumerator(Of T)` 手動操作 + 4-block Try/Catch パターン（`Finally` 内 `Await` は BC36943 で不可）
- メソッド内ローカル関数 → VB.NET 未対応（BC30026/30289）。`Private Async Function` にメソッド抽出
- `file-scoped namespace` / `top-level statements` → ブロック形式 `Namespace ... End Namespace` / `Module Program` + `Sub Main`

## ライセンス

MIT License — [LICENSE](LICENSE) 参照。

## 作者

[oreguchi](https://github.com/oreguchi)

## 変換元・派生元

- [managedcode/dotnet-skills](https://github.com/managedcode/dotnet-skills) — `dotnet-winforms`, `dotnet-project-setup` の原版
- dotnet-skills 公式プラグイン — `modern-csharp-coding-standards`, `csharp-concurrency-patterns`, `dependency-injection-patterns`, `OpenTelemetry-NET-Instrumentation` の原版

変換には [skill-conversion](https://github.com/oreguchi/skill-conversion) プラグインを使用：

- v1.0.0 / v2.0.0 / v2.1.0 の 5 スキル：`skill-conversion` v0.2.0 で変換
- v2.2.0 の `opentelemetry-net-instrumentation-vb-ja`：`skill-conversion` v1.0.1（high-utility profile + Catalog System）で変換
