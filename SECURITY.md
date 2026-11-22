# Security Policy

Flipper Emulator is an unofficial community project. It sets up emulators and downloads large third-party components, so it is written to keep the PC it runs on safe. This file describes what is protected, what is not, and how to report a problem.

## Supported versions

| Version | Security fixes |
|---|---|
| 14.1 (development build) and newer | yes |
| 14.0, 13.8 (development builds) | no - the Windows files do not start, please use 14.1 |
| 13.7 and older | no - please update |

## Reporting a vulnerability

Please **do not open a public issue** for a security problem. Use GitHub's private reporting instead: **Security > Report a vulnerability** in this repository (Private vulnerability reporting). If that is not available, open an issue that only says "security contact please" without details.

Please include the version, Windows, Linux (distribution, x86_64 / ARM64) or macOS (version, Apple silicon / Intel), the steps to reproduce and what an attacker would gain. You get an answer as soon as possible; fixes are published as a new version in `DevBuilds` and in the next release.

## Security model (since 13.8)

What the program protects against:

| Threat | Protection |
|---|---|
| Web pages in your browser that talk to the local services in the background (CSRF, DNS rebinding, cross-site WebSocket) | File Manager: every change needs a secret that only the File Manager page carries (`X-Flipper-Token`), JSON requests must be `application/json`, a foreign `Origin` / `Sec-Fetch-Site` is refused, no CORS, only local `Host` names. FlipCTL preview: a guard (`node --require`) forces `127.0.0.1`, refuses foreign `Host`, foreign `Origin` and cross-site requests (also WebSockets). |
| Other computers in the network | Every service listens on `127.0.0.1` only. The Flipper Zero emulator (Renode) used to open its monitor (which can run Python) and console on all interfaces; a small preload library (`flipper-localhost.so`, source `flipper-localhost.c`) now binds them to `127.0.0.1`. |
| Programs inside the emulated Flipper One (for example an app you installed) | They can reach `127.0.0.1` of the PC through QEMU's `10.0.2.2`. The Renode monitor and console therefore also need a secret per emulator start (`FLIPPER-AUTH`); everything else is disconnected. |
| Malicious files in the Flipper OS (HTML, SVG, scripts) | The File Manager never shows them as active pages: only harmless types are shown inline, everything is sent with `nosniff` and a sandboxing Content-Security-Policy. The page itself has a strict CSP with a nonce and cannot be framed. |
| Damaged or replaced downloads | See the table below. A download that fails its check is deleted and not used. |
| Self-update | Only release files of this repository (`github.com/<repo>/releases/download/...`), and only with the SHA-256 that GitHub publishes for the file. Without a published checksum the update is refused. |
| Linux administrator helper | See below. |

### Download verification

| Component | Check |
|---|---|
| Ubuntu WSL base system (Windows) | SHA-256 from `SHA256SUMS` next to the file on `cloud-images.ubuntu.com` (fail-closed) |
| Node.js (Windows and Linux) | SHA-256 from `SHASUMS256.txt` on `nodejs.org` (fail-closed; otherwise the system package is used) |
| zstd for Windows | fixed SHA-256 |
| 7-Zip MSI | fixed SHA-256 |
| QEMU for Windows | SHA-512 file of the mirror (fail-closed since 13.8) and a valid Windows signature if the installer is signed |
| Python (Flipper Zero window, Windows) | SHA-256 from `windows-<version>.json` on `python.org`, plus a valid Python Software Foundation signature on `python.exe`, `pythonw.exe` and `python3xx.dll` |
| PySide6 / shiboken6 (Windows) | fixed SHA-256 of the exact files and the SHA-256 PyPI publishes |
| PySide6, shiboken6, heatshrink2, pip (Linux) | `pip --require-hashes` with fixed versions and checksums, wheels only for Qt |
| Visual C++ runtime | valid Microsoft signature before it is started |
| Flipper Zero emulator source | fixed Git commit (`6b2443fc0065f102ea3045024b1d5e65f4d4c68c`), checked after the download |
| Flipper Zero firmware | Momentum: `sha256:` digest of the GitHub release asset; official: SHA-256 from `directory.json` of the update server. Without a published checksum the file is used with a warning (it only runs inside the emulator). |
| App store (File Manager) | git blob SHA-1 from the GitHub API or SHA-256 from jsDelivr, checked in the browser before the app is installed |
| Python 3.12 runtime (macOS installer) | python-build-standalone `20261021`, fixed SHA-256 per architecture, checked by the installer before it is unpacked (fail-closed) |
| PySide6 / shiboken6 6.12.0 (macOS installer) | fixed SHA-256 of the exact universal2 wheels from PyPI (fail-closed) |
| Multipass 1.17.0 (macOS) | fixed SHA-256 **and** Apple Developer ID signature of Canonical Group Limited (`X4QN7LTP59`, checked with `pkgutil --check-signature`) before macOS installs it |
| Flipper OS image | HTTPS from `dl-linux-images.flipp.dev` and the zstd frame checksum. Flipper Devices publishes no checksum for the nightly images that the program could check. |

Archives are unpacked with path checks (no "zip slip").

### Linux administrator helper

`/usr/local/lib/flipper-emulator/flipper-root` is the only part that runs as root.

- The sudo rule (`/etc/sudoers.d/flipper-emulator-<user>`) lists **exactly** the allowed commands and arguments (`version`, `deps`, `install-preview`, `extract-boot`, `guest-net`, `guest-shelf`, `guest-files Export|Import`, `guest-color [key]`, `remove-self`); nothing else can be run through it. The helper checks the number and form of the arguments again.
- It only works with the calling user's own image `~/.local/share/FlipperOneDemo/realos/flipper-os.img`. The image is opened once and from then on only used through that open file (a swapped path or a link to a system file is refused). While root works with it, the image belongs to root with mode `0600`; if any other program still has it open or mapped, the helper refuses to start. Afterwards the user gets the file back unchanged (also after a crash: the next start repairs it). Only one helper runs at a time.
- The image is mounted `nosuid,nodev` in a folder only root can enter, links that point out of the image are refused, and everything read from or written to the user's folders is done with the user's own rights.
- **Default ("quick" mode):** the listed commands run without a password, so the app keeps working with one click. **Password mode:** `flipper-emulator --helper-mode password` (or `FLIPPER_HELPER_MODE=password` for the installer) makes sudo ask for the password every time the helper is needed; the app shows a password window. `--helper-mode quick` switches back.

**Remaining risk:** mounting a file system image is done by the Linux kernel. A deliberately broken image could try to attack a bug in the kernel's btrfs code. The helper reduces this (only the user's own image, nobody may change it while it is mounted, exact command list), but it cannot remove it completely. If this matters to you, use password mode: then no root step can happen without your password.

### Windows

- The `.bat` runs its PowerShell part as a script block of itself (no `Invoke-Expression`) with `-ExecutionPolicy RemoteSigned` for that process only. Execution policy is not a security boundary; the content that runs is exactly the file you started.
- The `.exe` is a small launcher that contains the same `.bat`. It is **not code-signed** yet, so SmartScreen may warn. Check the SHA-256 against `SHA256SUMS.txt` in the build folder and the release notes. Signing is planned (SignPath Foundation for open-source projects or Azure Trusted Signing).
- Windows needs administrator rights only to enable WSL and, if missing, to install the Microsoft Visual C++ runtime.

### macOS (since 14.0, experimental)

- `FlipperEmulator-<version>.pkg` is **not signed or notarized** (that needs a paid Apple Developer account). macOS therefore blocks it at the first double-click and you allow it once under *System Settings > Privacy & Security > Open Anyway*. Check the SHA-256 against `SHA256SUMS.txt` first: `shasum -a 256 FlipperEmulator-14.1.pkg`.
- The package contains only the program files (Python code, pictures, fonts, the Linux installer for the virtual machine) and an install script, no Mac programs. The script runs as root during the installation: it downloads Python and Qt for Python with fixed checksums, puts `Flipper Emulator.app` into `/Applications`, a link on the desktop of the logged-in user, and logs every step to `/var/log/flipper-emulator-install.log`. Nothing is installed as a background service.
- The Linux parts run in a private Ubuntu virtual machine ("flipperone") managed by Multipass, the Mac counterpart of WSL. The app talks to it only over SSH with its own key (`~/Library/Application Support/FlipperOneDemo/vm/id_ed25519`, created on the Mac) and forwards the local ports to `127.0.0.1` on the Mac. The virtual machine's network is only reachable from this Mac. Inside it the same Linux version of Flipper Emulator runs with all protections listed above.
- Multipass needs administrator rights once (macOS password dialog) and stays installed after *Uninstall Everything*, like WSL on Windows.

### Known limits

- Software inside the emulators (Flipper OS, firmware, apps from the store or your own) is not reviewed by this project. Apps run inside the emulated Flipper One with the rights of its user and can use its internet connection.
- The FlipCTL preview server (`server.js`) and the Flipper Zero emulator are third-party code. They are protected from the network and from other web sites by the guard and the preload library, not changed.
- The third-party installer of the Flipper Zero emulator (`install.sh` at the fixed commit) installs its own Python packages and .NET SDK from PyPI and Microsoft.

## Recommended checks for contributors

Semgrep (`semgrep --config auto`), CodeQL (JavaScript/Python), ShellCheck for the shell scripts and PSScriptAnalyzer for the PowerShell part.
