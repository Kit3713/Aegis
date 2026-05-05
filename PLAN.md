# Project Plan — Draft v0.0

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

- A small, fixed number of named tiers (likely 3–5).
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
