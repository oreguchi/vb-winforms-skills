# 高度なDIパターン

DI拡張メソッドを使ったテスト、Akka.NETアクタースコープ管理、高度な登録パターン。

## 目次

- [テストでの利点](#テストでの利点)
- [Akka.NETアクタースコープ管理](#akkaneアクタースコープ管理)
- [共通パターン](#共通パターン)

## テストでの利点

`Add*`拡張メソッドの主な利点は、**本番の設定をテストで再利用**できることである。

### WebApplicationFactory

```vbnet
Public Class ApiTests
    Implements IClassFixture(Of WebApplicationFactory(Of Program))

    Private ReadOnly _factory As WebApplicationFactory(Of Program)

    Public Sub New(factory As WebApplicationFactory(Of Program))
        _factory = factory.WithWebHostBuilder(Sub(builder)
            builder.ConfigureServices(Sub(services)
                ' Add*メソッドで本番サービスはすでに登録済み
                ' テストで差異のある部分だけ上書きする

                ' メール送信をテストダブルに差し替える
                services.RemoveAll(Of IEmailSender)()
                services.AddSingleton(Of IEmailSender, TestEmailSender)()

                ' 外部決済プロセッサをフェイクに差し替える
                services.RemoveAll(Of IPaymentProcessor)()
                services.AddSingleton(Of IPaymentProcessor, FakePaymentProcessor)()
            End Sub)
        End Sub)
    End Sub

    <Fact>
    Public Async Function CreateOrder_SendsConfirmationEmail() As Task
        Dim client = _factory.CreateClient()
        Dim emailSender = TryCast(_factory.Services.GetRequiredService(Of IEmailSender)(), TestEmailSender)

        Await client.PostAsJsonAsync("/api/orders", New CreateOrderRequest())

        Assert.Single(emailSender.SentEmails)
    End Function
End Class
```

### Akka.Hosting.TestKit

```vbnet
Public Class OrderActorSpecs
    Inherits Akka.Hosting.TestKit.TestKit

    Protected Overrides Sub ConfigureAkka(builder As AkkaConfigurationBuilder, provider As IServiceProvider)
        ' 本番Akka設定を再利用する
        builder.AddOrderActors()

        ' Akka.Hosting.TestKit.TestKit には ConfigureServices のオーバーライドポイントがない。
        ' サービス登録は TestKit のコンストラクタ引数または ConfigureAkka 内で行う。
        ' 外部依存の差し替えは ConfigureAkka 内で provider 経由で対応する。
    End Sub

    <Fact>
    Public Async Function OrderActor_ProcessesPayment() As Task
        Dim orderActor = ActorRegistry.Get(Of OrderActor)()
        orderActor.Tell(New ProcessOrder(orderId))

        ExpectMsg(Of OrderProcessed)()
    End Function
End Class
```

### スタンドアロン単体テスト

```vbnet
Public Class UserServiceTests
    Private ReadOnly _provider As ServiceProvider

    Public Sub New()
        Dim services = New ServiceCollection()

        ' 本番の登録を再利用する
        services.AddUserServices()

        ' テスト用インフラを追加する
        services.AddSingleton(Of IUserRepository, InMemoryUserRepository)()

        _provider = services.BuildServiceProvider()
    End Sub

    <Fact>
    Public Async Function CreateUser_ValidData_Succeeds() As Task
        Dim service = _provider.GetRequiredService(Of IUserService)()
        Dim result = Await service.CreateUserAsync(New CreateUserRequest())

        Assert.True(result.IsSuccess)
    End Function
End Class
```

## Akka.NETアクタースコープ管理

**アクターはDIスコープ（Scoped）を自動的に持たない。** アクター内でスコープサービスが必要な場合は、`IServiceProvider`を注入して手動でスコープを作成する。

### パターン：メッセージごとのスコープ

```vbnet
Public NotInheritable Class AccountProvisionActor
    Inherits ReceiveActor

    Private ReadOnly _serviceProvider As IServiceProvider
    Private ReadOnly _mailingActor As IActorRef

    Public Sub New(
        serviceProvider As IServiceProvider,
        mailingActor As IRequiredActor(Of MailingActor))

        _serviceProvider = serviceProvider
        _mailingActor = mailingActor.ActorRef

        ReceiveAsync(Of ProvisionAccount)(AddressOf HandleProvisionAccount)
    End Sub

    Private Async Function HandleProvisionAccount(msg As ProvisionAccount) As Task
        ' このメッセージ処理用のスコープを作成する
        Using scope = _serviceProvider.CreateScope()

            ' スコープサービスを解決する
            Dim userManager = scope.ServiceProvider.GetRequiredService(Of UserManager(Of User))()
            Dim orderRepository = scope.ServiceProvider.GetRequiredService(Of IOrderRepository)()
            Dim emailComposer = scope.ServiceProvider.GetRequiredService(Of IPaymentEmailComposer)()

            ' スコープサービスで処理を行う
            Dim user = Await userManager.FindByIdAsync(msg.UserId)
            Dim order = Await orderRepository.CreateAsync(msg.Order)

            ' スコープがDisposeされるとDbContextがコミットされる
        End Using
    End Function
End Class
```

### このパターンが機能する理由

1. **メッセージごとに新鮮なDbContext** — エンティティトラッキングが古くならない
2. **適切なDisposeタイミング** — メッセージ処理後にコネクションが解放される
3. **分離** — あるメッセージのエラーが他のメッセージに影響しない
4. **テスト可能** — モックの`IServiceProvider`を注入できる

### アクター内のシングルトンサービス

ステートレスなサービスはスコープ不要のため、直接注入する。

```vbnet
Public NotInheritable Class NotificationActor
    Inherits ReceiveActor

    Private ReadOnly _linkGenerator As IEmailLinkGenerator  ' シングルトン — 問題なし！
    Private ReadOnly _mailingActor As IActorRef

    Public Sub New(
        linkGenerator As IEmailLinkGenerator,  ' 直接注入
        mailingActor As IRequiredActor(Of MailingActor))

        _linkGenerator = linkGenerator
        _mailingActor = mailingActor.ActorRef

        Receive(Of SendWelcomeEmail)(AddressOf Handle)
    End Sub
End Class
```

### Akka.DependencyInjectionリファレンス

- **Akka.DependencyInjection**: https://getakka.net/articles/actors/dependency-injection.html
- **Akka.Hosting**: https://github.com/akkadotnet/Akka.Hosting

## 共通パターン

### 条件付き登録

```vbnet
Imports Microsoft.Extensions.DependencyInjection
Imports System.Runtime.CompilerServices

Public Module ConditionalRegistrationExtensions
    <Extension>
    Public Function AddEmailServices(
        services As IServiceCollection,
        environment As IHostEnvironment) As IServiceCollection

        services.AddSingleton(Of IEmailComposer, MjmlEmailComposer)()

        If environment.IsDevelopment() Then
            services.AddSingleton(Of IEmailSender, MailpitEmailSender)()
        Else
            services.AddSingleton(Of IEmailSender, SmtpEmailSender)()
        End If

        Return services
    End Function
End Module
```

### ファクトリ登録

```vbnet
Imports Microsoft.Extensions.DependencyInjection
Imports System.Runtime.CompilerServices

Public Module FactoryRegistrationExtensions
    <Extension>
    Public Function AddPaymentServices(
        services As IServiceCollection,
        Optional configSection As String = "Stripe") As IServiceCollection

        services.AddOptions(Of StripeOptions)() _
            .BindConfiguration(configSection) _
            .ValidateOnStart()

        services.AddSingleton(Of IPaymentProcessor)(Function(sp)
            Dim options = sp.GetRequiredService(Of IOptions(Of StripeOptions))().Value
            Dim logger = sp.GetRequiredService(Of ILogger(Of StripePaymentProcessor))()

            Return New StripePaymentProcessor(options.ApiKey, options.WebhookSecret, logger)
        End Function)

        Return services
    End Function
End Module
```

### キー付きサービス（.NET 8以降）

```vbnet
Imports Microsoft.Extensions.DependencyInjection
Imports System.Runtime.CompilerServices

Public Module KeyedServiceExtensions
    <Extension>
    Public Function AddNotificationServices(services As IServiceCollection) As IServiceCollection
        services.AddKeyedSingleton(Of INotificationSender, EmailNotificationSender)("email")
        services.AddKeyedSingleton(Of INotificationSender, SmsNotificationSender)("sms")
        services.AddKeyedSingleton(Of INotificationSender, PushNotificationSender)("push")

        services.AddScoped(Of INotificationDispatcher, NotificationDispatcher)()

        Return services
    End Function
End Module
```
