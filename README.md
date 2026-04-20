# vb-winforms-skills

**[English](#english)** | **[日本語](#日本語)**

---

## English

Claude Code skills for building and modernizing **VB.NET Windows Forms** applications on modern .NET (8 / 9).

The five skills cover the full stack from solution layout to runtime concurrency. They are designed to be used together, but each one triggers independently on the topic it owns — so Claude picks up only what is relevant to the current task.

### What's inside

| Skill | Trigger topic |
|---|---|
| `vb-project-setup` | Solution layout, SDK-style `.vbproj`, Central Package Management, `global.json`, `.slnx` / `.slnf` |
| `vb-winforms` | Form / presenter separation (MVP), `.Designer.vb` discipline, data binding, validation, async UI |
| `vb-di-patterns` | `Microsoft.Extensions.DependencyInjection` wiring, extension-method composition, service lifetimes, child-form DI |
| `vb-concurrency-patterns` | Async/await on the UI thread, `CancellationToken`, `IProgress(Of T)`, escalation path to Channels / Akka.NET |
| `vb-coding-standards` | Idiomatic modern VB.NET, value objects as `Structure`, `Result` pattern, `Span(Of T)` / `Memory(Of T)`, anti-patterns |

All skills are authored in English. Japanese walkthroughs of the skill set are provided in the `docs/` directory for readers who prefer to start there:

- [`docs/skill-overview-ja.md`](docs/skill-overview-ja.md) — technical overview (JA)
- [`docs/skill-overview-beginner-ja.md`](docs/skill-overview-beginner-ja.md) — beginner-friendly overview (JA)

### Installing

#### Via marketplace (recommended)

Add the plugin's marketplace to Claude Code, then install:

```bash
/plugin marketplace add <path-or-URL-to-this-directory>
/plugin install vb-winforms-skills@vb-winforms-skills-marketplace
```

#### Manual install

Copy the `skills/` directory into one of the locations Claude Code scans for skills (for example, `~/.claude/skills/` or a project's `.claude/skills/`). Each skill is a directory with a top-level `SKILL.md` and optional `references/`.

### Design principles

- **Layered, non-overlapping skills.** `vb-project-setup` is the foundation; `vb-coding-standards` is the idiom layer; `vb-winforms` and `vb-di-patterns` shape the application architecture; `vb-concurrency-patterns` governs runtime behaviour. A change at one layer should not need to revisit another.
- **VB.NET-first.** Every C#-centric example from the original skill set has been translated or replaced with an idiomatic VB.NET equivalent, including the language features that have **no** VB.NET counterpart (records, `with` expressions, pattern matching, `await foreach`, primary constructors, etc.).
- **`Option Strict On` everywhere.** Samples assume strict compilation; any place where C# would silently coerce is made explicit.
- **WinForms reality.** Concurrency, DI lifetimes, and form communication patterns are all oriented around the single-UI-thread, event-driven model of Windows Forms.

### Skill invocation

Four of the five skills are **auto-invocable** — Claude picks them up when the conversation or codebase matches their trigger description. `vb-coding-standards` is explicit-only (`invocable: false`) because the ruleset is dense and would be distracting if applied on every VB.NET file touch; invoke it with `/vb-coding-standards` or by mentioning it by name when you want a style review.

### Compatibility

- Targets .NET 8 / 9 (and later). Older frameworks (.NET Framework, .NET Core 3.x) are out of scope except where the `vb-winforms/references/migration.md` guide explicitly addresses the migration itself.
- VB.NET language version 16.9 (the final released version). All samples stay within that ceiling.
- Designed for Windows Forms on Windows; WPF / MAUI / Blazor are out of scope.

### License

MIT

---

## 日本語

モダンな .NET (8 / 9) 上で **VB.NET Windows Forms** アプリケーションを構築・近代化するための Claude Code スキル集です。

5 つのスキルが、ソリューション構成からランタイムの並行処理まで、フルスタックをカバーします。併用を前提に設計されていますが、それぞれが自分の担当トピックで独立に発火するため、Claude は今のタスクに関係あるものだけを拾います。

### 収録内容

| スキル | 発火トピック |
|---|---|
| `vb-project-setup` | ソリューション構成、SDK 形式 `.vbproj`、Central Package Management、`global.json`、`.slnx` / `.slnf` |
| `vb-winforms` | Form / Presenter 分離 (MVP)、`.Designer.vb` の取り扱い、データバインディング、検証、非同期 UI |
| `vb-di-patterns` | `Microsoft.Extensions.DependencyInjection` の配線、拡張メソッドによる合成、サービスのライフタイム、子フォーム DI |
| `vb-concurrency-patterns` | UI スレッド上の Async/Await、`CancellationToken`、`IProgress(Of T)`、Channels / Akka.NET へのエスカレーション方針 |
| `vb-coding-standards` | モダンな VB.NET の書き方、`Structure` ベースの値オブジェクト、`Result` パターン、`Span(Of T)` / `Memory(Of T)`、アンチパターン |

スキル本体はすべて英語で書かれています。日本語で全体像を把握したい方のために、`docs/` ディレクトリにガイドを用意してあります:

- [`docs/skill-overview-ja.md`](docs/skill-overview-ja.md) — 技術者向け詳細版（日本語）
- [`docs/skill-overview-beginner-ja.md`](docs/skill-overview-beginner-ja.md) — はじめての方向け入門版（日本語）

### インストール

#### マーケットプレイス経由（推奨）

Claude Code にプラグインのマーケットプレイスを追加してからインストールします:

```bash
/plugin marketplace add <このディレクトリへのパスまたは URL>
/plugin install vb-winforms-skills@vb-winforms-skills-marketplace
```

#### 手動インストール

`skills/` ディレクトリを、Claude Code がスキルをスキャンする場所（例: `~/.claude/skills/` やプロジェクトの `.claude/skills/`）のいずれかにコピーしてください。各スキルは `SKILL.md` をトップに持つディレクトリで、任意で `references/` が添えられます。

### 設計方針

- **重ならないレイヤー構造。** `vb-project-setup` が土台、`vb-coding-standards` がイディオム層、`vb-winforms` と `vb-di-patterns` がアプリケーション構造、`vb-concurrency-patterns` がランタイム挙動を司ります。あるレイヤーの変更が他のレイヤーに波及しないように切り分けてあります。
- **VB.NET ファースト。** 元のスキル群にあった C# 中心の例は、すべて VB.NET 慣用の等価物へ書き換えるか置換してあります。VB.NET に対応物が **存在しない** 機能（records、`with` 式、パターンマッチング、`await foreach`、プライマリコンストラクタ等）も含めて対応表を用意しています。
- **`Option Strict On` を前提。** すべてのサンプルは厳格コンパイル前提。C# なら暗黙変換で済む箇所も、明示変換として書いてあります。
- **WinForms の現実に合わせる。** 並行処理、DI ライフタイム、フォーム間通信のパターンは、Windows Forms の「UI スレッドは 1 本 / イベント駆動」というモデルを前提に作られています。

### スキルの呼び出し

5 つのうち 4 つは **自動発火** します — Claude が会話やコードベースとトリガー記述を突き合わせて拾います。`vb-coding-standards` だけは明示呼び出し専用 (`invocable: false`) です。規約の密度が高いため、VB.NET ファイルに触るたびに発火すると邪魔になるからです。スタイルレビューが欲しいときは `/vb-coding-standards` で呼び出すか、スキル名を明示的に会話で言及してください。

### 対応範囲

- 対象は .NET 8 / 9 以降。古いフレームワーク（.NET Framework、.NET Core 3.x）は対象外ですが、`vb-winforms/references/migration.md` で移行手順そのものだけは扱っています。
- VB.NET 言語バージョン 16.9（最後のリリース版）。すべてのサンプルはその範囲内に収めています。
- Windows 上の Windows Forms 向けに設計されています。WPF / MAUI / Blazor は対象外です。

### ライセンス

MIT
