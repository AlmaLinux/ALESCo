# RFC: Build and Secure Boot-Sign a Mainline-Based Alternative Kernel

* **RFC Number:** `XXXX` (Assigned by the repository maintainers)
* **Author(s):** [Lance Albertson](lance@osuosl.org)
* **Status:** Draft
* **Created:** [2026-08-02 18:00 UTC]
* **Updated:** [2026-08-03 17:00 UTC]

## Abstract

This RFC proposes that AlmaLinux rebuild the
[CentOS Hyperscale SIG kernel](https://sigs.centos.org/hyperscale/contributing/kernel/)
— a rebuild of the Fedora/`kernel-ark` tree with an Enterprise Linux-shaped
configuration, currently Linux **7.1.3** — under the distinct package name
**`kernel-mainline`**, Secure Boot-sign it with AlmaLinux's existing key
material, and publish it in an opt-in, disabled-by-default repository as a
**Tech Preview**. It installs alongside the stock Enterprise Linux (EL) kernel,
never replacing it, and boots with Secure Boot **enabled** — something no
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
  * Ship a mainline-tracking kernel that installs alongside the stock EL
    kernel and never displaces it.
  * Enable hardware and kernel features that are impractical to backport into
    an EL kernel, particularly on fast-moving architectures such as riscv64.
  * Secure Boot-sign it so it boots with Secure Boot **enabled** where
    technically possible.
  * Publish it in an opt-in, disabled-by-default repository usable on both
    AlmaLinux Kitten and stable releases.
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
would genuinely be too much — every carried patch must meet all four criteria:

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
4. **Documented** in the package changelog and the "Deviations from RHEL"
   material, like every other AlmaLinux kernel deviation.

This also gives ALESCo's previously noted need — a process for users to
request hardware enablement — a concrete answer for this kernel: file the
request with the patch or upstream submission attached, and the criteria
above decide it.

### Naming and coexistence

Hyperscale ships as package `kernel` and wins by higher EVR — the opposite of
coexistence. AlmaLinux renames it via `SPECPACKAGE_NAME=kernel-mainline`, the
documented `kernel-ark` variable Red Hat uses for `kernel-automotive` (~10–15
spec lines; installed paths key on the kernel release string, so the two
kernels are naturally disjoint on disk). Coexistence rules:

1. **Drop `Provides: kernel = …`** and `Obsoletes: kernel-headers`/
   `kernel-cross-headers`, so DNF never selects `kernel-mainline` to satisfy a
   dependency on `kernel`. The uname-scoped provides (`kernel-uname-r`,
   `kernel-devel-uname-r`, `installonlypkg(kernel)`) are retained — they
   cannot collide with stock, and running-kernel protection and user module
   builds depend on them. The trade is deliberate: keeping the provide risks
   *silently* installing a Tech Preview kernel on supported systems, while
   dropping it confines breakage to unsupported mainline-only systems as a
   loud dependency error — and is a one-line revert if the reverse-dependency
   audit (Unresolved Questions) warrants.
2. **Disable the unprefixed userspace subpackages** (`perf`, `python3-perf`,
   `libperf`, `rtla`, `rv`, `kernel-tools`, `kernel-headers`, and related),
   which own unversioned paths and file-conflict with BaseOS — a defect the
   2024 prototype shipped. Hyperscale's own builds set the precedent. Newer
   tools remain available as prefixed companion packages (below).
3. **Install-only, never the default boot entry.** The stock kernel always
   remains installed as fallback; silently taking over the boot order (as
   both ELRepo's and the Kmods SIG's kernels do) is treated as a bug.
4. **Config baseline:** `CONFIG_SECONDARY_TRUSTED_KEYRING=y` and
   `CONFIG_SYSTEM_BLACKLIST_KEYRING=y` must match the stock kernel — the 2024
   prototype regressed both, which breaks MOK-enrolled module loading.

Running `kernel-mainline` exclusively (removing the stock kernel) is
**permitted but unsupported**: nothing blocks it, matching every distribution
with parallel kernel streams, but `Requires: kernel` becomes unsatisfiable,
the tested fallback is gone, and kABI-dependent kmods and Leapp will not work.
Documentation will recommend keeping a stock kernel installed.

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
RK3588-family boards) from `kernel-mainline` means deliberately enabling the
relevant platform drivers and DTBs, coordinated with the Hyperscale SIG. It
is worthwhile precisely because that support matures in mainline rather than
in any EL kernel, but it is scoped as config work, not an automatic property
of the rebuild.

### Secure Boot, SBAT, and disclosure

* A **dedicated signing certificate** under the AlmaLinux Secure Boot CA, per
  the [RFC 0004](0004-build-and-ship-nvidia-drivers.md) precedent — auditable
  and separable, with no expansion of CA-key access.
* A **distinct SBAT component**
  (`kernel-mainline.almalinux,1,AlmaLinux,kernel-mainline,<KVER>,mailto:security@almalinux.org`).
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

Version pinning follows the CentOS Kmods SIG's **stream-repository** model:
administrators pin a known-good series by installing the corresponding release
package rather than using `versionlock`, making "test and pin a known-good
version" — the core Tech Preview expectation — an ordinary repository
operation.

### Companion packages

Rule: ship a companion only if the kernel alone cannot deliver this RFC's
benefit without it, or if version skew loses real functionality — and name
what is declined, so scope is a decision rather than an accretion.

* **`linux-firmware`, rebuilt in lockstep (drop-in).** A mainline kernel
  enables drivers whose firmware postdates the EL snapshot (new amdgpu ASICs,
  Intel Wi-Fi `.ucode` revisions, NPU firmware); without it the driver loads
  and the device stays dead, hollowing out the hardware-enablement
  motivation. Upstream firmware is deliberately additive, and Red Hat itself
  rebases it mid-release. It ships under its stock name — all kernels share
  one `/usr/lib/firmware`, so side-by-side is not meaningful for firmware.
* **`kernel-mainline-perf` / `kernel-mainline-bpftool` (explicit swap).**
  Stock tools built from the EL kernel miss newer PMU events and cannot
  introspect newer BPF program types. These are built from the same SRPM under
  prefixed names with explicit `Conflicts:` on their stock counterparts;
  installing one is a deliberate `dnf swap`, never a side effect.
* **Declined: `systemd`, `selinux-policy`, `dracut`, `kpatch`.** Hyperscale
  couples its kernel to Fedora backports of the first three; AlmaLinux will
  not. A decade of ELRepo `kernel-ml` on stock dracut shows it unnecessary;
  `kpatch` has no livepatch stream to serve; `systemd` is the most invasive
  replacement possible (Hyperscale's documented SELinux problems trace to its
  systemd backport, not its kernel); and `selinux-policy` is system-global —
  it would govern the system even when booted into the stock kernel, and
  downgrading it requires a full relabel, defeating the fallback story. The
  cost of declining `selinux-policy` is stated under Drawbacks.

For users who enable the repository, `linux-firmware` is the **only** package
that updates without explicit installation; everything else lands only through
`dnf install` or `dnf swap`.

### Testing

Release gates, run Kitten-first:

1. **Boot test**, following the Kmods SIG pipeline shape (install
   `kernel-mainline-core`/`-modules`, reboot, assert `uname -r`); their
   pipeline already builds against `almalinux-$EL` mock roots.
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
2. Rename via `SPECPACKAGE_NAME=kernel-mainline` and apply the coexistence
   rules above.
3. Wire signing: AlmaLinux certificates, the current `pesign` identity, and
   the dedicated certificate; assert the lockdown and keyring configuration.
4. Mint the `kernel-mainline` SBAT component.
5. Stand up the repository and release package; populate with the kernel and
   companion packages.
6. Build and test on Kitten first (the staging precedent ALESCo approved for
   RFC 0005), then AlmaLinux 10, then AlmaLinux 9.
7. Publish the Tech Preview support statement and per-architecture tier table
   alongside the existing "Deviations from RHEL" material.
8. Disclose the new signed variant at the next shim-review submission.

### Compatibility

Self-contained and opt-in. Users who never enable the repository are
unaffected. For users who do, exactly one package changes without explicit
installation — the lockstep `linux-firmware` — and the stock kernel remains
the default boot entry and the fallback. Kernel↔userspace couplings are mostly
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
  out of scope. (`%kernel_module_package` also cannot currently detect a
  renamed kernel — follow-on work if ever wanted.)
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
  kernel compiles plus debuginfo per release, across Kitten, AlmaLinux 10 and
  AlmaLinux 9. Bounded by shipping exactly one alternative kernel, per
  ALESCo's stated constraint; carried enablement patches add rebase work,
  bounded by the owned-and-self-liquidating rule. Real numbers are needed
  from the Build System SIG.
* **The testing gap is real work.** ALTS installs RPMs in containers (which
  cannot boot a kernel) and openQA has no kernel or Secure Boot modules; the
  test plan above is a deliverable of this RFC, not an assumption.
* **Expanded signing surface.** A lockdown bypass in a fast-moving kernel is
  a plausible revocation event. Mitigated by the dedicated certificate, the
  distinct SBAT component (scoping revocation to `kernel-mainline` alone),
  and lockdown enforcement as a release gate.
* **Support-expectation risk.** "Signed by AlmaLinux" may read as "fully
  supported." Mitigated by the published Tech Preview statement: regressions
  expected, no kABI, no out-of-tree modules, pin a known-good stream, not for
  unattended production use.
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
    streams. **Documentation** — installation instructions, the support
    statement, and the tier table.
* **Policies and guidelines:** A published Tech Preview support statement
  (lifecycle, no kABI guarantee, exclusion of out-of-tree modules, per-arch
  signing tiers, pinning guidance).
* **Trademark approval:** N/A.
* **Board input:** This touches signing-key handling and may represent a
  significant resource increase; whether
  [board notification](../README.md#almalinux-os-foundation-board-input-as-a-blocker)
  is required before a vote should be confirmed during discussion.

## Unresolved Questions

* **Build and storage cost** — per-arch build time, builder saturation and
  mirror footprint need real numbers from the Build System SIG (this also
  settles the Board-input question).
* **Release scope** — Kitten → AlmaLinux 10 → AlmaLinux 9 is proposed; is
  AlmaLinux 9 worth including, given its older userspace?
* **Rolling-latest vs. longterm streams** — longterm (~1 disruptive
  transition/year vs. ~6) fits the Tech Preview tier better; needs a
  decision.
* **Default boot entry and `installonly_limit`** — installing
  `kernel-mainline` must not change the default boot entry; the mechanism,
  and the interaction of `installonly_limit` with two install-only kernel
  names, need empirical confirmation.
* **Mainline-only systems** — audit `Requires: kernel` reverse-dependencies
  across BaseOS/AppStream/EPEL, and confirm DNF's `protect_running_kernel`
  keys on the retained `kernel-uname-r` provide; if it instead needs the
  dropped `Provides: kernel`, a booted `kernel-mainline` could be silently
  removable — a must-fix before release.
* **Firmware cadence and packaging shape** — when the lockstep
  `linux-firmware` rebases, and whether it preserves EL's subpackage split or
  adopts Fedora's layout (affects upgrade paths in both directions).
* **ARM board scope and the Raspberry Pi kernel** — which aarch64 SBC
  platforms the configuration should enable, and the long-term relationship
  with `raspberrypi2-kernel4`: `kernel-mainline` complements the vendor-fork
  kernel initially, but the two could converge as mainline Raspberry Pi
  support matures. The AltArch SIG, which owns the SBC ports, should weigh
  in — as should it on who adjudicates carried enablement patches against the
  four criteria (likely the kernel configuration-policy reviewers).
* **Retirement policy** — the trigger and process for falling back to a
  direct `kernel-ark` rebase if the upstream SIG stalls.
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
