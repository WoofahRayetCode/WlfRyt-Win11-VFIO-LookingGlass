# Windows 11 gaming VM on CachyOS (VFIO + Looking Glass IDD)

This document describes the **working** Windows 11 gaming VM on this HP OMEN
laptop (AMD Ryzen 9 8940HX + Radeon 610M host, NVIDIA GeForce RTX 5060 Max-Q
guest). It covers:

1. How the stack works  
2. What we did chronologically (including failures)  
3. Troubleshooting that got us to a healthy state  
4. Daily use  
5. How to rebuild from scratch  

Last verified healthy: **2026-09-04 evening** (Looking Glass IDD on RTX 5060;
BattlEye hardening + 1 TiB disk expand + guest identity checks recorded below).

> **GitHub project folder:** [`~/Documents/GitHub/WlfRyt-Win11-VFIO-LookingGlass/`](file:///home/ericparsley/Documents/GitHub/WlfRyt-Win11-VFIO-LookingGlass/)  
> **Status card:** [`STATUS.txt`](STATUS.txt) (also linked from `~/Desktop/win11-vm-STATUS.txt`)  
> **Desktop runbook link:** [`~/Desktop/win11-vm-README.md`](file:///home/ericparsley/Desktop/win11-vm-README.md) → this README

Canonical docs live in the `WlfRyt-Win11-VFIO-LookingGlass` GitHub folder. Desktop and `/data/vms/win11/README.md` are convenience links/copies.

---

## 1. Hardware and roles

| Role | Hardware / path |
|------|-----------------|
| Host OS | CachyOS (Btrfs `subvol=/@`), Limine bootloader, kernel `linux-cachyos` |
| Host display + desktop audio | AMD Raphael iGPU `AMD Radeon 610M` (`amdgpu`) |
| Guest dGPU | `01:00.0` NVIDIA GB206M **RTX 5060 Max-Q** `[10de:2d19]` (HP subsystem `103c:8e35`) |
| Guest dGPU HDMI audio (stubbed, not in VM) | `01:00.1` `[10de:22eb]` — same **IOMMU group 15** as the GPU |
| Guest disk | `/data/vms/win11/win11.qcow2` on Sabrent 2TB `/data` (**1 TiB** virtual; ~42–47 GiB sparse host usage) |
| Guest RAM / CPU | ~14 GiB; 12 vCPU = 1 socket × 6 cores × 2 threads |
| Capture path | Looking Glass **IDD** (B7-826) → IVSHMEM `/dev/shm/looking-glass` (128 MiB) → `looking-glass-client` |
| Install ISOs (on disk, not attached) | `/data/vms/iso/Win11_English_x64.iso`, `virtio-win.iso` |

**Important laptop constraint:** this is Optimus/MUX-class hardware. The host
must keep the desktop on the AMD iGPU. The NVIDIA device is dedicated to the VM
via VFIO and must **not** drive the host session.

---

## 2. How it works (final architecture)

```
┌─────────────────────────────────────────────────────────────┐
│ CachyOS host                                                │
│  • Desktop / compositor on AMD Radeon 610M                  │
│  • looking-glass-client reads /dev/shm/looking-glass        │
│  • SPICE (optional backup) on 127.0.0.1:5900                │
│  • 01:00.0 + 01:00.1 bound to vfio-pci at boot              │
└───────────────────────────┬─────────────────────────────────┘
                            │ libvirt / QEMU / KVM
┌───────────────────────────▼─────────────────────────────────┐
│ Domain: win11                                               │
│  • OVMF Secure Boot + swtpm                                 │
│  • VirtIO disk, e1000e NIC, QXL kept for emergency SPICE    │
│  • hostdev: GPU 01:00.0 only (guest PCI bus 1 / slot 0)     │
│  • 01:00.1 stays on host vfio (not attached to guest)       │
│  • ivshmem-plain “looking-glass” 128 MiB                    │
│  • Fake battery ACPI SSDT (mobile NVIDIA Code 43 workaround)│
│  • CPU: host-passthrough + topoext + hypervisor=off         │
│  • kvm hidden + Hyper-V enlightenments, vendor_id=AuthenticAMD │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│ Windows 11 guest                                            │
│  • NVIDIA Game Ready driver owns RTX 5060                   │
│  • Looking Glass IDD creates virtual monitor on that GPU    │
│  • IDD writes frames into IVSHMEM                           │
│  • “Make LG the only monitor” → QXL/SPICE often goes black  │
└─────────────────────────────────────────────────────────────┘
```

### Boot-time VFIO (not live unbind)

`01:00.0` and `01:00.1` share IOMMU group 15. Live unbind of the HDMI audio
function from `snd_hda_intel` **hangs this host in D-state**. Therefore:

- Both PCI IDs are bound to `vfio-pci` **at boot** via:
  - `/etc/modprobe.d/vfio-nvidia-passthrough.conf`
  - Limine cmdline: `vfio-pci.ids=10de:2d19,10de:22eb`
- Host desktop never loads the NVIDIA stack (by design).
- Libvirt hooks still exist but, in boot-bound mode, mostly no-op / verify.

### Looking Glass IDD vs legacy Host

We tried the **legacy Looking Glass Host** first. It failed because DXGI/D12
only saw `Microsoft Basic Render Driver` (QXL) while NVIDIA had no usable
output for capture. **Looking Glass IDD** creates a virtual display that can
use the NVIDIA adapter as its render GPU — that is the working path.

Matching versions matter: host client is `looking-glass-git` **B7-826**; guest
IDD installer must be the matching bleeding **B7-826** build.

### Why QXL remains

QXL + SPICE stay in the domain as a **recovery console**. Once IDD enforces
“Looking Glass as the only monitor,” the SPICE window (`spicy`) is often black.
That is expected. Use Looking Glass as the primary view.

---

## 3. Chronological build log (what we actually did)

### Phase A — Install Windows without the dGPU

1. Created libvirt domain `win11`, OVMF + TPM, VirtIO disk, SPICE/QXL.  
2. Attached Windows + virtio-win ISOs.  
3. Installed Windows 11; loaded VirtIO storage/network drivers from the second CD.  
4. **Passthrough marker absent** so hooks would not touch NVIDIA during install.

### Early VFIO attempts (pain points)

1. **Dynamic GPU bind** while host still had NVIDIA modules → host UI freeze
   (audio kept playing). Bind was made **opt-in** via
   `/data/vms/win11/ENABLE_GPU_PASSTHROUGH`.  
2. GPU-only live bind worked; **audio live unbind hung** (D-state).  
3. Conclusion: boot-time `vfio-pci.ids` for **both** functions in group 15.

### Phase B — Attach GPU for the guest

1. Reboot into top-level CachyOS (`subvol=/@`, not a snapshot).  
2. Verified both NVIDIA functions on `vfio-pci`; host GL on AMD.  
3. Created `ENABLE_GPU_PASSTHROUGH`, redefined domain with GPU hostdev(s) +
   Looking Glass shmem, started VM.  
4. Installed NVIDIA Game Ready drivers in the guest (via SPICE/QXL).

### Looking Glass path

1. SPICE clipboard needed **spice-guest-tools** in Windows (`vdagent`
   channel was disconnected until then).  
2. Installed **legacy Looking Glass Host** → client waited forever:
   shared memory empty; Host log:
   `Failed to locate a valid output device` / only Basic Render Driver.  
3. Installed **Looking Glass IDD (B7-826)** instead; allowed disabling the
   old Host.  
4. IDD made SPICE go black; Looking Glass showed FPS but a black picture until
   PIN entry — login UI can be black while still accepting input.  
5. Client reported IDD in **software mode** (“no GPU found”) while Device
   Manager showed NVIDIA with a yellow bang.

### Code 43 / mobile NVIDIA fixes

Applied in order until the **display adapter** reported `ProblemCode=0`:

| Fix | Purpose |
|-----|---------|
| `kvm` hidden + Hyper-V `vendor_id=AuthenticAMD` | Hide obvious KVM / spoof CPU vendor for Hyper-V leaves |
| CPU `hypervisor` feature **disabled** | Clear hypervisor CPUID bit NVIDIA checks |
| **GPU-only** hostdev (do **not** attach `01:00.1`) | RTX 50-series reports of audio+GPU together breaking the driver |
| Guest GPU at **PCI bus 1 slot 0** | Placement some Optimus guides require |
| Fake battery **ACPI SSDT** (`SSDT1.dat`) | Max-Q drivers refuse to start without a battery |
| Dual start after config changes | Avoid false failures after XML changes |

VBIOS dump via sysfs ROM while the device is on `vfio-pci` **failed** (I/O
error) — common on this laptop; we did not need a romfile once the battery SSDT
+ hypervisor hide landed.

### Device Manager cleanup

- Yellow bang that remained was often **NVIDIA Platform Controllers and
  Framework (nvpcf) Code 31** — laptop EC/Dynamic Boost helper; **not** the
  GPU. Disabled it (shows Problem 22 = disabled by user).  
- Ghost duplicate RTX 5060 / old LSI SCSI / NVIDIA HD Audio devices removed.  
- Leftover **LSI Logic SCSI** controller (from an early ISO hotplug) removed
  from the libvirt domain.

### Final IDD reload

Even with RTX 5060 OK, IDD had started earlier in software mode. Reloading IDD
(toggle Looking Glass Indirect Display + restart helper service) produced:

```text
Selected render adapter NVIDIA GeForce RTX 5060 Laptop GPU (vendor 0x10de, device 0x2d19)
```

That is the healthy end state.

### ISO cleanup

Install ISOs and the temporary Looking Glass tools CD were ejected/detached from
the domain. Files remain under `/data/vms/iso/` if needed later.

---

## 4. Current host configuration (authoritative)

### Kernel cmdline (Limine — use stock CachyOS entry, not Snapshots)

Must include roughly:

```text
vfio-pci.ids=10de:2d19,10de:22eb amd_iommu=on iommu=pt ... rootflags=subvol=/@ ...
```

Verify after boot:

```bash
cat /proc/cmdline
lspci -nnk -d 10de:    # both should say: Kernel driver in use: vfio-pci
glxinfo -B | head      # AMD Radeon, not NVIDIA
```

### `/etc/modprobe.d/vfio-nvidia-passthrough.conf`

```conf
# Bind RTX 5060 Max-Q + HD Audio to vfio-pci at boot (IOMMU group 15).
# Dynamic unbind of 01:00.1 from snd_hda_intel hangs this host in D-state.
options vfio-pci ids=10de:2d19,10de:22eb disable_vga=1
softdep nvidia pre: vfio-pci
softdep nvidia_drm pre: vfio-pci
softdep nvidia_modeset pre: vfio-pci
softdep snd_hda_intel pre: vfio-pci
```

After editing: rebuild initramfs (`mkinitcpio -P` on this system) and reboot.

### Libvirt hooks

```text
/etc/libvirt/hooks/qemu.d/win11/prepare/begin/01-vfio-bind.sh
/etc/libvirt/hooks/qemu.d/win11/release/end/01-vfio-unbind.sh
```

Bind hook exports `VFIO_BOOT_BOUND=1` / `VFIO_SKIP_AUDIO=1` and calls
`/usr/local/bin/vfio-nvidia-bind.sh` (no-ops if already on vfio, still gated by
the marker file).

Unbind hook **leaves** devices on vfio when the modprobe conf exists (AMD
desktop stays correct).

### Passthrough marker

```bash
/data/vms/win11/ENABLE_GPU_PASSTHROUGH   # empty file; presence = opt-in
```

### Looking Glass client

Package: `looking-glass-git` (B7-826).

Config: `~/.looking-glass-client.ini`

```ini
[lgmp]
shmDevice=/dev/shm/looking-glass

[win]
fullScreen=yes
borderless=yes
showFPS=yes
alerts=yes

[input]
escapeKey=KEY_RIGHTCTRL
hideCursor=no
captureOnStart=no
rawMouse=yes
alwaysShowCursor=yes

[egl]
mapHDRtoSDR=yes
nvGain=0
peakLuminance=250

[spice]
input=yes
clipboard=yes
```

Escape / release capture: **Right Ctrl**.

### Critical guest XML features (summary)

```xml
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
  ...
  <features>
    <hyperv mode='custom'>
      ...
      <vendor_id state='on' value='AuthenticAMD'/>
      ...
    </hyperv>
    <kvm>
      <hidden state='on'/>
    </kvm>
  </features>
  <cpu mode='host-passthrough' check='none' migratable='on'>
    <topology sockets='1' dies='1' clusters='1' cores='6' threads='2'/>
    <cache mode='passthrough'/>
    <feature policy='require' name='topoext'/>
    <feature policy='disable' name='hypervisor'/>
  </cpu>
  ...
  <!-- GPU only — do not attach 01:00.1 -->
  <hostdev mode='subsystem' type='pci' managed='no'>
    <driver name='vfio'/>
    <source>
      <address domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
    </source>
    <address type='pci' domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
  </hostdev>
  <shmem name='looking-glass'>
    <model type='ivshmem-plain'/>
    <size unit='M'>128</size>
  </shmem>
  <qemu:commandline>
    <qemu:arg value='-acpitable'/>
    <qemu:arg value='file=/data/vms/win11/acpi/SSDT1.dat'/>
  </qemu:commandline>
</domain>
```

Dump live config anytime:

```bash
sudo virsh dumpxml win11 > /data/vms/win11/win11-live-$(date +%Y%m%d).xml
```

---

## 5. Daily use

```bash
# Confirm host still on AMD + NVIDIA on vfio (optional sanity)
lspci -nnk -d 10de: | rg 'Kernel driver'
glxinfo -B | rg -i renderer

sudo virsh start win11
looking-glass-client
```

- Shut down from **inside Windows** (or `sudo virsh shutdown win11`).  
- Do **not** expect NVIDIA to return to the host; it stays on vfio until you
  deliberately undo boot-time binding and reboot.  
- Prefer Looking Glass over `spicy`. SPICE black screen ≠ VM dead.

### Black login screen

IDD login UI can render black while still accepting input. Click Looking Glass,
type PIN/password, Enter. Night-vision gain (`egl:nvGain=1`) can help see a dim
login once; set back to `0` afterward.

---

## 6. Troubleshooting playbook

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Host freezes when starting VM / binding NVIDIA | Live module unload / wrong bind order | Reboot; use boot-time vfio ids; keep marker discipline |
| Live unbind of `01:00.1` hangs | IOMMU group 15 + `snd_hda_intel` D-state | Never live-unbind audio; boot-bind both IDs |
| `virsh` empty / missing hooks after boot | Booted a **Btrfs snapshot** | Reboot Limine → **CachyOS / linux-cachyos** (`subvol=/@`) |
| No copy/paste into SPICE | Missing spice-guest-tools | Install spice guest tools; only one SPICE client at a time |
| LG client: “Waiting for the source…” | Guest IDD/Host not writing IVSHMEM | Check IDD service/logs under `%ProgramData%\Looking Glass (IDD)\` |
| Legacy Host: no output device | Capturing QXL / no NVIDIA output | Use **Looking Glass IDD**, not legacy Host |
| SPICE black, LG has FPS but black | Desktop on IDD; login black | Type PIN blind; or check IDD “only monitor” topology |
| NVIDIA yellow bang / Code 43 | Mobile driver VM / battery / hypervisor checks | `hypervisor=off`, kvm hidden, fake battery SSDT, GPU-only, bus 1 |
| NVPCF Code 31 | Laptop platform helper in VM | Disable device (Problem 22 afterward is OK) |
| IDD “no GPU” / software mode while GPU OK | IDD started before driver ready | Reload IDD (toggle Indirect Display + helper) or reboot guest |
| Looking Glass mouse “dead” while waiting | Fullscreen client capturing with no frames | Right Ctrl; or kill client; use spicy until IDD healthy |
| Guest network weirdness | UFW vs `virbr0` | Allow virbr0; forward policy ACCEPT (see older note below) |

### UFW note (if DHCP dies)

If the guest gets no DHCP, host UFW may be blocking libvirt’s `virbr0`. Typical
fix: allow traffic on `virbr0`, allow forwarding between `virbr0` and the
uplink, set `DEFAULT_FORWARD_POLICY="ACCEPT"` in `/etc/default/ufw`, `ufw reload`.

### Useful guest log paths

```text
C:\ProgramData\Looking Glass (IDD)\looking-glass-idd.txt
C:\ProgramData\Looking Glass (IDD)\looking-glass-idd-service.txt
C:\Users\<you>\AppData\Local\Looking Glass (IDD)\looking-glass-idd-helper.txt
```

Healthy IDD line:

```text
Selected render adapter NVIDIA GeForce RTX 5060 Laptop GPU
```

Bad / software fallback:

```text
No hardware render adapter available; using SDR software mode
```

---

## 7. Rebuild guide (from scratch)

Approximate rebuild if the disk is wiped but host VFIO setup remains.

### 7.1 Host prerequisites

1. Packages: `qemu-desktop`, `libvirt`, `virt-manager` (optional), `edk2-ovmf`,
   `swtpm`, `looking-glass-git`, plus a SPICE client (`spicy` or virt-viewer).  
2. User in groups `libvirt` and `kvm`.  
3. IOMMU + vfio ids on cmdline and modprobe (section 4); rebuild initramfs; reboot
   into `subvol=/@`.  
4. Confirm AMD host + NVIDIA on vfio.  
5. Recreate hooks + `/usr/local/bin/vfio-nvidia-*.sh` (boot-bound variants).  
6. Recreate fake battery SSDT:

```bash
mkdir -p /data/vms/win11/acpi
echo 'U1NEVKEAAAAB9EJPQ0hTAEJYUENTU0RUAQAAAElOVEwYEBkgoA8AFVwuX1NCX1BDSTAGABBMBi5fU0JfUENJMFuCTwVCQVQwCF9ISUQMQdAMCghfVUlEABQJX1NUQQCkCh8UK19CSUYApBIjDQELcBcLcBcBC9A5C1gCCywBCjwKPA0ADQANTElPTgANABQSX0JTVACkEgoEAAALcBcL0Dk=' \
  | base64 -d > /data/vms/win11/acpi/SSDT1.dat
```

7. Client ini as in section 4.

### 7.2 Create disk + install domain

```bash
# Example — adjust size/path as needed
qemu-img create -f qcow2 /data/vms/win11/win11.qcow2 256G

# Define an install-time domain: OVMF secure boot, TPM, VirtIO disk,
# SPICE+QXL, Windows ISO + virtio-win ISO. Do NOT attach NVIDIA yet.
# Do NOT create ENABLE_GPU_PASSTHROUGH yet.
sudo virsh define /data/vms/win11/win11-install.xml   # or a fresh XML you write
sudo virsh start win11
spicy --uri=spice://127.0.0.1:5900
```

Install Windows; load VirtIO drivers; install `virtio-win-gt-x64.msi`;
spice-guest-tools for clipboard.

### 7.3 Switch to gaming profile

1. `sudo virsh shutdown win11` and wait for shut off.  
2. Edit domain (`sudo virsh edit win11` or define a prepared XML):
   - Add Looking Glass `<shmem>` 128M.  
   - Add **GPU-only** `<hostdev>` for `01:00.0` at guest `bus=0x01 slot=0x00`.  
   - Do **not** add `01:00.1`.  
   - CPU: `topoext` require + `hypervisor` disable.  
   - Features: kvm hidden + hyperv vendor_id AuthenticAMD.  
   - `xmlns:qemu` + `-acpitable file=/data/vms/win11/acpi/SSDT1.dat`.  
   - Keep QXL for recovery (or later set video model `none` if you accept no SPICE framebuffer).  
3. `touch /data/vms/win11/ENABLE_GPU_PASSTHROUGH`  
4. Start VM; install NVIDIA drivers via SPICE.  
5. Install **Looking Glass IDD matching the client** (B7-826 bleeding on this box).  
   Prefer IDD over legacy Host. Allow “disable old host”. Reboot guest if asked.  
6. `looking-glass-client` on the host.  
7. If NVIDIA Code 43: re-check section 3 table; reboot guest twice after XML
   changes.  
8. If IDD software mode but GPU OK: reload IDD (Device Manager toggle Looking
   Glass Indirect Display + restart “Looking Glass IDD Helper”) or reboot Windows.  
9. Disable NVPCF if it shows Code 31. Remove ghost devices. Eject install ISOs.

### 7.4 Reference XML snapshots in this directory

| File | Notes |
|------|-------|
| `win11-install.xml` | Early install-oriented snapshot (may be stale vs live domain) |
| `win11-phaseB-with-gpu.xml` | Mid-project GPU+audio attempt snapshot |
| `gpu-passthrough-snippet.xml` | Snippet used during Phase B edits |
| `win11.xml.bak-before-detach-*` | Accidental-detach recovery |
| **Live truth** | `sudo virsh dumpxml win11` |

Prefer dumping the **live** domain after any successful change.

---

## 8. Files and paths cheat sheet

```text
/data/vms/win11/win11.qcow2                 Guest disk
/data/vms/win11/ENABLE_GPU_PASSTHROUGH      Opt-in marker
/data/vms/win11/acpi/SSDT1.dat              Fake battery SSDT
/data/vms/win11/README.md                   This document
/data/vms/iso/                              Windows / virtio / tools ISOs
/etc/modprobe.d/vfio-nvidia-passthrough.conf
/etc/libvirt/hooks/qemu.d/win11/...
/usr/local/bin/vfio-nvidia-bind.sh
/usr/local/bin/vfio-nvidia-unbind.sh
~/.looking-glass-client.ini
~/Desktop/win11-vm-STATUS.txt               Short status / daily commands
/dev/shm/looking-glass                      IVSHMEM (created when VM runs)
```

Guest:

```text
C:\Program Files\Looking Glass (IDD)\
C:\ProgramData\Looking Glass (IDD)\looking-glass-idd.txt
```

---



## 9. Known limitations (accept these)

1. NVIDIA stays on vfio on the host whenever boot-bound config is active.  
2. Guest has **no** NVIDIA HDMI audio function (GPU-only passthrough). Use
   Looking Glass / SPICE / VirtIO audio paths instead.  
3. NVPCF remains disabled — Dynamic Boost-class features won’t work in-VM.  
4. Login screen may be black under IDD.  
5. Anti-cheat / some online games may still detect virtualization; SMBIOS
   spoofing or dual-boot may be required for specific titles.  
6. Do not boot the host from Limine **Snapshots** when expecting this VFIO
   layout.

---

## 10. Quick health checklist

```bash
# Host
cat /proc/cmdline | tr ' ' '\n' | rg 'vfio|iommu|subvol'
lspci -nnk -d 10de:
glxinfo -B | rg -i 'OpenGL renderer'
ls -la /data/vms/win11/ENABLE_GPU_PASSTHROUGH
sudo virsh list --all
ls -la /dev/shm/looking-glass

# Guest (PowerShell) — expect RTX ProblemCode=0 and IDD log selecting NVIDIA
Get-PnpDevice -Class Display | Format-Table Status, FriendlyName
Get-Content 'C:\ProgramData\Looking Glass (IDD)\looking-glass-idd.txt' -Tail 40
```

When those are green and `looking-glass-client` shows a responsive desktop, the
VM is in the intended “use Windows as if bare metal” state for this machine.

## 11. BattlEye / GTA Online hardening (applied 2026-09-04)

**Honest limit:** No configuration can *guarantee* BattlEye will allow GTA Online.
BattlEye is kernel-mode, updates often, and Rockstar’s ToS disallow circumventing
anti-cheat. Treat this as **best-effort VM concealment** for a personal gaming VM.
Prefer a throwaway / secondary Rockstar account until you are confident.

Desktop summary of the same evening work:
`~/Desktop/win11-vm-STATUS.txt`.

### What we already had
- `kvm` hidden, `hypervisor` CPUID disabled, Hyper-V `vendor_id=AuthenticAMD`
- GPU passthrough (harder for AC to dismiss as a soft VM)
- Looking Glass IDD (not a VNC-only toy VM)

### What we added for GTA V / BattlEye
| Change | Why |
|--------|-----|
| SMBIOS `sysinfo` spoofed as this HP OMEN (AMI F.13 / HP 8E35) | Removes default QEMU/BOCHS DMI strings |
| NIC MAC `ac:f4:66:28:f0:3d` (not `52:54:00` QEMU OUI) | Common easy detection |
| Disk serial + later ATA model/ver override | Avoids QEMU disk identity strings |
| Hyper-V: reset, reenlightenment, tlbflush, ipi, stimer-direct, ioapic=kvm, pmu | Looks more like Hyper-V-enlightened bare metal / hides KVM |
| QXL kept (video none tried, reverted) | `video none` removed the only non-IDD console and left us blind when IDD was slow; QXL stays for recovery. IDD “only monitor” still hides it from the desktop. |
| Removed virtio balloon (`model='none'`) | Minor paravirt tell |
| Fake-battery SSDT OEM IDs `HPQOEM`/`BATTERY` (was `BOCHS`/`BXPC`) | ACPI OEM string cleanup |
| **System disk VirtIO → SATA/AHCI** (`sda` / `ide-hd`) | Removes active “Red Hat VirtIO Disk” fingerprint |
| USB tablet input removed | Common VM HID tell; PS/2 + Looking Glass input remain |
| CPU `migratable='off'` | Avoids migratable CPUID trimming that can look synthetic |
| `qemu:override` on `sata0-0-0` | Sets `model=SAMSUNG MZVL21T0HCLR`, `ver=GXA75H0Q`, serial `S6XNNS0W20260904A` |

XML backups from this pass:
`win11-pre-harden-202609041801.xml`, `win11-hardened-202609041801.xml`,
`win11-live-post-harden-202609041811.xml`,
`win11-before-diskmodel-202609041840.xml`,
`win11-diskmodel-202609041840.xml`.

### Still visible / residual risk
- Looking Glass IVSHMEM / IDD devices are unusual hardware fingerprints.
- `virtio-serial` + SPICE vdagent channel still present (clipboard/recovery).
- QXL display adapter still present for recovery.
- Emulated TPM (swtpm), not host fTPM passthrough.
- Ghost/Unknown VirtIO Ethernet/SCSI/Balloon entries may remain in Device Manager
  until hidden devices are uninstalled.
- Timing / nested checks BattlEye may add later.

### Guest checks after reboot
In PowerShell (admin optional):

```powershell
Get-CimInstance Win32_ComputerSystem | Select Manufacturer, Model
Get-CimInstance Win32_BIOS | Select Manufacturer, SMBIOSBIOSVersion, SerialNumber
Get-CimInstance Win32_DiskDrive | Select Model, SerialNumber, InterfaceType, FirmwareRevision
getmac /v
(Get-CimInstance Win32_ComputerSystem).HypervisorPresent
# Expect HP / OMEN / AMI-like strings, MAC ac-f4-66-28-f0-3d,
# disk model SAMSUNG MZVL21T0HCLR (not QEMU HARDDISK), HypervisorPresent False
```

Also confirm Device Manager: storage is AHCI, NVIDIA ProblemCode=0, Looking Glass
IDD still selecting the RTX 5060.

### If BattlEye still kicks
1. Confirm NVIDIA still OK and IDD still on the RTX 5060.  
2. Consider stripping more recovery VirtIO (spice channel / virtio-serial) if you
   can live without SPICE clipboard.  
3. Host TPM passthrough (if you can free the fTPM from the host) — advanced.  
4. Dual-boot bare-metal Windows for Online; use the VM for single-player / other games.  
5. Do **not** use your main Rockstar account as the first test.


## 12. Evening follow-ups (2026-09-04): disk capacity + guest verification

### 12.1 “Move to 2TB SSD” clarification
The guest disk was **already** on the Sabrent Rocket Q4 2TB (`nvme1n1p1` → `/data`):

```text
/data/vms/win11/win11.qcow2
```

The Samsung 1TB (`nvme0n1`) only holds CachyOS/Btrfs. Windows felt small because the
**qcow2 virtual size was 256 GiB**, not because of host placement.

### 12.2 Expand qcow2 to 1 TiB
VM stopped (`virsh destroy` after soft powerdown ignored), then:

```bash
sudo qemu-img resize /data/vms/win11/win11.qcow2 1T
```

Result:
- Virtual size: **1 TiB**
- Host sparse usage: still ~42–47 GiB until the guest writes data
- `/data` free space remained ~1.3 TiB

Libvirt `domblkinfo` after restart reported Capacity `1099511627776`.

### 12.3 Why Extend Volume was greyed out
Windows Disk Management layout after resize:

```text
EFI/System → MSR/Reserved → C: (~255 GB) → Recovery (~730–854 MB) → Unallocated (~768 GB)
```

Extend requires **contiguous** free space immediately after C:. The Recovery
partition blocked it. Disk Management often only offers **Help** on Recovery
(no Delete in the GUI).

### 12.4 Guest fix that worked
1. `reagentc /disable` (already disabled).
2. `diskpart` on disk 0:
   - Partition 3 = Primary/C: (~254 GB) — **do not delete** (boot volume error if tried)
   - Partition 4 = Recovery (~854 MB) — delete with `delete partition override`
3. Extend C: into the freed + unallocated space — **succeeded**.

WinRE partition is currently absent. Optional later: recreate a small Recovery at
the end of the disk and re-enable `reagentc`.

### 12.5 In-guest BattlEye check results (same evening)
**Passed**
| Check | Observed |
|-------|----------|
| ComputerSystem | HP / OMEN Gaming Laptop 16-ap0xxx |
| BIOS | AMI F.13, serial 1H85430PWY |
| BaseBoard | HP 8E35 / PVAJT0387LJ2Z6 |
| MAC | AC-F4-66-28-F0-3D (e1000e / 82574L) |
| HypervisorPresent | False |
| NVIDIA RTX 5060 | OK |
| Looking Glass IDD | OK |
| Disk bus after VirtIO→SATA | IDE/SATA (not VirtIO) |

**Fixed afterward**
- Disk `Model` had been `QEMU HARDDISK` even on SATA. Spoofed with libvirt
  `qemu:override` on alias `sata0-0-0` so QEMU starts:
  `model=SAMSUNG MZVL21T0HCLR,ver=GXA75H0Q,serial=S6XNNS0W20260904A`.
- Re-verify in guest after that boot; if Windows caches the old name, reboot once
  or remove the old disk device entry.

**Residual / cleanup still useful**
- Active: Looking Glass + IVSHMEM, QXL, VirtIO Serial/SPICE, FwCfg, swtpm.
- Ghost Unknown devices seen: VirtIO Balloon, VirtIO Ethernet, VirtIO SCSI disk &
  controller, old QEMU DVD-ROM — uninstall via Device Manager (show hidden devices).

### 12.6 Next actions
1. Confirm Samsung disk model string in guest.  
2. Clean ghost VirtIO devices.  
3. Confirm IDD still selects RTX 5060.  
4. GTA Online test on a **secondary** Rockstar account only.


## 13. Network speed notes (discussed; no change applied)

### Current path
- Guest NIC: **e1000e** (Intel 82574L), MAC `ac:f4:66:28:f0:3d`
- Attachment: libvirt NAT (`<interface type='network'>` → `default` / `virbr0`)
- Host uplink during checks: WiFi `wlan0` (wired `eno1` was down)
- No libvirt `<bandwidth>` limiter on the interface
- QEMU guest agent not configured (cannot drive in-guest tests from host)

### What we learned
- Host WiFi radio looked strong (~2.4/1.9 Gbps PHY, ~-47 dBm).
- A single public 100 MB download on the host measured ~118 Mbps; that was
  treated as **server-limited**, not the host ceiling.
- User reports host throughput can reach **~1.5 Gbps**.
- With that host ceiling, the guest **e1000e** path is the likely VM bottleneck
  (emulated ~1 GbE; often well below multi-gig in practice). NAT overhead is
  secondary.

### BattlEye tradeoff (recorded decision)
| Option | Speed | BattlEye risk | Status |
|--------|-------|---------------|--------|
| Keep **e1000e** + NAT | Limited vs 1.5 Gbps host | Lower (current choice) | **Kept** |
| Host Ethernet only | Helps flaky WiFi; does not remove e1000e cap | Very low | Optional later |
| Guest **VirtIO** NIC (± vhost) | Largest guest-side win | Higher (VirtIO Ethernet tell) | **Not applied** |

Do **not** switch to VirtIO for downloads unless accepting the anti-detect hit.
Same MAC should be preserved on any future NIC model change.


## 14. Looking Glass client window (UI; config left unchanged)

`~/.looking-glass-client.ini` remains fullscreen/borderless by choice:

```ini
[win]
fullScreen=yes
borderless=yes
showFPS=yes
```

Escape / release key: **Right Ctrl**.

To minimize without editing config:
1. Press **Right Ctrl**
2. Disable fullscreen in the Looking Glass menu
3. Minimize the normal window

Optional windowed defaults were discussed and **not** applied (`fullScreen=no`,
`size=1280x720`, `showFPS=no`, `minimizeOnFocusLoss=yes`).


