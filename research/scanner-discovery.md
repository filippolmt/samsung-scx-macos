# Scanner discovery and addressing without editing `xerox_mfp.conf`

Research for issue [#6](https://github.com/filippolmt/samsung-scx-macos/issues/6)
(map: [#2](https://github.com/filippolmt/samsung-scx-macos/issues/2)).

Question: can the app find the SCX-4623FW on the LAN and scan from it without the user
editing `/opt/homebrew/etc/sane.d/xerox_mfp.conf`? Does `scanimage -d 'xerox_mfp:tcp <ip>'`
bypass that file? Does the printer announce itself over Bonjour/mDNS, SNMP or anything else?

Addresses below use the placeholder `192.168.1.50`.

## Short answer

1. **`-d 'xerox_mfp:tcp <ip>'` does not bypass the config file.** The backend opens only
   device names that it read from `xerox_mfp.conf`. A name that is not in the file fails
   with `SANE_STATUS_INVAL` ("Invalid argument"). Verified in the source; not tested on
   the device.
2. **The app does not need to touch the Homebrew file.** SANE finds every config file
   through `SANE_CONFIG_DIR`, so the app can write its own one-line `xerox_mfp.conf` to a
   temp dir and point `SANE_CONFIG_DIR` at it for that one `scanimage` call. Verified in
   the source; not tested on the device.
3. **SANE has no network discovery for this backend.** The man page says multicast
   autoconfiguration "is not implemented yet". In the code the `tcp` hook is a TODO stub.
   The app has to find the IP address itself.
4. **The printer does announce itself.** Samsung's user guide lists Bonjour, SLP, UPnP and
   SNMP v1/2/3. The app can find the printer with Bonjour (CUPS `dnssd`, `dns-sd`,
   `ippfind`) or with an SNMP broadcast, then use that address for the scanner on port
   9400. Nobody has captured what this particular unit announces yet: that needs the
   device (commands below).

## 1. How xerox_mfp resolves a device name

Sources: sane-backends at commit
[`cadda80b`](https://gitlab.com/sane-project/backends/-/tree/cadda80b9fec0d69728561aeae57eb8278f99c75)
(master on 2026-09-22).

- **The dll meta-backend splits the name at the first `:`.** It loads the backend
  `xerox_mfp` and passes it `tcp 192.168.1.50`
  ([`backend/dll.c` `sane_open`, L1264–1397](https://gitlab.com/sane-project/backends/-/blob/cadda80b9fec0d69728561aeae57eb8278f99c75/backend/dll.c#L1264)).
- **xerox_mfp's `sane_open` builds the device list first, then matches against it.** On
  first use it calls `sane_get_devices`. It then compares the requested name with
  `strcmp` against each entry of `devices_head`. If nothing matches it returns
  `SANE_STATUS_INVAL`. It never opens a name that is missing from the list
  ([`backend/xerox_mfp.c` L1173–1200](https://gitlab.com/sane-project/backends/-/blob/cadda80b9fec0d69728561aeae57eb8278f99c75/backend/xerox_mfp.c#L1173)).
- **The device list comes only from the config file.** `sane_get_devices` calls
  `sanei_configure_attach("xerox_mfp.conf", …)`, and each non-comment line becomes a
  device name. `tcp …` lines go to `tcp_configure_device`
  ([L1120–1160](https://gitlab.com/sane-project/backends/-/blob/cadda80b9fec0d69728561aeae57eb8278f99c75/backend/xerox_mfp.c#L1120)).
  Before listing a device, `list_one_device` opens it and runs an INQUIRY. A line whose
  scanner does not answer never makes it into the list
  ([L1042–1084](https://gitlab.com/sane-project/backends/-/blob/cadda80b9fec0d69728561aeae57eb8278f99c75/backend/xerox_mfp.c#L1042)).
- **The `tcp` hook does no discovery.** `tcp_configure_device` contains a
  `TODO: LAN scanners multicast discovery. devname would contain "tcp auto"` and just
  passes the line on
  ([`backend/xerox_mfp-tcp.c` L170–181](https://gitlab.com/sane-project/backends/-/blob/cadda80b9fec0d69728561aeae57eb8278f99c75/backend/xerox_mfp-tcp.c#L170)).
  The man page agrees: "Multicast autoconfiguration for LAN scanners is not implemented
  yet" ([sane-xerox_mfp(5)](http://www.sane-project.org/man/sane-xerox_mfp.5.html),
  LIMITATIONS).
- **The name must match the config line exactly, byte for byte.** Leading and trailing
  whitespace is stripped from config lines
  ([`sanei/sanei_config.c` `sanei_config_read`, L207](https://gitlab.com/sane-project/backends/-/blob/cadda80b9fec0d69728561aeae57eb8278f99c75/sanei/sanei_config.c#L207)).
  Nothing else is normalised, so `tcp 192.168.1.50 9400` in the file does **not** match
  `-d 'xerox_mfp:tcp 192.168.1.50'`. When no port is given, it defaults to `9400`
  ([`xerox_mfp-tcp.c` L96–121](https://gitlab.com/sane-project/backends/-/blob/cadda80b9fec0d69728561aeae57eb8278f99c75/backend/xerox_mfp-tcp.c#L96)).
- **The host may be a name.** `sanei_tcp_open` resolves it with `gethostbyname`
  ([`sanei/sanei_tcp.c` L66–84](https://gitlab.com/sane-project/backends/-/blob/cadda80b9fec0d69728561aeae57eb8278f99c75/sanei/sanei_tcp.c#L66)).
  The man page says the host "is passed through resolver, thus can be a dotted quad or a
  name from /etc/hosts or resolvable through DNS". On macOS the system resolver also
  answers `.local` Bonjour names, so a line such as `tcp <bonjour-name>.local` should
  survive DHCP address changes. **Needs on-device verification.**

So the repo's current setup works only because the README tells the user to append
`tcp $SCX_IP` to the Homebrew file. `bin/scx-scan`'s `-d` string has to match that
line, and it would fail if the line were missing or written differently.

## 2. Addressing without the Homebrew file: `SANE_CONFIG_DIR`

- **Search order.** Every backend finds its config file through
  `sanei_config_get_paths`. The search path is `$SANE_CONFIG_DIR` if set, otherwise
  `.:<sysconfdir>/sane.d`. If `SANE_CONFIG_DIR` ends in `:`, the default dirs are
  appended after it
  ([`sanei/sanei_config.c` L68, L77–127](https://gitlab.com/sane-project/backends/-/blob/cadda80b9fec0d69728561aeae57eb8278f99c75/sanei/sanei_config.c#L77)).
  `sanei_config_open` takes the **first** file it finds and ignores the rest
  (L130–165). Documented in
  [sane-dll(5)](http://www.sane-project.org/man/sane-dll.5.html), ENVIRONMENT:
  "setting SANE_CONFIG_DIR to "/tmp/config:" would result in directories tmp/config, .,
  and /usr/local/etc/sane.d being searched (in this order)."
- **So the app can bring its own config.** It writes a private `xerox_mfp.conf` holding
  only `tcp 192.168.1.50` and runs scanimage with `SANE_CONFIG_DIR="$TMP:"`. The trailing
  colon matters: without it the app would also need its own `dll.conf`. `dll.conf` and
  the other backends' files are still read from the Homebrew dir, and only
  `xerox_mfp.conf` is overridden:

  ```sh
  TMP=$(mktemp -d); trap 'rm -rf "$TMP"' EXIT
  printf 'tcp %s\n' "$SCX_IP" > "$TMP/xerox_mfp.conf"
  SANE_CONFIG_DIR="$TMP:" scanimage -d "xerox_mfp:tcp $SCX_IP" --source Flatbed …
  ```

  This removes the second copy of the address (the gotcha in `CLAUDE.md`). A
  `sane-backends` reinstall can then no longer break scanning.
- **The private file replaces the Homebrew one.** Its USB lines are therefore not read,
  which is fine for a network-only tool.
- **Side note.** The default path includes `.`, the working directory. A stray
  `xerox_mfp.conf` in whatever directory the user runs `scx-scan` from would take
  precedence over the Homebrew one. Setting `SANE_CONFIG_DIR` explicitly also rules this
  out.
- **The same approach should work with the SANE C API.** `SANE_CONFIG_DIR` is read once,
  on the first config lookup (`dir_list` is cached). An app that links libsane directly
  would have to set it in the environment before calling `sane_init`/`sane_get_devices`.
  **Not tested on the device.**

## 3. What the printer offers for discovery

Samsung, *SCX-4600 Series / SCX-4623 Series Multi Functional Printer User's Guide*
(hosted by HP, which took over Samsung's printer business):
[c05786777.pdf](https://h10032.www1.hp.com/ctg/Manual/c05786777.pdf).

- **Supported protocols.** Page 35, "Network environment", lists: TCP/IPv4, DHCP,
  BOOTP, **DNS, WINS, Bonjour, SLP, UPnP**, RAW, LPR, IPP, **SNMPv1/2/3**, HTTP(S), IPSec,
  and IPv6 with the same set.
- **Samsung's own Mac scanning path used Bonjour.** Page 66: for Mac OS X 10.5 Image
  Capture, the guide says to tick the machine under "Bonjour Devices". Samsung's own scan
  driver was installed for this, so the guide does not say which Bonjour service type
  carried the scanner, or whether the firmware advertises a scanner service at all.
- **Samsung's Scan Manager had "Auto detection on the network"** (pages 65 and 83), but
  the guide does not document the mechanism.

What an app can use on macOS, all standard and built in:

| Mechanism | Tool | Source | What it gives |
|---|---|---|---|
| Bonjour / DNS-SD | `dns-sd -B`, `dns-sd -L`, `ippfind`, `lpinfo --include-schemes dnssd -v` | [CUPS: Using network printers](https://www.cups.org/doc/network.html), [ippfind(1)](https://www.cups.org/doc/man-ippfind.html), [lpinfo(8)](https://www.cups.org/doc/man-lpinfo.html) | service instance name, `.local` host name, port, TXT record (`ty`, `product`, `adminurl`, …) |
| SNMP v1 broadcast | CUPS `snmp` backend, `lpinfo --include-schemes snmp -v` | [CUPS: Finding printers using SNMP](https://www.cups.org/doc/network.html), [snmp.conf(5)](https://www.cups.org/doc/man-cups-snmp.conf.html) | IP address, make and model from Host-Resources MIB `hrDeviceDescr` (.1.3.6.1.2.1.25.3.2.1.3.1) |

The app could pick the printer whose Bonjour `ty`/`product` or SNMP description contains
`SCX-4623` and take its address or `.local` name. It would then write the private
`xerox_mfp.conf` from §2 and confirm with `scanimage -L`, which runs the INQUIRY: the
reply's model string (`Samsung SCX-4623FW Serie`, as shown in the README) proves that
port 9400 speaks the xerox_mfp protocol. The repo already notes (`CLAUDE.md`, "No
AirScan/eSCL") that there is no `_uscan._tcp` scanner service to discover directly.

**Not known without the device:**

- which service types this unit actually registers (`_printer._tcp`,
  `_pdl-datastream._tcp`, `_ipp._tcp`, `_http._tcp`, a scanner type?);
- its TXT keys and host name;
- whether SNMP is enabled with the `public` community by default.

## 4. Commands for the owner to confirm on macOS

Run these with the printer on and on the same LAN. Redact the real addresses, MAC and
host names before pasting any output into an issue.

```sh
# A. Confirm -d does NOT bypass the config file (expect "Invalid argument"):
SANE_CONFIG_DIR=/nonexistent scanimage -d "xerox_mfp:tcp $SCX_IP" -n --source Flatbed

# B. Confirm a private config dir is enough (expect the device listed, then a dry run succeeds):
TMP=$(mktemp -d); printf 'tcp %s\n' "$SCX_IP" > "$TMP/xerox_mfp.conf"
SANE_CONFIG_DIR="$TMP:" scanimage -L
SANE_CONFIG_DIR="$TMP:" scanimage -d "xerox_mfp:tcp $SCX_IP" -n --source Flatbed; echo "exit $?"
SANE_DEBUG_XEROX_MFP=4 SANE_CONFIG_DIR="$TMP:" scanimage -L 2>&1 | grep -i 'config\|tcp'

# C. Exact-match check (expect failure: config has an explicit port, -d does not):
printf 'tcp %s 9400\n' "$SCX_IP" > "$TMP/xerox_mfp.conf"
SANE_CONFIG_DIR="$TMP:" scanimage -d "xerox_mfp:tcp $SCX_IP" -n --source Flatbed; echo "exit $?"

# D. What does the printer announce over Bonjour? (Ctrl-C each after a few seconds)
dns-sd -B _services._dns-sd._udp local.
dns-sd -B _printer._tcp local.
dns-sd -B _pdl-datastream._tcp local.
dns-sd -B _ipp._tcp local.
dns-sd -B _scanner._tcp local.
dns-sd -B _uscan._tcp local.
dns-sd -L "<instance name from above>" _pdl-datastream._tcp local.   # TXT record + host name
ippfind --ls

# E. What CUPS discovers (dnssd and snmp backends):
lpinfo --include-schemes dnssd,snmp -v
CUPS_DEBUG_LEVEL=2 /usr/libexec/cups/backend/snmp @LOCAL 2>&1 | tail -40

# F. Does a Bonjour host name work as the SANE host? (replace with the .local name from D)
printf 'tcp %s\n' "<bonjour-name>.local" > "$TMP/xerox_mfp.conf"
SANE_CONFIG_DIR="$TMP:" scanimage -L
rm -rf "$TMP"
```

The expected results for A–C follow from the source cited in §1–2. D–F are open
questions that only the device can answer.
