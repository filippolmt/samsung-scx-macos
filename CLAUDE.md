# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Documentation plus two Bash scripts that make a Samsung SCX-4623FW network multifunction
printer print and scan on macOS 26 (Apple Silicon). There is no build system, no test
suite, and no package manifest. Nothing compiles. The deliverables are `README.md`,
`bin/scx-scan` and `bin/scx-diagnose`.

Goal: a tool other SCX-4623FW owners can use. Do not hardcode new machine-specific network
data; prefer an environment variable or a config file when adding anything configurable.

## Verification

There are no tests. The only check is running the diagnostic script, which prints ✓/✗ for
network reachability, ports 9100/9400/80, CUPS queue state, pending jobs, and SANE scanner
detection:

```sh
bin/scx-diagnose
```

`bin/scx-diagnose` needs macOS (CUPS, BSD `ping`, `open`) and the printer on the LAN —
say so rather than reporting an untested change as verified. `bin/scx-scan` also needs
macOS (`open`, `sips`), but the scan itself only needs LAN reachability: on Linux,
`scanimage` from `sane-utils` works once `/etc/sane.d/xerox_mfp.conf` has
`tcp <printer-ip>`. Probe ports 9100/9400 with `bash -c '</dev/tcp/<printer-ip>/9400'`
before declaring the printer unreachable.

CI (`.github/workflows/check.yml`) runs shellcheck and shfmt on `bin/` and fails on any
private IPv4 address other than the `192.168.1.50` placeholder.

## Language

Everything in the repo is English: README prose, script and directory names, script
comments, user-facing strings, help text. Keep it that way. Replies to the user stay in
Italian.

## Gotchas

- **PCL, never PostScript.** The printer advertises `application/postscript` over IPP and
  that is false. With a PostScript driver it accepts the job, makes a noise, and discards
  it silently with no error. Use `drv:///sample.drv/generpcl.ppd`. If printing produces
  code pages or nothing at all, suspect the driver before the network.
- **No network data in the repo, ever.** Both scripts require `SCX_IP` and exit with an
  error when it is unset; there is no default address. If the environment lacks it they
  source the repo-root `.env`, which is gitignored and holds the real address locally.
  Documentation uses the placeholder `192.168.1.50` or `<printer-ip>`. Never commit a real
  LAN address, MAC address, router brand or hostname — not in prose, not in example output.
- **SANE keeps its own copy of the address** in `/opt/homebrew/etc/sane.d/xerox_mfp.conf`
  (line `tcp <printer-ip>`), outside the repo. That file is not version-controlled and a
  `sane-backends` reinstall wipes it — check it first when the scanner "disappears".
- **Symlinks in `/opt/homebrew/bin` may be stale.** The scripts were renamed
  (`comandi/scansiona` → `bin/scx-scan`, `comandi/diagnostica` → `bin/scx-diagnose`) and
  earlier links may still point at `~/Desktop/Stampante-Samsung/comandi/` or at the old
  names. The README has the two `ln -sf` commands.
- **Apple Silicon is assumed** throughout via the `/opt/homebrew` prefix. Intel Macs use
  `/usr/local`.
- **macOS 26 ships no Samsung printer drivers**, so the Settings "add printer" wizard finds
  nothing and the queue must be created manually with `lpadmin`. CUPS's
  `Printer drivers are deprecated` warning is expected and harmless.
- **No AirScan/eSCL and no duplex ADF.** Only Samsung's proprietary port 9400 works, via
  SANE's `xerox_mfp` backend. Double-sided originals need two passes.

## Recreating the print queue

```sh
lpadmin -p Samsung_SCX4623FW -E \
  -v "socket://$SCX_IP:9100" \
  -m drv:///sample.drv/generpcl.ppd \
  -D "Samsung SCX-4623FW" -L "Local network" \
  -o PageSize=A4 -o Resolution=600dpi -o Option1=True \
  -o printer-is-shared=false
lpoptions -d Samsung_SCX4623FW
```

Common remedies: paused queue → `cupsenable Samsung_SCX4623FW`; stuck jobs →
`cancel -a Samsung_SCX4623FW`; garbage or silent output → wrong driver, recreate the queue.
A major macOS upgrade can wipe CUPS queues.

## Shell style

- `#!/bin/bash`, 2-space indent, `then`/`do` on the same line.
- Configuration as uppercase constants at the top of the file.
- `getopts` for arguments, usage text in a `cat <<'EOF'` heredoc.
- Temp dirs via `TMP=$(mktemp -d)` with `trap 'rm -rf "$TMP"' EXIT`.
- Prefer graceful fallback chains (`img2pdf … || img2pdf …`) over hard failure.
- `scx-scan` uses `set -uo pipefail` deliberately without `-e`, because it handles its own
  failures with `||`.
- The `SC2086` at `set -- $entry` in `bin/scx-diagnose` is suppressed by a
  `# shellcheck disable` directive. That word splitting is intentional — it splits
  `"9100 print"` into port and label. Do not quote it.
- A `PostToolUse` hook runs `shfmt -i 2 -ci -bn -w` then `shellcheck` on any file edited
  under `bin/`. Both scripts are currently clean under both tools; keep them that way.

## Runtime dependencies

Declared only in the README's Installation section: `brew install sane-backends img2pdf`.
`scanimage` comes with `sane-backends`. The rest are macOS built-ins: `lpadmin`,
`lpstat`, `lpoptions`, `cupsenable`, `cancel`, `sips`, `open`, `nc`, `/sbin/ping`,
`readlink`, `dirname`.
