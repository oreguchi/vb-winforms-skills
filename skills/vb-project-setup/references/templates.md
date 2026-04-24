# プロジェクトテンプレート一覧

## 概要

.NET は `dotnet new` 経由でプロジェクトテンプレートを提供する。このリファレンスでは一般的なテンプレートとその典型的なユースケースを説明する。VB.NET プロジェクトを作成する場合は `-lang VB` オプションを付けることで VB.NET テンプレートが生成される。

---

## 利用可能なテンプレートの一覧表示

```bash
# インストール済みテンプレートをすべて表示
dotnet new list

# テンプレートを検索
dotnet new search webapi

# テンプレートパックをインストール
dotnet new install Microsoft.AspNetCore.SpaTemplates
```

---

## コンソールアプリケーション

### 基本コンソールアプリ

```bash
dotnet new console -lang VB -n MyApp -o src/MyApp
```

**生成される構造:**

```text
src/MyApp/
├── MyApp.vbproj
└── Program.vb
```

**典型的な `Program.vb`:**

```vbnet
' DI を使用したコンソールアプリ向けの最小ホスティング
Imports Microsoft.Extensions.DependencyInjection
Imports Microsoft.Extensions.Hosting

Module Program
    Sub Main(args As String())
        Dim builder = Host.CreateApplicationBuilder(args)

        builder.Services.AddSingleton(Of MyService)()

        Using host = builder.Build()
            host.Run()
        End Using
    End Sub
End Module
```

---

## クラスライブラリ

### 標準ライブラリ

```bash
dotnet new classlib -lang VB -n MyProduct.Core -o src/MyProduct.Core
```

### マルチターゲティング付きライブラリ

**生成された `.vbproj` を変更する:**

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFrameworks>net9.0;net8.0;netstandard2.0</TargetFrameworks>
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
    <PackageId>MyCompany.MyProduct.Core</PackageId>
    <Description>Core library for MyProduct</Description>
    <RootNamespace>MyProduct.Core</RootNamespace>
  </PropertyGroup>
</Project>
```

---

## ASP.NET Core Web API

### Minimal API

```bash
dotnet new webapi -lang VB -n MyProduct.Api -o src/MyProduct.Api --use-minimal-apis
```

**典型的な構造:**

```text
src/MyProduct.Api/
├── MyProduct.Api.vbproj
├── Program.vb
├── Properties/
│   └── launchSettings.json
├── appsettings.json
└── appsettings.Development.json
```

**Minimal API `Program.vb` の例:**

```vbnet
Imports Microsoft.AspNetCore.Builder
Imports Microsoft.AspNetCore.Http
Imports Microsoft.Extensions.DependencyInjection
Imports Microsoft.Extensions.Hosting

Module Program
    Sub Main(args As String())
        Dim builder = WebApplication.CreateBuilder(args)

        builder.Services.AddEndpointsApiExplorer()
        builder.Services.AddSwaggerGen()

        Dim app = builder.Build()

        If app.Environment.IsDevelopment() Then
            app.UseSwagger()
            app.UseSwaggerUI()
        End If

        app.UseHttpsRedirection()

        app.MapGet("/api/health", Function() Results.Ok(New With {.Status = "Healthy"})) _
           .WithName("HealthCheck") _
           .WithOpenApi()

        app.Run()
    End Sub
End Module
```

### コントローラーベース API

```bash
dotnet new webapi -lang VB -n MyProduct.Api -o src/MyProduct.Api --use-controllers
```

**コントローラーテンプレート:**

```vbnet
Imports Microsoft.AspNetCore.Mvc
Imports System.Threading

Namespace MyProduct.Api.Controllers

    <ApiController>
    <Route("api/[controller]")>
    Public Class ItemsController
        Inherits ControllerBase

        Private ReadOnly _itemService As IItemService

        Public Sub New(itemService As IItemService)
            _itemService = itemService
        End Sub

        <HttpGet>
        Public Async Function GetAll(ct As CancellationToken) As Task(Of ActionResult(Of IEnumerable(Of ItemDto)))
            Dim items = Await _itemService.GetAllAsync(ct)
            Return Ok(items)
        End Function

        <HttpGet("{id:guid}")>
        Public Async Function GetById(id As Guid, ct As CancellationToken) As Task(Of ActionResult(Of ItemDto))
            Dim item = Await _itemService.GetByIdAsync(id, ct)
            If item Is Nothing Then
                Return NotFound()
            Else
                Return Ok(item)
            End If
        End Function

        <HttpPost>
        Public Async Function Create(request As CreateItemRequest, ct As CancellationToken) As Task(Of ActionResult(Of ItemDto))
            Dim item = Await _itemService.CreateAsync(request, ct)
            Return CreatedAtAction(NameOf(GetById), New With {.id = item.Id}, item)
        End Function

    End Class

End Namespace
```

---

## ワーカーサービス

### バックグラウンドサービス

```bash
dotnet new worker -lang VB -n MyProduct.Worker -o src/MyProduct.Worker
```

**ワーカーテンプレート:**

```vbnet
Imports Microsoft.Extensions.Hosting
Imports Microsoft.Extensions.Logging
Imports Microsoft.Extensions.DependencyInjection
Imports System.Threading
Imports System.Threading.Tasks

Namespace MyProduct.Worker

    Public Class Worker
        Inherits BackgroundService

        Private ReadOnly _logger As ILogger(Of Worker)
        Private ReadOnly _scopeFactory As IServiceScopeFactory

        Public Sub New(logger As ILogger(Of Worker), scopeFactory As IServiceScopeFactory)
            _logger = logger
            _scopeFactory = scopeFactory
        End Sub

        Protected Overrides Async Function ExecuteAsync(stoppingToken As CancellationToken) As Task
            While Not stoppingToken.IsCancellationRequested
                _logger.LogInformation("Worker running at: {Time}", DateTimeOffset.Now)

                Using scope = _scopeFactory.CreateScope()
                    Dim service = scope.ServiceProvider.GetRequiredService(Of IMyService)()
                    Await service.ProcessAsync(stoppingToken)
                End Using

                Await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken)
            End While
        End Function

    End Class

End Namespace
```

---

## Windows Forms アプリケーション

### 基本 WinForms アプリ

```bash
dotnet new winforms -lang VB -n MyApp.UI -o src/MyApp.UI
```

**生成される `.vbproj`（重要プロパティ）:**

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net9.0-windows</TargetFramework>
    <UseWindowsForms>true</UseWindowsForms>
    <RootNamespace>MyApp.UI</RootNamespace>
  </PropertyGroup>
</Project>
```

**注意事項:**

- WinForms は Windows 限定のため、`TargetFramework` は `net9.0-windows` 形式（`-windows` サフィックス付き）を使用する。`Directory.Build.props` で `net9.0` を設定している場合は、WinForms プロジェクトのみ個別に上書きする。
- `<UseWindowsForms>true</UseWindowsForms>` は WinForms デザイナーと Windows Forms ランタイムを有効化するため必須。
- `<OutputType>WinExe</OutputType>` により、コンソールウィンドウを表示しない GUI 実行可能ファイルを生成する。

---

## Web アプリケーション

> **注記**: 本セクションのテンプレートのうち `dotnet new blazor` は VB.NET 公式テンプレートが提供されていない（`-lang VB` を付けても C# プロジェクトが生成される）。本セクションの Blazor サブセクションは C# 実装を参考情報として掲載する。`dotnet new webapp`（Razor Pages）は VB.NET テンプレート対応済み（下記参照）。

### Blazor Server

```bash
dotnet new blazor -n MyProduct.Web -o src/MyProduct.Web --interactivity Server
```

### Blazor WebAssembly

```bash
dotnet new blazor -n MyProduct.Web -o src/MyProduct.Web --interactivity WebAssembly
```

### Razor Pages

```bash
dotnet new webapp -lang VB -n MyProduct.Web -o src/MyProduct.Web
```

---

## テストプロジェクト

### xUnit テストプロジェクト

```bash
dotnet new xunit -lang VB -n MyProduct.Core.Tests -o tests/MyProduct.Core.Tests
```

**テストプロジェクト `.vbproj`:**

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <IsPackable>false</IsPackable>
    <IsTestProject>true</IsTestProject>
    <RootNamespace>MyProduct.Core.Tests</RootNamespace>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" />
    <PackageReference Include="xunit" />
    <PackageReference Include="xunit.runner.visualstudio" />
    <PackageReference Include="coverlet.collector" />
    <PackageReference Include="FluentAssertions" />
    <PackageReference Include="Moq" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\src\MyProduct.Core\MyProduct.Core.vbproj" />
  </ItemGroup>
</Project>
```

**テストクラステンプレート:**

```vbnet
Imports FluentAssertions
Imports Moq
Imports Xunit
Imports System.Threading

Namespace MyProduct.Core.Tests

    Public Class ItemServiceTests

        Private ReadOnly _repositoryMock As Mock(Of IItemRepository)
        Private ReadOnly _sut As ItemService

        Public Sub New()
            _repositoryMock = New Mock(Of IItemRepository)()
            _sut = New ItemService(_repositoryMock.Object)
        End Sub

        <Fact>
        Public Async Function GetByIdAsync_WhenItemExists_ReturnsItem() As Task
            ' Arrange
            Dim itemId = Guid.NewGuid()
            Dim expected = New Item With {.Id = itemId, .Name = "Test"}
            _repositoryMock _
                .Setup(Function(r) r.GetByIdAsync(itemId, It.IsAny(Of CancellationToken)())) _
                .ReturnsAsync(expected)

            ' Act
            Dim result = Await _sut.GetByIdAsync(itemId, CancellationToken.None)

            ' Assert
            result.Should().BeEquivalentTo(expected)
        End Function

        <Theory>
        <InlineData("")>
        <InlineData("   ")>
        <InlineData(Nothing)>
        Public Async Function CreateAsync_WhenNameInvalid_ThrowsArgumentException(name As String) As Task
            ' Arrange
            Dim request = New CreateItemRequest With {.Name = name}

            ' Act
            Dim act As Func(Of Task) = Async Function() Await _sut.CreateAsync(request, CancellationToken.None)

            ' Assert
            Await act.Should().ThrowAsync(Of ArgumentException)()
        End Function

    End Class

End Namespace
```

### NUnit テストプロジェクト

```bash
dotnet new nunit -lang VB -n MyProduct.Core.Tests -o tests/MyProduct.Core.Tests
```

### MSTest テストプロジェクト

```bash
dotnet new mstest -lang VB -n MyProduct.Core.Tests -o tests/MyProduct.Core.Tests
```

### 統合テストプロジェクト

```vbnet
' WebApplicationFactory ベースの統合テスト
Imports Microsoft.AspNetCore.Mvc.Testing
Imports Microsoft.Extensions.DependencyInjection
Imports Xunit

Namespace MyProduct.Api.IntegrationTests

    Public Class ItemsEndpointTests
        Implements IClassFixture(Of WebApplicationFactory(Of Program))

        Private ReadOnly _client As HttpClient

        Public Sub New(factory As WebApplicationFactory(Of Program))
            _client = factory _
                .WithWebHostBuilder(Sub(builder)
                    builder.ConfigureServices(Sub(services)
                        ' テスト用にサービスを差し替える
                        services.AddSingleton(Of IItemRepository, InMemoryItemRepository)()
                    End Sub)
                End Sub) _
                .CreateClient()
        End Sub

        <Fact>
        Public Async Function GetAll_ReturnsOkStatus() As Task
            ' Act
            Dim response = Await _client.GetAsync("/api/items")

            ' Assert
            response.EnsureSuccessStatusCode()
        End Function

    End Class

End Namespace
```

---

## ソリューションセットアップコマンド

### ソリューションとプロジェクトの作成

```bash
# ソリューションを作成
dotnet new sln -n MyProduct

# プロジェクトを作成
dotnet new classlib -lang VB -n MyProduct.Core -o src/MyProduct.Core
dotnet new classlib -lang VB -n MyProduct.Infrastructure -o src/MyProduct.Infrastructure
dotnet new webapi -lang VB -n MyProduct.Api -o src/MyProduct.Api --use-minimal-apis
dotnet new xunit -lang VB -n MyProduct.Core.Tests -o tests/MyProduct.Core.Tests
dotnet new xunit -lang VB -n MyProduct.Api.IntegrationTests -o tests/MyProduct.Api.IntegrationTests

# ソリューションにプロジェクトを追加
dotnet sln add src/MyProduct.Core/MyProduct.Core.vbproj
dotnet sln add src/MyProduct.Infrastructure/MyProduct.Infrastructure.vbproj
dotnet sln add src/MyProduct.Api/MyProduct.Api.vbproj
dotnet sln add tests/MyProduct.Core.Tests/MyProduct.Core.Tests.vbproj
dotnet sln add tests/MyProduct.Api.IntegrationTests/MyProduct.Api.IntegrationTests.vbproj

# プロジェクト参照を追加
dotnet add src/MyProduct.Infrastructure/MyProduct.Infrastructure.vbproj reference src/MyProduct.Core/MyProduct.Core.vbproj
dotnet add src/MyProduct.Api/MyProduct.Api.vbproj reference src/MyProduct.Core/MyProduct.Core.vbproj
dotnet add src/MyProduct.Api/MyProduct.Api.vbproj reference src/MyProduct.Infrastructure/MyProduct.Infrastructure.vbproj
dotnet add tests/MyProduct.Core.Tests/MyProduct.Core.Tests.vbproj reference src/MyProduct.Core/MyProduct.Core.vbproj
dotnet add tests/MyProduct.Api.IntegrationTests/MyProduct.Api.IntegrationTests.vbproj reference src/MyProduct.Api/MyProduct.Api.vbproj
```

---

## .NET Aspire アプリケーション

> **注記**: .NET Aspire AppHost および ServiceDefaults は VB.NET 公式テンプレートを提供していない（現時点では C# 専用）。Aspire ソリューションの AppHost と ServiceDefaults は C# のままにしつつ、業務ロジックを含むマイクロサービス（Web API / Worker 等）を VB.NET で実装して Aspire に追加する混在構成が一般的。

### Aspire AppHost

```bash
dotnet new aspire -n MyProduct
```

**作成される構造:**

```text
MyProduct/
├── MyProduct.AppHost/
│   ├── MyProduct.AppHost.csproj
│   └── Program.cs
├── MyProduct.ServiceDefaults/
│   ├── MyProduct.ServiceDefaults.csproj
│   └── Extensions.cs
└── MyProduct.sln
```

**AppHost `Program.cs`:**

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var cache = builder.AddRedis("cache");
var postgres = builder.AddPostgres("postgres")
    .AddDatabase("ordersdb");

var api = builder.AddProject<Projects.MyProduct_Api>("api")
    .WithReference(cache)
    .WithReference(postgres);

builder.AddProject<Projects.MyProduct_Web>("web")
    .WithReference(api);

builder.Build().Run();
```

### 既存の Aspire ソリューションへのサービス追加

```bash
# 新しい API プロジェクトを追加
dotnet new webapi -lang VB -n MyProduct.OrdersApi -o src/MyProduct.OrdersApi
dotnet sln add src/MyProduct.OrdersApi/MyProduct.OrdersApi.vbproj

# ServiceDefaults を参照
dotnet add src/MyProduct.OrdersApi/MyProduct.OrdersApi.vbproj reference src/MyProduct.ServiceDefaults/MyProduct.ServiceDefaults.csproj
```

---

## gRPC サービス

> **注記**: `dotnet new grpc` は VB.NET 公式テンプレートが提供されていない。`-lang VB` を付けても VB.NET プロジェクトは生成されず、C# プロジェクトが作成される。VB.NET で gRPC サービスを実装したい場合は、空の ASP.NET Core プロジェクトに `Grpc.AspNetCore` NuGet パッケージを追加し、`.proto` ファイルからの生成コードと連携させる必要がある。本セクションは C# 実装を参考情報として掲載する。

```bash
dotnet new grpc -n MyProduct.GrpcService -o src/MyProduct.GrpcService
```

**Proto ファイルテンプレート:**

```protobuf
syntax = "proto3";

option csharp_namespace = "MyProduct.GrpcService";

package orders;

service OrderService {
  rpc GetOrder (GetOrderRequest) returns (OrderResponse);
  rpc CreateOrder (CreateOrderRequest) returns (OrderResponse);
  rpc ListOrders (ListOrdersRequest) returns (stream OrderResponse);
}

message GetOrderRequest {
  string order_id = 1;
}

message CreateOrderRequest {
  string customer_id = 1;
  repeated OrderItem items = 2;
}

message OrderItem {
  string product_id = 1;
  int32 quantity = 2;
}

message OrderResponse {
  string order_id = 1;
  string customer_id = 2;
  repeated OrderItem items = 3;
  string status = 4;
}

message ListOrdersRequest {
  string customer_id = 1;
}
```

---

## ツールマニフェスト

### ツールマニフェストの作成

```bash
dotnet new tool-manifest
```

**`.config/dotnet-tools.json` が作成される:**

```json
{
  "version": 1,
  "isRoot": true,
  "tools": {}
}
```

### ローカルツールのインストール

```bash
dotnet tool install dotnet-ef
dotnet tool install dotnet-format
dotnet tool install dotnet-reportgenerator-globaltool
```

**更新後のマニフェスト:**

```json
{
  "version": 1,
  "isRoot": true,
  "tools": {
    "dotnet-ef": {
      "version": "9.0.0",
      "commands": ["dotnet-ef"]
    },
    "dotnet-format": {
      "version": "5.1.250801",
      "commands": ["dotnet-format"]
    },
    "dotnet-reportgenerator-globaltool": {
      "version": "5.3.10",
      "commands": ["reportgenerator"]
    }
  }
}
```

### ツールの復元

```bash
dotnet tool restore
```

---

## クイックリファレンス: 一般的なテンプレートオプション

| テンプレート | コマンド | 主なオプション |
|----------|---------|-------------|
| コンソール | `dotnet new console -lang VB` | `--framework` |
| クラスライブラリ | `dotnet new classlib -lang VB` | `--framework` |
| Web API | `dotnet new webapi -lang VB` | `--use-controllers`, `--use-minimal-apis`, `--auth` |
| Blazor | `dotnet new blazor` (VB.NET 非対応) | `--interactivity`, `--empty` |
| ワーカー | `dotnet new worker -lang VB` | `--framework` |
| xUnit | `dotnet new xunit -lang VB` | `--framework` |
| ソリューション | `dotnet new sln` | - |
| gitignore | `dotnet new gitignore` | - |
| editorconfig | `dotnet new editorconfig` | - |
| global.json | `dotnet new globaljson` | `--sdk-version`, `--roll-forward` |
