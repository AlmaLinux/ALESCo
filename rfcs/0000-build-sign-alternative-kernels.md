# RFC: Build and Secure Boot-Sign a Mainline-Based Alternative Kernel

* **RFC Number:** `XXXX` (Assigned by the repository maintainers)
* **Author(s):** [Lance Albertson](lance@osuosl.org)
* **Status:** Draft
* **Created:** [2026-08-02 18:00 UTC]
* **Updated:** [2026-08-16 19:00 UTC]

## Abstract

This RFC proposes that AlmaLinux rebuild the
[CentOS Hyperscale SIG kernel](https://sigs.centos.org/hyperscale/contributing/kernel/)
— a rebuild of the Fedora/`kernel-ark` tree with an Enterprise Linux-shaped
configuration, currently Linux **7.1.3** — Secure Boot-sign it with
AlmaLinux's existing key material, and publish it in a dedicated opt-in,
disabled-by-default repository as a **Tech Preview**. The package keeps the
name **`kernel`**: enabling the repository opts a system into the
mainline-based kernel as a full replacement for the stock Enterprise Linux
(EL) kernel — one that boots with Secure Boot **enabled**, something no
CentOS SIG kernel or third-party EL kernel can offer today.

This completes work AlmaLinux already started: `kernel-ml`/`kernel-lt`
packaging was built in AlmaLinux dist-git in April 2024, and ALESCo confirmed
"building works, signing works" in August 2024. That effort stalled for want of
an owner and a repository to ship into; this RFC supplies both. It supersedes
the earlier proposal in [PR #15](https://github.com/AlmaLinux/ALESCo/pull/15).

## Motivation

* **Problem Statement:**

  Users needing a kernel newer than the EL stream kernel must today either do
  without, leave AlmaLinux, or install a third-party kernel and **disable
  Secure Boot**: ELRepo's `kernel-ml`/`kernel-lt` are deliberately unsigned,
  the CentOS Kmods SIG documents its kernels as unsigned, and the Hyperscale
  kernel's own `vmlinuz` is signed only by a Red Hat *test* certificate.
  Disabling Secure Boot is not an option in regulated environments.

  Demand is concrete: 59 of the 161 bug IDs in the 500–660 range on
  [bugs.almalinux.org](https://bugs.almalinux.org/) are kernel driver backport
  requests, largely from Microsoft Azure, Google and AWS engineers. AlmaLinux's
  correct answer — "it must land in CentOS Stream/RHEL first" — routinely comes
  with a pointer to ELRepo `kernel-ml`, a workaround that costs the user
  Secure Boot.

  A second class of demand cannot be served by backports at all: enablement
  that depends on new subsystem infrastructure or ISA support. riscv64
  illustrates the dynamic. The Kitten 10 riscv64 port ships the stock EL
  kernel — Red Hat actively backports board support into the CentOS Stream 10
  kernel — yet 589 configuration symbols in the Hyperscale 7.1.3 riscv64
  config do not exist in 6.12 at all: SoC platforms that postdate it
  (Tenstorrent, Axiado, CIX), newer SoC drivers (SpacemiT K1 EMAC, ESWIN
  PCIe), new SBI infrastructure, and RISC-V user-mode control-flow integrity.
  None are backport candidates. The dynamic is the same for ARM boards:
  single-board computer support (Raspberry Pi, Rockchip RK3588-family
  devices) matures in mainline continuously, while AlmaLinux today serves the
  Raspberry Pi through a separate vendor-fork kernel — `raspberrypi2-kernel4`,
  built from the Raspberry Pi Foundation tree — and other popular boards only
  as far as the server-oriented EL kernel happens to reach. And it recurs for
  new GPU/NPU, Wi-Fi and platform support on x86_64.

  These two motivations do not overlap evenly: hardware enablement is the
  stronger argument on riscv64, which cannot be Secure Boot signed at all
  today; Secure Boot is the stronger argument on x86_64 and aarch64. The
  per-architecture table below states which benefit applies where.

  AlmaLinux is uniquely positioned to supply the signature. Its
  [Secure Boot CA](https://wiki.almalinux.org/development/private-keys/secure-boot.html)
  is embedded in the already-Microsoft-signed shim as a true CA
  (`Basic Constraints: CA:TRUE`), so an additional kernel can be signed with
  **no shim change, no Microsoft re-signing, and no new shim review**.

* **Goals:**
  * Ship a mainline-tracking kernel as an explicit, opt-in replacement for
    the stock EL kernel, delivered through a dedicated repository.
  * Enable hardware and kernel features that are impractical to backport into
    an EL kernel, particularly on fast-moving architectures such as riscv64.
  * Secure Boot-sign it so it boots with Secure Boot **enabled** where
    technically possible.
  * Publish it in an opt-in, disabled-by-default repository, initially for
    AlmaLinux 10 and AlmaLinux Kitten 10.
  * Define a clear Tech Preview support tier, including version-pinning
    guidance and an explicit statement of what is *not* promised.

## Detailed Design

### Why the Hyperscale kernel, and not pristine mainline

ALESCo's recorded position (2025-10-30) is to choose one kernel, most likely
between Hyperscale and ELRepo mainline. The Secure Boot requirement decides
it. The mechanism that *activates* lockdown under Secure Boot
(`CONFIG_LOCK_DOWN_IN_EFI_SECURE_BOOT` and
`drivers/firmware/efi/secureboot.c`) **does not exist upstream** — it is a
downstream patch series carried by Fedora/RHEL derivatives. A signed but
unpatched mainline kernel therefore boots under Secure Boot without ever
entering lockdown, leaving an unrestricted `kexec` path. shim-review has
already rejected exactly this claim from another EL rebuild distribution
([#339](https://github.com/rhboot/shim-review/issues/339)):

> "I had a kernel which had all patches applied, but then it didn't have the
> lockdown on enabled secure boot enabled, so one could simply `kexec` without
> any restrictions."

A later submission from the same vendor, signing longterm streams *with*
enforced lockdown, was accepted in June 2026
([#565](https://github.com/rhboot/shim-review/issues/565)).

The Hyperscale kernel carries the lockdown series with
`CONFIG_LOCK_DOWN_IN_EFI_SECURE_BOOT=y` on x86_64/aarch64, and is the only
candidate with kernel configurations for all five AlmaLinux architectures (the
Kmods SIG kernel has no Secure Boot machinery and no riscv64 or s390x support).
Its delta is small and auditable — 67 of 8,094 shared config symbols differ
from AlmaLinux 10, and its entire downstream delta over `kernel-ark` is eleven
changelog items with **zero kernel source patches** — meeting the standard
ALESCo set in 2024: "we have to be able to trust the full tree and trace back
all changes to the origin." [RFC 0005](0005-enable-btrfs-as-tech-preview.md)
already names a rebuilt Hyperscale kernel as its Btrfs fallback, and its
maintainer is an AlmaLinux kernel contributor.

AlmaLinux's own 2024 prototype confirms the danger of the pristine-mainline
path: the `a9-ml`/`a9-lt` branches (ELRepo-derived, unpatched kernel.org
source) enabled Secure Boot signing while their configs lack lockdown
(`CONFIG_LOCK_DOWN_KERNEL_FORCE_NONE=y`, no `LOCK_DOWN_IN_EFI_SECURE_BOOT`)
and disable the secondary trusted keyring. Signed-but-not-locked-down is
precisely the shape shim-review rejects; this RFC must not repeat it.

### Carried hardware-enablement patches

The rebuild itself stays patch-free — Hyperscale's zero-source-patch delta is
part of this RFC's trust argument. On top of that base, this RFC leaves room
for a small, bounded set of **hardware-enablement patches** (board support for
aarch64 SBCs or riscv64 platforms) that apply cleanly to the current kernel.
Precedent exists at every level: `kernel-ark` itself carries arm64 platform
errata, the Kmods SIG kernel inherits the same series, AlmaLinux's stock
*signed* kernel already carries PCI-ID re-enablement patches disclosed and
accepted at shim-review, and the riscv64 port applied the CentOS ISA SIG
enablement stack.

To keep this from growing into a vendor BSP kernel — the failure mode that
would genuinely be too much — every carried patch must meet all three criteria:

1. **Upstream-bound**: a backport of a commit already merged in a later
   mainline release or linux-next, or submitted upstream with a tracking
   link. Vendor BSP trees (e.g. the Raspberry Pi Foundation's, thousands of
   patches deep) are categorically out — that is what `raspberrypi2-kernel4`
   is for.
2. **Signing-neutral**: touches no Secure Boot, lockdown, module-signing or
   keyring code, and is disclosed at the next shim-review submission — the
   pattern AlmaLinux's PCI-ID patches already established.
3. **Owned and self-liquidating**: a named owner refreshes it across rebases
   or it is dropped; it is retired automatically once the enablement lands in
   the shipped kernel version.

This also gives ALESCo's previously noted need — a process for users to
request hardware enablement — a concrete answer for this kernel: file the
request with the patch or upstream submission attached, and the criteria
above decide it.

### Packaging model

The kernel keeps the name **`kernel`** and ships as a full replacement,
exactly as the Hyperscale SIG packages it: the mainline version is always
higher than the EL stream version, so on a system with the repository
enabled, DNF selects it and subsequent updates track it. This keeps the
packaging delta from Hyperscale near zero — no rename, no subpackage
surgery — and `perf`, `bpftool`, `kernel-tools` and `kernel-headers` update
in step with the kernel the same way. Three points stated plainly:

1. **Opting in happens at repository-enable time.** Enabling the repository
   and updating replaces the kernel and its tool subpackages; this is the
   intended, documented behavior, not a side effect. Opting back out means
   disabling the repository and reinstalling the stock kernel.
2. **Provenance stays visible in the NEVRA.** The release string keeps a
   distinctive marker (Hyperscale's `.hs` variant-stream pattern or an
   AlmaLinux equivalent), so `uname -r` and `rpm -q kernel` always show which
   kernel a system runs — important for bug triage. Previously installed
   stock kernels remain bootable as older install-only entries until they
   rotate out of `installonly_limit` retention; documentation will cover how
   to retain a stock fallback entry.
3. **Config baseline:** `CONFIG_SECONDARY_TRUSTED_KEYRING=y` and
   `CONFIG_SYSTEM_BLACKLIST_KEYRING=y` must match the stock kernel — the 2024
   prototype regressed both, which breaks MOK-enrolled module loading.

### Architectures and signing tier

All five architectures are in scope for **building**; signing is not uniform,
and this RFC states the tier explicitly rather than implying a blanket
guarantee:

| Architecture | Build | Kernel image signature | Lockdown under Secure Boot |
|---|---|---|---|
| **x86_64** | Yes | UEFI shim chain, `pesign`, AlmaLinux cert | Yes |
| **aarch64** (4K and 64K pages) | Yes | UEFI shim chain, `pesign`, AlmaLinux cert | Yes |
| **ppc64le** | Yes | Module signing (`rpm-sign --lkmsign`); platform integrity via IBM OPAL | No EFI lockdown path exists |
| **s390x** (incl. `zfcpdump`) | Yes | Module signing (`rpm-sign --lkmsign`) | Via the IPL secure flag |
| **riscv64** | Yes | **Kernel image unsigned**; modules signed | None available |

riscv64 cannot be Secure Boot signed today by anyone: it is absent from every
candidate's `secure_boot_arch`, `almalinux-sb-certs` ships no riscv64
certificate, AlmaLinux's own `shim-riscv64` is built unsigned, and no riscv64
lockdown patch exists upstream. It ships **build-only, unsigned** — still
worthwhile, as riscv64 is the strongest hardware-enablement case and its users
lose nothing relative to today. Note that Hyperscale has riscv64 configs but
has never built the architecture; AlmaLinux would be the first, and this is
real work rather than a proven rebuild. aarch64 **16K** page variants are out
of scope (EL-target configs exist only on Hyperscale's dormant Asahi branch,
with no CI coverage anywhere).

aarch64 board enablement is a configuration decision, not a given: the
EL-flavor config targets servers, so supporting SBCs (Raspberry Pi,
RK3588-family boards) from this kernel means deliberately enabling the
relevant platform drivers and DTBs, coordinated with the Hyperscale SIG. It
is worthwhile precisely because that support matures in mainline rather than
in any EL kernel, but it is scoped as config work, not an automatic property
of the rebuild.

### Secure Boot, SBAT, and disclosure

* A **dedicated signing certificate** under the AlmaLinux Secure Boot CA, per
  the [RFC 0004](0004-build-and-ship-nvidia-drivers.md) precedent — auditable
  and separable, with no expansion of CA-key access. This is the plan pending
  a validation test that the shim trust chain accepts a new leaf under the
  embedded CA as anticipated (shim trusts the CA itself rather than any
  specific leaf); reusing the existing signing certificate is the fallback.
* A **distinct SBAT component** (e.g.
  `kernel-mainline.almalinux,1,AlmaLinux,kernel-core,<KVER>,mailto:security@almalinux.org`
  — the component name is independent of the package name).
  SBAT revocation applies to component *names*: sharing the stock kernel's
  component would let a single lockdown bypass in a Tech Preview kernel force
  a revocation that renders every AlmaLinux kernel unbootable. Entries derived
  from an upstream distribution build are appended, never replaced, per
  shim-review guidance.
* Signing a new kernel variant is a **disclosure item at the next shim-review
  submission, not a re-review trigger** (new architectures require re-review;
  a variant does not). AlmaLinux's standing "we're following RHEL kernel"
  answer must be updated accordingly.

### Repository and version pinning

The 2024 effort was buildable and signed but had nowhere to ship — no kernel
repository definition exists in `almalinux-release` to this day. This RFC
specifies one: a dedicated **disabled-by-default** repository (with
`-debuginfo` and `-source` stanzas), enabled via an
`almalinux-release-kernel-mainline` package in Extras, following the pattern
of [RFC 0004](0004-build-and-ship-nvidia-drivers.md) and
[RFC 0001](0001-build-fedora-epel-for-almalinux-and-almalinux-kitten-x86_64_v2.md).
A single top-level repository tree serves both Kitten and stable releases.
AlmaLinux already ships a non-EL kernel exactly this way — the Raspberry Pi
kernel in its dedicated `raspberrypi` repository — so the pattern is proven.

**Initial scope is AlmaLinux 10 and Kitten 10 only.** AlmaLinux 9 is
excluded: CentOS Stream 9 — the build target for the Hyperscale EL9 kernel —
reaches end of life soon, and AlmaLinux 9 is already at the point in its
lifecycle where, under the expectation stated in Scope, the offering would be
winding down anyway.

**Cadence is rolling-latest, following the Hyperscale SIG directly.** The SIG
tracks the Fedora/`kernel-ark` releases (historically a median of about three
days behind each ark tag), and AlmaLinux rebuilds what the SIG publishes
rather than maintaining separate longterm streams of its own — the
lowest-maintenance option, with zero added divergence from the upstream SIG.
Administrators who need to hold a known-good version use standard DNF
`versionlock` or excludes, and previously installed kernels remain bootable
under install-only retention; "test before rolling forward" is the core Tech
Preview expectation.

### Companion packages

Rule: ship a companion only if the kernel alone cannot deliver this RFC's
benefit without it — and name what is declined, so scope is a decision rather
than an accretion.

* **`linux-firmware`, rebuilt in lockstep (drop-in).** A mainline kernel
  enables drivers whose firmware postdates the EL snapshot (new amdgpu ASICs,
  Intel Wi-Fi `.ucode` revisions, NPU firmware); without it the driver loads
  and the device stays dead, hollowing out the hardware-enablement
  motivation. Upstream firmware is deliberately additive, and Red Hat itself
  rebases it mid-release. It ships under its stock name — all kernels share
  one `/usr/lib/firmware`, so side-by-side is not meaningful for firmware.
  The Hyperscale SIG validates the approach: it ships its own Fedora-derived
  `linux-firmware` rebuild alongside its EL9 kernel.
* **Declined: `systemd`, `selinux-policy`, `dracut`, `kpatch`.** Hyperscale
  couples its kernel to Fedora backports of the first three; AlmaLinux will
  not. A decade of ELRepo `kernel-ml` on stock dracut shows it unnecessary;
  `kpatch` has no livepatch stream to serve; `systemd` is the most invasive
  replacement possible (Hyperscale's documented SELinux problems trace to its
  systemd backport, not its kernel); and `selinux-policy` is system-global —
  it would govern the system even when booted into the stock kernel, and
  downgrading it requires a full relabel, defeating the fallback story. The
  cost of declining `selinux-policy` is stated under Drawbacks.

Newer `perf`, `bpftool` and `kernel-tools` need no companion treatment under
the replacement model: they are subpackages of the kernel SRPM and update in
step with it. Enabling the repository therefore opts a system into the
kernel, its tool subpackages, and the lockstep `linux-firmware`; nothing
outside that set is touched.

### Testing

Release gates, run Kitten-first:

1. **Boot test**, following the Kmods SIG pipeline shape (install the
   repository's `kernel-core`/`kernel-modules`, reboot, assert `uname -r`);
   their pipeline already builds against `almalinux-$EL` mock roots.
2. **Structural checks** via `kernel-ark`'s portable `make dist-self-test`.
3. **A Secure Boot-enabled boot test** — no existing upstream pipeline runs
   one, and it is this kernel's entire value proposition.
4. **A lockdown assertion** after boot. CVE-2025-1272 — where an unrelated
   upstream init-order change silently broke Fedora/RHEL automatic lockdown —
   proves this can regress without local changes and must be a gate.
5. **A firmware regression check under the stock kernel**, since
   `/usr/lib/firmware` is shared by all installed kernels.

Red Hat's CKI gating is not reusable (Red Hat-internal); AlmaLinux supplies
its own. **No kABI stability is promised**: Hyperscale disables kABI checking
and mainline offers no ABI guarantee — defensible for a Tech Preview, and
stated plainly.

### Implementation Plan

1. Import the Hyperscale sources into AlmaLinux dist-git and apply the
   standing debranding patch.
2. Add the AlmaLinux packaging guard (the analogue of Hyperscale's
   `centos_hs` conditional), the release-string marker, and the config
   baseline above.
3. Wire signing: AlmaLinux certificates, the current `pesign` identity, and
   the dedicated certificate; assert the lockdown and keyring configuration.
4. Mint the distinct SBAT component.
5. Stand up the repository and release package; populate with the kernel and
   companion packages.
6. Build and test on Kitten first (the staging precedent ALESCo approved for
   RFC 0005), then AlmaLinux 10.
7. Publish the Tech Preview support statement, lifecycle expectations and
   per-architecture tier table in the AlmaLinux documentation.
8. Disclose the new signed variant at the next shim-review submission.

### Compatibility

Self-contained and opt-in at the repository level. Users who never enable the
repository are unaffected. Enabling it is the explicit decision to run the
mainline-based kernel: the kernel, its tool subpackages and the lockstep
`linux-firmware` then track the repository on normal updates, and previously
installed stock kernels remain bootable until they rotate out of install-only
retention. Kernel↔userspace couplings are mostly
low-risk: netlink userspace (`iproute2`, `ethtool`, `nftables`) is
forward-compatible by design, and EL SELinux policy permits unknown classes,
so a newer kernel produces informational messages rather than denials. The
genuinely coupled areas are listed under Drawbacks.

## Drawbacks

* **Out-of-tree modules, including NVIDIA, will not work.** Prebuilt kmods
  depend on EL kABI (AlmaLinux's own `kmod-nvidia-open` carries ~675 versioned
  symbol dependencies); mainline offers no such stability, the Kmods SIG
  builds NVIDIA kmods only for stock EL kernels, and NVIDIA's open modules
  have needed fixes on essentially every recent mainline release. Explicitly
  out of scope.
* **Userspace coupling.** Functionality requiring a newer `systemd` is
  unavailable, and new kernel interfaces run **unconfined** under the older
  SELinux policy — nothing breaks, but confinement coverage silently narrows
  as the kernel adds attack surface. Documented as a Tech Preview limitation.
* **The shared firmware directory.** The lockstep `linux-firmware` also
  serves — and could in principle regress — the stock kernel; hence the
  stock-kernel firmware gate. It also adds several hundred MB of mirror
  weight.
* **Single external maintainer.** Every Hyperscale kernel commit is authored
  by one person, and the release record shows a 7.5-month gap. Mitigations:
  that maintainer is an AlmaLinux contributor, and the tree is a thin delta
  over `kernel-ark`, which supports downstream flavors (the
  `kernel-automotive` precedent) should AlmaLinux need to rebase directly.
* **Build, mirror and maintenance cost.** Roughly eight additional full
  kernel compiles plus debuginfo per release, across Kitten and
  AlmaLinux 10. Bounded by shipping exactly one alternative kernel, per
  ALESCo's stated constraint; carried enablement patches add rebase work,
  bounded by the owned-and-self-liquidating rule. Real numbers are needed
  from the Build System SIG.
* **The testing gap is real work.** ALTS installs RPMs in containers (which
  cannot boot a kernel) and openQA has no kernel or Secure Boot modules; the
  test plan above is a deliverable of this RFC, not an assumption.
* **Expanded signing surface.** A lockdown bypass in a fast-moving kernel is
  a plausible revocation event. Mitigated by the dedicated certificate, the
  distinct SBAT component (scoping revocation to the mainline kernel alone),
  and lockdown enforcement as a release gate.
* **Support expectations.** As with everything AlmaLinux ships, support is
  community-based. The published Tech Preview statement sets accurate
  expectations for a fast-moving kernel: it tracks newer upstream releases,
  carries no kABI guarantee or out-of-tree module support, and
  production-sensitive deployments should hold tested versions and roll
  forward deliberately.
* **riscv64 ships unsigned** — stated in the tier table rather than glossed;
  riscv64 users lose nothing relative to today.

## Benefit to AlmaLinux

* **A capability nobody else offers.** A Secure Boot-signed,
  mainline-tracking kernel for Enterprise Linux does not exist anywhere
  today; AlmaLinux already owns the one asset — a shim-trusted CA — that
  closes the gap.
* **Hardware enablement with security intact**, preserving deployability
  where Secure Boot is mandated, plus a delivery path for enablement that
  backporting cannot reach (new SoCs, ISA extensions, ARM board support) —
  mattering most on riscv64 and aarch64 single-board computers.
* **A better answer to a recurring support burden** than "wait for Stream" or
  "use ELRepo and disable Secure Boot."
* **Completes work already paid for** — the 2024 packaging and the "building
  works, signing works" prototype — and reinforces AlmaLinux's "innovate on
  Enterprise Linux" position alongside Btrfs, frame pointers and the
  x86_64_v2 rebuild.

## Scope

* **Proposal Owners:**
  * **Lance Albertson** — champion (designated by ALESCo, 2026-06-04):
    design, coordination, and driving to completion. The 2024 effort's
    failure mode was the absence of a named owner.
  * **Kernel packaging** — the packagers who maintain AlmaLinux's kernel
    dist-git today (debranding, SBAT templates, the ppc64le KVM variant);
    kernel work in AlmaLinux is owned by named packagers, not a SIG, and this
    RFC reflects that.
  * **Kernel configuration policy** — reviewed by the contributors who own it
    (per RFC 0005), coordinating with the Hyperscale SIG so modules AlmaLinux
    needs stay enabled upstream.
* **Other Developers:**
  * **Build System SIG** — build targets, the Secure Boot build path, sign
    nodes, repository publication, and mirror onboarding.
  * **Secure Boot key holders** — issue the dedicated certificate; update the
    next shim-review submission.
  * **Community and testers** — validate builds and identify known-good
    versions. **Documentation** — installation instructions, the support
    statement, and the tier table.
* **Policies and guidelines:** A published Tech Preview support statement
  (community support, as with all AlmaLinux deliverables; no kABI guarantee;
  exclusion of out-of-tree modules; per-arch signing tiers; pinning
  guidance). The working lifecycle expectation is that a given major's
  mainline kernel is maintained for roughly the first 4.5–5 years of that
  major, ending around the major.9 release, with exact timing (± roughly six
  months) subject to further discussion.
* **Trademark approval:** N/A.
* **Board input:** This touches signing-key handling and may represent a
  significant resource increase; whether
  [board notification](../README.md#almalinux-os-foundation-board-input-as-a-blocker)
  is required before a vote should be confirmed during discussion.

## Unresolved Questions

* **Build and storage cost** — per-arch build time, builder saturation and
  mirror footprint need real numbers from the Build System SIG (this also
  settles the Board-input question).
* **Dedicated-certificate validation** — confirm the shim trust chain accepts
  a new leaf under the embedded CA as anticipated; reusing the existing
  signing certificate is the fallback if it does not.
* **Firmware packaging details** — the Hyperscale SIG's own `linux-firmware`
  rebuild (its `c9s-hs` branch: the Fedora dist-git rebuilt for EL, bumped at
  each upstream firmware release) is the natural template, settling both
  cadence and layout. Remaining: confirm the Fedora per-vendor subpackage
  layout upgrades cleanly over AlmaLinux 10's stock split, and whether Kitten
  needs the rebuild at all — CentOS Stream keeps its firmware comparatively
  current, which is why Hyperscale rebuilds firmware only for EL9.
* **ARM board scope and the Raspberry Pi kernel** — which aarch64 SBC
  platforms the configuration should enable, and the long-term relationship
  with `raspberrypi2-kernel4`: the mainline kernel complements the
  vendor-fork kernel initially, but the two could converge as mainline
  Raspberry Pi support matures. The AltArch SIG, which owns the SBC ports,
  should weigh in — as should it on who adjudicates carried enablement
  patches against the three criteria (likely the kernel configuration-policy
  reviewers).
* **Upstream stall contingency** — the trigger and process for falling back
  to a direct `kernel-ark` rebase if the Hyperscale SIG stalls (its release
  record includes a 7.5-month gap).
* **Coordination with ELRepo** — their maintainers have publicly offered to
  help; is a formal coordination statement worthwhile to avoid duplicated
  effort and user confusion?
* **History of the 2024 effort** — why it stopped (no reason is recorded
  anywhere), and whether any signed `kernel-lt` build reached users: one
  dist-git build tag exists, and those builds were signed **without**
  lockdown enforcement.

## Acknowledgments

* [James Reilly](https://github.com/hanthor), who authored the original
  proposal ([PR #15](https://github.com/AlmaLinux/ALESCo/pull/15)) this RFC
  builds on and supersedes.
* Yuriy Kohut and Eduard Abdullin for the 2024 `kernel-ml`/`kernel-lt`
  packaging work this proposal completes, and Andrew Lukoshko for reviewing
  it.
* Neal Gompa, for maintaining the CentOS Hyperscale SIG kernel and for the
  RFC 0005 Btrfs work this proposal builds on.
* The ALESCo members whose review of PR #15 identified the gaps this RFC sets
  out to close.
* The CentOS Hyperscale and Kmods SIG contributors, whose kernels and release
  automation this proposal reuses, and Phil Perry and the ELRepo maintainers,
  whose `kernel-ml`/`kernel-lt` work has served EL users for over a decade.
