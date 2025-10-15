# **RFC: Provide Secure Boot-Signed Alternative Kernels**

* **RFC Number:** 0007  
* **Author(s):** [James Reilly](https://github.com/hanthor)  
* **Status:** Draft  
* **Created:** [2025-10-16 04:00 UTC]  
* **Updated:** [2025-10-16 04:00 UTC]

## **Abstract**

This RFC proposes integrating and officially signing alternative kernels maintained by the CentOS Hyperscale SIG and kmods SIG with the AlmaLinux Secure Boot key, making them available as optional packages in AlmaLinux Kitten.

## **Motivation**

**Problem Statement:** Users with modern or specialized hardware (such as newer AMD APUs like Strix Halo) often require kernel versions newer than the base Enterprise Linux (EL) stream kernel for proper functionality, stability, and performance. While kernels from CentOS Special Interest Groups (SIGs) exist, they cannot be signed with the CentOS shim key. This forces users to choose between:

1. Sticking to the older, incompatible EL kernel.  
   2. Using the newer kernel but disabling Secure Boot.  
   3. Switching to a different distribution like Fedora.

Goals:

* Import and rebuild the kernel packages from the CentOS Hyperscale SIG and kmods SIG and sign these kernels using the official AlmaLinux Secure Boot key.  
* Strengthen AlmaLinux's value proposition and reputation by supporting modern hardware while maintaining Enterprise stability.

## **Design**

* Proposal Details:   
  We will leverage the existing kernel configurations from the CentOS SIGs and integrate them into the build and signing infrastructure.

The specific repositories targeted are:

1. **CentOS Hyperscale SIG Kernel:** https://gitlab.com/CentOS/Hyperscale/rpms/kernel  
2. **CentOS kmods SIG Kernel:** https://gitlab.com/CentOS/kmods/rpms/kernel

These kernels will be packaged with a distinct name (e.g., kernel-hyperscale-almalinux) to avoid conflicts with the base EL kernel and will be added to the AlmaLinux Kitten repositories as optional installs. Users will be able to install these kernels and retain Secure Boot integrity, which is a significant advantage over using the SIGs' kernels directly.

## **Drawbacks**

### **Increased Maintenance and Testing Burden**

Concern: Every new kernel stream we officially support and sign adds overhead to the build infrastructure  
Mitigation: The Hyperscale and kmods SIGs are established, high-quality projects. We are leveraging their work rather than creating an entirely new kernel configuration, significantly reducing the testing burden compared to a completely custom kernel, we can limit the number of releases and make explicit that these are experimental kernels i.e. untested. 

### **NVIDIA Kmods***

Concern: NVIDIA drivers will need to be built for each kernel stream, this is already a burdensome task for the existing kitten and 10.0 kernel releases. 
Mitigation: Either do not support nvidia drivers on these alternative kernels or if it's not too much of a burden, add them to the build trigger.


## **Benefit to AlmaLinux**

* **Enhanced Hardware Support:** Directly addresses the needs of users with new laptops, desktops, and specialized servers requiring recent kernel support.  
* **Value-Add:** Providing the official Secure Boot signature offers a crucial security feature that CentOS SIGs cannot currently provide.

## **Scope**

**Proposal Owners:**

* AlmaLinux Core Team: Update the build configuration to import SIG sources.  
* AlmaLinux Infrastructure Team: Configure/expand existing automated kernel build/rebuild/signing pipeline.  
* Others:  
* Review and test the signed kernels for stability before wide release  
* Update documentation regarding the installation and support scope for these alternative kernels.

## **Unresolved Questions**

* Should both hyperscale and kmods SIG kernels be synced over?  
* What is the desired frequency for importing and rebuilding new SIG releases (e.g., daily, weekly, per-release)?

## **Acknowledgments**

Thanks to the CentOS Hyperscale SIG and kmods SIG contributors for their excellent work in maintaining these alternative kernels.
