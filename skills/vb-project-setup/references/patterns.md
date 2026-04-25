# プロジェクト構造パターン

## ソリューションレイアウト

### 推奨ディレクトリ構造

```text
<repo-root>/
├── .config/
│   └── dotnet-tools.json
├── src/
│   ├── <ProductName>.Core/
│   ├── <ProductName>.Api/
│   └── <ProductName>.Web/
├── tests/
│   ├── <ProductName>.Core.Tests/
│   └── <ProductName>.Api.Tests/
├── samples/
│   └── <ProductName>.Sample/
├── docs/
├── Directory.Build.props
├── Directory.Build.targets
├── Directory.Packages.props
├── global.json
├── nuget.config
├── <SolutionName>.sln
└── README.md
```

### プロジェクト命名規約

| プロジェクト種別 | パターン | 例 |
|--------------|---------|---------|
| コアライブラリ | `<ProductName>.Core` | `Contoso.Orders.Core` |
| ドメイン層 | `<ProductName>.Domain` | `Contoso.Orders.Domain` |
| アプリケーション層 | `<ProductName>.Application` | `Contoso.Orders.Application` |
| インフラストラクチャ | `<ProductName>.Infrastructure` | `Contoso.Orders.Infrastructure` |
| Web API | `<ProductName>.Api` | `Contoso.Orders.Api` |
| Web フロントエンド | `<ProductName>.Web` | `Contoso.Orders.Web` |
| ワーカーサービス | `<ProductName>.Worker` | `Contoso.Orders.Worker` |
| ユニットテスト | `<ProjectName>.Tests` | `Contoso.Orders.Core.Tests` |
| 統合テスト | `<ProjectName>.IntegrationTests` | `Contoso.Orders.Api.IntegrationTests` |

---

## Directory.Build.props

`Directory.Build.props` は、そのディレクトリおよびサブディレクトリ内の全プロジェクトに対して MSBuild が自動的にインポートする。

### 基本テンプレート

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <LangVersion>latest</LangVersion>

    <!-- VB.NET では <Nullable>enable</Nullable> を通常は使わない。
         理由:
           - VB.NET は C# のような null 許容参照型のフロー解析を持たない
             （enable にしても静的チェック効果が限定的）
           - 参照型の null 許容は `Nothing` で暗黙的に表現する
           - 値型は従来どおり `Nullable(Of T)` / `T?` を使う
           - 代わりに Option Strict/Infer/Explicit を徹底するのが VB.NET 流の堅牢化
         必要なら個別プロジェクトで試せるが、デフォルトは非設定を推奨。 -->
    <OptionStrict>On</OptionStrict>
    <OptionInfer>On</OptionInfer>
    <OptionExplicit>On</OptionExplicit>
    <OptionCompare>Binary</OptionCompare>

    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <WarningsAsErrors />
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    <EnableNETAnalyzers>true</EnableNETAnalyzers>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
  </PropertyGroup>

  <!-- ライブラリ向けパッケージメタデータ -->
  <PropertyGroup>
    <Authors>Your Name or Organization</Authors>
    <Company>Your Company</Company>
    <Copyright>Copyright (c) $(Company) $([System.DateTime]::Now.Year)</Copyright>
    <RepositoryUrl>https://github.com/your-org/your-repo</RepositoryUrl>
    <RepositoryType>git</RepositoryType>
  </PropertyGroup>

  <!-- CI 向け決定論的ビルド -->
  <PropertyGroup Condition="'$(CI)' == 'true'">
    <ContinuousIntegrationBuild>true</ContinuousIntegrationBuild>
    <Deterministic>true</Deterministic>
  </PropertyGroup>
</Project>
```

### プロジェクト種別による条件付きプロパティ

```xml
<Project>
  <!-- 共有デフォルト -->
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <!-- VB.NET では <Nullable>enable</Nullable> は使用しない（基本テンプレート参照） -->
    <OptionStrict>On</OptionStrict>
  </PropertyGroup>

  <!-- テストプロジェクトのデフォルト -->
  <PropertyGroup Condition="$(MSBuildProjectName.EndsWith('.Tests'))">
    <IsPackable>false</IsPackable>
    <IsTestProject>true</IsTestProject>
  </PropertyGroup>

  <!-- ライブラリのデフォルト -->
  <PropertyGroup Condition="!$(MSBuildProjectName.EndsWith('.Tests')) AND !$(MSBuildProjectName.EndsWith('.Api')) AND !$(MSBuildProjectName.EndsWith('.Web'))">
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
  </PropertyGroup>
</Project>
```

### ネストした Directory.Build.props

子ディレクトリは親を明示的にインポートして拡張できる。

```xml
<!-- tests/Directory.Build.props -->
<Project>
  <Import Project="$([MSBuild]::GetPathOfFileAbove('Directory.Build.props', '$(MSBuildThisFileDirectory)../'))" />

  <PropertyGroup>
    <IsPackable>false</IsPackable>
    <IsTestProject>true</IsTestProject>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" />
    <PackageReference Include="xunit" />
    <PackageReference Include="xunit.runner.visualstudio" />
    <PackageReference Include="coverlet.collector" />
  </ItemGroup>
</Project>
```

---

## Directory.Build.targets

プロジェクトファイルが完全に評価された後に実行するロジックには `Directory.Build.targets` を使用する。

```xml
<Project>
  <!-- プロジェクト評価後に実行 -->
  <Target Name="PrintBuildInfo" BeforeTargets="Build">
    <Message Importance="High" Text="Building $(MSBuildProjectName) for $(TargetFramework)" />
  </Target>

  <!-- テスト命名規約を強制 -->
  <Target Name="ValidateTestProjectNaming" BeforeTargets="Build" Condition="'$(IsTestProject)' == 'true'">
    <Error Condition="!$(MSBuildProjectName.EndsWith('.Tests')) AND !$(MSBuildProjectName.EndsWith('.IntegrationTests'))"
           Text="Test projects must end with .Tests or .IntegrationTests" />
  </Target>
</Project>
```

---

## Central Package Management（CPM）

Central Package Management はパッケージバージョンを単一の `Directory.Packages.props` ファイルに集約する。

### CPM の有効化

リポジトリルートの `Directory.Packages.props` に追加する。

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    <CentralPackageTransitivePinningEnabled>true</CentralPackageTransitivePinningEnabled>
  </PropertyGroup>

  <ItemGroup>
    <!-- ランタイムパッケージ -->
    <PackageVersion Include="Microsoft.Extensions.Hosting" Version="9.0.0" />
    <PackageVersion Include="Microsoft.Extensions.DependencyInjection" Version="9.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Logging" Version="9.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Options" Version="9.0.0" />
    <PackageVersion Include="System.Text.Json" Version="9.0.0" />

    <!-- Entity Framework Core -->
    <PackageVersion Include="Microsoft.EntityFrameworkCore" Version="9.0.0" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore.SqlServer" Version="9.0.0" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore.Design" Version="9.0.0" />

    <!-- ASP.NET Core 追加パッケージ -->
    <PackageVersion Include="Swashbuckle.AspNetCore" Version="6.6.2" />

    <!-- テスト -->
    <PackageVersion Include="Microsoft.NET.Test.Sdk" Version="17.11.1" />
    <PackageVersion Include="xunit" Version="2.9.2" />
    <PackageVersion Include="xunit.runner.visualstudio" Version="2.8.2" />
    <PackageVersion Include="Moq" Version="4.20.72" />
    <PackageVersion Include="FluentAssertions" Version="6.12.1" />
    <PackageVersion Include="coverlet.collector" Version="6.0.2" />

    <!-- アナライザー -->
    <PackageVersion Include="StyleCop.Analyzers" Version="1.2.0-beta.556" />
    <PackageVersion Include="Roslynator.Analyzers" Version="4.12.6" />
  </ItemGroup>
</Project>
```

### CPM 有効時のプロジェクトファイル参照

CPM が有効の場合、プロジェクトファイルはバージョンなしでパッケージを参照する。

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.Extensions.Hosting" />
  <PackageReference Include="Microsoft.Extensions.Logging" />
</ItemGroup>
```

### バージョンオーバーライド

プロジェクトが異なるバージョンを必要とする場合は `VersionOverride` を控えめに使用する。

```xml
<ItemGroup>
  <PackageReference Include="Newtonsoft.Json" VersionOverride="13.0.3" />
</ItemGroup>
```

---

## global.json

再現性のあるビルドのために SDK バージョンを固定する。

```json
{
  "sdk": {
    "version": "9.0.100",
    "rollForward": "latestFeature",
    "allowPrerelease": false
  }
}
```

### ロールフォワードポリシー

| 値 | 動作 |
|-------|----------|
| `patch` | 指定または最高インストール済みパッチを使用 |
| `feature` | 指定または最高インストール済みフィーチャーバンドを使用 |
| `minor` | 指定または最高インストール済みマイナーを使用 |
| `major` | 最高インストール済み SDK を使用 |
| `latestPatch` | 最高インストール済みパッチを使用 |
| `latestFeature` | 最高インストール済みフィーチャーバンドを使用 |
| `latestMinor` | 最高インストール済みマイナーを使用 |
| `latestMajor` | 最高インストール済み SDK を使用 |
| `disable` | 完全一致のみ |

---

## nuget.config

パッケージソースと認証情報を設定する。

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" protocolVersion="3" />
    <!-- プライベートフィード -->
    <add key="github" value="https://nuget.pkg.github.com/your-org/index.json" />
  </packageSources>

  <packageSourceMapping>
    <packageSource key="nuget.org">
      <package pattern="*" />
    </packageSource>
    <packageSource key="github">
      <package pattern="YourOrg.*" />
    </packageSource>
  </packageSourceMapping>

  <!-- 環境変数経由の CI 認証情報 -->
  <packageSourceCredentials>
    <github>
      <add key="Username" value="%NUGET_USERNAME%" />
      <add key="ClearTextPassword" value="%NUGET_TOKEN%" />
    </github>
  </packageSourceCredentials>
</configuration>
```

---

## アナライザーとコードスタイル

### .editorconfig の基本

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 4
insert_final_newline = true
trim_trailing_whitespace = true

[*.vb]
dotnet_sort_system_directives_first = true
dotnet_separate_import_directive_groups = false

# IDE 診断
dotnet_diagnostic.IDE0005.severity = warning
dotnet_diagnostic.IDE0055.severity = warning

[*.{json,yml,yaml}]
indent_size = 2
```

### Directory.Build.props でのアナライザーパッケージ

```xml
<ItemGroup>
  <PackageReference Include="StyleCop.Analyzers" PrivateAssets="all" />
  <PackageReference Include="Roslynator.Analyzers" PrivateAssets="all" />
  <PackageReference Include="Microsoft.CodeAnalysis.NetAnalyzers" PrivateAssets="all" />
</ItemGroup>
```

---

## マルチターゲティング

### 単一プロジェクトによるマルチターゲティング

```xml
<PropertyGroup>
  <TargetFrameworks>net9.0;net8.0;netstandard2.0</TargetFrameworks>
</PropertyGroup>

<ItemGroup Condition="'$(TargetFramework)' == 'netstandard2.0'">
  <PackageReference Include="System.Text.Json" />
</ItemGroup>
```

### 条件付きコンパイル

```vbnet
#If NET9_0_OR_GREATER Then
    ' .NET 9+ 固有のコード
    ArgumentNullException.ThrowIfNull(value)
#Else
    ' 古いフレームワーク向けのフォールバック
    If value Is Nothing Then
        Throw New ArgumentNullException(NameOf(value))
    End If
#End If
```

---

## SourceLink と決定論的ビルド

デバッガー統合のために SourceLink を有効にする。

```xml
<PropertyGroup>
  <PublishRepositoryUrl>true</PublishRepositoryUrl>
  <EmbedUntrackedSources>true</EmbedUntrackedSources>
  <IncludeSymbols>true</IncludeSymbols>
  <SymbolPackageFormat>snupkg</SymbolPackageFormat>
</PropertyGroup>

<ItemGroup>
  <PackageReference Include="Microsoft.SourceLink.GitHub" PrivateAssets="all" />
</ItemGroup>
```

---

## ソリューションフィルター

部分的なソリューション読み込みのために `.slnf` ファイルを作成する。

```json
{
  "solution": {
    "path": "MyProduct.sln",
    "projects": [
      "src\\MyProduct.Core\\MyProduct.Core.vbproj",
      "src\\MyProduct.Api\\MyProduct.Api.vbproj",
      "tests\\MyProduct.Core.Tests\\MyProduct.Core.Tests.vbproj"
    ]
  }
}
```

開く方法: `dotnet sln open MyProduct.Api.slnf`
