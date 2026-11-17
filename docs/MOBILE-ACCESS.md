# Herdr mobile access — findings & current setup

How the official Herdr mobile app (`dev.herdr.app`, iOS/Android) connects, what
was wrong, and the working configuration on this machine.

## Architecture (from herdr.dev/mobile + herdr-core source)

- The phone app needs **no Herdr account** — it SSHes directly into a machine
  and runs `hc connect`, which speaks herdr-core's JSON-RPC protocol over
  stdin/stdout.
- Adding a machine = address + user + port. Tailscale hostname/IP works.
- The app shows **herdr-core** content: terminals, agents (Inbox), and spaces.
  It does **not** traverse hc "machines" (`machine.list` entries like `wsl`) —
  verified empirically: `wsl` never appeared inside the `<host>` entry.
- The old herdr TUI's panes are a different product and never appear.

## The desktop app's spaces are per-machine documents

herdr-app mirrors its layout into hc docs via `doc.set`/`doc.get` with
`name:"herdr-app"` and `on:<machine>` (`src/mirror_sync.rs`, `src/mirror.rs`):

- `<host>` (local machine) → `%LOCALAPPDATA%\herdr-core\default\docs.json`
- wsl machine → `~/.local/state/herdr-core/default/docs.json` inside Ubuntu

On this machine **all user spaces live on the `wsl` machine** — the local
doc correctly contains `"spaces":[]`. That is why the phone showed "nothing"
inside `<host>`: it was reading the right (empty) document.

Verified through `hc connect` on `<host>`:

```
doc.get {name:"herdr-app", on:"wsl"} → the real space → tab → terminal t2 (claude-code)
machine.list → wsl, state: connected, via wsl.exe, hc 0.1.21
```

## SSH path that works

Phone → Tailscale → Windows sshd :22 (user `<win-user>`, key auth) works —
auth and `herdr-core.exe connect` launch were both observed. It shows
`<host>`-local hc content only.

For WSL content, WSL got its own SSH endpoint:

- `openssh-server` installed in Ubuntu WSL; `ssh.service` enabled (systemd).
- `/etc/ssh/sshd_config.d/99-herdr-phone.conf`: `Port 2223`,
  `PasswordAuthentication no`, `KbdInteractiveAuthentication no`.
- `/home/<wsl-user>/.ssh/authorized_keys`: phone pubkey (+ the user's own
  `id_ed25519.pub`).
- `/usr/local/bin/hc` and `/usr/local/bin/herdr-core` → symlinks to
  `/home/<wsl-user>/.local/bin/hc` so the app's probe finds hc on PATH.
- WSL runs `networkingMode=mirrored` (`.wslconfig`), so WSL listeners are
  reachable on Windows localhost — but **not** on the Tailscale IP directly
  (adapter mirroring / Windows Firewall). Exposed via Tailscale Serve instead:

```
tailscale serve --bg --tcp=2223 tcp://localhost:2223
# persistent: tailnet → <host>:2223 → localhost:2223 → WSL sshd
```

Verified: `ssh -p 2223 <wsl-user>@localhost 'hc --version'` → `hc 0.1.21`.

## Phone app entries

| Machine | Address | User | Port | Shows |
|---|---|---|---|---|
| `<host>` | `<host>.<tailnet>.ts.net` / `<tailscale-ip>` | `<win-user>` | 22 | host-local hc (empty spaces; wsl terminals may still list) |
| wsl | same | **`<wsl-user>`** | **2223** | WSL hc → the actual spaces/tabs/agents |

Accept the new host key on port 2223 (it's WSL's sshd host key).

## Debugging notes (what was tried / ruled out)

- SSH auth on :22 always worked (ED25519 key, fingerprint
  `<key-fingerprint>`, from phone `<phone-tailscale-ip>`).
- The app probes `hc` then runs
  `cmd /d /c "if exist .\AppData\Local\Programs\herdr-core\herdr-core.exe (... connect) else ..."`
  via SSH exec — observed spawning and staying alive.
- A temporary debug sshd on :2222 (via `tailscale serve`) captured that command;
  one probe attempt returned `0x800700e8` — attributed to the unelevated
  user-context debug sshd, not the real service.
- `C:\Program Files\Git\bin` was added to user PATH so `sh`/`bash` resolve in
  SSH sessions.
- hc sessions were not the issue: `hc connect` with no `--session` reaches the
  default session — the same state dir the desktop app uses.
- All debug artifacts removed: :2222 sshd loop, `sshd-test/`, `watch22.*`,
  serve :2222. Only :2223 remains.

## Ops notes

- WSL sshd dies if the WSL VM stops (`wsl --shutdown`, reboot); systemd
  restarts `ssh.service` when WSL boots. The `wsl` hc machine connection
  (`keeps_running_after_logout`) usually keeps the VM alive.
- `tailscale serve --bg` config persists across reboots (stored in tailscaled).
- To remove the WSL endpoint later: `tailscale serve --tcp=2223 off`,
  `wsl -d Ubuntu -u root systemctl disable --now ssh`, delete
  `/etc/ssh/sshd_config.d/99-herdr-phone.conf`.
