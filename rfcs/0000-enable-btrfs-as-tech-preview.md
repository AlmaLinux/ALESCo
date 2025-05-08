# RFC: Enable Btrfs as a tech preview

* **RFC Number:** `XXXX` (Assigned by the repository maintainers)
* **Author(s):** [Michel Lind](michel@meta.com)
* **Status:** Draft
* **Created:** [YYYY-MM-DD hh:mm UTC]
* **Updated:** [YYYY-MM-DD hh:mm UTC]

## Abstract

Btrfs is enabled as a tech preview for AlmaLinux 10

## Motivation

* **Problem Statement:** CentOS Stream and Enterprise Linux distributions based on it, including AlmaLinux, currently lack Btrfs support - it is disabled in the kernel, and installer support and a lot of userspace tooling are missing. This makes the barrier of entry for using these distributions higher in organizations that have standardized on Btrfs.

* **Goals:**
  * Ship Btrfs as a kernel module
  * Offer Btrfs as an option in the installer
  * Ship the utilities needed to manage Btrfs

## Detailed Design

* **Proposal Details:** 
The kernel enablement should be self-explanatory.

For the installer, we can reference what Fedora did when enabling Btrfs (minus setting it to be the default)

  * [Btrfs by default](https://fedoraproject.org/wiki/Changes/BtrfsByDefault)
  * [Btrfs transparent compression](https://fedoraproject.org/wiki/Changes/BtrfsTransparentCompression)
  * [Fedora Cloud Btrfs by default](https://fedoraproject.org/wiki/Changes/FedoraCloudBtrfsByDefault)
  * [/var on subvolume for immutable variants](https://fedoraproject.org/wiki/Changes/VarSubvol4SilverblueKinoite) - relevant to AlmaLinux derivatives that want to ship an immutable distro using Btrfs

Utilities - apart from `btrfs-progs` which got added to EPEL because of a Golang dependency, EPEL can't host `btrfs` utilities because they will not be functional out of the box. And for compatibility reason AlmaLinux does not ship EPEL enabled by default anyway. So these will likely need to be branched and built for AlmaLinux. Looking at what CentOS Hyperscale SIG ships that are Btrfs related might be relevant here.

* **Implementation Plan:**
  * update kernel config to enable Btrfs in AlmaLinux Kitten
  * undo installer changes in Anaconda and its dependencies that disables Btrfs support. Can leverage the CentOS Hyperscale SIG packaging to avoid duplicating work.
  * undo disabled btrfs in various userspace packages
  * import userspace utilities that are needed by Btrfs users
  
* **Compatibility:** 
This should not break any compatibility for those not using Btrfs.

If it gets to the point where the Stream kernel sources can no longer support Btrfs (see below), a separate kernel can be used for those that need Btrfs support, rather than risking compatibility for non-Btrfs users

## Drawbacks

### Kernel diverging from upstream
Concern: Right now the EL10 kernel has not deviated much from upstream - later on in the lifecycle it might get more and more backports of separate subsystems, and keeping a feature enabled that Red Hat does not test for in the Stream kernel might get harder

Mitigation: When that happens AlmaLinux might need to ship the Hyperscale kernel (which closely tracks the Fedora configuration) rebuilt for AL. Wearing my Hyperscale hat we can work with the Alma kernel team to make sure the kernel modules that Alma needs are enabled in our kernel config

## Benefit to AlmaLinux

Combined with other recent features enabled in AlmaLinux (frame pointers, x86-64-v2 rebuild), this will strengthen AlmaLinux's reputation as an Enterprise Linux distribution where it is possible to innovate. It will also help organizations that use Btrfs more easily onboard Alma.

## Scope

* **Proposal Owners:**

  * Send PRs to enable Btrfs, e.g. for the following packages
    * anaconda
    * bcc
    * buildah
    * cockpit
    * ignition
    * kernel
    * libblockdev
    * libguestfs
    * osbuild
    * osbuild-composer
    * podman
    * pykickstart
    * python-blivet
    * skopeo
    * udisks2
    * virt-v2v

  * Request that Core SIG adds needed packages be pulled into the distribution e.g.
    * btrfs-progs
    * compsize
    
* **Other Developers:**
  * review and merge PRs
  * import needed new packages, likely from Fedora
  
* **Policies and guidelines:** N/A
* **Trademark approval:** N/A

## Unresolved Questions

* Are all the dependencies to Btrfs enabled already in the EL kernel?
  * If not, is the extra deviation and implications of that acceptable?


## Acknowledgments



