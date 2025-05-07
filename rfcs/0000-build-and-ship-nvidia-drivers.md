# RFC: Build NVIDIA Open Source Drivers and Ship Repo Config for AlmaLinux

* **RFC Number:** `0000`
* **Author(s):** [Jonathan Wright](jonathan@almalinux.org), [Neal Gompa](ngompa@almalinux.org)
* **Status:** Draft
* **Created:** [2025-05-07 15:00 UTC]
* **Updated:** [2025-05-07 15:00 UTC]

## Abstract

Build and ship NVIDIA's open source graphics drivers.  The kernel module should be signed for secure boot using a separate secure boot key under the AlmaLinux secure boot CA.  Ship a repo config to pull cuda and userspace components from Nvidia's own repositories.

## Motivation

* **Problem Statement:** Graphics drivers have always been a pain point for users in Linux and with GPUs now being used for more than just graphics, the pain is more real than ever.  Currently users must search around the web to find the drivers they want, unsure if the components they find will work seamlessly together or not.  We can and should solve this problem.
* **Goals:** The modules/drivers and repo configs shipped should permit users to have a relatively seamless experience with NVIDIA GPUs within AlmaLinux, with only a few `dnf` commands required to get up and running with fully supported drivers.

## Detailed Design

* **Proposal Details:** Provide an in-depth explanation of the proposed changes.
    * Build Nvidia's open source driver as a kmod. The legacy proprietary driver is out of scope for this.
      * It will be signed for secure boot using a dedicated secondary certificate that will be trusted by AlmaLinux kernel builds.
    * Ship a repo configuration file to pull user space and CUDA drivers from NVIDIA's dedicated repository.  This is required due to the licenses from NVIDIA.
      * Initially, this will leverage the negativo17.org NVIDIA userspace driver repository and migrate to the official NVIDIA driver repository later.
    * This RFC plans support specifically for AlmaLinux 10 initially (including Kitten), with AlmaLinux 9 to follow.  Support for AlmaLinux 8 is unlikely to happen without immense demand to justify the amount of backporting required to make it feasible.
    * Initially only the latest supported branch of the NVIDIA driver will be shipped and supported, though the implementation leaves room for multiple version trees to exist simultaneously.
* **Implementation Plan:** The binary RPMs built will exist in their own, optionally enabled, repository.  The release package to enable the repository will exist in the Extras repository (which is enabled by default), so that a single command could turn on everything required to then require only one more command to install the driver and associated userspace utilities.
* **Compatibility:** This is a self-contained change proposal.  Usage of the NVIDIA driver repository would remain optional.

## Drawbacks

* None known at this time - The use of these packages will remain optional.

## Benefit to AlmaLinux

AlmaLinux will become more accessible to users of NVIDIA GPUs - from small home users to HPC users alike, and everyone in between.  The ability to run the GPU drivers and use secure boot will increase security in environments, and may allow further adoption of AlmaLinux in environments that it cannot currently be deployed due to regulatory restrictions around secure boot being disabled.

## Scope

* **Proposal Owners:**
  * Design and guidance on implementation
  * Creation of `nvidia-open-kmod` package
  * Creation of `almalinux-nvidia-driver-release` package
* **Other Developers:**
  * Key holders
    * Creation of NVIDIA driver secure boot certificates
  * Core SIG members
    * Adding public certificate to AlmaLinux kernel builds
    * Build environment setup for building `nvidia-open-kmod` with signed kernel modules
* **Policies and guidelines:** N/A
* **Trademark approval:** N/A

## Acknowledgments
