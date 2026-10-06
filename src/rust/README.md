# askWAM Rust executable

This is a port of `../native/askwam.c`, using Microsoft's generated `windows`
WinRT bindings. It provides the standalone executable's complete CLI, account
enumeration/selection, silent token requests, and JSON result format. It does
not require .NET or MSAL. It has no BOF integration.

## Build on Windows

Install Rust (1.82 or newer), Visual Studio 2022 C++ Build Tools, and a Windows
10 or 11 SDK. Use a Windows x64 MSVC Rust toolchain:

```powershell
rustup default stable-x86_64-pc-windows-msvc
.\src\rust\build.ps1 -Configuration Release
.\src\rust\bin\Release\askwam.exe --help
```

These paths assume the repository root. In the standalone source archive,
`Cargo.toml` and `build.ps1` are at the archive root: run `build.ps1` there and
use `bin\Release\askwam.exe`.

The script uses the committed `Cargo.lock`, enables the static MSVC C runtime,
and copies the executable to `bin\Release\askwam.exe` (or `bin\Debug`). Rust
dependencies are compiled into the executable; WAM and WinRT are provided by
Windows. The static CRT setting is in `.cargo/config.toml`. For direct Cargo
commands, change to this directory so Cargo loads that configuration:

```powershell
Set-Location .\src\rust
cargo build --locked --release --target x86_64-pc-windows-msvc
cargo test --locked
```

Runtime target: Windows 10/11 x64. Account enumeration and username selection
use the WAM API introduced in Windows 10 version 1803. Actual token availability
depends on the current user's WAM accounts, client registration, consent, and
tenant policies.

## Usage and compatibility

```powershell
.\src\rust\bin\Release\askwam.exe --enum
.\src\rust\bin\Release\askwam.exe --account-id '<opaque account ID>' --hide
.\src\rust\bin\Release\askwam.exe --username 'user@example.com' --hide
.\src\rust\bin\Release\askwam.exe --resource https://graph.microsoft.com --hide
.\src\rust\bin\Release\askwam.exe --cae --hide
.\src\rust\bin\Release\askwam.exe --claims-json '{"access_token":{"xms_cc":{"values":["cp1"]}}}' --hide
```

Defaults match the C executable: Teams desktop/mobile client ID
`1fec8e78-bce4-4aaf-ab1b-5451cc387264`, Graph `.default`, `organizations`,
WAM's default account, a 30-second timeout, and visible token output.

- `--scope` sends its value followed by `openid offline_access profile` and
  sets `wam_compat=2.0`. `--resource` sends an empty scope and only the resource
  property. The modes are mutually exclusive.
- An opaque `--account-id` is used directly in `FindAccountAsync`. `--username`
  enumerates accounts for the client ID and requires exactly one match using
  Windows' case-insensitive ordinal comparison.
- `--enum` returns accounts without requesting a token. It cannot be combined
  with an account selector.
- `--cae` sets the predefined `xms_cc/cp1` claims object. It is mutually exclusive
  with `--claims-json`; custom claims are sent verbatim after validation.
- `--timeout-seconds` accepts 1–300, **per asynchronous operation**, matching
  the native executable. Provider discovery, account discovery, and token
  acquisition can each take this long. Polling is every 25 ms; expiry attempts
  cancellation. This is not a whole-process deadline and cannot interrupt a
  blocking synchronous COM call.
- `--hide` omits `accessToken`; it preserves token length and account metadata.
  Provider diagnostics include the error code and message length, never the
  message or claims challenge. Tokens are not decoded or validated locally.
- Like the C executable, only the first successful token response is reported.
- Like the C executable, `--scope` accepts a space-separated string; if repeated,
  the last value wins. This differs from the older .NET client's repeatable
  scope collection. The C client's UTF-16 scope length limit is retained.

Successful JSON goes to stdout, failures go to stderr. JSON keys and exit codes
match the native client; consumers must not depend on key order. Output uses
UTF-8 JSON with proper escaping, including non-ASCII account names. Results are
assembled before output, so account-read failures cannot leave partial JSON on
stdout.

| Exit code | Meaning |
| --- | --- |
| 0 | Success or help |
| 2 | Invalid arguments or claims |
| 20 | User interaction required; no interactive fallback |
| 30 | Account provider unavailable |
| 31 | Account discovery failed |
| 32 | Other WAM token error |
| 33 | Account not found |
| 34 | Username matched multiple accounts |
| 40 | Runtime failure, cancellation, timeout, unsupported platform, or output failure |

Timeout keeps the native JSON shape:
`{"status":"runtime_error","hresult":"0x80070102"}` and exit code 40.

Intentional input-handling improvements: claims must parse as a complete JSON
object, empty values and embedded NULs are rejected, missing option values are
reported clearly, and input errors use JSON. `-h` is also accepted. Malformed
claims are rejected before calling Windows and are not echoed in diagnostics.

The main implementation files are `src/options.rs` (CLI/request properties),
`src/wam.rs` (WinRT calls and COM lifetime), and `src/report.rs` (JSON contracts).
COM interfaces and HSTRINGs use the generated bindings' automatic ownership.
The few unsafe operations are documented: initializing/uninitializing WinRT,
Windows string comparison, and retrieving nullable interface results without
turning a successful null account/provider lookup into a runtime error.

## Validation and Windows handoff

The implementation was checked on Linux with 21 CLI/request/report tests,
Windows cross-compilation, formatting, and Clippy. Rust 1.82 passed the portable
tests and Windows MSVC type-checking; the current stable toolchain also passed
Windows GNU/MSVC type-checking and linked the GNU release executable.
The GNU cross-built executable
is a Windows x64 executable; its PE imports can be inspected separately from the
recommended MSVC build. **Live WAM behavior and the PowerShell/MSVC build need
Windows testing.** The portable tests do not authenticate or contact a tenant.

Suggested Windows checks against the C executable, using the same account,
client ID, and authority:

1. Compare `--enum` account IDs, usernames, and states, including non-ASCII names.
2. Request with the default account, an enumerated ID, and a unique username;
   test a missing ID/username and duplicate usernames when available.
3. Compare Graph scope mode and resource mode. Test a different client ID,
   scope/resource, and authority supported by your tenant.
4. Exercise `--cae` and a tenant-issued custom claims challenge. Compare actual
   token behavior; merely requesting `cp1` does not guarantee a CAE token.
5. Confirm `--hide` removes the token on success and that failures produce one
   JSON object on stderr with the expected exit code. Check an interaction-required
   account: no sign-in window should open.
6. Exercise timeout behavior when an operation actually remains pending; setting
   `--timeout-seconds 1` alone does not guarantee a timeout if WAM finishes sooner.

## API references used for the port

- [Microsoft's Rust WAM bindings](https://microsoft.github.io/windows-docs-rs/doc/windows/Security/Authentication/Web/Core/struct.WebAuthenticationCoreManager.html)
- [Microsoft Entra WAM API reference](https://learn.microsoft.com/en-us/entra/identity-platform/reference-entra-id-wam-api)
- [RoInitialize and its balancing requirements](https://learn.microsoft.com/en-us/windows/win32/api/roapi/nf-roapi-roinitialize)
- [Windows ordinal string comparison](https://learn.microsoft.com/en-us/windows/win32/api/stringapiset/nf-stringapiset-comparestringordinal)
- [MSAL issue discussing resource and wam_compat incompatibility](https://github.com/AzureAD/microsoft-authentication-library-for-dotnet/issues/4918)

The API calls were also checked against the downloaded `windows` 0.62.2 and
`windows-future` 0.3.2 generated sources. Scope/resource and `cp1` request semantics
are carried over from the native implementation; Windows/tenant testing remains
the evidence for policy-specific behavior.
