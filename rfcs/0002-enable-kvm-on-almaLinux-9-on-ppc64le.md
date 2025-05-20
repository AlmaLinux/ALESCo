# RFC: Enable KVM on AlmaLinux 9 on ppc64le

* **RFC Number:** `0002`
* **Author(s):** [Lance Albertson](lance@osuosl.org)
* **Status:** Accepted
* **Created:** [2025-02-14 17:25 UTC]
* **Updated:** [2025-02-14 17:25 UTC]

## Abstract

This RFC proposes enabling Kernel-based Virtual Machine (KVM) hypervisor support
on AlmaLinux 9 for the ppc64le architecture.

## Motivation

* **Problem Statement:**

   * Red Hat Enterprise Linux (RHEL) 9 removed support for KVM on the ppc64le
     architecture due to various technical challenges and resource allocation
     [priorities](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/considerations_in_adopting_rhel_9/index#ref_changes-to-kvm_assembly_virtualization).

* **Goals:**

   * Reintroduce KVM support on AlmaLinux 9 for ppc64le.
   * Ensure compatibility and stability for users running virtual machines on IBM Power Systems.
   * Support OpenStack cloud users who need KVM for their cloud infrastructure

## Detailed Design

* **Proposal Details:**

   * Kernel and QEMU Updates

      * Create AlmaLinux Kernel with KVM hypervisor support enabled for ppc64le
      * Update QEMU and other related packages to include the necessary patches
        and configurations for KVM support on ppc65le.
   * Testing and Validation

      * Conduct extensive testing to ensure stability and performance including
        running various workloads and tests on bare-metal IBM Power Systems.
      * Conduct extensive testing with OpenStack and it's underlying components
        to ensure stability and compatibility
   * Documentation

      * Provide comprehensive documentation to guide users on setting up and
        using KVM on ppc64le with AlmaLinux 9.

* **Implementation Plan:**

   * Package Updates: Update the kernel and QEMU packages in the AlmaLinux
     repositories.
   * Testing: Perform rigorous testing on different IBM Power Systems
     configurations with OpenStack integration
   * Release: Roll out the updated packages, repository and documentation to the
     AlmaLinux community.

* **Compatibility:**

   * Ensure that the updated packages and repositories do not interfere with
     existing AlmaLinux packages and repositories.

## Drawbacks

* Additional maintenance and support required for the ppc64le architecture.

## Benefit to AlmaLinux

* Community Support: Address the needs of users who rely on KVM for virtualization on IBM Power Systems.
* Marketing Opportunities

   * Position AlmaLinux as a versatile and robust alternative to RHEL, attracting more users and contributors.

## Scope

* **Proposal Owners:**

   * Design and guidance on implementation
   * Patch updates for kernel and QEMU packages
   * Testing on bare-metal systems
   * Ongoing maintenance and issues that may come up
* **Other Developers:**

   * Assistance with ensuring changes made follow AlmaLinux developers guidelines
* **Policies and guidelines:** N/A
* **Trademark approval:** N/A

## Unresolved Questions

* My initial testing showed some issues using qemu-kvm by default doesn't work and requires adding additional flags to
  work properly. I will need to do more testing and research to figure out why this is the case.

## Acknowledgments

* Jonathan Wright
