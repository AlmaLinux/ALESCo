# RFC: Use dynamic workspace indicator on GNOME

* **RFC Number:** `0003`
* **Author(s):** [Elkhan Mammadli](elkhan@almalinux.org),
* **Status:** Accepted
* **Created:** [2025-04-02 14:00 UTC]
* **Updated:** [2025-05-30 01:00 UTC]

## Abstract

Do not replace the dynamic workspace indicator with AlmaLinux OS logo as a part of branding.

## Motivation

* **Problem Statement:** [GNOME 45](https://release.gnome.org/45) introduced a [dynamic workspace indicator](https://release.gnome.org/45/activities-loop.webm) that replaced the "Activities" button. This indicator shows the number of workspaces and enables switching between them via mouse scrolling. Although the GNOME version on AlmaLinux OS 10 is 47, the dynamic workspace indicator is visually replaced by the AlmaLinux OS logo. Which means switching between the workspaces still possible via mouse scrolling on the AlmaLinux logo.
* **Goals:** For a complete experience, the dynamic workspace indicator should be consistent with GNOME's implementation.

## Detailed Design

* **Proposal Details:** Do branding without replacing the dynamic workspace indicator.
* **Implementation Plan:**
    - Implement a new patch for `gnome-shell` package to include AlmaLinux OS logo on the left side of the dynamic workspace indicator.
    - Automate the replacement of the branding upstream patch with the proposed one.
* **Compatibility:** Since AlmaLinux OS 10 not released yet. This change only affects the AlmaLinux OS Kitten 10.

## Drawbacks

N/A

## Benefit to AlmaLinux

This will improve the overall GUI experience for users familiar with GNOME from other distributions, including CentOS Stream 10, Fedora, and Ubuntu.

## Scope

* **Proposal Owners:**
    * Implementation and testing of the proposed patch.
* **Other Developers:**
    * Core SIG
        * Automation of building and releasing of `gnome-shell` package.
* **Policies and guidelines:** N/A
* **Trademark approval:** N/A

## Unresolved Questions

N/A

## Acknowledgments

* [Neal Gompa](ngompa@almalinux.org)
* [Javier Hernández](javi@almalinux.org)
