# WinForms のモダン .NET への移行

## 移行の概要

Windows Forms アプリケーションを .NET Framework からモダン .NET（6、7、8、9、10）へ移行することで、以下のメリットが得られる。
- パフォーマンスとメモリ効率の向上
- モダン VB.NET 言語機能へのアクセス
- システム全体へのランタイムインストール不要なサイドバイサイドデプロイ
- 継続的なサポートとセキュリティ更新
- 新しい WinForms 機能へのアクセス

## 事前調査

### 互換性分析

移行前に、アプリケーションの互換性を分析する。

```bash
# .NET Upgrade Assistant をインストールする
dotnet tool install -g upgrade-assistant

# プロジェクトを分析する
upgrade-assistant analyze MyWinFormsApp.vbproj

# または対話的なアップグレードを実行する
upgrade-assistant upgrade MyWinFormsApp.vbproj
```

### よくあるブロッカー

| ブロッカー | 影響 | 対処法 |
|---------|--------|------------|
| WCF クライアント | 変更が必要 | CoreWCF または gRPC を使用する |
| WCF サーバー | 非サポート | ASP.NET Core + gRPC に移行する |
| AppDomain | サポート限定 | AssemblyLoadContext で再設計する |
| Remoting | 非サポート | gRPC または REST API を使用する |
| Code Access Security | 非サポート | 削除または再設計する |
| Windows Workflow Foundation | 非サポート | Elsa 等のワークフローエンジンを使用する |
| Crystal Reports | 動作しない可能性あり | テストするか代替手段を使用する |

### 非推奨 API の確認

```vbnet
' 以下のパターンは潜在的な問題を示す:

' App.config の使用 — 移行が必要な場合がある
ConfigurationManager.AppSettings("MySetting")

' System.Web の参照 — 利用不可
System.Web.HttpUtility.UrlEncode(value)

' Drawing.Common の差異（非 Windows 環境）
System.Drawing.Image.FromFile(path)
```

## プロジェクトファイルの移行

### 移行前（.NET Framework）

```xml
<?xml version="1.0" encoding="utf-8"?>
<Project ToolsVersion="15.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <Import Project="$(MSBuildExtensionsPath)\$(MSBuildToolsVersion)\Microsoft.Common.props" />
  <PropertyGroup>
    <Configuration Condition=" '$(Configuration)' == '' ">Debug</Configuration>
    <Platform Condition=" '$(Platform)' == '' ">AnyCPU</Platform>
    <ProjectGuid>{GUID-HERE}</ProjectGuid>
    <OutputType>WinExe</OutputType>
    <RootNamespace>MyWinFormsApp</RootNamespace>
    <AssemblyName>MyWinFormsApp</AssemblyName>
    <TargetFrameworkVersion>v4.8</TargetFrameworkVersion>
  </PropertyGroup>
  <ItemGroup>
    <Reference Include="System" />
    <Reference Include="System.Core" />
    <Reference Include="System.Data" />
    <Reference Include="System.Drawing" />
    <Reference Include="System.Windows.Forms" />
    <!-- 他にも多数の参照 -->
  </ItemGroup>
  <ItemGroup>
    <Compile Include="Form1.vb">
      <SubType>Form</SubType>
    </Compile>
    <Compile Include="Form1.Designer.vb">
      <DependentUpon>Form1.vb</DependentUpon>
    </Compile>
    <!-- 他にも多数のコンパイル対象 -->
  </ItemGroup>
  <Import Project="$(MSBuildToolsPath)\Microsoft.VisualBasic.targets" />
</Project>
```

### 移行後（モダン .NET SDK 形式）

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net9.0-windows</TargetFramework>
    <UseWindowsForms>true</UseWindowsForms>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <ApplicationManifest>app.manifest</ApplicationManifest>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="9.0.0" />
    <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="9.0.0" />
  </ItemGroup>
</Project>
```

## ステップバイステップの移行手順

### ステップ 1: 新規プロジェクトの作成

```bash
# 新しい WinForms プロジェクトを作成する
dotnet new winforms -n MyWinFormsApp.Modern -f net9.0 --language VB

# または特定のテンプレートオプションを指定する
dotnet new winforms -n MyWinFormsApp.Modern --no-restore --language VB
```

### ステップ 2: ソースファイルのコピー

旧プロジェクトから以下のファイルをコピーする。
- すべての `.vb` ファイル（フォーム、クラス、コントロール）
- すべての `.resx` ファイル（リソース）
- すべての `.Designer.vb` ファイル
- アセット（画像、アイコンなど）

### ステップ 3: Program.vb の更新

```vbnet
' .NET Framework スタイル
Module Program
    <STAThread>
    Sub Main()
        Application.EnableVisualStyles()
        Application.SetCompatibleTextRenderingDefault(False)
        Application.Run(New MainForm())
    End Sub
End Module

' モダン .NET スタイル
Module Program
    <STAThread>
    Sub Main()
        ApplicationConfiguration.Initialize()
        Application.Run(New MainForm())
    End Sub
End Module

' DI 使用のモダン .NET スタイル
Module Program
    <STAThread>
    Sub Main()
        ApplicationConfiguration.Initialize()

        Dim host = Host.CreateDefaultBuilder() _
            .ConfigureServices(Sub(context, services)
                services.AddSingleton(Of MainForm)()
                services.AddTransient(Of ICustomerService, CustomerService)()
            End Sub) _
            .Build()

        Dim mainForm = host.Services.GetRequiredService(Of MainForm)()
        Application.Run(mainForm)
    End Sub
End Module
```

### ステップ 4: 構成の更新

`app.config` を `appsettings.json` に置き換える。

```json
{
  "ConnectionStrings": {
    "Default": "Server=...;Database=...;"
  },
  "AppSettings": {
    "MaxRetries": 3,
    "TimeoutSeconds": 30
  }
}
```

```vbnet
' 構成の読み込み
Public Class AppConfig
    Private ReadOnly _configuration As IConfiguration

    Public Sub New()
        _configuration = New ConfigurationBuilder() _
            .SetBasePath(AppContext.BaseDirectory) _
            .AddJsonFile("appsettings.json", optional:=False) _
            .AddJsonFile($"appsettings.{Environment.GetEnvironmentVariable("DOTNET_ENVIRONMENT")}.json", optional:=True) _
            .Build()
    End Sub

    Public ReadOnly Property ConnectionString As String
        Get
            Return _configuration.GetConnectionString("Default")
        End Get
    End Property

    Public ReadOnly Property MaxRetries As Integer
        Get
            Return _configuration.GetValue(Of Integer)("AppSettings:MaxRetries")
        End Get
    End Property
End Class
```

### ステップ 5: NuGet 参照の更新

packages.config を PackageReference に置き換える。

```xml
<!-- 旧 packages.config 形式 -->
<packages>
  <package id="Newtonsoft.Json" version="13.0.1" targetFramework="net48" />
  <package id="Dapper" version="2.0.123" targetFramework="net48" />
</packages>

<!-- 新 PackageReference 形式（.vbproj 内） -->
<ItemGroup>
  <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
  <PackageReference Include="Dapper" Version="2.1.35" />
</ItemGroup>
```

### ステップ 6: API 差異への対応

```vbnet
' BinaryFormatter — 推奨されなくなったため代替手段を使用する
' 旧
Dim formatter = New BinaryFormatter()
formatter.Serialize(stream, obj)

' 新 — System.Text.Json 等のシリアライザーを使用する
Dim json = JsonSerializer.Serialize(obj)
Await File.WriteAllTextAsync(path, json)

' System.Drawing の差異
' 旧 — すべての環境で動作した
Using bitmap = New Bitmap(path)
    ' ...
End Using

' 新 — デフォルトでは Windows のみ。クロスプラットフォームは SkiaSharp を使用する
' または以下のパッケージを追加する:
' <PackageReference Include="System.Drawing.Common" Version="8.0.0" />
```

### ステップ 7: アセンブリ情報の更新

`AssemblyInfo.vb` を削除し、プロジェクトプロパティを使用する。

```xml
<PropertyGroup>
  <AssemblyVersion>1.0.0.0</AssemblyVersion>
  <FileVersion>1.0.0.0</FileVersion>
  <Version>1.0.0</Version>
  <Company>My Company</Company>
  <Product>My WinForms App</Product>
  <Copyright>Copyright 2024</Copyright>
</PropertyGroup>
```

## よくある移行問題

### デザイナーの問題

```vbnet
' 問題: 移行後にデザイナーが読み込まれない
' 対処法: すべての依存関係が利用可能であることを確認し、再ビルドする

' 問題: ユーザーコントロールがツールボックスに表示されない
' 対処法: ソリューションをビルドし、ツールボックスを更新する

' 問題: リソースが読み込まれない
' 対処法: .resx ファイルのビルドアクションが正しいことを確認する
```

```xml
<!-- リソースが埋め込まれていることを確認する -->
<ItemGroup>
  <EmbeddedResource Update="Form1.resx">
    <DependentUpon>Form1.vb</DependentUpon>
  </EmbeddedResource>
</ItemGroup>
```

### サードパーティコントロール

```vbnet
' 移行前に互換性を確認する
' 多くのベンダーが .NET 6+ 対応版を提供している

' DevExpress、Telerik、Infragistics 等 — ベンダードキュメントを確認する
' 古い/放棄されたコントロール — 置き換えが必要な場合がある

' コントロールのソースコードが入手可能な場合は一緒に移行することを検討する
' または以下に置き換える:
' - 組み込み .NET コントロール
' - オープンソース代替品（ライセンスに注意）
' - カスタム実装
```

### データベースアクセス

```vbnet
' Entity Framework 6 から EF Core へ
' 旧（EF6）
Using context = New MyDbContext()
    Dim customers = context.Customers.Where(Function(c) c.IsActive).ToList()
End Using

' 新（EF Core）
Await Using context = New MyDbContext()
    Dim customers = Await context.Customers _
        .Where(Function(c) c.IsActive) _
        .ToListAsync()
End Using
```

### WCF クライアントの移行

```vbnet
' オプション 1: System.ServiceModel パッケージを使用する
' <PackageReference Include="System.ServiceModel.Http" Version="6.0.0" />

' オプション 2: 新しいクライアントを生成する
' dotnet-svcutil https://service.example.com/MyService?wsdl

' オプション 3: REST サービス向けに HTTP クライアントに置き換える
Public Class MyServiceClient
    Private ReadOnly _client As HttpClient

    Public Async Function GetCustomerAsync(id As Integer) As Task(Of Customer)
        Dim response = Await _client.GetAsync($"api/customers/{id}")
        response.EnsureSuccessStatusCode()
        Return Await response.Content.ReadFromJsonAsync(Of Customer)()
    End Function
End Class
```

## 高 DPI とモダン機能

### 高 DPI サポートの有効化

```vbnet
' Program.vb 内（ApplicationConfiguration.Initialize() に既に含まれている）
Application.SetHighDpiMode(HighDpiMode.PerMonitorV2)
```

```xml
<!-- app.manifest -->
<application xmlns="urn:schemas-microsoft-com:asm.v3">
  <windowsSettings>
    <dpiAware xmlns="http://schemas.microsoft.com/SMI/2005/WindowsSettings">true/pm</dpiAware>
    <dpiAwareness xmlns="http://schemas.microsoft.com/SMI/2016/WindowsSettings">PerMonitorV2</dpiAwareness>
  </windowsSettings>
</application>
```

### .NET 8/9 の新機能の活用

```vbnet
' ボタンコマンド（.NET 8+）
btnSave.Command = New RelayCommand(AddressOf Save, AddressOf CanSave)

' システムアイコン（.NET 8+）
Dim icon = SystemIcons.GetStockIcon(StockIconId.Info)

' 改善されたデータバインディング（.NET 9+）
' パフォーマンスとメモリ使用量の改善

' FolderBrowserDialog の改善
Using dialog = New FolderBrowserDialog With {
    .InitialDirectory = Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments),
    .ShowNewFolderButton = True,
    .UseDescriptionForTitle = True,
    .Description = "Select output folder"
}
End Using
```

## 移行後のテスト

### 機能テストチェックリスト

- [ ] アプリケーションがエラーなく起動する
- [ ] すべてのフォームが正しく開く
- [ ] デザイナーがすべてのフォームを読み込む
- [ ] データバインディングが正しく動作する
- [ ] 入力検証が期待通りに動作する
- [ ] データベース操作が動作する
- [ ] ファイル操作が動作する
- [ ] 印刷が動作する（該当する場合）
- [ ] サードパーティコントロールが機能する
- [ ] リソース（画像、アイコン）が読み込まれる
- [ ] ローカライズが動作する（該当する場合）
- [ ] 高 DPI が正しく表示される
- [ ] キーボードショートカットが動作する
- [ ] タブオーダーが正しい

### パフォーマンステスト

```vbnet
' 基本的な起動時間計測
Dim sw = Stopwatch.StartNew()
Application.Run(New MainForm())
Console.WriteLine($"Startup: {sw.ElapsedMilliseconds}ms")

' メモリ使用量の比較
' dotnet-counters または Visual Studio 診断ツールを使用する
' dotnet-counters monitor --process-id <PID>
```

## デプロイ

### フレームワーク依存デプロイ

```bash
# ターゲットマシンに .NET ランタイムが必要
dotnet publish -c Release -r win-x64 --self-contained false
```

### 自己完結型デプロイ

```bash
# ランタイムを含む。サイズは大きいが依存関係なし
dotnet publish -c Release -r win-x64 --self-contained true

# 単一ファイル（配布に推奨）
dotnet publish -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true

# トリム済み（サイズ削減、十分なテストが必要）
dotnet publish -c Release -r win-x64 --self-contained true -p:PublishTrimmed=true
```

```xml
<!-- 発行のプロジェクト設定 -->
<PropertyGroup>
  <RuntimeIdentifier>win-x64</RuntimeIdentifier>
  <SelfContained>true</SelfContained>
  <PublishSingleFile>true</PublishSingleFile>
  <IncludeNativeLibrariesForSelfExtract>true</IncludeNativeLibrariesForSelfExtract>
  <EnableCompressionInSingleFile>true</EnableCompressionInSingleFile>
</PropertyGroup>
```

## 段階的移行戦略

大規模なアプリケーションでは、段階的な移行を検討する。

### 1. 共有ライブラリアプローチ

```text
Solution/
├── MyApp.Core/                 # .NET Standard 2.0 - 共有
│   ├── Models/
│   ├── Services/
│   └── Interfaces/
├── MyApp.WinForms.Legacy/      # .NET Framework 4.8 - 旧 UI
│   └── References MyApp.Core
├── MyApp.WinForms.Modern/      # .NET 9 - 新 UI
│   └── References MyApp.Core
```

### 2. 機能ごとの移行

1. 共有ビジネスロジックを .NET Standard に移行する
2. 新しいモダン .NET WinForms プロジェクトを作成する
3. フォームを一つずつ移行する
4. 移行した各フォームを十分にテストする
5. 完了したら旧プロジェクトを廃止する

### 3. サイドバイサイド開発

```xml
<!-- 共有コードのマルチターゲット -->
<PropertyGroup>
  <TargetFrameworks>net48;net9.0-windows</TargetFrameworks>
</PropertyGroup>
```

```vbnet
' 必要に応じた条件付きコンパイル
#If NET48 Then
    ' .NET Framework 固有のコード
#Else
    ' モダン .NET のコード
#End If
```

## 参考資料

- [公式移行ガイド](https://learn.microsoft.com/ja-jp/dotnet/desktop/winforms/migration/)
- [.NET Upgrade Assistant](https://learn.microsoft.com/ja-jp/dotnet/core/porting/upgrade-assistant-overview)
- [破壊的変更](https://learn.microsoft.com/ja-jp/dotnet/core/compatibility/winforms)
- [Windows Forms の新機能](https://learn.microsoft.com/ja-jp/dotnet/desktop/winforms/whats-new/)
