# RFC: Build Fedora EPEL for AlmaLinux and AlmaLinux Kitten x86_64_v2

* **RFC Number:** `0001`
* **Author(s):** [Jonathan Wright](jonathan@almalinux.org), [Neal Gompa](ngompa@almalinux.org), [Andrew Lukoshko](alukoshko@almalinux.org)
* **Status:** Draft
* **Created:** [2025-01-29 16:00 UTC]
* **Updated:** [2025-01-29 16:00 UTC]

## Abstract

Rebuild Fedora's EPEL (Extra Packages for Enterprise Linux) to support x86_64_v2 in AlmaLinux 10, and AlmaLinux Kitten 10.  ALESCo unanimously approved a proposal to rebuild AlmaLinux 10 for x86_64_v2 as an additional architecture on 2024-07-24 in the first [meeting](https://wiki.almalinux.org/alesco/meeting-minutes/2024-07-24.html).

## Motivation

* **Problem Statement:** Users of AlmaLinux expect to have access to content provided by Fedora EPEL on all architectures that AlmaLinux itself is available on.  The x86_64_v2 variant is not available in upstream CentOS or RHEL and thus not available from Fedora EPEL, which impairs the usefulness of AlmaLinux 10 on x86_64_v2 systems.
* **Goals:** The x86_64_v2 variant should be as functional and useful as any other architecture AlmaLinux 10 is offered with the availability of EPEL content.

## Detailed Design

* **Proposal Details:** Provide an in-depth explanation of the proposed changes.
    * Pull changes/updates from upstream EPEL daily and rebuild the packages.  These built-packages will go into a dedicated repository which users can enable using `dnf install epel-release` just as in current AlmaLinux 8/9.
        * No changes to EPEL packages will be permitted downstream. Any changes needed to build in AlmaLinux for alternative architectures needs to be contributed to Fedora. We will build directly from Fedora SRPMS fetched via Bodhi composes.
    * We got feedback from the EPEL steering committee whose main concern was naming confusion.  We will name the actual package providing the repo `epel-release-almalinux-altarch` and make it `Provides` and `Conflicts` with `epel-release`. This package will only be available in the extras repo for architectures not served by Fedora.
    * Packages we rebuild and publish will have `.alma_altarch` added to their disttag for easy identification.
    * These packages will use a dedicated GPG signing key - not the base AlmaLinux OS package key.
    * The repository will be an optional rsync target for mirror owners and require them to configure mirroring for it.
    * EPEL10 will have 2 production branches - one targeting EL10 stable, and one targeting CentOS Stream, or in our case, EL10 stable and Kitten.
* **Implementation Plan:** We will ask Fedora Bodhi for the latest built package in stable, grab its SRPM, and build it.  ALBS "platforms" will be used to handle the multiple stable targets.  
* **Compatibility:** This is a self-contained change proposal.  Usage of this EPEL rebuild would remain optional and be disabled by default - just as upstream EPEL is within AlmaLinux already.

## Drawbacks

* The x86 builders will have more work building packages.  After the initial build this shouldn't be of any significant concern.
* Potential confusion between AlmaLinux's rebuild and upstream EPEL.
* More disk space required on mirrors that choose to mirror our EPEL rebuild.
* Extra work on the Core SIG.

## Benefit to AlmaLinux

This helps position AlmaLinux 10 x86_64_v2 on equal footing with the main AlmaLinux, and other EL builds which default to x86_64 v3.  Potential users of the x86_64_v2 variant have expressed reluctance to use AlmaLinux without EPEL being available.  EPEL is of large value to EL users in general and keeping it available to our x86_64_v2 users is important.

This also lays the groundwork for future alternative architecture work, as EPEL is an important aspect for delivering AlmaLinux on architectures that upstream CentOS does not support.

## Scope

* **Proposal Owners:**
  * Design and guidance on implementation
  * Creation of `epel-release-almalinux-altarch` package
* **Other Developers:**
  * Deploy and enable the EPEL builds for alternative architectures (Core SIG).
* **Policies and guidelines:** N/A
* **Trademark approval:** N/A

## Acknowledgments
* Alex Iribarren
* Javier Hernandez
* Ben Thomas
