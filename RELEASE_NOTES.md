# Release Notes

## 2.2.1 — 2026-04-28

**Patch: marketplace 名を `vb-winforms-skills-v2-local` から `vb-winforms-skills-marketplace` に修正**。

### 変更概要

v2.0.0 で誤って `vb-winforms-skills-v2-local`（local development の残骸）に rename されていた marketplace 名を、本来の規則 `vb-winforms-skills-marketplace` に戻しました。public 配信用としては `-local` サフィックスは不適切で、過去の v1.0.0 までの命名と一貫させるための修正です。

### 既存ユーザへの影響（v2.2.0 を `-v2-local` 名でインストール済みの場合）

以下の手順で再インストールしてください：

```
/plugin uninstall vb-winforms-skills@vb-winforms-skills-v2-local
/plugin marketplace remove vb-winforms-skills-v2-local
/plugin marketplace add oreguchi/vb-winforms-skills
/plugin install vb-winforms-skills@vb-winforms-skills-marketplace
/reload-plugins
```

スキル本体・機能・API には変更なし。インストール identifier のみの変更です。

### 影響範囲

- v2.2.0 公開から本 patch までの 1 時間程度に install したユーザのみ移行が必要
- v2.0.0 / v2.1.0 を `-v2-local` 名でインストール済みのユーザ（過去にも該当）も同様の手順で `-marketplace` 名に移行可能

---

## 2.2.0 — 2026-04-28

**Minor: skill-conversion v1.0.1 ドッグフーディング成果としての OpenTelemetry 計装スキル追加**。下位互換あり（既存利用者への破壊的変更なし）。

### 変更概要

`dotnet-skills:OpenTelemetry-NET-Instrumentation` v1.0.0（C#/EN）を skill-conversion v1.0.1（high-utility profile）で VB.NET + 日本語に変換した成果物を新規スキルとして取り込み。Phase 0–5 完走、Phase 5 fresh-session で 3/3 auto-fire PASS、独立 fresh-session レビューで「実用品質に到達」と判定済み。

### 新規スキル

- **`opentelemetry-net-instrumentation-vb-ja`**：VB.NET コードベースで OpenTelemetry 計装（トレース・メトリクス・命名規則・エラー処理・性能・API 設計）を実装するための指針。
  - 中核：ソース 17 コードブロック・全節を VB.NET 構文に完全変換（`Using ... End Using`、`Async Function ... As Task`、`KeyValuePair(Of String, Object)` 等）
  - 補強：`references/winforms-supplement.md`（~330 行）に WinForms 特有の補足
    - **§SDK セットアップ**：Generic Host 経由（`AddOpenTelemetry().WithTracing/WithMetrics`、`AddSqlClientInstrumentation`、`AddHttpClientInstrumentation`）と `Sdk.CreateTracerProviderBuilder` 直接版の両方
    - **§UI スレッドと Activity の境界**：`Async Sub` vs `Async Function ... As Task`、`Control.Invoke` 越境時の `Activity.Current` 不可視と明示キャプチャ対策、`InvalidOperationException`（クロススレッド）罠
    - **§長期稼働時のライフサイクル管理**：`ActivitySource`/`Meter` シングルトン化、`IHostedService`/`Application.ApplicationExit` での flush + Dispose、カーディナリティ累積
  - VB.NET 固有の落とし穴：`Dim activity As Activity = Activity.Current`（変数名と型名衝突回避）、`appHost`/完全修飾化（`Microsoft.Extensions.Hosting.Host` との衝突回避）、`ApplicationConfiguration.Initialize()` 不在の代替（`SetHighDpiMode`+`EnableVisualStyles`+`SetCompatibleTextRenderingDefault`）等を明記
- `invocable: true`（auto-fire 対応）

### ドリフト概要（新規スキル分）

- 承認追加 11 件（minor: 6 / moderate: 3 / heavy: 3）
  - Phase 1: 0 / Phase 2: 4 / Phase 3: 1 / Phase 4: 4 / Post-Phase-5 review: 2
- 全件 catalog 駆動 + register 監査済み、削除/歪曲なし（加算方向のみ）
- ドリフト判定：MODERATE（high-utility profile による Tier 2/3 取り込みが主因）

### 累計

- v2.0.0 までの累計: 19 件
- v2.1.0 で追加: 7 件
- **v2.2.0 で追加: 11 件**
- **合計: 37 件**

### v2.1.0 との互換性

- 既存 5 スキルの API・名前・`invocable` 値はすべて維持
- 追加のみで削除なし → 利用者から見て破壊的変更なし

---

## 2.1.0 — 2026-04-24

**Minor: バリアント比較レビュー（2026-04-24）の指摘を反映、1.0.0 の WinForms 特化章を統合**。下位互換あり（既存利用者への破壊的変更なし）。

### 変更概要

レビュー指摘に基づく改善 3 件 + 1.0.0 からの特化章取り込み。

### 改善点（β: レビュー指摘の解消）

- **`vb-di-patterns/SKILL.md`**: `AddOptionsWithValidateOnStart` 採用理由と「単純環境では従来 `.ValidateOnStart()` チェーンも可」両パターン併記コメントを追加（BC30521 発生条件を明示）
- **`vb-project-setup/references/patterns.md`**: `<Nullable>enable</Nullable>` を削除、代わりに `Option Strict/Infer/Explicit/Compare` を明示。「VB.NET は C# のような null 許容参照型のフロー解析を持たない」根拠を併記
- **`vb-winforms/SKILL.md`**: Workflow Step 3 の Program.vb DI 例を完全版（`<STAThread>` + `Module Program` + `Sub Main` + `ApplicationConfiguration.Initialize()`）に復元

### 追加機能（γ: 1.0.0 WinForms 特化章の取り込み）

- **`vb-winforms/references/patterns.md`**:
  - スレッド処理パターン強化（`OnHandleCreated` での `SynchronizationContext` キャプチャ、`Invoke`/`BeginInvoke`/`Post`/`Send` 4 手段の使い分け表、`ConfigureAwait(False)` の使い分け）
  - 新規節「IProgress(Of T) による進捗報告」（サービス側/フォーム側分離、複合 `ProgressInfo` 例、`Application.DoEvents` 代替）
  - 新規節「DI によるフォーム解決とフォーム間データ受け渡し」（DI 登録全体像、`Initialize` メソッド、`Func(Of TForm)` ファクトリー）
- **`vb-concurrency-patterns/SKILL.md`**:
  - 判断ツリーに「進捗報告」「キャンセル」「非 UI スレッドから UI 更新」3 分岐追加
  - 新規節「WinForms 特化: UI スレッドマーシャリングと進捗報告」（UI スレッドマーシャリング、`IProgress(Of T)`、キャンセルボタン連携、`Async Sub` 制約）
  - クイックリファレンス & エスカレーションパスへの WinForms 固有分岐追加

### ドリフト概要

- 承認追加（v2.1.0 で追加分） 7 件（minor: 1 / moderate: 4 / heavy: 2）
- v2.0.0 までの累計: 19 件 + v2.1.0 追加 7 件 = **26 件**
- ドリフト判定変化なし: **MODERATE**（合意済みの WinForms 特化統合は加味）

### v2.0.0 との互換性

- 既存スキルの API・名前・`invocable` 値はすべて維持
- 追加・拡張のみで削除なし → 利用者から見て破壊的変更なし

---

## 2.0.0 — 2026-04-24

**Breaking change: 全 5 スキルを C# 原版から VB.NET + 日本語に再変換**（1.0.0 との互換性なし）。

### 変更概要

- skill-conversion プラグイン v0.2.0 を用いた Phase 0〜5 フル実行による変換
- 変換ルールテーブル、承認追加レジスタ、3 軸 fixpoint レビュー（2 回連続ゼロ変更）、`dotnet build` による実行可能性検証、blind subagent による経験的評価、Phase 5 ハンドオフガイドを含む
- VB.NET 固有の制約（`Span(Of T)` 不可、`Finally` 内 `Await` 不可、メソッド内ローカル関数不可、など）を正確に反映

### 追加スキル

| スキル | 変換元 |
|---|---|
| `vb-winforms` | `dotnet-winforms` (managedcode-dotnet-skills) |
| `vb-project-setup` | `dotnet-project-setup` (managedcode-dotnet-skills) |
| `vb-coding-standards` | `modern-csharp-coding-standards` (dotnet-skills) |
| `vb-concurrency-patterns` | `csharp-concurrency-patterns` (dotnet-skills) |
| `vb-di-patterns` | `dependency-injection-patterns` (dotnet-skills) |

### 追加項目

- `.claude-plugin/plugin.json` — プラグインメタデータ
- `docs/plans/2026-04-23-vb-winforms-skills-conversion/` — 変換の完全なプランドキュメントとドリフトレポート
- `docs/plans/.../sample-build/` — スキルコード検証用サンプル VB.NET プロジェクト群

### 既知の制約

- `vb-winforms`, `vb-project-setup` は `invocable: true` — Phase 5 auto-fire test が user-pending
- `vb-coding-standards`, `vb-concurrency-patterns`, `vb-di-patterns` は `invocable: false`（ソース保持）— `Skill(<name>)` 明示呼び出しのみ

### ドリフト概要

- 承認追加 **19 件**（minor: 7 / moderate: 9 / heavy: 3）
- 全て `docs/plans/2026-04-23-vb-winforms-skills-conversion/approved-additions.md` に記録
- ドリフト判定: **MODERATE**（VB.NET と C# の言語的ギャップにより heavy な書き換えが 3 件必要だったが、ソースの主題・設計意図は保持）

## 1.0.0

(初版 — 本リリースで置き換え)
