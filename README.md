# Device Audit Scripts

Collects device/inventory information from employee computers and writes it to a
Markdown (`.md`) report. Two scripts produce the **same report layout**, so
Linux, macOS and Windows results can be compared side by side.

| File | Platform | Requirements |
| --- | --- | --- |
| `device-audit.sh` | Linux, macOS, other Unix | `bash`, standard coreutils |
| `DeviceAudit.ps1` | Windows | Windows PowerShell 5.1 or PowerShell 7+ |

## What gets collected

- **Overview** – OS name/version/build, kernel, architecture, host name, user, uptime, boot time, timezone, locale
- **Hardware** – model, serial number, CPU, cores, RAM, firmware/BIOS, UUID, asset tag, virtualization
- **Storage** – physical disks, partitions/volumes, file systems, per-volume usage
- **Network** – adapters, MAC addresses, IPv4/IPv6, gateway, DNS, domain, Wi-Fi SSID, proxy
- **Displays** – resolution, GPU/chipset, VRAM
- **Battery** – charge, health, cycle count (laptops only)
- **Security** – FileVault/BitLocker, firewall, SELinux/AppArmor, SIP/Gatekeeper, Defender, screen lock, admins
- **Software** – package/app counts, last update, plus a full app list with `-s` / `-IncludeSoftware`

Anything that cannot be read is reported as `_n/a_` instead of being omitted, so
every report has the same shape.

## Usage

### Linux / macOS

```bash
./device-audit.sh                      # writes device-audit-<host>-<timestamp>.md
./device-audit.sh -o report.md         # choose the output file
./device-audit.sh -s                   # also list every installed package/app
./device-audit.sh -s -q -o report.md   # quiet, with the full software list
./device-audit.sh -h                   # help
```

### Windows

```powershell
# Right-click PowerShell -> "Run as administrator" is recommended
.\DeviceAudit.ps1
.\DeviceAudit.ps1 -OutputFile .\report.md
.\DeviceAudit.ps1 -IncludeSoftware
```

If the script cannot run directly, allow local scripts for the current user:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

## Requirements and what is NOT required

**Nothing needs to be installed.** Both scripts use only tools that ship with the
operating system. If a tool is missing, the script still runs and reports
`_n/a_` for that field — it never fails because of a missing command.

| Platform | Really needed | Everything else is optional |
| --- | --- | --- |
| Linux / macOS | `bash`, `awk`, `sed`, `grep`, `tr`, `wc`, `cut`, `head`, `sort`, `uname`, `mktemp` | `lsblk`, `ip`, `df`, `nproc`, `xrandr`, `system_profiler`, `diskutil`, `pmset`, `fdesetup`, `scutil`, `dpkg-query`/`rpm`, `snap`, `flatpak`, `getenforce`, … |
| Windows | PowerShell 5.1+ (built in since Windows 10) | WMI/CIM cmdlets, `Get-NetFirewallProfile`, `Get-BitLockerVolume`, `Get-MpComputerStatus` (need admin for some) |

Notes on the optional tools:

- **Linux**: `lsblk` (from `util-linux`) gives disk details; without it the script
  falls back to `df`. `ip` (from `iproute2`) gives interface details; without it
  it falls back to `ifconfig`.
- **macOS**: uses the built-in `system_profiler`, `diskutil`, `pmset`, `scutil`.
  `lsblk`/`ip`/`df -hP` do **not** exist on macOS and are never required there.
- **The `-s` software list is capped at 400 packages** on Linux so the report
  stays readable.

### Not universal — be aware

- The **bash script covers Linux and macOS only**. It is not tested on BSD, Solaris,
  or busybox-based appliances, and the `IS_MAC`/`IS_LINUX` split means other
  Unix systems fall through to the generic branch with many `_n/a_` values.
- The **PowerShell script is unverified at runtime** — it was written against
  documented cmdlets but could not be executed in the build environment. Test it
  on one Windows machine before rolling it out.
- Values that need root/administrator (serial number, BIOS revision, BitLocker,
  SELinux status) are `_n/a_` for standard users. This is a permissions limit,
  not a bug.

## Notes and behaviour

- **No installation required, and no admin rights needed to run it.** However,
  running elevated gives noticeably better data: serial number, BIOS revision,
  BitLocker and SELinux status are `_n/a_` for a standard user. The report header
  records which privilege level it ran at.
- **It never hangs.** Every command runs with stdin closed, and known blocking
  commands (`dsconfigstatus`, `softwareupdate` on macOS) are killed after
  `DEVICE_AUDIT_TIMEOUT` seconds (default 3). On a real Linux or macOS machine
  the script finishes in a few seconds.
- **Safe to run on a laptop** - it only reads system information and writes one
  text file. It makes no changes to the machine.
- **Distribution**: copy the script to the machine (or run it from a USB/shared
  drive) and have the user run it, then collect the resulting `.md` file.

## Privacy note

The report contains identifiers such as serial number, MAC address and the
current user name. Treat the output as confidential and store it according to
your company's device-data policy.
