# Custom Windows x64 build

This fork removes the virtual-machine operation from the runtime session sequence. The Windows workflow builds Release x64 binaries and runs the existing `CompositionRootTests`, including the assertion for the session operation sequence.

## Download the build

Push to `codex/windows-no-vm` to start **Build Windows x64**, or run that workflow manually from the repository's Actions page. After a successful run, download and extract its `seb-windows-x64-<run number>` artifact. `BUILD.txt` records the source commit and workflow run.

The artifact contains:

- `Application/`: the complete application, browser engine, and dependencies. Launch `SafeExamBrowser.exe` from this directory.
- `Service/`: the Windows service executable and dependencies.
- `Configuration/`: `SEBConfigTool.exe` and its dependencies.
- `LICENSE.txt`: the upstream license.

Keep each directory intact. This is an unsigned application archive, not the upstream MSI installer or setup bundle. The workflow does not install or launch the application on the build runner.

## Run on Windows

Use 64-bit Windows 10 version 1803 or later with [.NET Framework 4.8 Runtime](https://dotnet.microsoft.com/download/dotnet-framework/net48) and the [Visual C++ 2015–2022 x64 Redistributable](https://learn.microsoft.com/en-us/cpp/windows/latest-supported-vc-redist). The archive does not install these prerequisites.

The default configuration requires the **Safe Exam Browser Service**. For a machine where that service is not already installed, move the extracted artifact to its permanent location and run these commands in an elevated Windows PowerShell prompt from the artifact directory:

```powershell
& "$env:WINDIR\Microsoft.NET\Framework64\v4.0.30319\InstallUtil.exe" `
    '.\Service\SafeExamBrowser.Service.exe'
if ($LASTEXITCODE -ne 0) { throw 'Service installation failed.' }
Start-Service -Name SafeExamBrowser
```

The project's service installer registers `SafeExamBrowser` with automatic startup under LocalSystem. If the service already exists, these commands are not an upgrade procedure; use the service installation appropriate to that machine. No service policy is changed by this fork. After the service is available, launch `Application\SafeExamBrowser.exe`.

The upstream native integrity module (`seb_x64.dll`) is not provided in this source repository and is not included by this workflow. The existing build emits a warning when it is missing; the managed integrity wrapper logs unavailable-module warnings. Native integrity-dependent features are therefore unavailable in this archive. No integrity implementation or policy is modified by the build workflow.

## Build locally

Use Windows with Visual Studio 2022 or its Build Tools, the .NET desktop build tools, .NET Framework 4.8 Developer Pack, and NuGet CLI. In a Developer PowerShell prompt at the repository root:

```powershell
$solutionDirectory = (Get-Location).Path + '\'
nuget restore SafeExamBrowser.sln -NonInteractive
if ($LASTEXITCODE -ne 0) { throw 'NuGet restore failed.' }

foreach ($project in @(
    'SafeExamBrowser.Runtime',
    'SafeExamBrowser.Client',
    'SafeExamBrowser.Service',
    'SebWindowsConfig'
)) {
    & msbuild "$project\$project.csproj" /m /verbosity:minimal `
        /p:Configuration=Release /p:Platform=x64 /p:LangVersion=latest `
        "/p:SolutionDir=$solutionDirectory"
    if ($LASTEXITCODE -ne 0) { throw "Build failed: $project" }
}
```

The Client post-build step copies its browser dependencies and output into `SafeExamBrowser.Runtime\bin\x64\Release`. The workflow packages that entire directory and the separate Service and Configuration outputs. It builds individual projects because the full Release solution also builds WiX installer projects whose signing steps require the upstream signing certificate.

The regression test results are available as the separate `windows-test-results-<run number>` artifact. These tests verify runtime component wiring; they do not constitute an interactive Windows or virtual-machine launch test.
