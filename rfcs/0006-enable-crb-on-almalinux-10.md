# RFC: Enable CRB by default for AlmaLinux 10+

* **RFC Number:** `0006`
* **Author(s):** [Neal Gompa](ngompa@almalinux.org)
* **Status:** Accepted
* **Created:** [2025-05-13 13:45 UTC]
* **Updated:** [2025-05-29 23:00 UTC]

## Abstract

Enable the CRB repository by default on AlmaLinux 10 onward.

## Motivation

* **Problem Statement:** The full content set is not available by default, which is a problem when people enable EPEL, as it requires the CRB repository. This results in issues as people complain when they enable EPEL that packages are broken when they are not.
* **Goals:** Reduce user confusion and support burden with using common third-party repositories that leverage CRB content at runtime.

## Detailed Design

* **Proposal Details:** The CRB repository will be changed from disabled-by-default to enabled-by-default.
* **Implementation Plan:** Update the `almalinux-crb.repo` file in `almalinux-release` to enable the CRB repo.
* **Compatibility:** No negative impact, it's purely additive.

## Drawbacks

One more repository is loaded by default, which slightly slows down metadata download on constrained systems.

## Benefit to AlmaLinux

This ensures the complete content set is offered and avoids issues with third party repositories that depend on CRB content.
Notably, Fedora Extra Packages for Enterprise Linux (EPEL) requires CRB and users regularly forget to turn it on and have dependency issues.
With EPEL 10, `epel-release` will grow a conditional dependency on `selinux-policy-extra` that is [shipped in CRB](https://gitlab.com/redhat/centos-stream/rpms/selinux-policy/-/merge_requests/199),
and if the repository is not enabled when `epel-release` is installed, the dependency will never be evaluated due to how DNF works for conditional weak dependencies.
Enabling CRB by default makes things smoother and easier for users, and we do not have to burden ourselves with dealing with users trying to figure out inscrutable
errors in the default configuration.

## Scope

* **Proposal Owners:** Pull request to `almalinux-release` to enable CRB on a10/a10s
* **Other Developers:** N/A
* **Policies and guidelines:** N/A
* **Trademark approval:** N/A

## Unresolved Questions

N/A.

## Acknowledgments

N/A.
