# Release Notes

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
