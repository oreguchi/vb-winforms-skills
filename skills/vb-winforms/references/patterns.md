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

## スレッド処理パターン

### 安全な UI 更新

```vbnet
Public Partial Class DataForm
    Inherits Form

    Private ReadOnly _syncContext As SynchronizationContext

    Public Sub New()
        InitializeComponent()
        _syncContext = SynchronizationContext.Current
    End Sub

    Private Async Function ProcessInBackgroundAsync() As Task
        ' バックグラウンド処理を開始する
        Dim data = Await Task.Run(Function() LoadExpensiveData())

        ' await により WinForms コンテキストでは既に UI スレッド上にある
        dgvData.DataSource = data
    End Function

    ' ファイアアンドフォーゲットまたは手動スレッドの場合
    Private Sub StartBackgroundWork()
        Task.Run(Sub()
            Dim result = DoWork()

            ' UI スレッドに戻す
            _syncContext.Post(Sub(state) lblResult.Text = result, Nothing)
        End Sub)
    End Sub

    ' InvokeRequired アプローチ
    Private Sub UpdateStatusSafe(status As String)
        If InvokeRequired Then
            Invoke(Sub() UpdateStatusSafe(status))
            Return
        End If
        lblStatus.Text = status
    End Sub
End Class
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
