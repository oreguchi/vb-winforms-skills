# VB.NET WinForms Migration to Modern .NET

## Migration Overview

Migrating VB.NET Windows Forms applications from .NET Framework to modern .NET (6, 7, 8, 9, 10) provides:
- Better performance and memory efficiency
- Access to modern VB.NET language features
- Side-by-side deployment without system-wide runtime
- Continued support and security updates
- Access to new WinForms features

## Prerequisites Assessment

### Compatibility Analysis

Before migrating, analyze your application for compatibility:

```bash
# Install the .NET Upgrade Assistant
dotnet tool install -g upgrade-assistant

# Analyze project
upgrade-assistant analyze MyWinFormsApp.vbproj

# Or run interactive upgrade
upgrade-assistant upgrade MyWinFormsApp.vbproj
```

### Common Blockers

| Blocker | Impact | Mitigation |
|---------|--------|------------|
| WCF Client | Requires change | Use CoreWCF or gRPC |
| WCF Server | Not supported | Migrate to ASP.NET Core + gRPC |
| AppDomain | Limited support | Redesign with AssemblyLoadContext |
| Remoting | Not supported | Use gRPC or REST APIs |
| Code Access Security | Not supported | Remove or redesign |
| Windows Workflow Foundation | Not supported | Use Elsa or other workflow engine |
| Crystal Reports | May not work | Test or use alternative |

### Check for Deprecated APIs

```vb
' These patterns indicate potential issues:

' App.config usage — may need migration
ConfigurationManager.AppSettings("MySetting")

' System.Web references — not available in modern .NET
System.Web.HttpUtility.UrlEncode(value)

' Drawing.Common differences on non-Windows
System.Drawing.Image.FromFile(path)
```

## Project File Migration

### Before (.NET Framework)

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
    <!-- Many more references -->
  </ItemGroup>
  <ItemGroup>
    <Compile Include="Form1.vb">
      <SubType>Form</SubType>
    </Compile>
    <Compile Include="Form1.Designer.vb">
      <DependentUpon>Form1.vb</DependentUpon>
    </Compile>
    <!-- Many more compile items -->
  </ItemGroup>
  <Import Project="$(MSBuildToolsPath)\Microsoft.VisualBasic.targets" />
</Project>
```

### After (Modern .NET SDK-Style)

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net9.0-windows</TargetFramework>
    <UseWindowsForms>true</UseWindowsForms>
    <ApplicationManifest>app.manifest</ApplicationManifest>
    <!-- Note: <ImplicitUsings> is C#-only and has no VB.NET equivalent.
         For project-level namespace imports in VB.NET, use <Import Include="Namespace.Name"/>
         inside an <ItemGroup> (shown below). This is not the same feature as C# implicit usings,
         but serves the analogous role of avoiding per-file Imports statements. -->
  </PropertyGroup>

  <ItemGroup>
    <!-- Project-level imports (VB.NET equivalent of per-project namespace defaults) -->
    <Import Include="System" />
    <Import Include="System.Collections.Generic" />
    <Import Include="System.Linq" />
    <Import Include="System.Windows.Forms" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="9.0.0" />
    <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="9.0.0" />
  </ItemGroup>
</Project>
```

**Note:** The `<Nullable>enable</Nullable>` property from the C# SDK-style template is a C#-specific nullable reference types (NRT) feature. VB.NET has its own nullable value type support (`Integer?`, `Nullable(Of T)`) but does not use the NRT compiler annotation system. Omit `<Nullable>enable</Nullable>` from VB.NET project files.

## Step-by-Step Migration

### Step 1: Create New Project

```bash
# Create new VB.NET WinForms project
dotnet new winforms -lang VB -n MyWinFormsApp.Modern -f net9.0

# Or use specific template features
dotnet new winforms -lang VB -n MyWinFormsApp.Modern --no-restore
```

### Step 2: Copy Source Files

Copy these files from the old project:
- All `.vb` files (forms, classes, controls)
- All `.resx` files (resources)
- All `.Designer.vb` files
- Assets (images, icons, etc.)

### Step 3: Update Program.vb

```vb
' .NET Framework style
Module Program
    <STAThread>
    Sub Main()
        Application.EnableVisualStyles()
        Application.SetCompatibleTextRenderingDefault(False)
        Application.Run(New MainForm())
    End Sub
End Module

' Modern .NET style
Module Program
    <STAThread>
    Sub Main()
        ApplicationConfiguration.Initialize()
        Application.Run(New MainForm())
    End Sub
End Module

' Modern .NET with DI
Imports Microsoft.Extensions.DependencyInjection
Imports Microsoft.Extensions.Hosting

Module Program
    <STAThread>
    Sub Main()
        ApplicationConfiguration.Initialize()

        Dim host = Host.CreateDefaultBuilder() _
            .ConfigureServices(Sub(context, services)
                                   services.AddTransient(Of MainForm)()
                                   services.AddTransient(Of ICustomerService, CustomerService)()
                               End Sub) _
            .Build()

        Dim mainForm = host.Services.GetRequiredService(Of MainForm)()
        Application.Run(mainForm)
    End Sub
End Module
```

### Step 4: Update Configuration

Replace `app.config` with `appsettings.json`:

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

```vb
' Reading configuration
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

### Step 5: Update NuGet References

Replace packages.config with PackageReference:

```xml
<!-- Old packages.config style -->
<packages>
  <package id="Newtonsoft.Json" version="13.0.1" targetFramework="net48" />
  <package id="Dapper" version="2.0.123" targetFramework="net48" />
</packages>

<!-- New PackageReference style in .vbproj -->
<ItemGroup>
  <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
  <PackageReference Include="Dapper" Version="2.1.35" />
</ItemGroup>
```

### Step 6: Handle API Differences

```vb
' BinaryFormatter — no longer recommended, use alternatives
' Old
Dim formatter As New BinaryFormatter()
formatter.Serialize(stream, obj)

' New — use System.Text.Json or other serializers
Dim json = JsonSerializer.Serialize(obj)
Await File.WriteAllTextAsync(path, json)

' System.Drawing differences
' Old — worked everywhere
Using bitmap As New Bitmap(path)
    ' ...
End Using

' New — Windows-only by default, use SkiaSharp for cross-platform
' Or add package reference:
' <PackageReference Include="System.Drawing.Common" Version="8.0.0" />
```

### Step 7: Update Assembly Info

Remove `AssemblyInfo.vb` and use project properties:

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

## Common Migration Issues

### Designer Issues

```vb
' Issue: Designer fails to load after migration
' Solution: Ensure all dependencies are available and rebuild

' Issue: User controls not showing in toolbox
' Solution: Build solution, then refresh toolbox

' Issue: Resources not loading
' Solution: Ensure .resx files have correct build action
```

```xml
<!-- Ensure resources are embedded -->
<ItemGroup>
  <EmbeddedResource Update="Form1.resx">
    <DependentUpon>Form1.vb</DependentUpon>
  </EmbeddedResource>
</ItemGroup>
```

### Third-Party Controls

```vb
' Check compatibility before migration
' Many vendors provide .NET 6+ compatible versions

' DevExpress, Telerik, Infragistics, etc. — check vendor documentation
' Older/abandoned controls — may need replacement

' If control source is available, consider migrating it too
' Or replace with:
' - Built-in .NET controls
' - Open-source alternatives (be mindful of licensing)
' - Custom implementations
```

### Database Access

```vb
' Entity Framework 6 to EF Core
' Old (EF6)
Using context As New MyDbContext()
    Dim customers = context.Customers.Where(Function(c) c.IsActive).ToList()
End Using

' New (EF Core) — VB does not support Await Using; dispose via Try/Finally
Dim context As New MyDbContext()
Try
    Dim customers = Await context.Customers _
        .Where(Function(c) c.IsActive) _
        .ToListAsync()
Finally
    Await context.DisposeAsync()
End Try
```

### WCF Client Migration

```vb
' Option 1: Use System.ServiceModel packages
' <PackageReference Include="System.ServiceModel.Http" Version="6.0.0" />

' Option 2: Generate new client
' dotnet-svcutil https://service.example.com/MyService?wsdl

' Option 3: Replace with HTTP client for REST services
Public Class MyServiceClient
    Private ReadOnly _client As HttpClient

    Public Async Function GetCustomerAsync(id As Integer) As Task(Of Customer)
        Dim response = Await _client.GetAsync($"api/customers/{id}")
        response.EnsureSuccessStatusCode()
        Return Await response.Content.ReadFromJsonAsync(Of Customer)()
    End Function
End Class
```

## High-DPI and Modern Features

### Enable High-DPI Support

```vb
' In Program.vb — ApplicationConfiguration.Initialize() already applies high-DPI settings.
' Prefer configuring DPI via <ApplicationHighDpiMode> in the .vbproj instead of
' calling Application.SetHighDpiMode explicitly.
ApplicationConfiguration.Initialize()
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

### Use New .NET Features

```vb
' Button commands (.NET 9+)
btnSave.Command = New RelayCommand(AddressOf Save, AddressOf CanSave)

' System icons (.NET 7+)
Dim icon = SystemIcons.GetStockIcon(StockIconId.Info)

' Improved data binding (.NET 9+)
' Better performance and memory usage

' FolderBrowserDialog improvements (.NET 5+)
Using dialog As New FolderBrowserDialog With {
    .InitialDirectory = Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments),
    .ShowNewFolderButton = True,
    .UseDescriptionForTitle = True,
    .Description = "Select output folder"
}
    ' ...
End Using
```

## Testing After Migration

### Functional Testing Checklist

- [ ] Application launches without errors
- [ ] All forms open correctly
- [ ] Designer loads all forms
- [ ] Data binding works correctly
- [ ] Validation behaves as expected
- [ ] Database operations work
- [ ] File operations work
- [ ] Printing works (if applicable)
- [ ] Third-party controls function
- [ ] Resources (images, icons) load
- [ ] Localization works (if applicable)
- [ ] High-DPI displays correctly
- [ ] Keyboard shortcuts work
- [ ] Tab order is correct

### Performance Testing

```vb
' Basic startup timing
Dim sw = Stopwatch.StartNew()
Application.Run(New MainForm())
Console.WriteLine($"Startup: {sw.ElapsedMilliseconds}ms")

' Memory usage comparison
' Use dotnet-counters or Visual Studio diagnostics
' dotnet-counters monitor --process-id <PID>
```

## Deployment

### Framework-Dependent Deployment

```bash
# Requires .NET runtime on target machine
dotnet publish -c Release -r win-x64 --self-contained false
```

### Self-Contained Deployment

```bash
# Includes runtime, larger but no dependencies
dotnet publish -c Release -r win-x64 --self-contained true

# Single file (recommended for distribution)
dotnet publish -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true

# Trimmed (smaller size, test thoroughly)
dotnet publish -c Release -r win-x64 --self-contained true -p:PublishTrimmed=true
```

```xml
<!-- Project settings for publishing -->
<PropertyGroup>
  <RuntimeIdentifier>win-x64</RuntimeIdentifier>
  <SelfContained>true</SelfContained>
  <PublishSingleFile>true</PublishSingleFile>
  <IncludeNativeLibrariesForSelfExtract>true</IncludeNativeLibrariesForSelfExtract>
  <EnableCompressionInSingleFile>true</EnableCompressionInSingleFile>
</PropertyGroup>
```

## Gradual Migration Strategy

For large applications, consider incremental migration:

### 1. Shared Library Approach

```text
Solution/
├── MyApp.Core/                 # .NET Standard 2.0 — shared
│   ├── Models/
│   ├── Services/
│   └── Interfaces/
├── MyApp.WinForms.Legacy/      # .NET Framework 4.8 — old UI
│   └── References MyApp.Core
├── MyApp.WinForms.Modern/      # .NET 9 — new UI
│   └── References MyApp.Core
```

### 2. Feature-by-Feature Migration

1. Migrate shared business logic to .NET Standard
2. Create new modern .NET VB.NET WinForms project
3. Migrate forms one at a time
4. Test each migrated form thoroughly
5. Retire old project when complete

### 3. Side-by-Side Development

```xml
<!-- Multi-targeting for shared code -->
<PropertyGroup>
  <TargetFrameworks>net48;net9.0-windows</TargetFrameworks>
</PropertyGroup>
```

```vb
' Conditional compilation when needed
#If NET48 Then
    ' .NET Framework specific code
#Else
    ' Modern .NET code
#End If
```

## Resources

- [Official Migration Guide](https://learn.microsoft.com/en-us/dotnet/desktop/winforms/migration/)
- [.NET Upgrade Assistant](https://learn.microsoft.com/en-us/dotnet/core/porting/upgrade-assistant-overview)
- [Breaking Changes](https://learn.microsoft.com/en-us/dotnet/core/compatibility/winforms)
- [What's New in Windows Forms](https://learn.microsoft.com/en-us/dotnet/desktop/winforms/whats-new/)
