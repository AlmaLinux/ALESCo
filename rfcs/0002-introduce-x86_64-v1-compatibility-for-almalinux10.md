# RFC: Introduce x86_64-v1 compatibility for AlmaLinux 10

* **RFC Number:** `0002`
* **Author(s):** [Bohdan Khomutskyi](bogdan@khomutsky.com)
* **Status:** Draft
* **Created:** [2025-03-21 16:50 UTC]
* **Updated:** [2025-03-21 16:50 UTC]

## Abstract

Build another version of AlmaLinux to support x86_64-v1.

## Motivation

* **Problem Statement:** no option to run enterprise linux on the old hardware. 
* **Goals:** provide users of old hardware a choice to run enterprise linux. This is useful for those, who want free
security updates for the OS running on the old hardware.

## Detailed Design

* **Proposal Details:** Introduce compatibility with x86_64-v1 in AlmaLinux 10.
* **Implementation Plan:** Do another rebuild, similar to x86_64-v2.
* **Compatibility:** This change will increase compatibility with old systems, specifically old Core 2 Duo, which are
still good for light desktop use.

## Drawbacks

Increased use of storage, CPU time and network bandwidth for the rebuild.

## Benefit to AlmaLinux

Allowing users to transition from Debian to Alma. Covering a larger share of potential users. Giving owners of retro
hardware to run the latest and greatest Linux.

## Scope

* **Proposal Owners:** Bohdan Khomutskyi
* **Other Developers:** None at the moment.
* **Policies and guidelines:** N/A
* **Trademark approval:** N/A

## Unresolved Questions

None: the AlmaLinux team has experience rebuilding for x86-64-v2. The same approach can be applied for v1.

## Acknowledgments

None at this moment.
