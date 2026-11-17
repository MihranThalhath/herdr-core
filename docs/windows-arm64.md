# Native Windows ARM64 build

herdr-core (`hc`) builds natively for `aarch64-pc-windows-msvc` (PE `0xAA64`) on
Windows ARM64 hardware. Upstream release manifests only ship `windows-x86_64`.

## Toolchain

| Piece | Notes |
|---|---|
| Rust | repo-pinned toolchain |
| MSVC | VS with `Hostarm64\arm64\link.exe` |
| Windows SDK | 10.0.26100+ |
| Zig 0.16.0 | **x86_64 build** — native ARM64 Zig crashes; x86_64 runs under emulation |
| LLVM/libclang | aarch64 Windows build; `libclang.dll` for bindgen, MSVC-aware `clang` for ring |

Environment (PowerShell):

```powershell
$env:ZIG = "C:\path\to\zig-x86_64\zig.exe"
$env:LIBCLANG_PATH = "C:\path\to\llvm\bin"
$env:RUSTFLAGS = "-C target-feature=+crt-static"   # optional; see below
$env:PATH = "C:\path\to\llvm\bin;$env:PATH"
```

`.cargo/config.toml` already sets `+crt-static` for `aarch64-pc-windows-msvc`
(mirroring the existing x86_64 section), so `RUSTFLAGS` is redundant but harmless.

## Build

```powershell
git submodule update --init --recursive   # vendor/ghostty (fresh upstream)
cargo build --release                     # → target\release\herdr-core.exe
```

`hc --version` → `hc <version>`.

## Install (manual — upstream scripts are x64-only)

`distribution/install.ps1`, `install-local.ps1` and `fetch-conpty.ps1` only
handle `windows-x86_64`. Install layout used for ARM64
(`%LOCALAPPDATA%\Programs\herdr-core\`):

```
herdr-core.exe                 # ARM64
hc.exe                         # hardlink to herdr-core.exe
conpty\conpty.dll              # ARM64 — nupkg runtimes/win-arm64/native/
conpty\arm64\OpenConsole.exe   # ARM64 — nupkg build/native/runtimes/arm64/
conpty\x64\OpenConsole.exe     # x64 — nupkg build/native/runtimes/x64/
LICENSE, THIRD-PARTY-NOTICES
```

The ConPTY nupkg is `Microsoft.Windows.Console.ConPTY.1.24.260710001`
(`https://github.com/microsoft/terminal/releases/download/v1.24.11911.0/`),
pinned in the herdr CLI repo's `packaging/windows/conpty.json`.

hc loads `conpty\conpty.dll` beside the exe because inbox ConPTY swallows the
OSC 7501 responses core depends on (`src/platform/windows.rs`).

Append the install dir to user PATH and broadcast the env change so `hc`
resolves in new terminals.

## WSL / machines

Register a WSL distro as a machine so spaces can live inside it:

```powershell
hc machine add --via 'wsl.exe -d Ubuntu -e sh -c' --name wsl --install
```

hc installs itself inside the distro on first connect; folder paths are
Linux paths (`/home/user/project`), not `\\wsl$\...`.

## Limitations

- `hc update` cannot work: `this_target()` reports `windows-aarch64` and no
  such release asset exists upstream. Rebuild + recopy to update.
- Requires the x86_64 Zig under emulation for ghostty-vt.
