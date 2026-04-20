# Project Structure Patterns

## Solution Layout

### Recommended Directory Structure

```text
<repo-root>/
├── .config/
│   └── dotnet-tools.json
├── src/
│   ├── <ProductName>.Core/
│   ├── <ProductName>.App/
│   └── <ProductName>.Forms/
├── tests/
│   ├── <ProductName>.Core.Tests/
│   └── <ProductName>.App.Tests/
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

### Project Naming Conventions

| Project Type | Pattern | Example |
|--------------|---------|---------|
| Core library | `<ProductName>.Core` | `Contoso.Orders.Core` |
| Domain layer | `<ProductName>.Domain` | `Contoso.Orders.Domain` |
| Application layer | `<ProductName>.Application` | `Contoso.Orders.Application` |
| Infrastructure | `<ProductName>.Infrastructure` | `Contoso.Orders.Infrastructure` |
| Windows Forms app | `<ProductName>.App` | `Contoso.Orders.App` |
| Worker service | `<ProductName>.Worker` | `Contoso.Orders.Worker` |
| Unit tests | `<ProjectName>.Tests` | `Contoso.Orders.Core.Tests` |
| Integration tests | `<ProjectName>.IntegrationTests` | `Contoso.Orders.App.IntegrationTests` |

---

## Directory.Build.props

`Directory.Build.props` is automatically imported by MSBuild for all projects in its directory and subdirectories.

### Basic Template

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net9.0-windows</TargetFramework>
    <LangVersion>latest</LangVersion>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <WarningsAsErrors />
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    <EnableNETAnalyzers>true</EnableNETAnalyzers>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
  </PropertyGroup>

  <!-- Package metadata for libraries -->
  <PropertyGroup>
    <Authors>Your Name or Organization</Authors>
    <Company>Your Company</Company>
    <Copyright>Copyright (c) $(Company) $([System.DateTime]::Now.Year)</Copyright>
    <RepositoryUrl>https://github.com/your-org/your-repo</RepositoryUrl>
    <RepositoryType>git</RepositoryType>
  </PropertyGroup>

  <!-- Deterministic builds for CI -->
  <PropertyGroup Condition="'$(CI)' == 'true'">
    <ContinuousIntegrationBuild>true</ContinuousIntegrationBuild>
    <Deterministic>true</Deterministic>
  </PropertyGroup>
</Project>
```

**VB.NET-specific notes:**

- `<Nullable>enable</Nullable>` is a C# nullable reference types feature and does **not** apply to VB.NET. Do not include it in VB.NET project files.
- `<ImplicitUsings>enable</ImplicitUsings>` is also C#-specific (it generates a `GlobalUsings.g.cs` file). Use `Imports` statements at the top of each `.vb` file, or place shared `Imports` in a dedicated module instead.
- `<LangVersion>` for VB.NET uses different version numbers than C#. VB 16.9 is the current stable language version (shipped with .NET 5+); Microsoft has indicated VB language evolution is effectively frozen at 16.9. Earlier VB 16.0 shipped with .NET Core 3.0. Use `<LangVersion>latest</LangVersion>` to get the latest supported version for your SDK.

### VB-critical project-level MSBuild properties

VB.NET has four project-wide compiler options that should normally be set in the `.vbproj` (or a shared `Directory.Build.props`) rather than sprinkled as per-file `Option` headers:

```xml
<PropertyGroup>
  <OptionStrict>On</OptionStrict>
  <OptionExplicit>On</OptionExplicit>
  <OptionInfer>On</OptionInfer>
  <OptionCompare>Binary</OptionCompare>
</PropertyGroup>
```

- `OptionStrict=On` prevents implicit narrowing conversions (e.g., `Object` to `Integer`); this is the single most important setting for catching type errors at compile time.
- `OptionExplicit=On` requires all variables to be declared with `Dim` / `Private` / `Public` before use.
- `OptionInfer=On` allows local type inference such as `Dim x = 42` (the compiler infers `Integer`).
- `OptionCompare=Binary` gives culture-invariant `=` / `<>` string comparison (byte-for-byte); the alternative `Text` would apply locale-aware case-insensitive comparison to every `=` and `<>` on strings, which is rarely what you want.

Setting these at the project level means every `.vb` file in the project uses the same defaults without needing individual `Option Strict On` headers at the top of each file.

### Conditional Properties by Project Type

```xml
<Project>
  <!-- Shared defaults for VB.NET projects -->
  <PropertyGroup>
    <TargetFramework>net9.0-windows</TargetFramework>
    <LangVersion>latest</LangVersion>
  </PropertyGroup>

  <!-- Test project defaults -->
  <PropertyGroup Condition="$(MSBuildProjectName.EndsWith('.Tests'))">
    <IsPackable>false</IsPackable>
    <IsTestProject>true</IsTestProject>
  </PropertyGroup>

  <!-- Library defaults: generate XML docs for libraries -->
  <PropertyGroup Condition="!$(MSBuildProjectName.EndsWith('.Tests')) AND !$(MSBuildProjectName.EndsWith('.App'))">
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
  </PropertyGroup>
</Project>
```

### Nested Directory.Build.props

Child directories can extend the parent by importing it explicitly:

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

Use `Directory.Build.targets` for logic that runs after the project file is fully evaluated.

```xml
<Project>
  <!-- Run after project evaluation -->
  <Target Name="PrintBuildInfo" BeforeTargets="Build">
    <Message Importance="High" Text="Building $(MSBuildProjectName) for $(TargetFramework)" />
  </Target>

  <!-- Enforce test naming convention -->
  <Target Name="ValidateTestProjectNaming" BeforeTargets="Build" Condition="'$(IsTestProject)' == 'true'">
    <Error Condition="!$(MSBuildProjectName.EndsWith('.Tests')) AND !$(MSBuildProjectName.EndsWith('.IntegrationTests'))"
           Text="Test projects must end with .Tests or .IntegrationTests" />
  </Target>
</Project>
```

---

## Central Package Management (CPM)

Central Package Management consolidates package versions into a single `Directory.Packages.props` file.

### Enabling CPM

Add to `Directory.Packages.props` at the repository root:

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    <CentralPackageTransitivePinningEnabled>true</CentralPackageTransitivePinningEnabled>
  </PropertyGroup>

  <ItemGroup>
    <!-- Runtime packages -->
    <PackageVersion Include="Microsoft.Extensions.Hosting" Version="9.0.0" />
    <PackageVersion Include="Microsoft.Extensions.DependencyInjection" Version="9.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Logging" Version="9.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Options" Version="9.0.0" />
    <PackageVersion Include="System.Text.Json" Version="9.0.0" />

    <!-- Windows Forms extras -->
    <PackageVersion Include="Microsoft.EntityFrameworkCore" Version="9.0.0" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore.SqlServer" Version="9.0.0" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore.Design" Version="9.0.0" />

    <!-- Testing -->
    <PackageVersion Include="Microsoft.NET.Test.Sdk" Version="17.11.1" />
    <PackageVersion Include="xunit" Version="2.9.2" />
    <PackageVersion Include="xunit.runner.visualstudio" Version="2.8.2" />
    <PackageVersion Include="Moq" Version="4.20.72" />
    <PackageVersion Include="FluentAssertions" Version="6.12.1" />
    <PackageVersion Include="coverlet.collector" Version="6.0.2" />

    <!-- Analyzers -->
    <PackageVersion Include="Microsoft.CodeAnalysis.VisualBasic.Analyzers" Version="3.3.3" />
    <PackageVersion Include="Roslynator.Analyzers" Version="4.12.6" />
  </ItemGroup>
</Project>
```

**Analyzer note:** `StyleCop.Analyzers` is C#-only and does not support VB.NET. Use `Microsoft.CodeAnalysis.VisualBasic.Analyzers` for VB.NET-specific Roslyn analyzer rules. `Roslynator.Analyzers` has partial VB.NET coverage and is generally safe to include.

### Project File References with CPM

When CPM is enabled, project files reference packages without versions:

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.Extensions.Hosting" />
  <PackageReference Include="Microsoft.Extensions.Logging" />
</ItemGroup>
```

### Version Overrides

Use `VersionOverride` sparingly when a project requires a different version:

```xml
<ItemGroup>
  <PackageReference Include="Newtonsoft.Json" VersionOverride="13.0.3" />
</ItemGroup>
```

---

## global.json

Pin the SDK version for reproducible builds:

```json
{
  "sdk": {
    "version": "9.0.100",
    "rollForward": "latestFeature",
    "allowPrerelease": false
  }
}
```

### Roll-Forward Policies

| Value | Behavior |
|-------|----------|
| `patch` | Require exact major.minor.feature.patch; otherwise roll forward to a higher patch in the same feature band. |
| `feature` | Require exact major.minor.feature.patch; otherwise roll forward to a higher feature+patch in the same minor band. |
| `minor` | Require exact major.minor.feature.patch; otherwise roll forward to a higher minor+feature+patch in the same major band. |
| `major` | Require exact major.minor.feature.patch; otherwise roll forward to a higher major. |
| `latestPatch` | Require exact major.minor.feature; pick highest installed patch. |
| `latestFeature` | Require exact major.minor; pick highest installed feature+patch. |
| `latestMinor` | Require exact major; pick highest installed minor+feature+patch. |
| `latestMajor` | Pick highest installed SDK regardless of major version. |
| `disable` | Require exact SDK version specified. |

---

## nuget.config

Configure package sources and credentials:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" protocolVersion="3" />
    <!-- Private feeds -->
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

  <!-- CI credentials via environment variables -->
  <packageSourceCredentials>
    <github>
      <add key="Username" value="%NUGET_USERNAME%" />
      <add key="ClearTextPassword" value="%NUGET_TOKEN%" />
    </github>
  </packageSourceCredentials>
</configuration>
```

---

## Analyzers and Code Style

### .editorconfig Basics

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 4
insert_final_newline = true
trim_trailing_whitespace = true

[*.{cs,vb}]
dotnet_sort_system_directives_first = true
dotnet_separate_import_directive_groups = false

[*.vb]
# VB.NET-specific editor settings
dotnet_diagnostic.IDE0005.severity = warning
dotnet_diagnostic.IDE0055.severity = warning

[*.{json,yml,yaml}]
indent_size = 2
```

**Note:** The `.editorconfig` `[*.cs]` section supports C#-specific style rules such as `csharp_style_namespace_declarations` and `csharp_style_var_for_built_in_types`. For VB.NET, use `[*.vb]` and rely on `dotnet_diagnostic.*` entries and VB.NET's own IDE diagnostics. Many `csharp_*` settings are silently ignored in VB.NET projects.

### Analyzer Packages in Directory.Build.props

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.CodeAnalysis.VisualBasic.Analyzers" PrivateAssets="all" />
  <PackageReference Include="Roslynator.Analyzers" PrivateAssets="all" />
  <PackageReference Include="Microsoft.CodeAnalysis.NetAnalyzers" PrivateAssets="all" />
</ItemGroup>
```

---

## Multi-Targeting

### Single Project Multi-Targeting

```xml
<PropertyGroup>
  <TargetFrameworks>net9.0-windows;net8.0-windows;net48</TargetFrameworks>
</PropertyGroup>

<ItemGroup Condition="'$(TargetFramework)' == 'net48'">
  <PackageReference Include="System.Text.Json" />
</ItemGroup>
```

**VB.NET and `LangVersion` with multi-targeting:** VB.NET uses different version numbers than C#. The VB version that ships with a given .NET SDK is fixed: for example, .NET 5+ SDKs include VB 16.9. Using `<LangVersion>latest</LangVersion>` is safe for all targets in a multi-targeting scenario because the VB compiler resolves it per SDK. You do not need to set different `LangVersion` values per target framework in VB.NET.

### Conditional Compilation

```vb
#If NET9_0_OR_GREATER Then
    ' .NET 9+ specific code
    ArgumentNullException.ThrowIfNull(value)
#Else
    ' Fallback for older frameworks
    If value Is Nothing Then
        Throw New ArgumentNullException(NameOf(value))
    End If
#End If
```

---

## Source Link and Deterministic Builds

Enable source link for debugger integration:

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

SourceLink is language-agnostic and works identically for VB.NET `.vbproj` files.

---

## Solution Formats: `.sln`, `.slnx`, and `.slnf`

Starting with Visual Studio 17.10 and the .NET 8+ SDK, `.slnx` is the new unified, XML-based solution format intended as a modern alternative to the legacy `.sln` text format. It is produced and consumed by `dotnet sln` and recent Visual Studio builds, and is designed to be easier to diff and merge than `.sln`. The classic `.sln` format remains fully supported, and `.slnf` continues to be the solution-filter format for loading only a subset of the projects in a solution. You can keep a single `.sln` (or `.slnx`) as the source of truth and use `.slnf` files alongside it for focused workflows.

## Solution Filters

Create `.slnf` files for partial solution loading:

```json
{
  "solution": {
    "path": "MyProduct.sln",
    "projects": [
      "src\\MyProduct.Core\\MyProduct.Core.vbproj",
      "src\\MyProduct.App\\MyProduct.App.vbproj",
      "tests\\MyProduct.Core.Tests\\MyProduct.Core.Tests.vbproj"
    ]
  }
}
```

Build a filter with: `dotnet build MyProduct.App.slnf` (open in Visual Studio to use the filtered solution view).
