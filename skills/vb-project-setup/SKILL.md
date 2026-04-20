---
name: vb-project-setup
description: "Create or reorganize VB.NET solutions with clean project boundaries, repeatable SDK settings, and a maintainable baseline for libraries, apps, tests, CI, and local development."
compatibility: "Best for new VB.NET repositories or structural refactors of existing VB.NET solutions."
---

# VB.NET Project Setup

## Trigger On

- creating a new VB.NET solution or restructuring an existing one
- setting up `Directory.Build.props`, shared package management, or repo-wide defaults
- defining project layout for VB.NET apps, libraries, and test projects

## Workflow

1. Start from the app model and deployment target, then choose the smallest correct SDK and target framework set.
2. Use solution folders and project names that reflect bounded contexts or product areas, not temporary implementation details.
3. Centralize shared build settings, analyzer rules, and package versions where it reduces duplication without hiding important differences. Note: VB.NET does not use `<Nullable>enable</Nullable>` (that is a C# nullable reference types feature); VB.NET has its own nullability model via `Nothing` and `Nullable(Of T)`.
4. Create test projects and CI hooks early so new projects do not drift into unverified templates.
5. Prefer project references and composition over circular dependencies or utility dumping grounds.
6. Document the local build, test, and run path in repo docs or `AGENTS.md` when the workflow is not obvious.

## Deliver

- a coherent VB.NET solution structure
- shared build defaults that are easy to reason about
- starter quality and testing hooks for future work

## Validate

- projects have explicit responsibility boundaries
- shared MSBuild settings do not accidentally override platform-specific needs
- a new contributor can build and test the repo without guessing

## References

- [patterns.md](references/patterns.md): solution layout conventions, `Directory.Build.props` / `Directory.Build.targets`, VB project-level `Option*` properties (`OptionStrict` / `OptionExplicit` / `OptionInfer` / `OptionCompare`), Central Package Management, `global.json` with `rollForward` semantics, `nuget.config`, analyzers and `.editorconfig`, multi-targeting, source link, and solution formats (`.sln` / `.slnx` / `.slnf`)
- [templates.md](references/templates.md): `dotnet new` support matrix for VB.NET (console, class library, Windows Forms, WPF, xUnit / NUnit / MSTest), constraints on `worker` and `webapi` (no `-lang VB`; use `classlib` or `console` + `BackgroundService`), language-agnostic templates (`sln`, `gitignore`, `editorconfig`, `globaljson`, `tool-manifest`), and a quick-reference command table
