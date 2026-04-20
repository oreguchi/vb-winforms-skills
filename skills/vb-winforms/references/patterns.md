# VB.NET WinForms Patterns Reference

## Architectural Patterns

### MVP (Model-View-Presenter)

MVP is the recommended pattern for VB.NET WinForms applications that need testability and separation of concerns.

**Structure:**
- **Model**: Domain entities and business logic
- **View**: Form implementing an interface, handles UI concerns only
- **Presenter**: Mediates between Model and View, contains presentation logic

**Key Characteristics:**
- View is passive and raises events
- Presenter subscribes to view events and updates view properties
- Presenter can be unit tested without UI
- View interface enables mocking

```vb
' View contract
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

        AddHandler _view.LoadRequested, AddressOf OnLoadRequested
        AddHandler _view.SaveRequested, AddressOf OnSaveRequested
        AddHandler _view.CancelRequested, Sub(s, e) _view.Close()
    End Sub

    Private Async Sub OnLoadRequested(sender As Object, e As EventArgs)
        Await LoadOrderAsync()
    End Sub

    Private Async Sub OnSaveRequested(sender As Object, e As EventArgs)
        Await SaveOrderAsync()
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

**VB.NET interface implementation note:** VB.NET requires an explicit `Implements` clause on each member. The form that implements `IOrderView` must declare each property, event, and method with its `Implements` clause:

```vb
Public Class OrderForm
    Implements IOrderView

    Public Property OrderId As Integer Implements IOrderView.OrderId
    Public Property CustomerName As String Implements IOrderView.CustomerName
    Public Property Total As Decimal Implements IOrderView.Total

    Public WriteOnly Property Lines As IEnumerable(Of OrderLine) Implements IOrderView.Lines
        Set(value As IEnumerable(Of OrderLine))
            ' populate grid
        End Set
    End Property

    Public Event LoadRequested As EventHandler Implements IOrderView.LoadRequested
    Public Event SaveRequested As EventHandler Implements IOrderView.SaveRequested
    Public Event CancelRequested As EventHandler Implements IOrderView.CancelRequested

    Public Sub ShowValidationError(field As String, message As String) Implements IOrderView.ShowValidationError
        ' set ErrorProvider on the relevant control
    End Sub

    Public Sub ClearValidationErrors() Implements IOrderView.ClearValidationErrors
        ' clear ErrorProvider
    End Sub

    Private Sub btnLoad_Click(sender As Object, e As EventArgs) Handles btnLoad.Click
        RaiseEvent LoadRequested(Me, EventArgs.Empty)
    End Sub

    Private Sub btnSave_Click(sender As Object, e As EventArgs) Handles btnSave.Click
        RaiseEvent SaveRequested(Me, EventArgs.Empty)
    End Sub

    Private Sub btnCancel_Click(sender As Object, e As EventArgs) Handles btnCancel.Click
        RaiseEvent CancelRequested(Me, EventArgs.Empty)
    End Sub
End Class
```

### MVVM (Model-View-ViewModel)

MVVM can be used in VB.NET WinForms with data binding, though MVP is generally preferred for WinForms. Use when:
- Heavy data binding requirements
- Sharing ViewModels between WinForms and WPF
- Team is familiar with MVVM from other frameworks

```vb
Imports System.ComponentModel
Imports System.Runtime.CompilerServices
Imports System.Windows.Input

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

A stricter variant of MVP where the view contains zero logic:
- All decisions made by presenter
- View only exposes properties and events
- Maximum testability, minimum view code

```vb
' Passive view — no logic at all
Public Class CustomerForm
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

' Presenter controls everything
Public Class CustomerPresenter
    Private ReadOnly _view As ICustomerView

    Public Sub New(view As ICustomerView)
        _view = view
        _view.SaveEnabled = False

        AddHandler _view.FirstNameChanged, Sub(s, e) UpdateSaveEnabled()
        AddHandler _view.LastNameChanged, Sub(s, e) UpdateSaveEnabled()
    End Sub

    Private Sub UpdateSaveEnabled()
        _view.SaveEnabled = Not String.IsNullOrWhiteSpace(_view.FirstName) AndAlso
                            Not String.IsNullOrWhiteSpace(_view.LastName)
    End Sub
End Class
```

## Data Binding Patterns

### Master-Detail Binding

Common pattern for list-detail UIs:

```vb
Public Class MasterDetailForm
    Inherits Form

    ' _orderService is assumed to be injected via the constructor or a DI container.
    Private ReadOnly _orderService As IOrderService
    Private ReadOnly _masterSource As New BindingSource()
    Private ReadOnly _detailSource As New BindingSource()

    Public Sub New(orderService As IOrderService)
        _orderService = orderService
        InitializeComponent()

        ' Link detail to master
        _detailSource.DataSource = _masterSource
        _detailSource.DataMember = "OrderLines" ' Navigation property

        dgvOrders.DataSource = _masterSource
        dgvOrderLines.DataSource = _detailSource

        ' Detail controls bind to detail source
        txtLineDescription.DataBindings.Add("Text", _detailSource, "Description")
        txtLineQuantity.DataBindings.Add("Text", _detailSource, "Quantity")
    End Sub

    Private Async Function LoadAsync() As Task
        Dim orders = Await _orderService.GetAllWithLinesAsync()
        _masterSource.DataSource = New BindingList(Of Order)(orders.ToList())
    End Function
End Class
```

### Two-Way Binding with Validation

```vb
Public Class EditForm
    Inherits Form

    Private ReadOnly _bindingSource As New BindingSource()
    Private ReadOnly _errorProvider As New ErrorProvider()

    Private Sub SetupBindings(customer As Customer)
        _bindingSource.DataSource = customer

        ' Two-way binding with format and parse.
        ' Note: e.Value is Object. The ?. chain returns Nothing when e.Value is Nothing,
        ' otherwise returns a trimmed String. Assigning String (or Nothing) back to Object
        ' is a widening conversion, so this is safe under Option Strict On.
        Dim nameBinding As New Binding("Text", _bindingSource, "Name", True)
        AddHandler nameBinding.Format,
            Sub(s, e)
                Dim trimmed As String = e.Value?.ToString()?.Trim()
                e.Value = CType(trimmed, Object)
            End Sub
        AddHandler nameBinding.Parse,
            Sub(s, e)
                Dim trimmed As String = e.Value?.ToString()?.Trim()
                e.Value = CType(trimmed, Object)
            End Sub
        txtName.DataBindings.Add(nameBinding)

        ' Binding with null handling
        txtEmail.DataBindings.Add("Text", _bindingSource, "Email",
            True, DataSourceUpdateMode.OnPropertyChanged, String.Empty)

        ' Checkbox binding
        chkActive.DataBindings.Add("Checked", _bindingSource, "IsActive",
            True, DataSourceUpdateMode.OnPropertyChanged)

        ' ComboBox binding
        cboCategory.DataSource = _categories
        cboCategory.DisplayMember = "Name"
        cboCategory.ValueMember = "Id"
        cboCategory.DataBindings.Add("SelectedValue", _bindingSource, "CategoryId")
    End Sub
End Class
```

### Observable Collection Pattern

```vb
Public Class ObservableList(Of T)
    Inherits BindingList(Of T)

    Public Sub AddRange(items As IEnumerable(Of T))
        Dim oldRaise = MyBase.RaiseListChangedEvents
        MyBase.RaiseListChangedEvents = False
        Try
            For Each item In items
                Add(item)
            Next
        Finally
            MyBase.RaiseListChangedEvents = oldRaise
        End Try
        MyBase.ResetBindings()
    End Sub
End Class
```

`BindingList(Of T)` already exposes a `RaiseListChangedEvents` property; toggle it in a `Try/Finally` to suppress notifications during bulk inserts, then call `ResetBindings()` once. Reinventing this with a custom flag and an `OnListChanged` override is unnecessary.

## Validation Patterns

### Centralized Validation

```vb
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
                                           Dim err = validator()
                                           _errorProvider.SetError(control, If(err, String.Empty))
                                           If Not String.IsNullOrEmpty(err) Then
                                               e.Cancel = True
                                           End If
                                       End Sub
    End Sub

    Public Function ValidateAll() As Boolean
        Dim isValid = True
        For Each kvp In _validators
            Dim err = kvp.Value()
            _errorProvider.SetError(kvp.Key, If(err, String.Empty))
            If Not String.IsNullOrEmpty(err) Then
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

' Usage
Public Class CustomerForm
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

### IDataErrorInfo Validation

VB.NET supports the `IDataErrorInfo` interface for model-level validation tied to data binding. Note: the `switch` expression used in the C# original is not available in VB.NET; use `Select Case` or `If`/`ElseIf` chains instead.

```vb
Imports System.ComponentModel
Imports System.Runtime.CompilerServices

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

    ' IDataErrorInfo implementation
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
            Return String.IsNullOrEmpty(Me(NameOf(Name))) AndAlso
                   String.IsNullOrEmpty(Me(NameOf(Email)))
        End Get
    End Property

    Public Event PropertyChanged As PropertyChangedEventHandler Implements INotifyPropertyChanged.PropertyChanged

    Protected Sub OnPropertyChanged(<CallerMemberName> Optional name As String = Nothing)
        RaiseEvent PropertyChanged(Me, New PropertyChangedEventArgs(name))
    End Sub
End Class
```

## Form Communication Patterns

### Mediator Pattern

For complex multi-form coordination:

```vb
Public Interface IFormMediator
    Sub Register(Of TMessage)(handler As Action(Of TMessage))
    Sub Send(Of TMessage)(message As TMessage)
End Interface

Public Class FormMediator
    Implements IFormMediator

    Private ReadOnly _handlers As New Dictionary(Of Type, List(Of [Delegate]))()

    Public Sub Register(Of TMessage)(handler As Action(Of TMessage)) Implements IFormMediator.Register
        Dim t = GetType(TMessage)
        If Not _handlers.ContainsKey(t) Then
            _handlers(t) = New List(Of [Delegate])()
        End If
        _handlers(t).Add(handler)
    End Sub

    Public Sub Send(Of TMessage)(message As TMessage) Implements IFormMediator.Send
        Dim t = GetType(TMessage)
        Dim handlers As List(Of [Delegate]) = Nothing
        If _handlers.TryGetValue(t, handlers) Then
            For Each handler In handlers.Cast(Of Action(Of TMessage))()
                handler(message)
            Next
        End If
    End Sub
End Class

' Messages — VB.NET does not have C# records; use a Class with ReadOnly properties
' set in the constructor. Value equality must be implemented manually if needed.
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

' Usage
Public Class CustomerListForm
    Inherits Form

    Private ReadOnly _mediator As IFormMediator

    Public Sub New(mediator As IFormMediator)
        InitializeComponent()
        _mediator = mediator

        AddHandler dgvCustomers.SelectionChanged, Sub(s, e)
                                                      Dim c = TryCast(dgvCustomers.CurrentRow?.DataBoundItem, Customer)
                                                      If c IsNot Nothing Then
                                                          _mediator.Send(New CustomerSelectedMessage(c.Id))
                                                      End If
                                                  End Sub
    End Sub
End Class

Public Class CustomerDetailForm
    Inherits Form

    Private ReadOnly _mediator As IFormMediator

    Public Sub New(mediator As IFormMediator)
        InitializeComponent()
        _mediator = mediator
        _mediator.Register(Of CustomerSelectedMessage)(Sub(msg) LoadCustomer(msg.CustomerId))
    End Sub
End Class
```

**Note on records:** The C# original used `record` types (`CustomerSelectedMessage`, `CustomerUpdatedMessage`). VB.NET does not have records. Use a `Class` with `ReadOnly Property` members initialized in the constructor. Value equality (`Equals`, `GetHashCode`) must be implemented manually when needed.

### Parent-Child Form Pattern

```vb
Public Class MainForm
    Inherits Form

    Public Sub OpenCustomerEditor(customer As Customer)
        Using editor As New CustomerEditorForm(customer)
            AddHandler editor.CustomerSaved, AddressOf OnCustomerSaved

            If editor.ShowDialog(Me) = DialogResult.OK Then
                RefreshCustomerList()
            End If
        End Using
    End Sub

    Private Sub OnCustomerSaved(sender As Object, e As CustomerSavedEventArgs)
        statusLabel.Text = $"Customer {e.Customer.Name} saved"
    End Sub
End Class

Public Class CustomerEditorForm
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

    ' BindCustomer and UpdateCustomerFromControls are illustrative stubs.
    ' In a real form, bind controls to _customer (e.g., via DataBindings) and
    ' copy edited values back before saving.
    Private Sub BindCustomer()
        Throw New NotImplementedException()
    End Sub

    Private Sub UpdateCustomerFromControls()
        Throw New NotImplementedException()
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

## Threading Patterns

### Safe UI Updates

```vb
Public Class DataForm
    Inherits Form

    Private _syncContext As SynchronizationContext

    Public Sub New()
        InitializeComponent()
        ' NOTE: Do not capture SynchronizationContext.Current here. The handle
        ' has not been created yet and Application.Run has not installed the
        ' WindowsFormsSynchronizationContext, so Current may be Nothing or
        ' the wrong context. Capture it in OnHandleCreated instead.
    End Sub

    Protected Overrides Sub OnHandleCreated(e As EventArgs)
        MyBase.OnHandleCreated(e)
        ' Safe to capture now: the WinForms SynchronizationContext is installed
        ' and the form's handle exists on the UI thread.
        _syncContext = SynchronizationContext.Current
    End Sub

    Private Async Function ProcessInBackgroundAsync() As Task
        ' Start background work
        Dim data = Await Task.Run(Function() LoadExpensiveData())

        ' Already on UI thread when the continuation resumes, because
        ' ConfigureAwait(False) is not applied and the Await started on the
        ' UI thread (WinForms SynchronizationContext captures it).
        dgvData.DataSource = data
    End Function

    ' For fire-and-forget or manual threading
    Private Sub StartBackgroundWork()
        Task.Run(Sub()
                     Dim result = DoWork()

                     ' Post back to UI thread
                     _syncContext.Post(Sub(_)
                                           lblResult.Text = result
                                       End Sub, Nothing)
                 End Sub)
    End Sub

    ' Extension method approach
    Private Sub UpdateStatusSafe(status As String)
        If InvokeRequired Then
            Invoke(Sub() UpdateStatusSafe(status))
            Return
        End If
        lblStatus.Text = status
    End Sub
End Class
```

### Cancellation Pattern

```vb
Public Class LongOperationForm
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
        Dim progress As New Progress(Of Integer)(Sub(p) progressBar.Value = p)

        For i As Integer = 0 To items.Count - 1
            ct.ThrowIfCancellationRequested()
            Await ProcessItemAsync(items(i))
            CType(progress, IProgress(Of Integer)).Report((i + 1) * 100 \ items.Count)
        Next
    End Function

    Protected Overrides Sub OnFormClosing(e As FormClosingEventArgs)
        If _cts IsNot Nothing Then
            _cts.Cancel()
            e.Cancel = True ' Prevent close until operation stops
            ' After _cts.Cancel() the task will wind down asynchronously.
            ' Close is cancelled this once; the user re-clicks close when the
            ' operation has finished (or track the running Task and call
            ' Me.Close() from its continuation to close programmatically).
        End If
        MyBase.OnFormClosing(e)
    End Sub
End Class
```
