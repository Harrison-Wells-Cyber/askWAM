# askWAM

askWAM lists Windows work or school accounts and requests Microsoft Entra
access tokens through Windows Web Account Manager (WAM), without opening a
sign-in window.

## Build

Use Windows 10 (1803 or newer) or Windows 11, x64. Install Visual Studio 2022
C++ Build Tools with a Windows 10 or 11 SDK.

From the repo folder, run in PowerShell:

```powershell
.\src\native\build.ps1 -Configuration Release
Set-Location .\src\native\bin\Release
```

## Run

List accounts:

```powershell
.\askwam.exe --enum
```

Request a Microsoft Graph token using the default account:

```powershell
.\askwam.exe
```

Choose an account or request a token for another resource:

```powershell
.\askwam.exe --username user@example.com
.\askwam.exe --account-id '<ID from --enum>'
.\askwam.exe --resource https://graph.microsoft.com
```

By default, askWAM uses the Microsoft Teams client ID and prints the token.
Use `--client-id` to choose another client, `--scope` to set scopes instead of
a resource, and `--hide` to leave the token out of the output.
Run `.\askwam.exe --help` for all options.

## Other versions

Run these build commands from the repo folder.

- **Rust:** Install Rust 1.82 or newer with the x64 MSVC toolchain, plus the
  C++ Build Tools and Windows SDK above. Run
  `.\src\rust\build.ps1 -Configuration Release`, then
  `.\src\rust\bin\Release\askwam.exe`. See the [Rust README](src/rust/README.md).
- **.NET:** Install the .NET SDK, .NET Framework 4.8, and Windows SDK
  10.0.26100.0. Run
  `dotnet build .\src\dotnet\AskWAM.csproj --configuration Release`, then
  `.\src\dotnet\bin\Release\net48\askwam.exe`.
- **BOF:** Use the C++ Build Tools and Windows SDK above. Run
  `.\src\bof\build.ps1 -Configuration Release`, load
  `src\bof\askwam.cna` in Cobalt Strike, then run `askwam --enum`.
  See the [BOF README](src/bof/README.md) for other loaders.
