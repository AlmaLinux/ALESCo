# RFC: Use dynamic workspace indicator on GNOME

* **RFC Number:** `0002`
* **Author(s):** [Elkhan Mammadli](elkhan@almalinux.org)
* **Status:** Draft
* **Created:** [2025-04-02 14:00 UTC]
* **Updated:** [2025-04-02 14:00 UTC]

## Abstract

Do not replace the dynamic workspace indicator with AlmaLinux OS logo.

## Motivation

* **Problem Statement:** [GNOME 45](https://release.gnome.org/45) introduced a [dynamic workspace indicator](https://release.gnome.org/45/activities-loop.webm) that replaced the "Activities" button. This indicator shows the number of workspaces and enables switching between them via mouse scrolling. Although the GNOME version on AlmaLinux OS 10 is 47, the dynamic workspace indicator is visually replaced by the AlmaLinux OS logo. Which means switching between the workspaces still possible via mouse scrolling on the AlmaLinux logo.
* **Goals:** For a complete experience, the dynamic workspace indicator should be consistent with GNOME's implementation.

## Detailed Design

* **Proposal Details:** Remove the branding patch from the [gnome-shell](https://git.almalinux.org/rpms/gnome-shell) package.
* **Implementation Plan:**
    - Remove `0001-panel-Use-branding-in-activities-button.patch` from the spec file of `gnome-shell` package.
    - Automate the removal of the patch each time package sources are synchronized.
* **Compatibility:** Since AlmaLinux OS 10 not released yet. This change only affects the AlmaLinux OS Kitten 10.

## Drawbacks

* N/A

## Benefit to AlmaLinux

This will improve the overall GUI experience for users familiar with GNOME from other distributions, including CentOS Stream 10, Fedora, and Ubuntu.

## Scope

* **Proposal Owners:**
* **Other Developers:**
* **Policies and guidelines:** N/A
* **Trademark approval:** N/A

## Unresolved Questions

Identify any open issues or areas needing further investigation.

## Acknowledgments

Credit individuals who contributed to the RFC.
