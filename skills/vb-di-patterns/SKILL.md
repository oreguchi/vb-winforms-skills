---
name: vb-di-patterns
description: IServiceCollection拡張メソッドを使用してDI登録を整理する。関連するサービスをコンポーザブルなAdd*メソッドにグループ化し、Program.vbをクリーンに保つとともに、テストで設定を再利用可能にする。VB.NET構文では<Extension>属性とModuleを使用して拡張メソッドを定義する。
invocable: false
---

# 依存性注入（DI）パターン

## このスキルを使う場面

次の場面で使用する。
- ASP.NET CoreアプリケーションでサービスのDI登録を整理したい
- 数百行の登録コードで肥大化したProgram.vbやStartup.vbを解消したい
- 本番環境とテストでサービス設定を再利用したい
- Microsoft.Extensions.DependencyInjectionと統合するライブラリを設計したい

## 参考ファイル

- [advanced-patterns.md](advanced-patterns.md): DI拡張メソッドを使ったテスト、Akka.NETアクタースコープ管理、条件付き登録・ファクトリ登録・キー付き登録パターン

---

## 問題

整理しないと、Program.vbは管理不能になる。

```vbnet
' BAD: 200行以上の無秩序な登録
Dim builder = WebApplication.CreateBuilder(args)

builder.Services.AddScoped(Of IUserRepository, UserRepository)()
builder.Services.AddScoped(Of IOrderRepository, OrderRepository)()
builder.Services.AddScoped(Of IProductRepository, ProductRepository)()
builder.Services.AddScoped(Of IUserService, UserService)()
' ... さらに150行 ...
```

問題点：関連する登録を見つけにくい、境界が不明瞭、テストで再利用できない、マージコンフリクトが発生しやすい。

---

## 解決策：拡張メソッドによるコンポジション

関連する登録を拡張メソッドにグループ化する。

```vbnet
' GOOD: クリーンでコンポーザブルなProgram.vb
Dim builder = WebApplication.CreateBuilder(args)

builder.Services _
    .AddUserServices() _
    .AddOrderServices() _
    .AddEmailServices() _
    .AddPaymentServices() _
    .AddValidators()

Dim app = builder.Build()
```

---

## 拡張メソッドパターン

### 基本構造

```vbnet
Imports System.Runtime.CompilerServices
Imports Microsoft.Extensions.DependencyInjection

Namespace MyApp.Users

    Public Module UserServiceCollectionExtensions
        <Extension>
        Public Function AddUserServices(services As IServiceCollection) As IServiceCollection
            services.AddScoped(Of IUserRepository, UserRepository)()
            services.AddScoped(Of IUserReadStore, UserReadStore)()
            services.AddScoped(Of IUserWriteStore, UserWriteStore)()
            services.AddScoped(Of IUserService, UserService)()
            services.AddScoped(Of IUserValidationService, UserValidationService)()

            Return services
        End Function
    End Module

End Namespace
```

### 設定パラメータあり

```vbnet
Imports System.Runtime.CompilerServices
Imports Microsoft.Extensions.DependencyInjection

Namespace MyApp.Email

    Public Module EmailServiceCollectionExtensions
        <Extension>
        Public Function AddEmailServices(
            services As IServiceCollection,
            Optional configSectionName As String = "EmailSettings") As IServiceCollection

            services.AddOptionsWithValidateOnStart(Of EmailOptions)() _
                .BindConfiguration(configSectionName) _
                .ValidateDataAnnotations()

            services.AddSingleton(Of IMjmlTemplateRenderer, MjmlTemplateRenderer)()
            services.AddSingleton(Of IEmailLinkGenerator, EmailLinkGenerator)()
            services.AddScoped(Of IUserEmailComposer, UserEmailComposer)()
            services.AddScoped(Of IEmailSender, SmtpEmailSender)()

            Return services
        End Function
    End Module

End Namespace
```

---

## ファイル構成

拡張メソッドは、登録するサービスの近くに配置する。

```
src/
  MyApp.Api/
    Program.vb                           # 全Add*メソッドを組み合わせる
  MyApp.Users/
    Services/
      UserService.vb
    UserServiceCollectionExtensions.vb   # AddUserServices()
  MyApp.Orders/
    OrderServiceCollectionExtensions.vb  # AddOrderServices()
  MyApp.Email/
    EmailServiceCollectionExtensions.vb  # AddEmailServices()
```

**命名規則**: `{Feature}ServiceCollectionExtensions.vb` をそのフィーチャーのサービスと同じ場所に置く。

---

## 命名規則

| パターン | 用途 |
|---------|---------|
| `Add{Feature}Services()` | 汎用フィーチャー登録 |
| `Add{Feature}()` | 明確な場合の短縮形 |
| `Configure{Feature}()` | 主にオプション設定を行う場合 |
| `Use{Feature}()` | ミドルウェア（IApplicationBuilder用） |

---

## テストでの利点

`Add*` パターンを使うと**本番の設定をテストで再利用**でき、差異のある部分だけ上書きできる。WebApplicationFactory、Akka.Hosting.TestKit、スタンドアロンのServiceCollectionと組み合わせて使用できる。完全なテスト例については[advanced-patterns.md](advanced-patterns.md)の該当セクションを参照。

---

## 階層的拡張メソッド

大規模なアプリケーションでは、拡張メソッドを階層的に組み合わせる。

```vbnet
Imports Microsoft.Extensions.DependencyInjection
Imports System.Runtime.CompilerServices

Public Module AppServiceCollectionExtensions
    <Extension>
    Public Function AddAppServices(services As IServiceCollection) As IServiceCollection
        Return services _
            .AddDomainServices() _
            .AddInfrastructureServices() _
            .AddApiServices()
    End Function
End Module

Public Module DomainServiceCollectionExtensions
    <Extension>
    Public Function AddDomainServices(services As IServiceCollection) As IServiceCollection
        Return services _
            .AddUserServices() _
            .AddOrderServices() _
            .AddProductServices()
    End Function
End Module
```

---

## Akka.Hosting統合

同じパターンはAkka.NETアクター設定にも使用できる。

```vbnet
Public Module OrderActorExtensions
    <Extension>
    Public Function AddOrderActors(builder As AkkaConfigurationBuilder) As AkkaConfigurationBuilder
        Return builder _
            .WithActors(Sub(system, registry, resolver)
                Dim orderProps = resolver.Props(Of OrderActor)()
                Dim orderRef = system.ActorOf(orderProps, "orders")
                registry.Register(Of OrderActor)(orderRef)
            End Sub)
    End Function
End Module

' Program.vbでの使用例
builder.Services.AddAkka("MySystem", Sub(akkaBuilder, sp)
    akkaBuilder _
        .AddOrderActors() _
        .AddInventoryActors() _
        .AddNotificationActors()
End Sub)
```

完全なAkka.Hostingパターンは`akka-hosting-actor-patterns`スキルを参照。

---

## アンチパターン

### NG：すべてをProgram.vbに登録する

```vbnet
' BAD: 200行以上の登録コードが並ぶ肥大化したProgram.vb
```

### NG：過度に汎用的な拡張メソッドを作る

```vbnet
' BAD: 名前が曖昧で何が登録されるか伝わらない
<Extension>
Public Function AddServices(services As IServiceCollection) As IServiceCollection
    ' ...
End Function
```

### NG：重要な設定を隠蔽する

```vbnet
' BAD: 設定が埋もれている
<Extension>
Public Function AddDatabase(services As IServiceCollection) As IServiceCollection
    services.AddDbContext(Of AppDbContext)(Sub(options)
        options.UseSqlServer("hardcoded-connection-string")  ' 隠蔽！
    End Sub)
    Return services
End Function

' GOOD: 設定を明示的に受け取る
<Extension>
Public Function AddDatabase(
    services As IServiceCollection,
    connectionString As String) As IServiceCollection

    services.AddDbContext(Of AppDbContext)(Sub(options)
        options.UseSqlServer(connectionString)
    End Sub)
    Return services
End Function
```

---

## ベストプラクティス一覧

| プラクティス | 効果 |
|----------|---------|
| 関連サービスを`Add*`メソッドにグループ化する | Program.vbをクリーンに保ち、境界を明確にする |
| 拡張メソッドを登録するサービスの近くに置く | 見つけやすく保守しやすい |
| `IServiceCollection`を返してチェーン可能にする | FluentなAPI |
| 設定パラメータを受け取る | 柔軟性の確保 |
| 命名規則を統一する（`Add{Feature}Services`） | 発見しやすさの向上 |
| テストで本番の拡張メソッドを再利用する | 信頼性向上、重複削減 |

---

## ライフタイム管理

| ライフタイム | 使用場面 | 例 |
|----------|----------|----------|
| **シングルトン（Singleton）** | ステートレス、スレッドセーフ、生成コストが高い | 設定、HttpClientファクトリ、キャッシュ |
| **スコープ（Scoped）** | リクエストごとのステート、データベースコンテキスト | DbContext、リポジトリ、ユーザーコンテキスト |
| **一時（Transient）** | 軽量、ステートフル、生成コストが低い | バリデータ、短命ヘルパー |

```vbnet
' SINGLETON: ステートレスサービス、安全に共有
services.AddSingleton(Of IMjmlTemplateRenderer, MjmlTemplateRenderer)()

' SCOPED: データベースアクセス、リクエストごとのステート
services.AddScoped(Of IUserRepository, UserRepository)()

' TRANSIENT: 安価で短命
services.AddTransient(Of CreateUserRequestValidator)()
```

**スコープサービスはスコープを必要とする。** ASP.NET CoreはHTTPリクエストごとにスコープを作成する。バックグラウンドサービスやアクター内では手動でスコープを作成する必要がある。

アクタースコープ管理パターンは[advanced-patterns.md](advanced-patterns.md)を参照。

---

## よくある間違い

### スコープサービスをシングルトンに注入する

```vbnet
' BAD: シングルトンがスコープサービスをキャプチャ — 古くなったDbContext！
Public Class CacheService  ' シングルトンとして登録
    Private ReadOnly _repo As IUserRepository  ' スコープ — 起動時にキャプチャされる！
End Class

' GOOD: IServiceProviderを注入し、操作ごとにスコープを作成する
Public Class CacheService
    Private ReadOnly _serviceProvider As IServiceProvider

    Public Async Function GetUserAsync(id As String) As Task(Of User)
        Using scope = _serviceProvider.CreateScope()
            Dim repo = scope.ServiceProvider.GetRequiredService(Of IUserRepository)()
            Return Await repo.GetByIdAsync(id)
        End Using
    End Function
End Class
```

### バックグラウンド処理でスコープを作成しない

```vbnet
' BAD: スコープサービスにスコープなし
Public Class BadBackgroundService
    Inherits BackgroundService

    Private ReadOnly _orderService As IOrderService  ' スコープ — 例外が発生する！
End Class

' GOOD: 処理単位ごとにスコープを作成する
Public Class GoodBackgroundService
    Inherits BackgroundService

    Private ReadOnly _scopeFactory As IServiceScopeFactory

    Protected Overrides Async Function ExecuteAsync(ct As CancellationToken) As Task
        Using scope = _scopeFactory.CreateScope()
            Dim orderService = scope.ServiceProvider.GetRequiredService(Of IOrderService)()
            ' ...
        End Using
    End Function
End Class
```

---

## リソース

- **Microsoft.Extensions.DependencyInjection**: https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection
- **Akka.Hosting**: https://github.com/akkadotnet/Akka.Hosting
- **Akka.DependencyInjection**: https://getakka.net/articles/actors/dependency-injection.html
- **オプションパターン**: `microsoft-extensions-configuration`スキルを参照
