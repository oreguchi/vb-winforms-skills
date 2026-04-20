# VB.NET スキル 5 点セット 完全ガイド

> 本ドキュメントは `vb-winforms-skills` プラグインに含まれる 5 つの VB.NET Windows Forms 向け Claude Code スキルの内容を、LOB / 業務アプリ開発の文脈で理解できる形にまとめたものです。

---

## 目次

1. [全体像 — なぜ 5 つに分けたか](#全体像)
2. [スキル相関図と使い分け](#相関図)
3. [各スキル詳細解説](#各スキル詳細解説)
    - [vb-project-setup — 土台づくり](#1-vb-project-setup--土台づくり)
    - [vb-winforms — UI 構造の整理](#2-vb-winforms--ui-構造の整理)
    - [vb-di-patterns — 依存性の組み立て方](#3-vb-di-patterns--依存性の組み立て方)
    - [vb-concurrency-patterns — 固まらない UI](#4-vb-concurrency-patterns--固まらない-ui)
    - [vb-coding-standards — モダンな書き方](#5-vb-coding-standards--モダンな書き方)
4. [業務での活かし方](#業務での活かし方)
5. [実用 Tips](#実用-tips)

---

<a id="全体像"></a>
## 1. 全体像 — なぜ 5 つに分けたか

VB.NET Windows Forms を使う業務アプリ開発に必要な知識を、**重ならないレイヤー** として 5 つのテーマへ分割しました。

```
┌─────────────────────────────────────────────────────────┐
│  レベル 0：環境                                         │
│  ─ vb-project-setup（ソリューション構成、SDK、CPM）     │
├─────────────────────────────────────────────────────────┤
│  レベル 1：言語                                         │
│  ─ vb-coding-standards（Option Strict、C# 機能の代替）  │
├─────────────────────────────────────────────────────────┤
│  レベル 2：設計                                         │
│  ─ vb-winforms（MVP、データバインディング、検証）      │
│  ─ vb-di-patterns（DI コンテナ、ライフタイム設計）      │
├─────────────────────────────────────────────────────────┤
│  レベル 3：動き方                                       │
│  ─ vb-concurrency-patterns（Async、キャンセル、並行）  │
└─────────────────────────────────────────────────────────┘
```

下から順に「**どこで動かす（環境）→ どう書く（言語）→ どう組む（設計）→ どう動かす（実行時）**」と重なっていくイメージです。

---

<a id="相関図"></a>
## 2. スキル相関図と使い分け

### 依存関係

```
vb-project-setup  ─┐
                   ├─▶  全スキルの前提（ソリューションの土台）
vb-coding-standards ─┤
                   │
vb-winforms  ──────┤
                   ├─▶  UI 層の設計
vb-di-patterns ────┘
                   
vb-concurrency-patterns ─▶  UI 層と、その下の I/O・計算処理
```

### どの場面でどれを呼ぶか

| 場面 | 呼ばれるスキル |
|---|---|
| 新規プロジェクトを作りたい | `vb-project-setup` |
| `.sln` を `.slnx` にしたい、CPM を導入したい | `vb-project-setup` |
| フォームが巨大になりテスト不可能 | `vb-winforms` |
| `.Designer.vb` を手で編集してしまった | `vb-winforms` |
| `Program.vb` で DI を始めたい | `vb-di-patterns` |
| 子フォームを `New` で開いたら注入が効かない | `vb-di-patterns` |
| 「処理中 UI が固まる」 | `vb-concurrency-patterns` |
| キャンセル・進捗付きで I/O を回したい | `vb-concurrency-patterns` |
| `C#` のレコードや `init` を VB でどう書く？ | `vb-coding-standards` |
| コードレビュー基準を明文化したい | `vb-coding-standards` |

### 自動発火 vs 明示呼び出し

| スキル | 発火条件 |
|---|---|
| vb-project-setup | 自動（「VB.NET ソリューション」「プロジェクト構成」等で） |
| vb-winforms | 自動（「Windows Forms」「フォーム」等で） |
| vb-di-patterns | 自動（「依存性注入」「MEDI」「ServiceCollection」等で） |
| vb-concurrency-patterns | 自動（「非同期」「固まる」「Async/Await」等で） |
| **vb-coding-standards** | **明示呼び出しのみ**（`/vb-coding-standards` または自然文で指定） |

`vb-coding-standards` だけ `invocable: false` にしてあります。理由は、「書き方の基準」は横断的で常時適用したくないから。必要なときだけ参照する構成です。

---

<a id="各スキル詳細解説"></a>
## 3. 各スキル詳細解説

---

### 1. vb-project-setup — 土台づくり

#### 目的

新規 VB.NET ソリューションを **モダンな構成** で作るための手順書。.NET Framework 時代と異なり、`.csproj` 相当の SDK スタイル `.vbproj` を中心にした構成を徹底します。

#### 主な教え

##### (A) ディレクトリ構成

```
MyProduct/
├── .config/dotnet-tools.json    ← dotnet CLI ツールのローカル管理
├── src/
│   ├── MyProduct.Core/          ← UI 非依存のドメインロジック
│   └── MyProduct.App/           ← Windows Forms 本体
├── tests/
│   └── MyProduct.Core.Tests/    ← xUnit
├── Directory.Build.props         ← 全プロジェクト共通のビルド設定
├── Directory.Packages.props      ← NuGet バージョン一括管理（CPM）
├── global.json                   ← .NET SDK バージョン固定
├── .editorconfig
├── .gitignore
└── MyProduct.sln
```

**ポイント**: `MyProduct.Core` を `net9.0`（`-windows` 無し）にすることで、**UI から切り離したテスト可能な層** を作れる。これが MVP や DI を成立させる前提。

##### (B) Central Package Management（CPM）

- 従来: 各 `.vbproj` に `<PackageReference Include="xunit" Version="2.9.2" />` と **バージョンを個別記述**
- CPM: `Directory.Packages.props` に **一箇所で全バージョンを管理**、各プロジェクトはバージョン無しで `Include` だけ書く

```xml
<!-- Directory.Packages.props -->
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <PackageVersion Include="xunit" Version="2.9.2" />
    <PackageVersion Include="Microsoft.Extensions.DependencyInjection" Version="9.0.0" />
  </ItemGroup>
</Project>
```

「**バージョン不整合でビルドは通るが挙動が違う**」という厄介な問題を根絶できます。

##### (C) VB 固有の MSBuild プロパティ

C# の `<Nullable>enable</Nullable>` や `<ImplicitUsings>enable</ImplicitUsings>` は **VB では使えません**。代わりに下記:

```xml
<PropertyGroup>
  <OptionStrict>On</OptionStrict>      ← 暗黙型変換禁止（最重要）
  <OptionExplicit>On</OptionExplicit>  ← 変数宣言必須
  <OptionInfer>On</OptionInfer>        ← Dim x = ... の型推論
  <OptionCompare>Binary</OptionCompare>← 文字列比較を厳密に
</PropertyGroup>
```

**`OptionStrict` は絶対 On にしてください**。LOB で一番怖い「`Object` 型に何でも入って実行時まで型エラーに気付かない」を防げます。

##### (D) `global.json` の rollForward

```json
{
  "sdk": {
    "version": "9.0.100",
    "rollForward": "latestFeature"
  }
}
```

| rollForward | 動作 |
|---|---|
| `patch` / `feature` / `minor` / `major` | 指定のバージョンを **正確に要求**、無ければ上位にロールフォワード |
| `latestPatch` / `latestFeature` / `latestMinor` / `latestMajor` | インストール済みの **最新** を採用 |
| `disable` | ロールフォワード禁止、完全一致のみ |

現場では `latestFeature` が実用的。パッチ更新（9.0.100 → 9.0.103）は自動で乗る、機能更新（9.1.x）も乗るが、メジャー変更（10.x）はブロック。

##### (E) `.slnx` と `.slnf`

- `.sln` — 従来の複雑なテキスト形式
- `.slnx` — **新しい XML 形式**（VS 17.10 / SDK 9.0.200+ で使用可能）、差分が読みやすい
- `.slnf` — **ソリューションフィルタ**、大規模ソリューションで一部プロジェクトだけロードする

プロジェクト数が多いソリューションでは `.slnf` で特定プロジェクト群だけ開くと起動が爆速になります。

##### (F) `dotnet new` の VB 対応

| テンプレート | VB 対応 |
|---|:---:|
| `console` | ✅ |
| `classlib` | ✅ |
| `winforms` | ✅ |
| `wpf` | ✅ |
| `xunit` / `mstest` / `nunit` | ✅ |
| `webapi` / `blazor*` / `maui*` / `worker` / `aspire*` | ❌ C#/F# のみ |

**注意**: VB では `--use-program-main` オプションは使えません（Top-level statements 自体が VB に無い）。

---

### 2. vb-winforms — UI 構造の整理

#### 目的

WinForms のフォームコードが肥大化する典型パターンを **MVP（Model-View-Presenter）** で分解し、テスト可能・保守可能にする。

#### 主な教え

##### (A) MVP パターン

**これが WinForms スキルの核心です。**

```
  [Form]              [Presenter]           [Service]
     │                    │                     │
     │ RaiseEvent SaveClicked                    │
     ├────────────────────▶                     │
     │                    │ Await SaveAsync()    │
     │                    ├────────────────────▶│
     │                    │                     │
     │ ShowError(msg)     │                     │
     ◀────────────────────┤                     │
     │                    │                     │
```

- **View（Form）**: ボタンクリック等の UI イベントを `RaiseEvent` して Presenter に流す、それだけ
- **Presenter**: ビジネスロジック、UI には `ICustomerView` インターフェース経由で指示
- **Service**: DB / API アクセス

この分離により:
- **Presenter を単体テスト可能**（`ICustomerView` をモックするだけ、Form 不要）
- Form はただの「入力/出力のルーター」
- Service を差し替えればオフラインテストも可能

##### (B) `.Designer.vb` を触らない

- Designer は **VS の GUI で編集する前提**
- 手で書き換えると次回 GUI で開いたとき**書き戻される**
- イベントハンドラの追加も GUI から（または `AddHandler` をコード側で）
- 「Designer.vb に一行だけ変更を…」はアンチパターン

##### (C) データバインディング

手動でコントロール値を詰める代わりに `BindingSource` を使う:

```vb
' INotifyPropertyChanged 実装済みのモデル
Public Class Customer
    Implements INotifyPropertyChanged
    ' Name, Email などのプロパティは SetField ヘルパ経由で通知
End Class

' フォーム側
Private _bindingSource As New BindingSource()

Private Sub Form_Load(...) Handles MyBase.Load
    _bindingSource.DataSource = New Customer()
    txtName.DataBindings.Add("Text", _bindingSource, NameOf(Customer.Name))
End Sub
```

片方向バインディング（モデル → UI）だけでなく、双方向（UI → モデル）も自動。大量のコントロール同期コードが消えます。

##### (D) `ErrorProvider` による検証

```vb
Private _errorProvider As New ErrorProvider(Me)

Private Sub txtName_Validating(...) Handles txtName.Validating
    If String.IsNullOrWhiteSpace(txtName.Text) Then
        _errorProvider.SetError(txtName, "氏名は必須です")
        e.Cancel = True
    Else
        _errorProvider.SetError(txtName, "")
    End If
End Sub

Private Sub btnSave_Click(...) Handles btnSave.Click
    If Not Me.ValidateChildren() Then Return  ' 全 Validating を一括発火
    ' 保存処理
End Sub
```

`MessageBox` で「エラーです」を出す代わりに、**対象コントロールの隣に ! アイコンを表示**。LOB の業務画面にぴったり。

##### (E) .NET 8/9 の WinForms 新機能

- `ApplicationConfiguration.Initialize()` — .NET 6+ で `EnableVisualStyles()` 等をまとめてくれる
- `SystemIcons.GetStockIcon(...)` — .NET 7+ で OS 標準アイコンを取得
- `Button.Command` プロパティ — **.NET 9+**、MVVM の `ICommand` バインディングがついに可能に

##### (F) .NET Framework → .NET 8/9 移行

`references/migration.md` に以下を詳述:
- `<OutputType>WinExe</OutputType>` と `<UseWindowsForms>true</UseWindowsForms>` の組
- `app.config` → `appsettings.json` への移行
- ClickOnce の継続可否
- `Await Using` は VB に無いので `Try/Finally + Await DisposeAsync()`

---

### 3. vb-di-patterns — 依存性の組み立て方

#### 目的

`Microsoft.Extensions.DependencyInjection`（以下 MEDI）を **WinForms で正しく使う**。Autofac や Ninject とは微妙に違う挙動を踏まえた設計方針を示します。

#### 主な教え

##### (A) `Program.vb` での基本構成

```vb
Imports Microsoft.Extensions.DependencyInjection

Module Program
    <STAThread>
    Sub Main()
        ApplicationConfiguration.Initialize()

        Dim services As New ServiceCollection()
        services.AddCustomerServices()  ' 後述の拡張メソッド
        services.AddForms()

        Using sp = services.BuildServiceProvider(validateScopes:=True)
            Application.Run(sp.GetRequiredService(Of MainForm)())
        End Using
    End Sub
End Module
```

**`validateScopes:=True` は必ず付ける**。これがないと、起動時は気付かず、ランタイムでおかしな挙動に遭遇してから原因究明という地獄に入ります。

##### (B) 拡張メソッドで登録を分割

```vb
Public Module CustomerServiceCollectionExtensions
    <Extension()>
    Public Function AddCustomerServices(services As IServiceCollection) _
        As IServiceCollection
        services.AddScoped(Of ICustomerRepository, CustomerRepository)()
        services.AddScoped(Of ICustomerService, CustomerService)()
        Return services
    End Function
End Module
```

機能ごとにファイル分割することで `Program.vb` が爆発するのを防ぐ。`Return services` があるのでメソッドチェーンできます。

##### (C) `Func(Of T)` は **明示登録が必要**

**ここが一番ハマりやすい**。Autofac では自動提供される `Func(Of T)` が MEDI には無いので、自分で登録:

```vb
' 登録
services.AddTransient(Of CustomerEditForm)()
services.AddSingleton(Of Func(Of CustomerEditForm))(
    Function(sp) Function() sp.GetRequiredService(Of CustomerEditForm)())

' 利用側（メインフォームのコンストラクタで注入）
Public Sub New(editFormFactory As Func(Of CustomerEditForm))
    _editFormFactory = editFormFactory
End Sub

' ボタンクリックで新しいフォームを毎回生成
Private Sub btnEdit_Click(...) Handles btnEdit.Click
    Using dlg = _editFormFactory()
        dlg.ShowDialog(Me)
    End Using
End Sub
```

子フォームを `New CustomerEditForm()` で直接開くと **DI をバイパス** します。必ずファクトリ経由で。

##### (D) ライフタイム選び

| 種類 | いつ使う | 例 |
|---|---|---|
| **Singleton** | アプリ起動から終了まで 1 つ、状態なし | `ILogger(Of T)`、設定、HTTP クライアントファクトリ |
| **Scoped** | 「作業単位」ごとに 1 つ | `DbContext`、リポジトリ |
| **Transient** | 呼ばれるたびに新規、軽量 | バリデータ、ダイアログ用フォーム |

##### (E) 長命フォーム × Scoped サービスの罠

`MainForm` はアプリ終了まで生き続ける。ここに `DbContext` を直接注入すると、同じ `DbContext` が何時間も生き、**追跡エンティティがメモリに溜まって破綻** します。

解決策は `IServiceScopeFactory`:

```vb
Public Sub New(scopeFactory As IServiceScopeFactory)
    _scopeFactory = scopeFactory
End Sub

Private Async Sub btnSave_Click(...) Handles btnSave.Click
    Using scope = _scopeFactory.CreateScope()
        Dim service = scope.ServiceProvider.GetRequiredService(Of ICustomerService)()
        Await service.SaveAsync(...)
    End Using  ' ここで DbContext も Dispose
End Sub
```

**操作ごとに新しいスコープを切る**。これが WinForms × DI の正解。

---

### 4. vb-concurrency-patterns — 固まらない UI

#### 目的

「処理中に UI が固まる」を根絶し、キャンセル・進捗表示・並行処理を **安全に** 書けるようにする。

#### 主な教え

##### (A) 基本の三点セット

```vb
' サービス側
Public Async Function ImportAsync(
    files As IReadOnlyList(Of String),
    progress As IProgress(Of ImportProgress),
    ct As CancellationToken) As Task
    
    For i = 0 To files.Count - 1
        ct.ThrowIfCancellationRequested()
        Await ProcessOneAsync(files(i), ct).ConfigureAwait(False)
        progress?.Report(New ImportProgress With {.Percent = ..., .Message = ...})
    Next
End Function

' フォーム側
Private Async Sub btnStart_Click(...) Handles btnStart.Click
    _cts = New CancellationTokenSource()
    Dim reporter = New Progress(Of ImportProgress)(
        Sub(p) 
            progressBar1.Value = p.Percent
            lblStatus.Text = p.Message
        End Sub)
    
    Try
        Await _service.ImportAsync(files, reporter, _cts.Token)
    Catch ex As OperationCanceledException
        lblStatus.Text = "キャンセルされました"
    End Try
End Sub
```

**重要**:
- `Progress(Of T)` は **UI スレッドで `New` する**。作った瞬間の `SynchronizationContext` を掴むので、コールバックが自動で UI スレッドに戻る
- `ConfigureAwait(False)` はライブラリ側（Service）だけ。UI 側は **絶対に付けない**（戻ってこなくなる）

##### (B) `Async Sub` の厳格な使いどころ

- ✅ OK: **イベントハンドラ**（`btnStart_Click` 等）
- ❌ NG: 通常メソッド（例外で **アプリが落ちる**）

通常メソッドは `Async Function ... As Task` にする。戻り値が要らなくても `Task` を返すべき。

##### (C) 絶対にやってはいけないこと

| 避けるべき | 理由 | 代替 |
|---|---|---|
| `Application.DoEvents()` | 再入で壊れる | `Async/Await` |
| `.Result` / `.Wait()` を UI で | **デッドロック** | `Await` |
| `BackgroundWorker` | レガシー、捨てるべき | `Async/Await` + `IProgress` |
| `Thread.Sleep` | スレッド専有 | `Await Task.Delay(...)` |
| `Task.Run(Function() SomeAsync(...))` | `Task(Of Task)` で例外観測不可 | 直接 `Await SomeAsync(...)` |

##### (D) 高度なパターン（reference/advanced-concurrency.md）

現場で出てくる場面別:

- **複数の I/O を並行**: `Task.WhenAll` / `Task.WhenAny`
- **スレッドセーフな状態更新**: `Interlocked` / `lock`（VB では `SyncLock`）/ `SemaphoreSlim`
- **ストリーミング処理**: `IAsyncEnumerable(Of T)` + `Await For Each` （※ VB では `For Each ... In await ... ToListAsync()` などで代替）
- **Producer / Consumer**: `System.Threading.Channels`
- **リアクティブ**: Rx.NET（`Observable.FromAsync` + `SelectMany` のパターン）
- **アクターモデル**: Akka.NET（複雑な状態機械に）

##### (E) CPU バウンド処理だけ `Task.Run`

画像処理や数値計算のように **I/O 待ちが無い重い処理** は:

```vb
Public Async Function ProcessImageAsync(img As Bitmap) As Task(Of Bitmap)
    Return Await Task.Run(Function() DoHeavyImageProcessing(img))
End Function
```

I/O を待つ処理（DB, ファイル, HTTP）は **すでに非同期** なので `Task.Run` で包む必要なし。逆に包むと余計なスレッド消費になります。

---

### 5. vb-coding-standards — モダンな書き方

#### 目的

C# 側で確立した「モダン .NET コーディング規約」を **VB.NET で実現可能な形に翻訳** したもの。C# の新機能が次々登場する中、VB で何ができて何ができないかを明確化。

**唯一 `invocable: false` のスキル**。必要なときだけ `/vb-coding-standards` で明示呼び出し。

#### 主な教え

##### (A) C# 機能 ↔ VB 代替早見表

| C# 機能 | VB での代替 |
|---|---|
| `record` | `Class` + `Implements IEquatable(Of T)` を手動 |
| `init` セッター | 読み取り専用 `ReadOnly` プロパティ + コンストラクタ |
| `with { ... }` 式 | `Dim copy = original.Clone() With { .X = 1 }` （`With` 代入） |
| パターンマッチング `switch` 式 | `Select Case` / `If...ElseIf...` / `TypeOf ... Is` |
| `required` プロパティ | コンストラクタで強制 |
| `ref` / `ref readonly` パラメータ | **VB も `ByRef` あり**（`ref` 相当は可能、`ref readonly` は不可） |
| ref 戻り値 | **不可** |
| `await foreach` | **不可**（`await Enumerable.ToListAsync()` で代替） |
| `await using` | **不可**（`Try/Finally + Await DisposeAsync()`） |
| `yield return` | **不可**（コレクションを返す） |
| top-level statements | **不可**（`Module Program` + `Sub Main` 必須） |
| file-scoped namespace | **不可**（ブロックスタイルのみ） |
| 生文字列リテラル `"""..."""` | **不可**（`&` 結合か `XML` リテラル） |
| コレクション式 `[1, 2, 3]` | **不可**（`New List(Of Integer) From {1, 2, 3}`） |
| primary constructor | **不可** |
| `stackalloc` | **不可**（消費者側のみ利用可能） |
| Nullable Reference Types | **非対応** |
| `<ImplicitUsings>` | **C# 専用**（VB は `<ItemGroup><Import Include="..."/></ItemGroup>`） |

##### (B) `ValueTask(Of T)` の VB 書き方

C# の `ValueTask.FromResult(value)` は VB で型推論エラーになりやすい:

```vb
' ❌ Option Strict On で怒られる
Return ValueTask.FromResult(cached)

' ✅
Return New ValueTask(Of Order)(cached)
```

##### (C) 値オブジェクトは `Structure`

```vb
Public Structure Money
    Implements IEquatable(Of Money)

    Public ReadOnly Property Amount As Decimal
    Public ReadOnly Property Currency As String

    Public Sub New(amount As Decimal, currency As String)
        Me.Amount = amount
        Me.Currency = currency
    End Sub

    Public Overloads Function Equals(other As Money) As Boolean _
        Implements IEquatable(Of Money).Equals
        Return Amount = other.Amount AndAlso Currency = other.Currency
    End Function
End Structure
```

C# の `record struct` と同等の効果を、手動実装で得る。面倒だが型安全性は同等。

##### (D) `Result(Of T, TError)` パターン

例外を乱用する代わりに、エラーを型で表現:

```vb
Public NotInheritable Class Result(Of T, TError)
    Public ReadOnly Property IsSuccess As Boolean
    Public ReadOnly Property Value As T
    Public ReadOnly Property [Error] As TError

    Private Sub New(isSuccess As Boolean, value As T, err As TError)
        Me.IsSuccess = isSuccess
        Me.Value = value
        Me.[Error] = err
    End Sub

    Public Shared Function Success(value As T) As Result(Of T, TError)
        Return New Result(Of T, TError)(True, value, Nothing)
    End Function

    Public Shared Function Failure(err As TError) As Result(Of T, TError)
        Return New Result(Of T, TError)(False, Nothing, err)
    End Function
End Class
```

業務ロジックのエラー（バリデーション違反、在庫不足、権限なし）を例外で投げると、呼び出し側がすべて Try/Catch で受けることになります。Result 型にすれば **コンパイラが処理漏れを指摘** できる。

##### (E) Span/Memory/ArrayPool（消費者側）

- VB は `stackalloc` や `ref struct` を **作れない** が、**使うことはできる**
- `ReadOnlySpan(Of Char)` を受け取る API（`Integer.Parse`, `String.Split` 等）は VB から普通に呼べる
- `ArrayPool(Of T).Shared.Rent` で大きなバッファを借りる、ベンチマーク上有効な場面のみ使う

##### (F) Reference ファイル群

`vb-coding-standards` には 4 つの reference ファイル:

1. `value-objects-and-patterns.md` — 値オブジェクト、Widening/Narrowing、分岐パターン
2. `performance-and-api-design.md` — Span/Memory/ArrayPool、API 設計
3. `composition-and-error-handling.md` — 継承より合成、Result 型
4. `anti-patterns-and-reflection.md` — 反射回避、よくあるアンチパターン

---

<a id="業務での活かし方"></a>
## 4. 業務での活かし方

### LOB / 業務アプリでよくある場面

| 現場の痛み | どのスキルが効くか |
|---|---|
| 外部装置や外部サービスとの通信中に画面が固まる | **vb-concurrency-patterns** |
| 長時間処理をキャンセル可能にしたい | **vb-concurrency-patterns** |
| フォームが 3000 行を超えた | **vb-winforms**（MVP で分解） |
| プロジェクト間でライブラリバージョンが食い違う | **vb-project-setup**（CPM） |
| 新機能テストのため DB をモック化したい | **vb-di-patterns**（`ICustomerRepository` を差し替え） |
| C# の新機能記事を読んで VB でどう書けば？ | **vb-coding-standards** |
| .NET Framework 4.8 から .NET 8 に移行 | **vb-project-setup** + **vb-winforms/migration** |
| 外部連携部分のエラー処理がぐちゃぐちゃ | **vb-coding-standards**（Result 型） |

### 導入順序の提案

新規プロジェクトなら:
1. **vb-project-setup** でソリューション土台
2. **vb-di-patterns** で DI コンテナ設置
3. **vb-winforms** で MVP を適用
4. **vb-concurrency-patterns** を必要な箇所に
5. **vb-coding-standards** はコードレビューのとき随時参照

既存プロジェクトの改善なら:
1. まず **vb-concurrency-patterns** で「固まる」を退治（体感効果が大きい）
2. 次に **vb-winforms** で巨大フォームから MVP 抽出
3. **vb-di-patterns** で依存関係を整理
4. 最後に **vb-project-setup** でソリューション構造を現代化

---

<a id="実用-tips"></a>
## 5. 実用 Tips

### トリガー言葉の例

自動発火させるには「自然な日本語の依頼文」で OK。無理に英単語を入れる必要はありません。

- ✅ 「Windows Forms で MVP パターンを使いたい」→ `vb-winforms`
- ✅ 「VB.NET で DI を導入したい」→ `vb-di-patterns`
- ✅ 「非同期処理で UI が固まらないようにしたい」→ `vb-concurrency-patterns`
- ✅ 「VB.NET プロジェクトをモダンに作りたい」→ `vb-project-setup`

### 複数スキルの同時発火

WinForms + DI のようにテーマが重なる質問をすると **複数のスキルが同時に発火** します。これは正常動作。互いに補完しあって回答します。

### reference ファイルの読み込みタイミング

Claude は SKILL.md を最初に読み、**必要に応じて** reference ファイルを読みます。全部読まないことで高速応答しつつ、深い話題が来たら深掘り。

### スキルの更新

スキル本体は Claude Code のプラグインとしてインストールされます。内容を改変したい場合は、プラグインのソースをローカルに取得して編集し、ローカルマーケットプレイスとしてインストールし直せば反映できます。

---

## おわりに

この 5 スキルは **VB.NET WinForms 開発で必要になる知識の「索引」** です。Claude が自動で呼び出してくれるので、**何を覚えているかを常に意識する必要はありません**。代わりに「こういう場面なら自動で正しい知識が引かれる」という安心感が得られます。

気になる点・追加したい内容・改善案があれば、Issue / PR でお知らせください。
