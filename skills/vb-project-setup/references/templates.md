# Common VB.NET Project Templates

## Overview

.NET provides project templates via `dotnet new`. This reference covers templates that support VB.NET and their typical use cases. Most VB.NET project templates accept `-lang VB` (for example, `console`, `classlib`, `winforms`, `wpf`, `xunit`, `nunit`, `mstest`). Language-agnostic templates (`sln`, `gitignore`, `editorconfig`, `globaljson`, `tool-manifest`) do not take a language flag — they produce the same output regardless of project language.

---

## Listing Available Templates

```bash
# List all installed templates
dotnet new list

# Filter for VB.NET-supported templates
dotnet new list --language VB

# Search for templates
dotnet new search classlib

# Install a template pack
dotnet new install <TemplatePack>
```

---

## Console Applications

### Basic Console App

```bash
dotnet new console -lang VB -n MyApp -o src/MyApp
```

**Generated structure:**

```text
src/MyApp/
├── MyApp.vbproj
└── Program.vb
```

**Typical `Program.vb`:**

```vb
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

**Note:** VB.NET does not support top-level statements (a C# 9+ feature). Use `Module Program` with `Sub Main()` instead.

---

## Class Libraries

### Standard Library

```bash
dotnet new classlib -lang VB -n MyProduct.Core -o src/MyProduct.Core
```

### Library with Multi-Targeting

**Modify the generated `.vbproj`:**

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFrameworks>net9.0;net8.0;netstandard2.0</TargetFrameworks>
    <LangVersion>latest</LangVersion>
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
    <PackageId>MyCompany.MyProduct.Core</PackageId>
    <Description>Core library for MyProduct</Description>
  </PropertyGroup>
</Project>
```

---

## Windows Forms Applications

### WinForms App

```bash
dotnet new winforms -lang VB -n MyProduct.App -o src/MyProduct.App
```

**Generated structure:**

```text
src/MyProduct.App/
├── MyProduct.App.vbproj
├── Form1.vb
├── Form1.Designer.vb
└── Program.vb
```

**Typical `.vbproj`:**

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net9.0-windows</TargetFramework>
    <UseWindowsForms>true</UseWindowsForms>
    <LangVersion>latest</LangVersion>
    <RootNamespace>MyProduct.App</RootNamespace>
  </PropertyGroup>
</Project>
```

---

## WPF Applications

### WPF App

```bash
dotnet new wpf -lang VB -n MyProduct.WpfApp -o src/MyProduct.WpfApp
```

**Note:** VB.NET supports WPF project creation via `dotnet new wpf -lang VB`, but XAML-based WPF development is primarily a C# ecosystem. VB.NET WPF projects generate a `.vbproj` with `<UseWPF>true</UseWPF>`. XAML files themselves are language-agnostic. Code-behind files use `.xaml.vb` extensions and standard VB.NET syntax.

---

## Test Projects

### xUnit Test Project

```bash
dotnet new xunit -lang VB -n MyProduct.Core.Tests -o tests/MyProduct.Core.Tests
```

**Test project `.vbproj`:**

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <LangVersion>latest</LangVersion>
    <IsPackable>false</IsPackable>
    <IsTestProject>true</IsTestProject>
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

**Test class template:**

```vb
Imports FluentAssertions
Imports Moq
Imports Xunit

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
            _repositoryMock.Setup(Function(r) r.GetByIdAsync(itemId, It.IsAny(Of CancellationToken)())).ReturnsAsync(expected)

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
            Dim act = Async Function() Await _sut.CreateAsync(request, CancellationToken.None)

            ' Assert
            Await act.Should().ThrowAsync(Of ArgumentException)()
        End Function

    End Class

End Namespace
```

### NUnit Test Project

```bash
dotnet new nunit -lang VB -n MyProduct.Core.Tests -o tests/MyProduct.Core.Tests
```

### MSTest Test Project

```bash
dotnet new mstest -lang VB -n MyProduct.Core.Tests -o tests/MyProduct.Core.Tests
```

---

## Solution Setup Commands

### Create Solution and VB.NET Projects

```bash
# Create solution
dotnet new sln -n MyProduct

# Create projects
dotnet new classlib -lang VB -n MyProduct.Core -o src/MyProduct.Core
dotnet new classlib -lang VB -n MyProduct.Infrastructure -o src/MyProduct.Infrastructure
dotnet new winforms -lang VB -n MyProduct.App -o src/MyProduct.App
dotnet new xunit -lang VB -n MyProduct.Core.Tests -o tests/MyProduct.Core.Tests

# Add projects to solution
dotnet sln add src/MyProduct.Core/MyProduct.Core.vbproj
dotnet sln add src/MyProduct.Infrastructure/MyProduct.Infrastructure.vbproj
dotnet sln add src/MyProduct.App/MyProduct.App.vbproj
dotnet sln add tests/MyProduct.Core.Tests/MyProduct.Core.Tests.vbproj

# Add project references
dotnet add src/MyProduct.Infrastructure/MyProduct.Infrastructure.vbproj reference src/MyProduct.Core/MyProduct.Core.vbproj
dotnet add src/MyProduct.App/MyProduct.App.vbproj reference src/MyProduct.Core/MyProduct.Core.vbproj
dotnet add src/MyProduct.App/MyProduct.App.vbproj reference src/MyProduct.Infrastructure/MyProduct.Infrastructure.vbproj
dotnet add tests/MyProduct.Core.Tests/MyProduct.Core.Tests.vbproj reference src/MyProduct.Core/MyProduct.Core.vbproj
```

---

## Tool Manifest

### Create Tool Manifest

```bash
dotnet new tool-manifest
```

**Creates `.config/dotnet-tools.json`:**

```json
{
  "version": 1,
  "isRoot": true,
  "tools": {}
}
```

### Install Local Tools

```bash
dotnet tool install dotnet-ef
dotnet tool install dotnet-format
dotnet tool install dotnet-reportgenerator-globaltool
```

**Updated manifest:**

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

### Restore Tools

```bash
dotnet tool restore
```

---

## Templates Not Available in VB.NET

The following templates do **not** support `-lang VB` and should not be used for VB.NET projects:

| Template | Reason |
|----------|---------|
| `blazor`, `blazorserver`, `blazorwasm` | Blazor requires C# for component code-behind; no VB.NET support |
| `maui`, `maui-blazor` | .NET MAUI targets are C# and XAML only; no VB.NET support |
| `webapp` (Razor Pages) | Razor Pages and MVC use C#-specific code generation; no VB.NET support |
| `mvc` | Same as above |
| `grpc` | gRPC service templates are C#-only |
| `worker` | The Worker Service template (`dotnet new worker`) does not support `-lang VB`; implement background services in VB.NET by referencing `Microsoft.Extensions.Hosting` and inheriting from `BackgroundService` manually |
| `aspire`, `aspire-*` | .NET Aspire templates are C# only |
| `webapi` | `dotnet new webapi` is C#/F#-only and does **not** support `-lang VB`. If you need VB for web work, create a `classlib` with `-lang VB` and host it manually (e.g., reference `Microsoft.AspNetCore.App` from a C# host project or wire up `WebApplication` in C# and consume VB class libraries) |
| ML.NET templates | All ML.NET scaffolding templates are C# only |

For background services in VB.NET, create a `classlib` or `console` project and inherit from `BackgroundService` directly:

```vb
Imports Microsoft.Extensions.Hosting

Public Class MyWorker
    Inherits BackgroundService

    Protected Overrides Async Function ExecuteAsync(stoppingToken As CancellationToken) As Task
        While Not stoppingToken.IsCancellationRequested
            ' Do work here
            Await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken)
        End While
    End Function

End Class
```

---

## Quick Reference: VB.NET-Supported Templates

| Template | Command | Key Options |
|----------|---------|-------------|
| Console | `dotnet new console -lang VB` | `--framework` |
| Class Library | `dotnet new classlib -lang VB` | `--framework` |
| Windows Forms | `dotnet new winforms -lang VB` | `--framework` |
| WPF | `dotnet new wpf -lang VB` | `--framework` |
| xUnit | `dotnet new xunit -lang VB` | `--framework` |
| NUnit | `dotnet new nunit -lang VB` | `--framework` |
| MSTest | `dotnet new mstest -lang VB` | `--framework` |
| Solution | `dotnet new sln` | — |
| gitignore | `dotnet new gitignore` | — |
| editorconfig | `dotnet new editorconfig` | — |
| global.json | `dotnet new globaljson` | `--sdk-version`, `--roll-forward` |

> Note: `--use-program-main` is a C#-only flag and does not apply to VB. The VB `console` template always generates an explicit `Module Program` with `Sub Main` — there are no top-level statements in VB.
