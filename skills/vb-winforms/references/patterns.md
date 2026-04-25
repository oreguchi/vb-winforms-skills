# WinForms パターンリファレンス

## アーキテクチャパターン

### MVP（Model-View-Presenter）

MVP は、テスト容易性と関心の分離が必要な WinForms アプリケーションに推奨されるパターンである。

**構造:**
- **Model**: ドメインエンティティとビジネスロジック
- **View**: インターフェースを実装したフォーム。UI の関心事のみを担う
- **Presenter**: Model と View の仲介役。プレゼンテーションロジックを保持する

**主な特徴:**
- View はパッシブでイベントを発生させる
- Presenter がビューのイベントを購読し、ビューのプロパティを更新する
- Presenter は UI なしでユニットテスト可能
- View インターフェースによってモック化が可能

```vbnet
' View コントラクト
Public Interface IOrderView
    Property OrderId As Integer
    Property CustomerName As String
    Property Total As Decimal
    WriteOnly Property Lines As IEnumerable(Of OrderLine)

    Event LoadRequested As EventHandler
    Event SaveRequested As EventHandler
    Event CancelRequested As EventHandler

    Sub Close()
    Sub ShowValidationError(field As String, message As String)
    Sub ClearValidationErrors()
End Interface

' Presenter
Public Class OrderPresenter
    Private ReadOnly _view As IOrderView
    Private ReadOnly _repository As IOrderRepository
    Private _currentOrder As Order

    Public Sub New(view As IOrderView, repository As IOrderRepository)
        _view = view
        _repository = repository

        AddHandler _view.LoadRequested, Async Sub(s, e) Await LoadOrderAsync()
        AddHandler _view.SaveRequested, Async Sub(s, e) Await SaveOrderAsync()
        AddHandler _view.CancelRequested, Sub(s, e) _view.Close()
    End Sub

    Private Async Function LoadOrderAsync() As Task
        _currentOrder = Await _repository.GetByIdAsync(_view.OrderId)
        If _currentOrder IsNot Nothing Then
            _view.CustomerName = _currentOrder.CustomerName
            _view.Total = _currentOrder.Total
            _view.Lines = _currentOrder.Lines
        End If
    End Function

    Private Async Function SaveOrderAsync() As Task
        _view.ClearValidationErrors()

        If String.IsNullOrWhiteSpace(_view.CustomerName) Then
            _view.ShowValidationError("CustomerName", "Customer name is required")
            Return
        End If

        If _currentOrder IsNot Nothing Then
            _currentOrder.CustomerName = _view.CustomerName
            Await _repository.SaveAsync(_currentOrder)
            _view.Close()
        End If
    End Function
End Class
```

### MVVM（Model-View-ViewModel）

MVVM はデータバインディングを使って WinForms でも利用できるが、WPF でより一般的である。以下の場合に使用する。
- 重厚なデータバインディング要件がある
- WinForms と WPF の間で ViewModel を共有する
- チームが他のフレームワークで MVVM に慣れている

```vbnet
Public Class OrderViewModel
    Implements INotifyPropertyChanged

    Private _customerName As String = String.Empty
    Private _total As Decimal
    Private _isBusy As Boolean

    Public Property CustomerName As String
        Get
            Return _customerName
        End Get
        Set(value As String)
            _customerName = value
            OnPropertyChanged()
        End Set
    End Property

    Public Property Total As Decimal
        Get
            Return _total
        End Get
        Set(value As Decimal)
            _total = value
            OnPropertyChanged()
        End Set
    End Property

    Public Property IsBusy As Boolean
        Get
            Return _isBusy
        End Get
        Set(value As Boolean)
            _isBusy = value
            OnPropertyChanged()
            OnPropertyChanged(NameOf(IsNotBusy))
        End Set
    End Property

    Public ReadOnly Property IsNotBusy As Boolean
        Get
            Return Not IsBusy
        End Get
    End Property

    Public Property SaveCommand As ICommand
    Public Property LoadCommand As ICommand

    Public Event PropertyChanged As PropertyChangedEventHandler Implements INotifyPropertyChanged.PropertyChanged

    Protected Sub OnPropertyChanged(<CallerMemberName> Optional name As String = Nothing)
        RaiseEvent PropertyChanged(Me, New PropertyChangedEventArgs(name))
    End Sub
End Class
```

### Passive View

View にロジックをまったく持たせない、MVP のより厳格なバリアントである。
- すべての判断はプレゼンターが行う
- View はプロパティとイベントのみを公開する
- 最大のテスト容易性、最小のビューコード

```vbnet
' Passive View — ロジックをまったく持たない
Public Partial Class CustomerForm
    Inherits Form
    Implements ICustomerView

    Public Property FirstName As String Implements ICustomerView.FirstName
        Get
            Return txtFirstName.Text
        End Get
        Set(value As String)
            txtFirstName.Text = value
        End Set
    End Property

    Public Property LastName As String Implements ICustomerView.LastName
        Get
            Return txtLastName.Text
        End Get
        Set(value As String)
            txtLastName.Text = value
        End Set
    End Property

    Public Property SaveEnabled As Boolean Implements ICustomerView.SaveEnabled
        Get
            Return btnSave.Enabled
        End Get
        Set(value As Boolean)
            btnSave.Enabled = value
        End Set
    End Property

    Public Event FirstNameChanged As EventHandler Implements ICustomerView.FirstNameChanged
    Public Event LastNameChanged As EventHandler Implements ICustomerView.LastNameChanged
    Public Event SaveClicked As EventHandler Implements ICustomerView.SaveClicked

    Public Sub New()
        InitializeComponent()
        AddHandler txtFirstName.TextChanged, Sub(s, e) RaiseEvent FirstNameChanged(Me, e)
        AddHandler txtLastName.TextChanged, Sub(s, e) RaiseEvent LastNameChanged(Me, e)
        AddHandler btnSave.Click, Sub(s, e) RaiseEvent SaveClicked(Me, e)
    End Sub
End Class

' Presenter がすべてを制御する
Public Class CustomerPresenter
    Private ReadOnly _view As ICustomerView

    Public Sub New(view As ICustomerView)
        _view = view
        _view.SaveEnabled = False

        AddHandler _view.FirstNameChanged, Sub(s, e) UpdateSaveEnabled()
        AddHandler _view.LastNameChanged, Sub(s, e) UpdateSaveEnabled()
    End Sub

    Private Sub UpdateSaveEnabled()
        _view.SaveEnabled = Not String.IsNullOrWhiteSpace(_view.FirstName) _
                         AndAlso Not String.IsNullOrWhiteSpace(_view.LastName)
    End Sub
End Class
```

## データバインディングパターン

### マスター/詳細バインディング

リスト/詳細 UI に共通のパターン。

```vbnet
Public Partial Class MasterDetailForm
    Inherits Form

    Private ReadOnly _masterSource As New BindingSource()
    Private ReadOnly _detailSource As New BindingSource()

    Public Sub New()
        InitializeComponent()

        ' 詳細をマスターにリンクする
        _detailSource.DataSource = _masterSource
        _detailSource.DataMember = "OrderLines" ' ナビゲーションプロパティ

        dgvOrders.DataSource = _masterSource
        dgvOrderLines.DataSource = _detailSource

        ' 詳細コントロールを詳細ソースにバインドする
        txtLineDescription.DataBindings.Add("Text", _detailSource, "Description")
        txtLineQuantity.DataBindings.Add("Text", _detailSource, "Quantity")
    End Sub

    Private Async Function LoadAsync() As Task
        Dim orders = Await _orderService.GetAllWithLinesAsync()
        _masterSource.DataSource = New BindingList(Of Order)(orders.ToList())
    End Function
End Class
```

### 双方向バインディングと入力検証

```vbnet
Public Partial Class EditForm
    Inherits Form

    Private ReadOnly _bindingSource As New BindingSource()
    Private ReadOnly _errorProvider As New ErrorProvider()

    Private Sub SetupBindings(customer As Customer)
        _bindingSource.DataSource = customer

        ' フォーマットとパースを伴う双方向バインディング
        Dim nameBinding = New Binding("Text", _bindingSource, "Name", True)
        AddHandler nameBinding.Format, Sub(s, e) e.Value = e.Value?.ToString()?.Trim()
        AddHandler nameBinding.Parse, Sub(s, e) e.Value = e.Value?.ToString()?.Trim()
        txtName.DataBindings.Add(nameBinding)

        ' null 処理を伴うバインディング
        txtEmail.DataBindings.Add("Text", _bindingSource, "Email",
            True, DataSourceUpdateMode.OnPropertyChanged, String.Empty)

        ' チェックボックスのバインディング
        chkActive.DataBindings.Add("Checked", _bindingSource, "IsActive",
            True, DataSourceUpdateMode.OnPropertyChanged)

        ' コンボボックスのバインディング
        cboCategory.DataSource = _categories
        cboCategory.DisplayMember = "Name"
        cboCategory.ValueMember = "Id"
        cboCategory.DataBindings.Add("SelectedValue", _bindingSource, "CategoryId")
    End Sub
End Class
```

### 監視可能コレクションパターン

```vbnet
Public Class ObservableList(Of T)
    Inherits BindingList(Of T)

    Private _raiseListChangedEvents As Boolean = True

    Public Sub AddRange(items As IEnumerable(Of T))
        _raiseListChangedEvents = False
        Try
            For Each item In items
                Add(item)
            Next
        Finally
            _raiseListChangedEvents = True
            ResetBindings()
        End Try
    End Sub

    Protected Overrides Sub OnListChanged(e As ListChangedEventArgs)
        If _raiseListChangedEvents Then
            MyBase.OnListChanged(e)
        End If
    End Sub
End Class
```

## 入力検証パターン

### 集中型バリデーション

```vbnet
Public Class FormValidator
    Private ReadOnly _errorProvider As ErrorProvider
    Private ReadOnly _validators As New Dictionary(Of Control, Func(Of String))()

    Public Sub New(form As Form)
        _errorProvider = New ErrorProvider(form)
        _errorProvider.BlinkStyle = ErrorBlinkStyle.NeverBlink
    End Sub

    Public Sub AddRule(control As Control, validator As Func(Of String))
        _validators(control) = validator
        AddHandler control.Validating, Sub(s, e)
            Dim errorMsg = validator()
            _errorProvider.SetError(control, If(errorMsg, String.Empty))
            If Not String.IsNullOrEmpty(errorMsg) Then
                e.Cancel = True
            End If
        End Sub
    End Sub

    Public Function ValidateAll() As Boolean
        Dim isValid = True
        For Each kvp In _validators
            Dim errorMsg = kvp.Value()
            _errorProvider.SetError(kvp.Key, If(errorMsg, String.Empty))
            If Not String.IsNullOrEmpty(errorMsg) Then
                isValid = False
            End If
        Next
        Return isValid
    End Function

    Public Sub ClearAll()
        For Each control In _validators.Keys
            _errorProvider.SetError(control, String.Empty)
        Next
    End Sub
End Class

' 使用例
Public Partial Class CustomerForm
    Inherits Form

    Private ReadOnly _validator As FormValidator

    Public Sub New()
        InitializeComponent()

        _validator = New FormValidator(Me)
        _validator.AddRule(txtName, Function()
            If String.IsNullOrWhiteSpace(txtName.Text) Then Return "Name is required"
            Return Nothing
        End Function)
        _validator.AddRule(txtEmail, Function()
            If Not txtEmail.Text.Contains("@"c) Then Return "Invalid email format"
            Return Nothing
        End Function)
        _validator.AddRule(txtAge, Function()
            Dim age As Integer
            If Not Integer.TryParse(txtAge.Text, age) OrElse age < 0 OrElse age > 150 Then
                Return "Age must be between 0 and 150"
            End If
            Return Nothing
        End Function)
    End Sub

    Private Sub btnSave_Click(sender As Object, e As EventArgs) Handles btnSave.Click
        If _validator.ValidateAll() Then
            SaveCustomer()
        End If
    End Sub
End Class
```

### IDataErrorInfo による入力検証

```vbnet
Public Class Customer
    Implements IDataErrorInfo
    Implements INotifyPropertyChanged

    Private _name As String = String.Empty
    Private _email As String = String.Empty

    Public Property Name As String
        Get
            Return _name
        End Get
        Set(value As String)
            _name = value
            OnPropertyChanged()
        End Set
    End Property

    Public Property Email As String
        Get
            Return _email
        End Get
        Set(value As String)
            _email = value
            OnPropertyChanged()
        End Set
    End Property

    ' IDataErrorInfo の実装
    Public ReadOnly Property [Error] As String Implements IDataErrorInfo.Error
        Get
            Return String.Empty
        End Get
    End Property

    Default Public ReadOnly Property Item(columnName As String) As String Implements IDataErrorInfo.Item
        Get
            Select Case columnName
                Case NameOf(Name)
                    If String.IsNullOrWhiteSpace(Name) Then Return "Name is required"
                Case NameOf(Email)
                    If Not String.IsNullOrEmpty(Email) AndAlso Not Email.Contains("@"c) Then Return "Invalid email"
            End Select
            Return String.Empty
        End Get
    End Property

    Public ReadOnly Property IsValid As Boolean
        Get
            Return String.IsNullOrEmpty(Me(NameOf(Name))) _
               AndAlso String.IsNullOrEmpty(Me(NameOf(Email)))
        End Get
    End Property

    Public Event PropertyChanged As PropertyChangedEventHandler Implements INotifyPropertyChanged.PropertyChanged

    Protected Sub OnPropertyChanged(<CallerMemberName> Optional name As String = Nothing)
        RaiseEvent PropertyChanged(Me, New PropertyChangedEventArgs(name))
    End Sub
End Class
```

## フォーム間通信パターン

### Mediator パターン

複雑なマルチフォーム連携のためのパターン。

```vbnet
Public Interface IFormMediator
    Sub Register(Of TMessage)(handler As Action(Of TMessage))
    Sub Send(Of TMessage)(message As TMessage)
End Interface

Public Class FormMediator
    Implements IFormMediator

    Private ReadOnly _handlers As New Dictionary(Of Type, List(Of System.Delegate))()

    Public Sub Register(Of TMessage)(handler As Action(Of TMessage)) Implements IFormMediator.Register
        Dim type = GetType(TMessage)
        If Not _handlers.ContainsKey(type) Then
            _handlers(type) = New List(Of System.Delegate)()
        End If
        _handlers(type).Add(handler)
    End Sub

    Public Sub Send(Of TMessage)(message As TMessage) Implements IFormMediator.Send
        Dim type = GetType(TMessage)
        Dim handlers As List(Of System.Delegate) = Nothing
        If _handlers.TryGetValue(type, handlers) Then
            For Each handler In handlers.Cast(Of Action(Of TMessage))()
                handler(message)
            Next
        End If
    End Sub
End Class

' メッセージ
' VB.NET に record の直接対応なし。Class + ReadOnly プロパティ + コンストラクタで代替する。
Public Class CustomerSelectedMessage
    Public ReadOnly Property CustomerId As Integer
    Public Sub New(customerId As Integer)
        Me.CustomerId = customerId
    End Sub
End Class

Public Class CustomerUpdatedMessage
    Public ReadOnly Property Customer As Customer
    Public Sub New(customer As Customer)
        Me.Customer = customer
    End Sub
End Class

' 使用例
Public Partial Class CustomerListForm
    Inherits Form

    Private ReadOnly _mediator As IFormMediator

    Public Sub New(mediator As IFormMediator)
        _mediator = mediator

        AddHandler dgvCustomers.SelectionChanged, Sub(s, e)
            Dim c = TryCast(dgvCustomers.CurrentRow?.DataBoundItem, Customer)
            If c IsNot Nothing Then
                _mediator.Send(New CustomerSelectedMessage(c.Id))
            End If
        End Sub
    End Sub
End Class

Public Partial Class CustomerDetailForm
    Inherits Form

    Private ReadOnly _mediator As IFormMediator

    Public Sub New(mediator As IFormMediator)
        _mediator = mediator
        _mediator.Register(Of CustomerSelectedMessage)(Sub(msg) LoadCustomer(msg.CustomerId))
    End Sub
End Class
```

### 親子フォームパターン

```vbnet
Public Partial Class MainForm
    Inherits Form

    Public Sub OpenCustomerEditor(customer As Customer)
        Using editor = New CustomerEditorForm(customer)
            AddHandler editor.CustomerSaved, AddressOf OnCustomerSaved

            If editor.ShowDialog(Me) = DialogResult.OK Then
                RefreshCustomerList()
            End If
        End Using
    End Sub

    Private Sub OnCustomerSaved(sender As Object, e As CustomerSavedEventArgs)
        ' 保存通知を処理する
        statusLabel.Text = $"Customer {e.Customer.Name} saved"
    End Sub
End Class

Public Partial Class CustomerEditorForm
    Inherits Form

    Public Event CustomerSaved As EventHandler(Of CustomerSavedEventArgs)

    Private ReadOnly _customer As Customer

    Public Sub New(customer As Customer)
        InitializeComponent()
        _customer = customer
        BindCustomer()
    End Sub

    Private Sub btnSave_Click(sender As Object, e As EventArgs) Handles btnSave.Click
        If ValidateChildren() Then
            UpdateCustomerFromControls()
            RaiseEvent CustomerSaved(Me, New CustomerSavedEventArgs(_customer))
            DialogResult = DialogResult.OK
            Close()
        End If
    End Sub
End Class

Public Class CustomerSavedEventArgs
    Inherits EventArgs

    Public ReadOnly Property Customer As Customer

    Public Sub New(customer As Customer)
        Me.Customer = customer
    End Sub
End Class
```

### DI によるフォーム解決とフォーム間データ受け渡し

<!-- v2.1.0: 1.0.0 からの WinForms 特化章取り込み -->

`ServiceProvider` を直接参照してフォームを生成すると DI コンテナへの依存が散らばるため、`IServiceProvider` をルートフォームにのみ注入し、子フォームはファクトリーまたは `IServiceProvider.GetRequiredService` で解決する。

```vbnet
' Program.vb — DI 登録の全体像
Module Program
    <STAThread>
    Sub Main()
        ApplicationConfiguration.Initialize()

        Dim services As New ServiceCollection()

        ' サービス登録
        services.AddSingleton(Of ICustomerService, CustomerService)()
        services.AddSingleton(Of IFormMediator, FormMediator)()

        ' フォーム登録: メインフォームは Singleton、子フォームは Transient
        services.AddSingleton(Of MainForm)()
        services.AddTransient(Of CustomerEditorForm)()
        services.AddTransient(Of OrderListForm)()

        Using sp = services.BuildServiceProvider()
            Application.Run(sp.GetRequiredService(Of MainForm)())
        End Using
    End Sub
End Module

' メインフォーム: IServiceProvider を受け取り子フォームを解決する
Public Partial Class MainForm
    Inherits Form

    Private ReadOnly _sp As IServiceProvider

    Public Sub New(sp As IServiceProvider)
        InitializeComponent()
        _sp = sp
    End Sub

    Private Sub btnOpenEditor_Click(sender As Object, e As EventArgs) Handles btnOpenEditor.Click
        ' Transient なので呼ぶたびに新しいインスタンスが作られる
        Using editor = _sp.GetRequiredService(Of CustomerEditorForm)()
            If editor.ShowDialog(Me) = DialogResult.OK Then
                RefreshList()
            End If
        End Using
    End Sub
End Class
```

**子フォームへのデータ受け渡し**: DI で解決したフォームにデータを渡すには、専用のメソッド（`Initialize`）かプロパティで設定する。コンストラクタ引数として渡すと DI の Transient 登録と競合するため避ける。

```vbnet
' 子フォーム: Initialize メソッドでデータを受け取る
Public Partial Class CustomerEditorForm
    Inherits Form

    Private ReadOnly _customerService As ICustomerService
    Private _customer As Customer

    ' DI コンテナからはこのコンストラクタが使われる
    Public Sub New(customerService As ICustomerService)
        InitializeComponent()
        _customerService = customerService
    End Sub

    ' フォームを開く前に呼び出してデータを設定する
    Public Sub Initialize(customer As Customer)
        _customer = customer
        BindFormToCustomer()
    End Sub

    Private Sub BindFormToCustomer()
        If _customer IsNot Nothing Then
            txtName.Text = _customer.Name
            txtEmail.Text = _customer.Email
        End If
    End Sub

    Private Async Sub btnSave_Click(sender As Object, e As EventArgs) Handles btnSave.Click
        If ValidateChildren() Then
            _customer.Name = txtName.Text
            _customer.Email = txtEmail.Text
            Await _customerService.SaveAsync(_customer)
            DialogResult = DialogResult.OK
            Close()
        End If
    End Sub
End Class

' 呼び出し側: 解決 → Initialize → ShowDialog の順
Private Sub OpenEditorForCustomer(customer As Customer)
    Using editor = _sp.GetRequiredService(Of CustomerEditorForm)()
        editor.Initialize(customer)
        If editor.ShowDialog(Me) = DialogResult.OK Then
            RefreshList()
        End If
    End Using
End Sub
```

**モーダルフォームとファクトリーパターン**: フォームの生成ロジックが複雑な場合は `Func(Of TForm)` をファクトリーとして登録すると、`IServiceProvider` をフォームに渡さずに済む。

```vbnet
' DI 登録: Func(Of CustomerEditorForm) をファクトリーとして登録
services.AddTransient(Of CustomerEditorForm)()
services.AddSingleton(Of Func(Of CustomerEditorForm))(
    Function(sp) Function() sp.GetRequiredService(Of CustomerEditorForm)())

' メインフォーム: IServiceProvider の代わりにファクトリーを注入する
Public Partial Class MainForm
    Inherits Form

    Private ReadOnly _editorFactory As Func(Of CustomerEditorForm)

    Public Sub New(editorFactory As Func(Of CustomerEditorForm))
        InitializeComponent()
        _editorFactory = editorFactory
    End Sub

    Private Sub OpenEditor(customer As Customer)
        Using editor = _editorFactory()
            editor.Initialize(customer)
            editor.ShowDialog(Me)
        End Using
    End Sub
End Class
```

## スレッド処理パターン

<!-- v2.1.0: 1.0.0 からの WinForms 特化章取り込み -->

### 安全な UI 更新

Windows Forms は単一 UI スレッドを前提にしている。UI スレッド以外のスレッドから UI コントロールを操作すると `InvalidOperationException`（「Cross-thread operation not valid」）が発生する。

**Async/Await を使っている場合**: フォームのイベントハンドラから `Await` を呼び出すと、WinForms の `SynchronizationContext` が自動的に継続を UI スレッドに戻す。`ConfigureAwait(False)` を付けない限り、`Await` の後は UI スレッド上にある。

```vbnet
' フォームのイベントハンドラ — Await 後も UI スレッド上に戻る
Private Async Sub btnLoad_Click(sender As Object, e As EventArgs) Handles btnLoad.Click
    btnLoad.Enabled = False
    Try
        ' UI SynchronizationContext をキャプチャする
        ' タスク完了後、継続は UI スレッドで実行される
        Dim data = Await _service.LoadAsync()
        lblResult.Text = data.Summary   ' 安全: UI スレッド上
    Finally
        btnLoad.Enabled = True
    End Try
End Sub
```

**Async/Await を使っていない場合**（タイマーコールバック、シリアルポートイベント、バックグラウンドスレッドなど）は、明示的にマーシャリングが必要。

```vbnet
Public Partial Class DataForm
    Inherits Form

    Private _syncContext As SynchronizationContext

    Public Sub New()
        InitializeComponent()
        ' 注意: コンストラクタでは SynchronizationContext.Current を
        ' キャプチャしない。この時点ではハンドルが作成されておらず、
        ' Application.Run が WindowsFormsSynchronizationContext を
        ' インストールする前のため、Current が Nothing または
        ' 誤ったコンテキストになる可能性がある。
        ' OnHandleCreated でキャプチャする。
    End Sub

    Protected Overrides Sub OnHandleCreated(e As EventArgs)
        MyBase.OnHandleCreated(e)
        ' ここでキャプチャするのが安全: UI スレッド上でフォームの
        ' ハンドルが存在し、WinForms の SynchronizationContext がインストール済み
        _syncContext = SynchronizationContext.Current
    End Sub

    Private Async Function ProcessInBackgroundAsync() As Task
        ' バックグラウンド処理を開始する
        Dim data = Await Task.Run(Function() LoadExpensiveData())

        ' ConfigureAwait(False) を付けていないため、
        ' 継続は WinForms コンテキスト（UI スレッド）で実行される
        dgvData.DataSource = data
    End Function

    ' タイマーやシリアルポートなど非 UI スレッドから呼ばれる場合
    Private Sub OnSensorReading(value As Double)
        If lblSensor.InvokeRequired Then
            ' BeginInvoke: 非同期（呼び出し元スレッドをブロックしない）
            ' 高頻度更新（センサーストリーム、ログ）に推奨
            lblSensor.BeginInvoke(Sub() lblSensor.Text = value.ToString("F3"))
        Else
            lblSensor.Text = value.ToString("F3")
        End If
    End Sub

    ' SynchronizationContext 経由（Control 参照を持たないコードから）
    Private Sub StartBackgroundWork()
        Task.Run(Sub()
            Dim result = DoWork()
            ' Post: 非同期（BeginInvoke と同等）
            ' Send: 同期（Invoke と同等、UI スレッドをブロックするため注意）
            _syncContext.Post(Sub(state) lblResult.Text = result, Nothing)
        End Sub)
    End Sub

    ' InvokeRequired による再帰パターン
    Private Sub UpdateStatusSafe(status As String)
        If InvokeRequired Then
            Invoke(Sub() UpdateStatusSafe(status))
            Return
        End If
        lblStatus.Text = status
    End Sub
End Class
```

**使い分けの目安:**

| 手段 | 動作 | 推奨場面 |
|------|------|----------|
| `Control.Invoke` | 同期（UI スレッドの完了を待つ） | 呼び出し元が UI 更新の完了を確認する必要がある場合 |
| `Control.BeginInvoke` | 非同期（呼び出し元をブロックしない） | 高頻度更新（センサー、ログ）など |
| `SynchronizationContext.Post` | 非同期（`BeginInvoke` と同等） | `Control` 参照を持たないライブラリコード |
| `SynchronizationContext.Send` | 同期（`Invoke` と同等） | 同期が必須の場合（デッドロックに注意） |

**ライブラリコードでの `ConfigureAwait(False)`:**

フォームのコードでは `ConfigureAwait(False)` を付けない（UI スレッドに戻る必要があるため）。フォームから呼ばれるクラスライブラリでは、特定の SynchronizationContext に依存しないよう `ConfigureAwait(False)` を付ける。

```vbnet
' ライブラリコード: ConfigureAwait(False) を付けて呼び出し元の
' コンテキストに依存しないようにする
Public Async Function LoadAsync() As Task(Of Data)
    Dim json = Await _http.GetStringAsync(_url).ConfigureAwait(False)
    Return JsonSerializer.Deserialize(Of Data)(json)
End Function
```

### IProgress(Of T) による進捗報告

`IProgress(Of T)` はバックグラウンド処理からフォームへの進捗報告の標準的な手段である。`Progress(Of T)` は UI スレッド上でコンストラクトされると UI SynchronizationContext をキャプチャするため、コールバックは自動的に UI スレッドで実行される。手動の `Invoke` は不要。

```vbnet
' サービス側: IProgress(Of T) を受け取る（ライブラリとして設計）
Public Async Function ProcessFilesAsync(
    files As IReadOnlyList(Of String),
    progress As IProgress(Of Integer),
    ct As CancellationToken) As Task

    For i = 0 To files.Count - 1
        ct.ThrowIfCancellationRequested()
        Await ProcessOneAsync(files(i), ct).ConfigureAwait(False)
        progress?.Report(CInt((i + 1) / files.Count * 100))
    Next
End Function

' フォーム側: UI スレッドで Progress(Of T) を作成する
Private Async Sub btnStart_Click(sender As Object, e As EventArgs) Handles btnStart.Click
    Dim reporter As IProgress(Of Integer) = New Progress(Of Integer)(
        Sub(pct)
            ' UI スレッドで実行される。コントロールに安全にアクセスできる
            progressBar1.Value = pct
            lblPercent.Text = $"{pct}%"
        End Sub)

    Await _service.ProcessFilesAsync(_files, reporter, CancellationToken.None)
End Sub
```

**複合的な進捗状態**（パーセント＋メッセージ＋推定完了時間など）を報告する場合は、スカラー `Integer` の代わりに専用の `Class` または `Structure` パラメータを使う。

```vbnet
Public Class ProgressInfo
    Public ReadOnly Property Percent As Integer
    Public ReadOnly Property Message As String
    Public Sub New(percent As Integer, message As String)
        Me.Percent = percent
        Me.Message = message
    End Sub
End Class

' サービス側
progress?.Report(New ProgressInfo(pct, $"処理中: {fileName}"))

' フォーム側
Dim reporter As IProgress(Of ProgressInfo) = New Progress(Of ProgressInfo)(
    Sub(info)
        progressBar1.Value = info.Percent
        lblStatus.Text = info.Message
    End Sub)
```

**`Application.DoEvents` を使わない:** レガシーコードでよく見られる `Application.DoEvents()` による UI 更新は再帰的なメッセージポンプを引き起こし、バグの原因になる。`Async/Await` + `IProgress(Of T)` + `CancellationToken` の組み合わせに置き換える。

```vbnet
' 悪い例: Application.DoEvents — 再帰的メッセージポンプ、状態が半壊したまま
' ボタンクリックなどが中途半端なタイミングで発火し得る
For i = 0 To 10000
    Process(i)
    Application.DoEvents()
Next

' 良い例: Task.Run + IProgress(Of T) + CancellationToken
Private Async Sub btnProcess_Click(sender As Object, e As EventArgs) Handles btnProcess.Click
    Dim progress As IProgress(Of Integer) = New Progress(Of Integer)(
        Sub(pct) progressBar1.Value = pct)

    Await Task.Run(
        Sub()
            For i = 0 To 10000
                Process(i)
                progress.Report(CInt(i / 10000.0 * 100))
            Next
        End Sub)
End Sub
```

### キャンセルパターン

```vbnet
Public Partial Class LongOperationForm
    Inherits Form

    Private _cts As CancellationTokenSource

    Private Async Sub btnStart_Click(sender As Object, e As EventArgs) Handles btnStart.Click
        _cts = New CancellationTokenSource()
        btnStart.Enabled = False
        btnCancel.Enabled = True

        Try
            Await ProcessDataAsync(_cts.Token)
            MessageBox.Show("Completed")
        Catch ex As OperationCanceledException
            MessageBox.Show("Cancelled")
        Finally
            btnStart.Enabled = True
            btnCancel.Enabled = False
            _cts.Dispose()
            _cts = Nothing
        End Try
    End Sub

    Private Sub btnCancel_Click(sender As Object, e As EventArgs) Handles btnCancel.Click
        _cts?.Cancel()
    End Sub

    Private Async Function ProcessDataAsync(ct As CancellationToken) As Task
        Dim items = Await GetItemsAsync()
        Dim progress = New Progress(Of Integer)(Sub(p) progressBar.Value = p)

        For i As Integer = 0 To items.Count - 1
            ct.ThrowIfCancellationRequested()
            Await ProcessItemAsync(items(i))
            DirectCast(progress, IProgress(Of Integer)).Report((i + 1) * 100 \ items.Count)
        Next
    End Function

    Protected Overrides Sub OnFormClosing(e As FormClosingEventArgs)
        If _cts IsNot Nothing Then
            _cts.Cancel()
            e.Cancel = True ' 操作が停止するまで閉じるのを防ぐ
            ' または: 閉じる前にキャンセルの完了を待機する
        End If
        MyBase.OnFormClosing(e)
    End Sub
End Class
```
