# Decoding the boot log of a 2010-era HP ProLiant (DL380 G7)

A field guide to every scary-looking kernel message this machine prints at
boot, what each one actually means, and which ones have real software
workarounds. Tested on Arch Linux, Xeon X5675-class (X58/Tylersburg), BIOS
P67 (2013-07-02 — the final release HP ever shipped).

TL;DR: **all of them are harmless**, and the kernel already works around most
before you finish reading them. Three had actionable fixes; they are listed
with their fixes below.

## The messages

| Message | Verdict | What it means |
|---|---|---|
| `ACPI BIOS Warning (bug): Invalid length for FADT/Pm1aControlBlock: 32, using default 16` | cosmetic | HP's firmware table declares wrong register widths; ACPICA silently corrects to defaults on every boot. |
| `ERST: Failed to get Error Log Address Range.` | cosmetic → fixed by param | Firmware provides no APEI/ERST error-log region; kernel probes it anyway and fails. Silence permanently with `erst_disable` on the kernel cmdline (see below). |
| `[Firmware Bug]: the BIOS has corrupted hw-PMU resources (MSR 38d is 330)` | cosmetic | BIOS leaves perf-counter MSRs dirty; kernel resets them, only the 'bus cycles' profiling event is lost. |
| `DMAR-IR: ... interrupt remapping is being disabled` | informational | Intel 5520/X58 "Tylersburg" chipset erratum: BIOS enabled IRQ remapping on known-flaky silicon, kernel turns it off for stability. Only affects PCI passthrough into VMs. |
| `virt/tdx: TDX not supported by the host platform` | informational | Kernel probing for a 2021+ feature on 2008 silicon. |
| `MDS CPU bug present and SMT on, data leak possible` | real, manageable | CPU has the Microarchitectural Data Sampling flaw and no mitigating microcode exists for it. Full fix is `nosmt`; the surgical software alternative is [mds-core-wall](https://github.com/example/mds-core-wall) (cpuset partitioning of untrusted tenants). |
| `kvm_intel: VM_EXIT_LOAD_IA32_PERF_GLOBAL_CTRL does not work properly. Using workaround` | harmless | Known Nehalem/Westmere erratum; KVM applies an automatic workaround. |
| `pci_bus XXXX: No busn resource found for root bus, will use [bus XX-ff]` | harmless | Ancient ACPI didn't declare PCI root-bus number ranges; kernel guesses. |
| `radeon ... vgaarb: deactivate vga console` | normal | DRM driver taking over from legacy VGA console — happens on every modern Linux system. |
| `radeon ... format RG16 little-endian ... not supported` / `No compatible format found` | cosmetic | ES1000 framebuffer lacks a probed 16-bit format; automatic fallback. |
| `ACPI Warning: SystemIO range ... conflicts with OpRegion (\SGPE)` + `lpc_ich: Resource conflict(s) found affecting gpio_ich` | harmless | Firmware declares an I/O region overlapping the LPC GPIO controller; kernel skips that GPIO driver. Nothing user-visible. |
| `hpsa ... can't disable ASPM` / `Logical aborts not supported` / `HP SSD Smart Path aborts not supported` / `can't handle SMP requests` | informational | Capability flags of the Smart Array RAID controller — features it simply doesn't have. |
| `[sda] Write cache: disabled, read cache: enabled, doesn't support DPO or FUA` | correct behavior | `sda` is a logical drive on the Smart Array, not a raw disk. Caching lives in the controller's battery-backed cache; per-disk flush directives don't apply to volumes. Data safety is *better* this way. |
| `systemd[1]: TPM PCR Measurements skipped, unmet condition check ConditionSecurity=measured-os` | expected | systemd measured-boot units require booting a measured UKI with a TPM policy; a standard install doesn't, so they skip. Not errors. |
| `perf: interrupt took too long (...), lowering kernel.perf_event_max_sample_rate` | self-tuning | Kernel throttling the perf sampler on older CPUs under load. |
| `bridge: filtering via arp/ip/ip6tables is no longer available by default` | informational | Modern kernels don't filter bridged frames unless `br_netfilter` is loaded; Docker loads it itself when needed. |

## Software workarounds applied

### 1. Stop probing broken ERST every boot

```console
$ sudo sed -i 's|^GRUB_CMDLINE_LINUX_DEFAULT=.*|GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet erst_disable"|' /etc/default/grub
$ sudo grub-mkconfig -o /boot/grub/grub.cfg
```

### 2. MDS without losing SMT

See [mds-core-wall](../mds-core-wall): Docker containers walled onto dedicated
cores via `docker.slice`, KVM guests via `machine.slice`.

### 3. EGL/DRI3 warnings from the ES1000

See [mesa-software-rendering](../mesa-software-rendering):
`LIBGL_ALWAYS_SOFTWARE=1` in `/etc/environment`.

## Why firmware bugs can't be "fixed" in Linux

The FADT lengths, PMU MSRs, ERST table etc. live in BIOS ROM. The kernel
patches around each one in RAM at runtime — which is exactly why you see the
warnings: they document workarounds being applied. The last BIOS HP released
for this generation is P67 (July 2013); there will be no newer one.

MIT licensed; see [LICENSE](LICENSE).
