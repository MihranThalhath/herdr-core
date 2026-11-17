# Building Herdr for native Windows ARM64

How `herdr` (CLI v0.9.5), `herdr-app` (v0.1.16) and `herdr-core`/`hc` (v0.1.21)
were built **natively** (`aarch64-pc-windows-msvc`, PE machine `0xAA64`) on an
ARM64 Windows laptop. Upstream only ships `windows-x86_64` binaries.

Workspace root: `C:\dev\herdr`
(`herdr-app` and `herdr-core` are cloned *inside* the `herdr` repo dir, so the
root `rust-toolchain.toml` pins Rust 1.96.1 for all three.)

> Versions below are what was built/tested, not pins — for rebuilds use the
> latest release tags (or `master`) of each repo; the toolchain stays the same.

## Host toolchain

| Piece | Value |
|---|---|
| OS / arch | Windows 11 ARM64 (`aarch64-pc-windows-msvc`) |
| Rust | 1.96.1 via root `rust-toolchain.toml` (channel + clippy + rustfmt) |
| MSVC | VS 18 Community, MSVC 14.50.35717, `Hostarm64\arm64\link.exe` |
| Windows SDK | 10.0.26100.0 (has `arm64\fxc.exe` for shaders) |
| Zig | 0.16.0 **x86_64** at `C:\Users\<user>\tools\zig-x86_64\zig.exe` |
| LLVM/libclang | 23.1.5 aarch64 at `C:\Users\<user>\tools\llvm` (`bin\libclang.dll`) |

### Downloads used

- **Zig x86_64**: `zig-x86_64-windows-0.16.0.zip` → `C:\Users\<user>\tools\zig-x86_64\`
  - Do NOT use native ARM64 Zig (`tools\zig\`, `zig-aarch64-windows-0.16.0.zip`)
    — it segfaults on this machine. The x86_64 build runs under Prism emulation.
- **LLVM**: `llvm-aarch64.tar.zst` → `C:\Users\<user>\tools\llvm\`
  (official LLVM Windows aarch64 payload; provides `libclang.dll` for bindgen
  and an MSVC-aware `clang` for `ring`'s build script)
- **ConPTY nupkg**:
  `https://github.com/microsoft/terminal/releases/download/v1.24.11911.0/Microsoft.Windows.Console.ConPTY.1.24.260710001.nupkg`
  sha256 `9382ad7becb7e4d84e300578d8e4f4df28f43d979d9055d978c42913c47e0e9d`
  (pinned in `packaging/windows/conpty.json`; ships
  `runtimes/win-arm64/native/conpty.dll` and
  `build/native/runtimes/arm64/OpenConsole.exe`)
- **Node v24.14.0** (`tools\node-v24.14.0-win-x64`) — for repo TS scripts/tests;
  not needed for `cargo build` itself.

### Build environment (PowerShell)

```powershell
$env:ZIG = "C:\Users\<user>\tools\zig-x86_64\zig.exe"
$env:LIBCLANG_PATH = "C:\Users\<user>\tools\llvm\bin"
$env:RUSTFLAGS = "-C target-feature=+crt-static"
$env:PATH = "C:\Users\<user>\tools\llvm\bin;$env:PATH"   # ring build.rs needs clang on PATH
```

`+crt-static` produces a static-CRT exe (no VC++ redist needed). For
`herdr-core` the same flag was added permanently to
`herdr-core\.cargo\config.toml`:

```toml
[target.aarch64-pc-windows-msvc]
rustflags = ["-C", "target-feature=+crt-static"]
```

## 1. herdr CLI — v0.9.5

```powershell
cd C:\dev\herdr
git checkout v0.9.5
cargo build --release                     # → target\release\herdr.exe (~22 MB, 0xAA64)
```

On ARM64 the CLI uses the OS system ConPTY (the app-local ConPTY loader is
x86_64-only upstream), so the bare exe runs fine.

### Packaging + install (installer-layout zip)

`herdr-windows-aarch64-v0.9.5.zip` mimics what `distribution/install.ps1`
validates:

```
herdr.exe
conpty\conpty.dll                 (ARM64, nupkg runtimes/win-arm64/native/)
conpty\arm64\OpenConsole.exe      (ARM64, nupkg build/native/runtimes/arm64/)
conpty\x64\OpenConsole.exe        (x64, nupkg build/native/runtimes/x64/)
herdr-conpty.json                 (marker w/ per-file sha256 per packaging/windows/conpty.json)
THIRD-PARTY-NOTICES\              (licenses from the nupkg)
```

```powershell
pwsh -NoProfile -ExecutionPolicy Bypass -File ".\distribution\install.ps1" `
  -LocalPackagePath ".\herdr-windows-aarch64-v0.9.5.zip" `
  -LocalPackageFormat zip -LocalPackageIdentity "0.9.5" `
  -LocalPackageSha256 "<sha256 of the zip>"
```

Installed once, later uninstalled at user request (junctions + release dir +
PATH entry removed). `target\release\herdr.exe` remains.

> `herdr update` cannot work — manifests have no `windows-aarch64` asset; the
> local-package install mode is the update path.

## 2. herdr-app — v0.1.16

```powershell
cd C:\dev\herdr
git clone https://github.com/herdrdev/herdr-app herdr-app
cd herdr-app
git submodule update --init --recursive   # vendor/ghostty @ 1e5cdb865807abc990181efb8ebfa9122955cce2
# env vars from "Build environment" above
cargo build --release --locked           # → target\release\herdr-app.exe (~53 MB, 0xAA64)
```

`vendor/ghostty` builds libghostty-vt via Zig
(`zig build -Demit-lib-vt -Doptimize=ReleaseFast -Dcpu=baseline -Dtarget=aarch64-windows-msvc`);
bindgen needs libclang (`LIBCLANG_PATH` + LLVM `bin` on PATH).

Install:

```powershell
pwsh -NoProfile -ExecutionPolicy Bypass -File scripts\install-windows.ps1
# or: just --shell pwsh --shell-arg -Command install
```

Installs to `%LOCALAPPDATA%\Programs\herdr-app\` + Start Menu shortcut + user
PATH. Already installed and verified running.

## 3. herdr-core (hc) — v0.1.21

```powershell
cd C:\dev\herdr
git clone https://github.com/herdrdev/herdr-core herdr-core
cd herdr-core
git submodule update --init --recursive   # vendor/ghostty (fresh upstream)
# add the [target.aarch64-pc-windows-msvc] crt-static section to .cargo\config.toml
# env vars from "Build environment" above (RUSTFLAGS optional — config.toml covers it)
cargo build --release                     # → target\release\herdr-core.exe (~8.1 MB, 0xAA64)
```

`hc --version` → `hc 0.1.21`.

### Install (manual — upstream scripts are x64-only)

`install.ps1` / `install-local.ps1` / `fetch-conpty.ps1` only handle
`windows-x86_64`, so install was replicated into
`%LOCALAPPDATA%\Programs\herdr-core\`:

```
herdr-core.exe                 (ARM64)
hc.exe                         (hardlink to herdr-core.exe)
conpty\conpty.dll              (ARM64, nupkg runtimes/win-arm64/native/)
conpty\arm64\OpenConsole.exe   (ARM64)
conpty\x64\OpenConsole.exe     (x64, kept for layout compatibility)
LICENSE, THIRD-PARTY-NOTICES
```

hc loads `conpty\conpty.dll` beside the exe because Windows' inbox ConPTY
swallows the OSC 7501 responses core needs
(`crates\herdr-core\src\platform\windows.rs`). PATH was appended and the
env-change broadcast sent, so `hc` works in new terminals.

> `hc update` won't work (`this_target()` → `windows-aarch64`, no upstream
> asset). Rebuild + recopy to update.

## 4. herdr-core inside WSL (the `wsl` machine)

```powershell
hc machine add --via 'wsl.exe -d Ubuntu -e sh -c' --name wsl --install
```

hc installs itself in the distro on first connect →
`/home/<user>/.local/bin/hc` (Linux aarch64, `hc 0.1.21`), lingering enabled so
the core survives logout.

## Verify

```powershell
.\target\release\herdr.exe --version                    # herdr 0.9.5
.\herdr-app\target\release\herdr-app.exe --version
.\herdr-core\target\release\herdr-core.exe --version    # hc 0.1.21

# PE machine check — want 0xAA64 (ARM64), 0x8664 = x64
$b=[IO.File]::ReadAllBytes("path\to\exe"); $pe=[BitConverter]::ToInt32($b,0x3C)
'0x{0:x4}' -f [BitConverter]::ToUInt16($b,$pe+4)
```

## Known limitations

- No `windows-aarch64` release assets upstream → no auto-update for any of the
  three; rebuild manually.
- Native ARM64 Zig 0.16.0 crashes → x86_64 Zig under emulation is required for
  every Zig step (ghostty-vt for app and core, anything in vendor/).
- `herdr` CLI pane terminals use OS ConPTY on ARM64; the bundled conpty.dll is
  inert on ARM64 (loader is x64-only upstream).
- herdr-core rule: never start a core from an agent shell — it inherits that
  shell's env into every terminal it opens. Tests use `HC_SESSION` + scratch
  XDG paths.
