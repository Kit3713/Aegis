# Project Plan — Draft v0.0

> **Status:** Pre-alpha. This document is a vision and architecture draft, not a specification.
> **Purpose:** Consolidate decisions made so far, surface open questions, and set up a v0.1.0 roadmap.

---

## 1. What this is

An opinionated, atomic, security-by-default Linux desktop operating system targeting RHEL-family stability (AlmaLinux base) with a KDE user environment for the visible install. The system ships with a fixed-shape disk architecture, LUKS2+integrity full-disk encryption, SELinux MLS enforced at the cube perimeter, signed VM templates ("cubes") for compartmentalization, a broker-mediated IPC framework between cubes, and a duress-triggered destructive boot path ("Pyrrhic Boot"). Plausible deniability — a hidden OS sharing the encrypted data partition with the visible one — is a toggleable install option.

The product is the opinionated architecture itself. Users do not configure SELinux, LUKS, or the partition layout. They confirm a hardware tier, set passphrases, pick which cubes to instantiate, and the system takes care of the rest.

There is **no declarative-spec / config-language layer** inside this OS. The opinionated architecture is shipped as code. Power-user customization (custom bootc images, user-built templates, etc.) lives outside the OS proper and is documented as a community pattern, not as supported product surface.

---

## 2. Threat model

The system is designed against a layered threat model. It does not claim protection against every attacker, but defines an explicit cost-of-attack curve.

- **Evil maid / brief physical access.** Powered-off device taken for minutes-to-hours. Adversary has the disk image and can attempt offline cracking or boot-chain tampering.
- **Remote network and exploit attackers.** Standard internet-facing exposure: drive-by exploits, malicious downloads, network-borne lateral movement attempts against a single device.
- **Persistent malware.** Code that lands on the system and tries to survive reboot.
- **Cold-boot and DMA.** Opportunistic memory-scraping while the system is on or recently powered down.
- **Forensic / compelled inspection.** A trained examiner with the disk image and time. Not necessarily a state-level adversary — could be customs, civil discovery, or a divorce subpoena. They will look for crypto containers and they will ask about them.

The system is **not** designed to defeat:

- A nation-state-level adversary with hardware implants, supply-chain compromise, or unbounded budget. Finding the hidden OS is acknowledged as possible at this tier — the design goal is cost asymmetry: expensive to find, much more expensive to access.
- Compelled disclosure under jurisdictions where refusal to provide a passphrase is itself prosecutable (UK RIPA-style). The hidden OS feature lowers but does not eliminate this risk.
- An attacker with continuous physical possession over long periods.
- The user themselves choosing to install untrusted code in a worker cube and granting it broker access.

---

## 3. Core design principles

1. **Opinionated security floor, period.** SELinux enforcing, LUKS2+integrity, immutable root via dm-verity, signed boot chain, signed cube templates. Non-negotiable. The baseline cannot be downgraded.
2. **Forensically unsuspicious by default.** AlmaLinux + KDE reads as a stability-focused conservative user. Long gaps between updates are normal for AlmaLinux. The visible install looks like nothing in particular.
3. **Plausible deniability as a feature, not a research project.** Two LUKS partitions with identical crypto profiles. The data partition's "free" space is mathematically indistinguishable from a hidden OS, because that's exactly what it might be.
4. **Atomic and (where possible) immutable.** State transitions happen as complete units. Read-only `/usr` enforced by composefs + dm-verity. Updates ship as image transactions via bootc.
5. **Compartmentalization over hardening.** Workloads run in cubes (VMs). The host is small and boring. Breaking a cube costs you that cube, not the system.
6. **Hardware-tier graceful degradation.** A user with no TPM still gets a meaningfully secure system. A user with Coreboot/Heads gets the strongest version. Same OS, same install media, different ceiling. No hardware gating — Tier 0 is a first-class deployment.
7. **Host opinionated, cubes sovereign, perimeter non-negotiable.** The host enforces the security architecture without compromise. What runs inside a cube is the user's domain — the project does not dictate cube internals beyond the signed-template defaults. Templates are a convenience, not the only path. Standalone cubes, user-built templates, and disposable cubes are all first-class architectural citizens. The MLS perimeter at the cube boundary is enforced uniformly regardless of cube origin.
8. **No declarative layer.** The opinionated architecture is shipped as code, not as a spec to be evaluated. Users do not write configuration files to install or to manage the system.

---

## 4. Fixed architecture (decided)

These are settled. Items here are not subject to per-install configuration; they are what the OS *is*.

### 4.1 Base platform

- **Host distribution:** AlmaLinux (RHEL 9.x family target initially; 10.x once tooling matures).
- **Decoy/visible OS:** straight AlmaLinux + KDE Plasma. Forensically unsuspicious; same ecosystem as the host for operational consistency.
- **Init:** systemd (RHEL native; not relitigating).
- **Container/VM runtime:** libvirt + QEMU/KVM for cubes. Cloud Hypervisor for thin service cubes (netvm, audio cube). Rootless Podman + quadlets available for in-cube workloads if the user chooses.
- **Image lifecycle:** bootc / OSTree-based atomic updates with composefs + fs-verity for `/usr`.
- **Mandatory access control:** SELinux in enforcing mode. MLS labels enforced at the cube perimeter (disk images, qemu processes, vsock channels, NICs, broker pipes). Inside-cube hardening is the user's choice; the perimeter is the project's responsibility.
- **Bootloader:** sd-boot with signed UKIs. Custom Secure Boot keys (PK/KEK/db) optional but recommended at install time.

### 4.2 Disk layout

Fixed, identical across all installs (this is core to the deniability story — variation in layout is itself a forensic signal).

```
/dev/<disk>
  ├─ p1  ESP                              1 GiB    FAT32, signed UKIs only
  ├─ p2  LUKS2+integrity — system part.   ~80 GiB  AES-XTS-512, Argon2id, HMAC-SHA256
  └─ p3  LUKS2+integrity — data part.     rest     identical crypto profile to p2
```

**`/boot` lives inside the system LUKS partition, not as a separate unencrypted partition.** Kernel and initramfs are encrypted at rest; only the ESP is unencrypted on disk.

**Both LUKS containers use identical crypto parameters.** Any deviation is itself a fingerprint. Same cipher, same KDF, same Argon2id memory/time costs, same integrity mode.

Sizes scale with disk capacity. Structure does not change.

### 4.3 System partition LVM (inside p2)

```
VG: system
  ├─ lv_boot         1 GiB    ext4                          /boot
  ├─ lv_sysroot_a   12 GiB    erofs + composefs + verity    /usr (active deployment)
  ├─ lv_sysroot_b   12 GiB    erofs + composefs + verity    /usr (standby for A/B)
  ├─ lv_var_state    4 GiB    XFS                            /var/lib (broker, libvirt defs, sealed key blobs)
  ├─ lv_var_log      4 GiB    XFS                            /var/log (host journal, rotated, capped)
  └─ lv_var_audit    4 GiB    XFS                            /var/log/audit (auditd, append-only via chattr +a)
```

`/var/tmp` is tmpfs. `/dev/shm` is tmpfs. `/home` does not exist on the host (the host has no users beyond an admin account; user state lives in cubes).

Sizes above are sketches — final sizing depends on auditd retention policy, libvirt definition footprint, and image size. See §10.

### 4.4 Data partition (inside p3)

```
VG: data
  ├─ lv_cubes_pool   ~90% of partition    LVM thin pool
  │     │
  │     ├─ Templates (read-only thin LVs of signed composefs images):
  │     │   ├─ tmpl_workstation
  │     │   ├─ tmpl_comms
  │     │   ├─ tmpl_browser
  │     │   ├─ tmpl_usb
  │     │   ├─ tmpl_netvm
  │     │   └─ tmpl_disposable
  │     │
  │     └─ Live cubes (CoW thin snapshots of templates, or independent thin LVs for standalone cubes)
  │
  └─ "Free space" (unallocated VG extents) — when hidden OS is enabled, hosts the
     hidden OS in extents that have no LVM metadata claim on them. Cover story
     (true regardless): "free space in the VG for future growth."
```

The hidden OS, when enabled, lives in extents the LVM metadata does not claim. From `vgs`/`pvs`, these appear as ordinary unallocated VG free space. The hidden OS's LUKS header is detached and stored per the hardware tier (see §6).

### 4.5 Cubes

Cubes are KVM virtual machines. They are the unit of compartmentalization and the security boundary in this OS. Cubes are always VMs, never containers — the security boundary lives at the hypervisor perimeter where sVirt enforcement, IOMMU isolation, and separate kernels apply. Containers are first-class *inside* cubes (rootless podman + quadlets) but never as the cube primitive.

Four cube creation paths, all first-class at the architecture layer:

| Type | Backing | Updates | Default? |
|---|---|---|---|
| **Template-derived** | CoW thin snapshot of signed template LV | rebase to new template version | yes |
| **Disposable** | tmpfs only, fresh per boot from template | implicit — always current | yes |
| **Standalone** | own thin LV, no template parent, full RW | traditional package mgmt inside the cube | opt-in |
| **Custom template** | user-built signed image, becomes a template root that user-derived cubes can CoW from | user-managed | opt-in |

The first two are the opinionated path and ship in v0.1. Standalone cubes and custom templates are post-v0.1 (architecture supports them; tooling lands later).

The host's view of all four is uniform: each cube is a labeled qemu process with a labeled disk image, MLS-enforced at the perimeter regardless of cube origin.

#### Cube identity

Each cube has an ed25519 identity keypair held in its vTPM (provided by swtpm). Public key registered with broker at cube creation. Every cube → broker request is authenticated via challenge-response. vTPM state protection scales by hardware tier:

- **Tier 0:** vTPM state encrypted with key derived from host master passphrase
- **Tier 1+:** vTPM state sealed against host TPM PCRs
- **Tier 2+:** additionally wrapped by USB-held key

Attestation (PCR quote of cube boot state) is a Tier 1+ feature. Certain operations (T4 cube unlock, declassification gestures, cube template rebase) require fresh attestation and are simply unavailable on Tier 0 hardware.

### 4.6 MLS tiers

Five tiers, ranked by restriction. Applies uniformly to all cube types including service cubes.

| Tier | Name | Network | Inter-cube data flow | Persistence | Use cases |
|---|---|---|---|---|---|
| **T0** | Disposable | edge netvm, default-deny + broad allowlist | prompt-with-friction across tiers, easy intra-tier | tmpfs, ephemeral | sketchy URLs, untrusted attachments, USB cube |
| **T1** | Standard | edge netvm, normal policy | auto-allow intra-tier (audited), prompt cross-tier | persistent | daily driver, dev, browsing, edge netvm itself |
| **T2** | Trusted | edge or VPN service netvm | prompt every event, even intra-tier | persistent | banking, password mgmt, Tor netvm, comms cubes |
| **T3** | Sensitive | VPN/Tor service netvm only | prompt + audit every event, declassification required to write down | persistent, often app-encrypted too | work confidential, signing keys, identity docs |
| **T4** | Lockdown | none by default; opt-in escape hatches with high friction | NO automatic flow either direction; manual export only | persistent | crown-jewel vaults, malware analysis, forensic samples |

**Tier semantics:**

- T0–T3 use Bell-LaPadula dominance: read-down free, write-up audited, write-down requires declassification gesture.
- **T4 sits outside the BLP lattice** — strict isolation, no automatic flow either direction regardless of dominance. Vault-style protection (don't let secrets escape) and quarantine-style containment (don't let malware escape) share the same enforcement; different intent, same discipline.
- Compartments apply at every tier including T4 — two T4 cubes do not share data with each other unless compartments dominate.
- T4 network options (each requires explicit per-cube opt-in, none default): pinhole allowlist, fake-network service netvm (INetSim-style for malware analysis), operator-mediated manual bridge.

**Tier dial is universal:** edge netvm = T1, USB cube = T0, Tor netvm = T2, vault cubes = T3 or T4 by user choice, malware-analysis cube = T4 always. Tier reflects the cube's role and risk.

Tier renaming for cosmetic preferences (UNCLASSIFIED/CONFIDENTIAL/SECRET style) is a system setting; the underlying semantics are fixed.

### 4.7 Broker

A host-side daemon that mediates inter-cube communication and any cube-to-host action. It is the single chokepoint for all cross-cube data flow.

**Channel model.** The broker exposes typed channels, each with independent policy:

| Channel | Direction | Default policy | Notes |
|---|---|---|---|
| Clipboard text | bidirectional | prompt + dominance check | size cap (e.g., 1 MB) |
| Clipboard image | bidirectional | prompt + dominance check | image format whitelist |
| File copy | one-shot | prompt + dominance check | size cap, lands in `/home/user/Incoming/` on dest |
| URL open | one-shot | prompt + allowlist target cube | "open this in browser cube" |
| Audio stream | persistent | explicit grant per session | per-cube source/sink |
| Video stream | persistent | explicit grant per session | camera handling |
| Notifications | one-shot | always allowed up, prompt down | cube wants to surface a notification |
| Screen share | persistent | explicit grant per session | rare, high-friction |
| Backup-out | persistent | tier-dependent | source-side encrypted blob → backup cube |

**No shared filesystems, no bidirectional sync, no arbitrary IPC.** If two cubes need a persistent connection, they use the network with the netvm in path.

**Per-pair consent modes** (configurable per cube-pair per channel): always-prompt, prompt-once-per-session, always-allow (only for explicitly-designated trusted same-tier pairs), always-deny.

**Consent prompts** are drawn by the host compositor (unfakeable by cubes), show source/destination cube identity + tier color, payload summary, and channel type.

**Audit:** every consent decision and every transfer is recorded by auditd to `lv_var_audit`.

The broker's own threat model deserves its own document. Restart semantics, attack surface, and recovery from broker compromise are open questions (see §10).

### 4.8 Networking

Layered architecture. Mandatory security floor + composable services + per-cube routing policy.

**Edge netvm (mandatory, always-in-path):**
- Owns the physical NIC(s) via PCIe passthrough. The host has no NIC, no IP stack, no resolver.
- Acts as router between cubes — each cube on its own network segment, default-deny cube-to-cube, explicit allowlist for inter-cube flows.
- Suricata IDS/IPS + Zeek inline.
- nftables firewall, default-deny egress.
- DNS resolver: DoH/DoT upstream, local cache with offline-resilience, IPv6 leak prevention (lock or tunnel, never half), mDNS/LLMNR blocked.
- Proteus L2/identity spoofing, per-source-cube profile.
- Captive portal mode (controlled, time-bounded).
- Per-cube bandwidth accounting.
- Suricata/Zeek output to ring buffer with vsock export to host audit volume (or to logging cube if user has provisioned one).

**Service netvms (zero or more, user-composable thin VMs on Cloud Hypervisor):**
- VPN providers (Proton, Mullvad, etc.), Tor (Whonix-gateway-style with stream isolation), Tailscale, bare WireGuard, mesh overlays.
- User-built service netvms supported, with user's own signing key.
- Chains compose by routing one service netvm through another.

**Per-cube policy** (broker-enforced at cube wire-up): which netvm to route through (default: edge direct), egress destination allowlist, inter-cube flow allowlist, spoofing profile, killswitch fail mode (default: closed), bandwidth cap.

**Host updates flow via broker** from a destination-restricted `update-cube` (egress allowlist: signed-image registry only, mandatory signature + verity-hash verification before staging). The host's "network access" reduces to *the broker hands me a signed image from a known-good cube*.

### 4.9 Display server

Per-cube Wayland compositor with minimal buffer-handoff to a host compositor that owns all window decoration and input routing.

- Each cube runs its own minimal Wayland compositor (labwc or custom slim variant). Apps inside the cube get full Wayland API surface and don't know they're nested.
- Cube compositor renders to a virtio-gpu buffer. Buffer + dimensions + damage rect + cursor pose cross to host via vsock-based handoff protocol. Nothing else crosses — no Wayland objects, no shared-memory maps, no GL contexts.
- Buffer **copy** at the boundary (not memory map). Explicit trust transition.
- Host compositor wraps each cube buffer in chrome showing the cube's MLS tier color and label. Cubes cannot draw outside their buffer rect, cannot fake decoration, cannot impersonate another cube's color.
- Input events routed by host compositor based on focus. Cubes never receive events they're not focused for.
- Cursor is host-drawn (real cursor over cube windows), preventing fake-cursor attacks.
- GPU-passthrough cubes own a real display output and bypass the host compositor entirely. Switching back requires hotkey + hardware-token confirmation.

Built on existing pieces (labwc + virtio-gpu + vsock + wlroots) rather than custom protocol.

**Default UX:** workspace-per-cube; cube color follows everywhere (chrome, taskbar, alt-tab, workspace switcher, notifications).

### 4.10 Audio and camera

Dedicated **audio cube** owns the audio hardware via PCIe passthrough; brokers per-cube virtual sources/sinks via PipeWire over vsock.

- Audio cube is T1.
- Source-side encryption for any persistent recording.
- **Output (speakers):** moderate concern. Available to most cubes by user opt-in, prompted on first use, can be granted persistent.
- **Input (microphone):** high privacy concern. Available only to comms-class cubes by explicit per-session grant (default: ask every session).
- Host compositor draws an unfakeable always-on indicator whenever any cube has live mic access (color-coded by cube identity, fixed screen position cubes cannot draw over).
- Hardware mute switch cuts audio at the audio cube level, not at the receiving cube — guarantees physical mute regardless of cube state.

**Camera follows the same pattern**, sibling cube (or same cube if the device is integrated webcam+mic). Default: only comms-compartment T2 cubes can request camera, per-session grant.

**Per-tier audio defaults:**
- T0: no audio.
- T1: speaker output on user opt-in. Mic input denied.
- T2: same as T1, except comms-compartment cubes can request mic with per-session grant.
- T3: no audio by default; opt-in with declassification-class friction.
- T4: no audio, no exceptions. Audio is a sidechannel that bypasses data-flow controls.

**Bluetooth audio** is out of scope for the audio cube; lives in its own dedicated BT cube when enabled.

### 4.11 Logging

Minimal essential logging by default; aggregating logging cube is opt-in, not built-in.

**Default (no logging cube):**
- auditd → `lv_var_audit` (append-only). Kernel security events, broker consent decisions, tamper counter increments, Pyrrhic Boot triggers, attestation failures, USB cube device events.
- Per-cube journald stays inside each cube. Cube sovereignty — host doesn't aggregate.
- Edge netvm Suricata/Zeek writes to ring buffer inside the netvm; recent events queryable from netvm; expires on rotation.

**Optional patterns** (user opts in by spinning up a cube):
- Aggregating logging cube (T2) running Vector or Fluent Bit, configured to forward events from sources via broker.
- Heavy SIEM cube (Wazuh, OpenObserve, Loki+Grafana) for users who want it.
- Offsite log shipping via Tailscale to a user-owned SIEM.

The mandatory audit volume is the security floor. The logging cube is an *optional aggregation point*, not load-bearing infrastructure.

### 4.12 Backup and recovery

Two-layer model.

**Local recovery (always on):** LVM thin snapshots on schedule, per-cube retention. Same-disk, fast restore for "I deleted a file" or "this update broke my cube." Not real backup.

**Off-cube backup (configurable per tier):** dedicated backup cube transports source-side-encrypted blobs to user-chosen destination (local NAS, removable disk, cloud via netvm). Backup cube only ever sees ciphertext.

Per-cube backup keys wrapped by master backup key, derived from a separate recovery passphrase printed offline at install. Loss of host ≠ loss of backups.

**Per-tier backup policy:**
- T0: none (ephemeral).
- T1: scheduled, default destination, encrypted client-side.
- T2: more frequent, separate per-cube keys.
- T3: encrypted with cube-specific key, restricted destination.
- T4: opt-in **per-cube** (vault yes, quarantine never).

**Restore-to-disposable verification:** after every successful backup, the backup cube spins up a disposable verification cube and restores the latest backup into it. Validates count/hash/health probe. On failure, alert immediately. Backup is not "done" until verified.

---

## 5. Installer model

A custom, fixed-shape, two-phase installer. Not an Anaconda fork. Not a declarative-config evaluator.

### 5.1 Phase 1 — text-mode, security-critical setup

Runs in early install environment, no Wayland, minimal attack surface.

- Hardware detection (TPM, IOMMU, Secure Boot keys writability, USB, audio, GPU, NIC count).
- Tier auto-detection and recommendation (T0/1/2/3 — same disk shape regardless).
- Disk selection.
- CSPRNG disk wipe (universal, applied regardless of whether user opts into deniability).
- Fixed partition layout applied: ESP + system LUKS+integrity + data LUKS+integrity (sizes scale, structure doesn't).
- LUKS+integrity format with shared cipher/KDF profile across both partitions.
- Passphrase configuration: main, recovery, optional duress (Pyrrhic Boot), optional hidden-OS.
- **Decoy install:** straight AlmaLinux + KDE Plasma into the system LUKS partition (normal LVM + ext4 — wait, this is wrong; the system partition is the hardened OS's, not the decoy's. The decoy IS the visible OS. See §10.)
- LUKS keyslot setup (TPM seal at Tier 1+, USB header location at Tier 2+).
- sd-boot install + signed UKI generation.
- Custom Secure Boot key enrollment (optional, recommended).
- bootc image pull and composefs/verity tree build into `lv_sysroot_a`.
- Reboot.

### 5.2 Phase 2 — Wayland-native, post-first-boot setup wizard

- Host admin account creation.
- Edge netvm configuration (NIC selection for passthrough, basic firewall posture).
- Audio cube setup if hardware present.
- Default cube creation walkthrough (creates a starter workstation cube; optional comms cube).
- Recovery passphrase generation and display (user must record before continuing).
- Optional: lived-in synthesizer for decoy if user chose deniability in Phase 1.
- Tier renaming if user prefers different labels (cosmetic only).

### 5.3 What is NOT customizable at install

- Filesystem architecture (fixed)
- LUKS profile (fixed)
- MLS tier model (fixed — opinionated MLS *is* the product)
- Disk shape (fixed for deniability)
- Cube template internals (project-signed; immutable)
- Broker policy framework (defaults ship; runtime tuning only)

---

## 6. Hardware tier model

Same OS, same install media, four ceilings. The user does not pay for unavailable hardware in lost security guarantees beyond what the hardware itself prevents.

| Capability                        | Tier 0 (baseline)            | Tier 1 (+TPM)             | Tier 2 (+USB)             | Tier 3 (+Coreboot/Heads) |
|-----------------------------------|------------------------------|---------------------------|---------------------------|--------------------------|
| Host LUKS unlock                  | Passphrase                   | TPM+PIN                   | USB+PIN                   | TPM+PIN+measured boot    |
| Hidden OS header location         | File in decoy `/var/lib`     | TPM-sealed file           | USB-only                  | USB + TPM-sealed         |
| AEM detection                     | None                         | PCR mismatch on tamper    | —                         | Heads attestation        |
| Tamper counter                    | EFI variable (signed)        | TPM NV counter            | Same                      | TPM NV counter w/ Heads  |
| Pyrrhic Boot trigger              | Duress passphrase            | + tamper threshold        | Same                      | Same                     |
| Custom Secure Boot keys           | Optional, recommended        | Yes                       | Yes                       | N/A (Heads handles)      |
| Cube attestation                  | Identity only, no attestation | Full PCR attestation     | Same                      | Same + Heads anchor      |
| T4 cube unlock                    | Unavailable                  | Available                 | Available                 | Available                |

Tier 0 still gets full-disk authenticated encryption, immutable system, audit isolation, hidden OS with disk-only deniability, and a Pyrrhic Boot path. The degradation curve is real but acceptable. **No hardware is required for any tier; users opt in to features their hardware supports.**

---

## 7. Pyrrhic Boot

A duress passphrase entered at boot triggers a destructive sequence. Three stages of increasing time-cost:

**Stage 0 (~50 ms): keyslot annihilation**
```
cryptsetup luksErase --batch-mode /dev/disk/by-partlabel/data
shred -n 1 -z /path/to/hidden_os_header (if exists)
```
After Stage 0, even if the wipe is interrupted by a yanked power cord, the data is unrecoverable. No keyslot, no key, no decryption path.

**Stage 1 (seconds): metadata storms**
```
blkdiscard /dev/data
blkdiscard /dev/cubes_pool
```
On NVMe with TRIM, returns sectors to controller's free list almost instantly.

**Stage 2 (minutes-hours, background): I/O wipe**
```
dd if=/dev/urandom of=/dev/<disk> bs=1M
```
Sequential overwrite of raw device. Runs under cover of the post-duress visible OS.

**Cover behavior** (Stage 2 happens while user is logged into something) is open — see §10.

**Trigger mechanisms beyond duress passphrase** (Tier 1+): tamper counter exceeded, PCR mismatch detected. Thresholds are open questions.

**Recovery from accidental Pyrrhic Boot:** none, by design. Documented in install warnings.

---

## 8. Hidden OS / deniability

Toggleable at install. When enabled:

- The hardened atomic OS lives in unallocated VG extents inside `data` partition's LVM. No LUKS header on disk for the hidden region — the region is filled with CSPRNG output at install time so its contents are indistinguishable from never-allocated space.
- Header is detached and stored per hardware tier (file/TPM/USB).
- The visible/decoy install is straight AlmaLinux + KDE Plasma, lived-in to survive behavioral analysis.
- Two-passphrase boot flow: decoy passphrase → AlmaLinux+KDE; hidden passphrase + header location → hardened cube OS.
- Whether the hidden OS is a full atomic image with all cubes or a stripped variant is open (current lean: full atomic — option a from May 5).

When disabled, the data partition's free space is genuinely free space — but the cover story still works because there is no way to tell from the disk alone.

**Forensic posture:** without the unlock material, the disk presents as a hardened AlmaLinux + KDE install with whole-disk LUKS+integrity, LVM, normal lived-in filesystem, and one partition (`data`) containing a thin pool with some unallocated extents. The "why is there free space in the VG?" question has the unfalsifiable answer "future growth / overprovisioning" which is *literally true* of every LVM-managed system. A determined examiner with sufficient curiosity may notice the size of unallocated space is consistent with another OS; this is acknowledged in the threat model. The cost of going from "noticed" to "accessed" is the cost asymmetry the design funds.

---

## 9. Path to v0.1.0 — proposed scope

The first tagged release should be **the smallest thing that boots, encrypts, and demonstrates the architecture** — not a full-featured OS. Suggested v0.1.0 deliverables:

1. **Bootable installer ISO.** Phase 1 hardware detection (TPM only initially), Phase 2 fixed install. Tier 0 and Tier 1 only. No USB or Heads.
2. **Two LUKS+integrity partitions, identical profiles.** LVM inside p2 with `lv_boot`, `lv_sysroot_a`, `lv_var_state`, `lv_var_log`, `lv_var_audit`. Thin pool inside p3.
3. **Atomic AlmaLinux + KDE base** with composefs+verity on `/usr` and SELinux enforcing. Visible OS is the decoy.
4. **MLS lattice present but minimal:** host label + T1 (Standard) cubes only. T0/T2/T3/T4 architecturally reserved but no shipped templates yet.
5. **One signed cube template:** "general-purpose worker" (workstation-class). Demonstrates the cube boot path and broker handshake.
6. **Edge netvm shipping with a basic config:** nftables default-deny egress, DoH resolver, no Suricata/Zeek/Proteus yet (all v0.2+).
7. **Broker daemon** with smallest viable policy: clipboard text and file transfer between host and the single cube. No cube-to-cube. No audio. No video.
8. **Per-cube Wayland compositor + buffer handoff** for the workstation cube. Host compositor with simple chrome. No advanced WM features.
9. **Pyrrhic Boot via duress passphrase** as the only trigger. Stage 0 + Stage 1 only — Stage 2 cover behavior deferred.
10. **No hidden OS support yet.** Architecture leaves the room; provisioning lands in v0.2.
11. **Custom Secure Boot key enrollment** as optional install step.
12. **Documentation:** install guide, threat model, "what this is and isn't" page.

**Deliberately NOT in v0.1.0:** hidden OS, multi-cube broker policies, audio cube, camera cube, USB cube, AEM detection, USB unlock, Heads integration, user-defined cubes / custom templates / standalone cubes, fleet update infrastructure, recovery passphrase complexity beyond a single static one, Suricata/Zeek/Proteus in netvm, service netvms (VPN/Tor/Tailscale), backup framework, logging cube.

---

## 10. Open questions

These are the calls that need to be made — by you, with input — before v0.1.0 is buildable. Grouped roughly from "must answer first" to "can defer."

### 10.1 Identity and naming

1. **Project name.** The May 5 conversation surfaced candidates (Limes, Lattice, Caisson, Castrum, Aegis, etc.) but didn't lock one. Public name + kernel/branding name needed before the public repo opens.
2. **License.** MIT, GPLv2 (matches kernel/AlmaLinux ecosystem), AGPL for the broker daemon specifically? Settle before code lands publicly.
3. **Trademark posture.** Trademark the name now or leave it free?

### 10.2 Scope and audience

4. **Server fork.** Earlier conversations described a server-focused variant. Is the desktop OS the only product for the foreseeable future, or is server a v0.5+ track that should be visible in the architecture from day one?
5. **Single-user assumption.** Is the visible decoy strictly single-user, or do we want multi-user from the start? Multi-user changes a lot about MLS, auditing, and cube ownership.
6. **Defense/government posture.** Is the project actively pursuing eventual defense use (CC certification path, FIPS modules, etc.) or is "secure for serious users" the ceiling?

### 10.3 Architecture details still to finalize

7. **Phase 1 installer's decoy/host ordering.** §5.1 has a confusion point: the system LUKS (p2) houses the *hardened* OS, but the visible install (decoy) is AlmaLinux+KDE. Where exactly does the decoy live on disk? Two readings exist in the May 5 conversation; pin one.
   - **Reading A:** decoy lives in p2's system partition; hardened cube OS lives in p3's data partition unallocated extents (decoy is the default boot).
   - **Reading B:** hardened cube OS is the visible system in p2; the decoy is a separately-booted environment hidden similarly to how the hardened-OS-as-hidden case would be.
   - Reading A is what the project description implies and what the deniability story needs. §5.1 needs a rewrite to reflect this clearly: the decoy is in p2; the hardened OS lives in p3.
8. **`lv_var_*` sizing.** 4 GiB each was a sketch. Real sizing depends on auditd retention policy, libvirt definition footprint, etc.
9. **Hidden OS contents.** Full atomic image with different cubes (option a, the lean) or stripped variant (option b)? Storage cost vs. operational cleanliness.
10. **Per-cube LUKS layer inside the cube pool.** Defense-in-depth at ~20% IOPS cost. With Tier 0, per-cube keys would have to derive from the master passphrase or sit plaintext in `lv_var_state`. Tier-conditional?
11. **Boot-inside-LUKS confirmed.** §4.2 has `/boot` as an LV inside the system LUKS. GRUB is replaced by sd-boot which can do cryptodisk decode. Confirm sd-boot's path for this and lock in.
12. **Filesystem choice on the data partition's cube backing.** Thin pool LVs are block-device-shaped. Per-cube filesystem inside is the cube's choice (template-defined). Confirm.

### 10.4 Cubes and broker

13. **Initial cube template set.** What ships in v0.1? In v1.0? Browser, documents, comms, dev, scratch, vault?
14. **Cube template signing infrastructure.** Project key, key rotation procedure, where the public key lives (in the OS image, on the install media, both?).
15. **Broker policy expression.** Default policies ship as code. Per-pair settings adjustable in the running system — through what UI? A plain GUI panel; not a config language.
16. **Broker restart semantics.** What happens to active cube connections when the broker restarts? What does broker compromise look like and how do we contain it?
17. **User-defined cubes / custom templates / standalone cubes.** Architecture supports them; tooling lands when? v0.3? v0.5?

### 10.5 Installer

18. **Installer implementation language.** Rust (favored for memory safety), Go, Python, or plain shell + dialog?
19. **Tier 1 unlock UX.** systemd-cryptenroll for TPM2+PIN — confirm. PCR set: 0, 2, 4, 7, 11 from May 5. Stick with that?
20. **Recovery passphrase model.** One static recovery passphrase per install? Multiple? Time-limited? What happens if it leaks?
21. **Duress passphrase entry.** Where in the boot flow? Same prompt as the main passphrase, or distinct? How do we prevent accidental entry?

### 10.6 Pyrrhic Boot

22. **Stage 2 cover behavior.** Pristine first-boot, "normal" desktop with no data, or something else? Affects how convincing the post-duress story is.
23. **Tamper-threshold trigger thresholds.** PCR mismatch and tamper counter — what numbers? User-configurable or fixed?
24. **Admin-policy trigger.** Is there a remote/admin trigger for Pyrrhic Boot, or duress-passphrase-only? Remote triggers are powerful and dangerous.

### 10.7 Updates and lifecycle

25. **Update channel infrastructure.** Mirror AlmaLinux's? Run our own? Cloudflare R2 / mirror network?
26. **Update verification.** OSTree signatures, additional project signing, both?
27. **Update cadence.** Match AlmaLinux's stability cadence (months) or faster for security fixes?
28. **Cube template updates.** How are updated templates pushed? Are running cubes updated in place, replaced on next boot (default), or pinned (per cube policy)?
29. **End-of-life policy.** AlmaLinux 9 EOL is 2032. What's the project's commitment? Migration path to Alma 10?

### 10.8 Build, CI, distribution

30. **Build infrastructure.** GitHub Actions? Self-hosted runner? How is the verity-signed `/usr` image built reproducibly?
31. **ISO distribution.** GitHub Releases? Separate mirror? Torrent? Signed checksums obviously.
32. **Reproducible builds.** Aspirational from day one, or a v1.0+ goal?

### 10.9 Documentation and community

33. **Documentation site.** README + wiki, or a static site (mdBook, Hugo, Docusaurus)?
34. **Threat model document.** Public from day one. Audience: security researchers, prospective users, both?
35. **Contribution policy.** Do we accept community PRs against the cube template set? Against the broker? Against the installer? How is template signing handled if so?
36. **Bug bounty / vuln disclosure.** Aspirational note in the README from v0.1, or wait until there's something worth reporting?

### 10.10 Hard-mode questions worth flagging now

37. **Legal review of deniability features.** Hidden OS in some jurisdictions is essentially fine; in others (UK RIPA, France in some readings, others) refusing to disclose can be prosecuted. Should the README carry a jurisdictional warning?
38. **Export classification.** Strong crypto + MLS + designed-for-defense framing may trigger US export controls (EAR, ITAR-adjacent). Worth a one-line check before public release.
39. **The "this looks like criminal evasion software" problem.** The honest framing — protecting journalists, dissidents, abuse survivors, professionals with confidential client data — needs to be on the front page of the README, not buried.
40. **Scope discipline.** Realistically, building v0.1 to the spec above is a year-plus of evening work alongside cert study, the resume site, and field work. The v0.1 scope in §9 is already aggressive; don't expand it before v0.1 ships.

---

## 11. What I'd push back on if I were external review

A few things to think about before this becomes a public commitment:

- **The MLS-by-default story is a heavy lift.** SELinux MLS policy authoring is a specialized skill, and getting it wrong produces a system that's either constantly broken or silently permissive. The project either needs deep SELinux expertise or needs to pull from an existing MLS reference policy and harden incrementally. The "perimeter-only" framing is correct and defensible, but the policy still needs to be written.
- **The cube model overlaps significantly with Qubes OS.** The differentiation story (RHEL base, signed templates, broker model with explicit channel typing, full deniability story, atomic immutability via composefs+bootc, MLS at the perimeter, T4 isolation outside BLP, Wayland-native compositor architecture, hardware tier model) is real, but should be articulated clearly so this doesn't read as "Qubes but with my preferences." Qubes-the-product has thousands of person-years of work in it. The differentiators need to be on the front page.
- **The broker is a single point of trust.** If it's compromised, every cube's IPC is compromised. The broker's own threat model deserves its own document. Restart semantics, recovery from compromise, and the broker's update path are non-trivial.
- **"Forensically unsuspicious" and "ships hidden OS support" are in tension.** A forensic examiner who knows the OS exists knows that any install of it *might* have a hidden volume. The deniability is "I might have one but you can't prove it from the disk," not "you'd never know to look." Worth being precise about in the README.
- **Scope vs. timeline.** Be honest with yourself about what one person on evenings can ship, then halve it for v0.1. The §9 minimum may still be a year of work.
- **"No declarative layer" is a real commitment.** The temptation to add a small DSL for broker policy or cube template definition will be strong. Resist or be explicit about scope creep when it happens.

---

## 12. Immediate next steps (proposed)

When you pick this back up:

1. Decide §10.1 (name, license) — these gate the public repo.
2. Resolve §10.3 question 7 (decoy/hardened OS partition assignment) — this is a §5.1 confusion that would block installer code.
3. Pick the v0.1 minimum viable architecture from §9 + §10.10 (40) — what's the smallest demo that proves the concept?
4. Start a `THREAT_MODEL.md` from §2 — it's already 80% written above.
5. Stub a `ROADMAP.md` with v0.1 / v0.2 / v0.5 / v1.0 milestones once §9 firms up.
6. Open a GitHub Discussions thread (or a `decisions/` directory of ADRs) to log the answers to §10 as they're made, so this draft can be retired in favor of decision records.

---

*End of draft. Anything important I missed should be added by you on next pass; anything wrong here should be struck through rather than deleted, so the reasoning trail is visible.*# Project Plan — Draft v0.0

> **Status:** Pre-alpha. This document is a vision and architecture draft, not a specification.
> **Purpose:** Consolidate decisions made so far, surface open questions, and set up a v0.1.0 roadmap.

---

## 1. What this is

An opinionated, atomic, security-by-default Linux desktop operating system targeting RHEL-family stability (AlmaLinux base) with a KDE user environment. The system ships with a fixed-shape disk architecture, LUKS2 full-disk encryption, SELinux MLS, signed VM templates ("cubes") for compartmentalization, a broker-mediated IPC framework between cubes, and a duress-triggered destructive boot path ("Pyrrhic Boot"). Plausible deniability — a hidden OS sharing the encrypted data partition with the visible one — is a toggleable install option.

The product is the opinionated architecture itself. Users do not configure SELinux, LUKS, or the partition layout. They confirm a hardware tier, set passphrases, pick which cubes to instantiate, and the system takes care of the rest.

There is **no declarative-spec / config-language layer** inside this OS. That problem is explicitly out of scope. (Note: this is a deliberate boundary against the Ironclad project — Ironclad is the declarative compiler for general-purpose secure Linux; this OS is the shipped-as-code, no-spec-required appliance.)

---

## 2. Threat model

The system is designed against a layered threat model. It does not claim protection against every attacker, but defines an explicit cost-of-attack curve.

- **Evil maid / brief physical access.** Powered-off device taken for minutes-to-hours. Adversary has the disk image and can attempt offline cracking or boot-chain tampering.
- **Remote network and exploit attackers.** Standard internet-facing exposure: drive-by exploits, malicious downloads, network-borne lateral movement attempts against a single device.
- **Persistent malware.** Code that lands on the system and tries to survive reboot.
- **Cold-boot and DMA.** Opportunistic memory-scraping while the system is on or recently powered down.
- **Forensic / compelled inspection.** A trained examiner with the disk image and time. Not necessarily a state-level adversary — could be customs, civil discovery, or a divorce subpoena. They will look for crypto containers and they will ask about them.

The system is **not** designed to defeat:

- A nation-state-level adversary with hardware implants, supply-chain compromise, or unbounded budget.
- Compelled disclosure under jurisdictions where refusal to provide a passphrase is itself prosecutable (UK RIPA-style). The hidden OS feature lowers but does not eliminate this risk.
- An attacker with continuous physical possession over long periods.
- The user themselves choosing to install untrusted code in a worker cube and granting it broker access.

---

## 3. Core design principles

1. **Opinionated security floor, period.** SELinux enforcing, LUKS2, immutable root, signed boot chain, signed cube templates. Non-negotiable. The baseline cannot be downgraded.
2. **Forensically unsuspicious by default.** AlmaLinux + KDE reads as a stability-focused conservative user. Long gaps between updates are normal for AlmaLinux. The visible install looks like nothing in particular.
3. **Plausible deniability as a feature, not a research project.** Two LUKS partitions with identical crypto profiles. The data partition's "free" space is mathematically indistinguishable from a hidden OS, because that's exactly what it might be.
4. **Atomic and (where possible) immutable.** State transitions happen as complete units. Read-only `/usr` enforced by dm-verity. Updates ship as image transactions.
5. **Compartmentalization over hardening.** Workloads run in cubes (VMs). The host is small and boring. Breaking a cube costs you that cube, not the system.
6. **Hardware-tier graceful degradation.** A user with no TPM still gets a meaningfully secure system. A user with Coreboot/Heads gets the strongest version. The same OS, same install media, different ceiling.
8. **No declarative layer.** The opinionated architecture is shipped as code. Users do not write specs to install or to manage the system. Power-user customization (custom bootc images, etc.) lives outside the OS proper.

---

## 4. Fixed architecture (decided)

These are settled. Items here are not subject to per-install configuration; they are what the OS *is*.

### 4.1 Base platform

- **Host distribution:** AlmaLinux (RHEL 9.x family target initially; 10.x once tooling matures).
- **Desktop environment:** KDE Plasma. Chosen for forensic unsuspiciousness and a stable widget set that synthesizes a "lived-in" appearance well.
- **Init:** systemd (RHEL native; not relitigating).
- **Container/VM runtime:** libvirt + QEMU/KVM for cubes. Rootless Podman available for in-cube workloads.
- **Image lifecycle:** bootc / OSTree-based atomic updates.
- **Mandatory access control:** SELinux in enforcing mode with an opinionated MLS policy.

### 4.2 Disk layout

Fixed, identical across all installs (this is core to the deniability story — variation in layout is itself a forensic signal).

```
/dev/<disk>
  ├─ p1  EFI System Partition         1 GiB    FAT32, signed binaries only
  ├─ p2  /boot                        1 GiB    ext4, measured kernel+initramfs
  ├─ p3  LUKS2 — system partition     ~30 GiB  AES-XTS-512, Argon2id, hmac-sha256
  └─ p4  LUKS2 — data partition       rest     identical crypto profile to p3
```

(Naming above uses `p3`/`p4` for clarity; in the May 5 conversation these were called `p2`/`p3`. Resolve naming before writing installer code.)

**Both LUKS containers use identical crypto parameters.** Any deviation is itself a fingerprint. Same cipher, same KDF, same Argon2id memory/time costs, same integrity mode.

### 4.3 LVM layout (inside p3 — system partition)

```
VG: system
  ├─ lv_root         XFS, mounted /
  ├─ lv_usr          XFS + dm-verity (read-only, signed)
  ├─ lv_home         XFS
  ├─ lv_var          XFS
  ├─ lv_var_state    XFS  (libvirt definitions, broker state, sealed key blobs)
  ├─ lv_var_log      XFS  (host journal/syslog, capped rotation)
  ├─ lv_var_audit    XFS  (auditd append-only)
  └─ lv_var_tmp      XFS  (volatile)
```

`lv_var_state`, `lv_var_log`, and `lv_var_audit` are sized at ~4 GiB each based on the May 5 sketch. Final sizing is an open question (see §10).

### 4.4 Data partition (p4)

Inside p4 lives the **cube pool** — the storage backing for all cube disk images, user data, and (if enabled) the hidden OS. The pool is presented to the host as additional LVM, with cube images as logical volumes.

If the hidden OS feature is enabled at install time, the hidden OS's LUKS2 header is stored either as an unallocated extent inside the visible LVM or as a sealed file in `lv_var_state` (tier-dependent — see §6).

### 4.5 Cubes

Cubes are KVM virtual machines with a project-signed root image and per-cube state. They are the unit of compartmentalization.

- **Project-signed templates ship immutable.** A "browser cube," "documents cube," "comms cube," etc. are defined by templates the project owns and signs. Users cannot modify template internals.
- **Per-cube state** lives in the data partition.
- **Cubes communicate via the broker** (§4.7). Cubes do not get arbitrary host access, arbitrary network access, or arbitrary access to each other.
- **User-defined cubes** (custom worker VMs) are a future feature; v0.1 ships with the project-signed template set only.

### 4.6 MLS tiers

The opinionated MLS lattice is fixed. Concrete tier names and semantic mapping to cubes is one of the larger open questions (see §10), but the structural commitments are:

- A small, fixed number of named tiers.
- The host runs at a defined tier (likely the highest, or a separate "host" label outside the user lattice).
- Cubes are assigned to tiers by the project-signed template metadata; users do not choose a cube's tier ad-hoc.
- Cross-tier data movement requires broker mediation. There is no shared filesystem between tiers.

### 4.7 Broker

A host-side daemon that mediates inter-cube communication and any cube-to-host action. Default policies ship with the OS; per-cube-pair consent settings are adjustable in the running system.

The broker is the single chokepoint for:

- Clipboard between cubes
- File transfer between cubes
- Network access requests from cubes
- Device passthrough (USB, audio, camera) to cubes

If the broker dies or is compromised, the security model degrades. Hardening, restart semantics, and the broker's own attack surface are open questions.

---

## 5. Installer model

A custom, fixed-shape, two-phase installer. Not an Anaconda fork. Not a declarative-config evaluator.

### 5.1 Phase 1 — hardware detection and tier selection

- Detects: TPM 2.0 presence, IOMMU support, Secure Boot keys writability, CPU virtualization extensions, presence of an inserted USB key for tier-2 unlock.
- Proposes a hardware tier (0–3, see §6).
- User can confirm or downgrade. (Upgrading above what hardware supports is not possible.)

### 5.2 Phase 2 — fixed-shape install

User-facing choices in phase 2:

- Whether to enable hidden OS / deniability features.
- Passphrases (main, recovery, duress, hidden if enabled).
- Whether to enroll custom Secure Boot keys (PK/KEK/db).
- Account creation.
- Initial cube selection (which signed templates to instantiate).
- Optional cosmetic tier rename.

Everything else — partitioning, LUKS, LVM, SELinux policy install, broker bootstrap, signed verity image — runs without user input.

### 5.3 What is NOT customizable at install

- Filesystem architecture
- LUKS profile
- MLS tier model
- Disk shape
- Cube template internals (project-signed; immutable)
- Broker policy framework (defaults ship; runtime tuning only)

---

## 6. Hardware tier model

Same OS, same install media, four ceilings. The user does not pay for unavailable hardware in lost security guarantees beyond what the hardware itself prevents.

| Capability                        | Tier 0 (baseline)            | Tier 1 (+TPM)             | Tier 2 (+USB)             | Tier 3 (+Coreboot/Heads) |
|-----------------------------------|------------------------------|---------------------------|---------------------------|--------------------------|
| Host LUKS unlock                  | Passphrase                   | TPM+PIN                   | USB+PIN                   | TPM+PIN+measured boot    |
| Hidden OS header location         | File in decoy `/var/lib`     | TPM-sealed file           | USB-only                  | USB + TPM-sealed         |
| AEM detection                     | None                         | PCR mismatch on tamper    | —                         | Heads attestation        |
| Tamper counter                    | EFI variable (signed)        | TPM NV counter            | Same                      | TPM NV counter w/ Heads  |
| Pyrrhic Boot trigger              | Duress passphrase            | + tamper threshold        | Same                      | Same                     |
| Custom Secure Boot keys           | Optional, recommended        | Yes                       | Yes                       | N/A (Heads handles)      |

Tier 0 still gets full-disk authenticated encryption, immutable system, audit isolation, hidden OS with disk-only deniability, and a Pyrrhic Boot path. The degradation curve is real but acceptable.

---

## 7. Pyrrhic Boot

A duress passphrase entered at boot triggers a destructive sequence. The current sketch:

1. **Stage 0 (~50 ms):** `cryptsetup luksErase` on p4. Destroys the data partition's LUKS header. Cube pool and hidden OS (if any) gone.
2. **Stage 1:** Continue boot into a clean visible OS as if nothing happened. The user is logged into a system with no cubes and no data.
3. **Stage 2 (open):** What does the visible OS show? A pristine first-boot? An "everything is fine" desktop? This affects the cover story under examination.

The cover story for the now-empty data partition is: "I provisioned the partition with random data; LVM hasn't allocated those extents yet." This is true for the visible LVM after Pyrrhic Boot, and unfalsifiable without unlock material.

**Trigger mechanisms beyond duress passphrase** (Tier 1+): tamper counter exceeded, PCR mismatch detected, or admin policy. All open as to thresholds.

---

## 8. Hidden OS / deniability

Toggleable at install. When enabled:

- A second LUKS2 container is created **inside p4's free space**, with a separate header.
- The hidden OS contents are atomic and immutable, same as the visible OS.
- Whether the hidden OS is a full atomic image with different cubes, or a stripped variant with only essential cubes, is an open question (May 5 lean: full atomic, option a).
- Header location is tier-dependent (§6).

When disabled, the data partition's free space is genuinely free space — but the cover story still works because there is no way to tell from the disk alone.

---

## 9. Path to v0.1.0 — proposed scope

The first tagged release should be **the smallest thing that boots, encrypts, and demonstrates the architecture** — not a full-featured OS. Suggested v0.1.0 deliverables:

1. **Bootable installer ISO.** Phase 1 hardware detection (TPM only initially), phase 2 fixed install. Tier 0 and Tier 1 only. No USB or Heads.
2. **Two LUKS partitions, identical profiles, LVM inside p3.** No hidden OS yet — that's v0.2.
3. **Atomic AlmaLinux + KDE base** with dm-verity on `/usr` and SELinux enforcing. MLS lattice present but possibly minimal (host + one user tier).
4. **One signed cube template.** Likely a "general-purpose worker" cube. Demonstrates the cube boot path and broker handshake.
5. **Broker daemon** with the smallest viable policy: clipboard, file transfer between host and the single cube. No cube-to-cube yet.
6. **Pyrrhic Boot via duress passphrase** as the only trigger. Stage 0 + stage 1 only — stage 2 cover behavior deferred.
7. **Custom Secure Boot key enrollment as optional** install step.
8. **Documentation:** install guide, threat model, "what this is and isn't" page.

What v0.1.0 deliberately does **not** include: hidden OS, multi-cube broker policies, AEM detection, USB unlock, Heads integration, user-defined cubes, fleet update infrastructure, recovery passphrase complexity beyond a single static one.

---

## 10. Open questions

These are the calls that need to be made — by you, with input — before v0.1.0 is buildable. They are grouped roughly from "must answer first" to "can defer."

### 10.1 Identity and naming

1. **Project name.** ATL/Anti-Trust Linux, the original name, was decided to be too on-the-nose; the May 5 conversation didn't land on a replacement. What's the public name? What's the kernel/branding name vs. the marketing name?
2. **Relationship to Ironclad.** Same GitHub org? Same repo? Cross-link in READMEs? Are these explicitly sibling projects under one umbrella, or independent?
3. **License.** MIT (matches Ironclad)? GPLv2 (matches kernel/AlmaLinux ecosystem)? AGPL for the broker daemon specifically?
4. **Trademark posture.** Trademark the name now or leave it free?

### 10.2 Scope and audience

5. **Server fork.** Earlier conversations described a server-focused variant. Is the desktop OS the only product for the foreseeable future, or is server a v0.5+ track that should be visible in the architecture from day one?
6. **Single-user assumption.** Is the visible OS strictly single-user, or do we want multi-user from the start? Multi-user changes a lot about MLS, auditing, and cube ownership.
7. **Defense/government posture.** Is the project actively pursuing eventual defense use (CC certification path, FIPS modules, etc.) or is "secure for serious users" the ceiling?

### 10.3 Architecture details still to finalize

8. **Partition naming convention.** `p3`/`p4` here vs. `p2`/`p3` in the May 5 chat. Pick one.
9. **`lv_var_*` sizing.** 4 GiB each was a sketch. Real sizing depends on auditd retention policy, libvirt definition footprint, etc.
10. **Hidden OS contents.** Full atomic image with different cubes (option a, the lean) or stripped variant (option b)? Storage cost vs. operational cleanliness.
11. **Per-cube LUKS layer inside the cube pool.** Defense-in-depth at ~20% IOPS cost. With Tier 0, per-cube keys would have to derive from the master passphrase or sit plaintext in `lv_var_state`. Is the IOPS cost worth it? Is there a tier-conditional answer?
12. **MLS lattice — concrete tiers.** How many? What are they called? Is the host its own label or the top of the user lattice? How are cube templates assigned tiers?
13. **Boot LV vs. boot partition.** Earlier discussion settled on an unencrypted ext4 boot partition outside the LUKS container. Confirm and lock in.
14. **Filesystem choice on the data partition.** XFS like the system partition? Btrfs for snapshot support? ZFS is probably out for license reasons but worth naming explicitly.

### 10.4 Cubes and broker

15. **Initial cube template set.** What ships in v0.1? In v1.0? Browser, documents, comms, dev, scratch, vault?
16. **Cube template signing infrastructure.** Project key, key rotation procedure, where the public key lives (in the OS image, on the install media, both?).
17. **Broker policy language.** How are policies expressed? Static config files? A small DSL? (Note tension with §3 principle 8 — "no declarative layer." A broker policy file isn't the same as a system-spec language, but it's a slope.)
18. **Broker restart semantics.** What happens to active cube connections when the broker restarts? What does broker compromise look like and how do we contain it?
19. **User-defined cubes.** When does this land? v0.5? v1.0? Never?

### 10.5 Installer

20. **Installer implementation language.** Rust (matches Ironclad)? Python? Plain shell + dialog?
21. **Live ISO vs. anaconda-style two-boot install.** The May 5 conversation rejected forking Anaconda but didn't pick a positive alternative.
22. **Tier 1 unlock UX.** systemd-cryptenroll for TPM2+PIN — confirm. PCR set: 0, 2, 4, 7, 11 from May 5. Stick with that?
23. **Recovery passphrase model.** One static recovery passphrase per install? Multiple? Time-limited? What happens if it leaks?
24. **Duress passphrase entry.** Where in the boot flow? Same prompt as the main passphrase, or distinct? How do we prevent accidental entry?

### 10.6 Pyrrhic Boot

25. **Stage 2 cover behavior.** Pristine first-boot, "normal" desktop with no data, or something else?
26. **Tamper-threshold trigger thresholds.** PCR mismatch and tamper counter — what numbers? User-configurable or fixed?
27. **Admin-policy trigger.** Is there a remote/admin trigger for Pyrrhic Boot, or duress-passphrase-only? Remote triggers are powerful and dangerous.
28. **Recovery from accidental Pyrrhic Boot.** If a user fat-fingers the duress passphrase, the data is gone. What's the off-ramp? Is there one? (Probably not, by design — but should be stated explicitly.)

### 10.7 Updates and lifecycle

29. **Update channel infrastructure.** Mirror AlmaLinux's? Run our own? Cloudflare R2 / mirror network?
30. **Update verification.** ostree signatures, additional project signing, both?
31. **Update cadence.** Match AlmaLinux's stability cadence (months) or faster for security fixes?
32. **Cube template updates.** How are updated templates pushed? Are running cubes updated in place or replaced on next boot?
33. **End-of-life policy.** AlmaLinux 9 EOL is 2032. What's the project's commitment? Migration path to Alma 10?

### 10.8 Build, CI, distribution

34. **Build infrastructure.** GitHub Actions? Self-hosted runner on `mnas`? How is the verity-signed `/usr` image built reproducibly?
35. **ISO distribution.** GitHub Releases? Separate mirror? Torrent? Signed checksums obviously.
36. **Reproducible builds.** Aspirational from day one, or a v1.0+ goal?

### 10.9 Documentation and community

37. **Documentation site.** README + wiki, or a static site (mdBook, Hugo, Docusaurus)?
38. **Threat model document.** Public from day one. Who's the audience — security researchers, prospective users, both?
39. **Contribution policy.** Do we accept community PRs against the cube template set? Against the broker? Against the installer? How is template signing handled if so?
40. **Bug bounty / vuln disclosure.** Aspirational note in the README from v0.1, or wait until there's something worth reporting?

### 10.10 Hard-mode questions worth flagging now

41. **Legal review of deniability features.** Hidden OS in some jurisdictions is essentially fine; in others (UK RIPA, France in some readings, others) refusing to disclose can be prosecuted. Should the README carry a jurisdictional warning?
42. **Export classification.** Strong crypto + MLS + designed-for-defense framing may trigger US export controls (EAR, ITAR-adjacent). Worth a one-line check before public release.
43. **The "this looks like criminal evasion software" problem.** The honest framing — protecting journalists, dissidents, abuse survivors, professionals with confidential client data — needs to be on the front page of the README, not buried.
44. **Scope discipline.** Realistically, building v0.1 to the spec above is a year-plus of evening work alongside cert study, the resume site, Ironclad, and field work. Is there a smaller v0.1 that still demonstrates the core idea? "Just AlmaLinux + KDE + signed verity + one cube + duress wipe" might be enough.

---

## 11. What I'd push back on if I were external review

A few things to think about before this becomes a public commitment:

- The MLS-by-default story is a heavy lift. SELinux MLS policy authoring is a specialized skill, and getting it wrong produces a system that's either constantly broken or silently permissive. The project either needs deep SELinux expertise or needs to pull from an existing MLS reference policy and harden incrementally.
- The cube model overlaps significantly with Qubes OS. The differentiation story (RHEL base, signed templates, broker model, deniability, atomic immutability, MLS) is real, but should be articulated clearly so this doesn't read as "Qubes but with my preferences." Qubes-the-product has thousands of person-years of work in it.
- The broker is a single point of trust. If it's compromised, every cube's IPC is compromised. The broker's own threat model deserves its own document.
- "Forensically unsuspicious" and "ships hidden OS support" are in tension. A forensic examiner who knows the OS exists knows that any install of it *might* have a hidden volume. The deniability is "I might have one but you can't prove it from the disk," not "you'd never know to look." Worth being precise about in the README.
- Scope vs. timeline. Be honest with yourself about what one person on evenings can ship, then halve it for v0.1.

---

## 12. Immediate next steps (proposed)

When you pick this back up:

1. Decide §10.1 (name, license, repo relationship to Ironclad) — these gate everything else.
2. Pick the v0.1 minimum viable architecture from §9 + §10.10 (4) — what's the smallest demo that proves the concept?
3. Start a `THREAT_MODEL.md` from §2 — it's already 80% written above.
4. Stub a `ROADMAP.md` with v0.1 / v0.2 / v0.5 / v1.0 milestones once §9 firms up.
5. Open a GitHub Discussions thread (or a `decisions/` directory of ADRs) to log the answers to §10 as they're made, so this draft can be retired in favor of decision records.

---

*End of draft. — written from the May 5 secure-filesystem conversation. Anything important I missed should be added by you on next pass; anything wrong here should be struck through rather than deleted, so the reasoning trail is visible.*
