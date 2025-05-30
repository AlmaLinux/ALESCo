# AlmaLinux ALESCo RFCs (Request for Comments)

Welcome to the AlmaLinux Engineering Steering Committee's (ALESCo) RFC repository. This repository is the central place for proposing, discussing, and documenting significant changes to AlmaLinux.

## Table of Contents

- [Introduction](#introduction)
- [When to Submit an RFC](#when-to-submit-an-rfc)
- [RFC Process Overview](#rfc-process-overview)
- [How to Submit an RFC](#how-to-submit-an-rfc)
- [RFC Lifecycle](#rfc-lifecycle)
- [RFC Template](#rfc-template)
- [Contributing Guidelines](#contributing-guidelines)
  - [Code of Conduct](#code-of-conduct)
- [License](#license)
- [Contact](#contact)
- [Additional Notes](#additional-notes)
- [Frequently Asked Questions](#frequently-asked-questions)
  - [How long is the review period?](#how-long-is-the-review-period)
  - [Who decides whether an RFC is accepted?](#who-decides-whether-an-rfc-is-accepted)
  - [Can I withdraw my RFC?](#can-i-withdraw-my-rfc)

## Introduction

The AlmaLinux RFC process is a formal way to propose significant changes, enhancements, or new features to the AlmaLinux project. It aims to foster open collaboration, transparency, and comprehensive documentation of decisions made by ALESCo and the broader community.

## When to Submit an RFC

Consider submitting an RFC if you are proposing:

- Major changes to the operating system or its components.
    - EX, Changing the default desktop environment
- New features or functionalities.
    - EX, Adding SPICE support
- Any change that requires community consensus or impacts a large portion of the project.
    - EX, Replacing the use of OpenSSL with LibreSSL

For minor issues like bug fixes or small enhancements, please use the appropriate [AlmaLinux issue tracker](https://bugs.almalinux.org/) or contribute directly to the relevant repository.

## RFC Process Overview

1. **Proposal Drafting**: Author drafts the RFC using the provided template.
2. **Submission**: RFC is submitted as a pull request to this repository.
3. **Accepted for Discussion**: Community and ALESCo members provide feedback, with active revision of the proposal expected. 
	As part of this step, ALESCo will also send an email to the AlmaLinux OS Foundation board with a link to the RFC pull request, indicating whether or not the board input is expected to be a blocker for the vote.
5. **Final Comment Period (FCP)**: A last call for comments before the decision. The board will also get a notification of an impending vote here, in case they would like to provide input.
6. **Decision Making**: ALESCo votes on the RFC.
7. **Implementation**: Accepted RFCs are merged and scheduled for implementation.

## AlmaLinux OS Foundation Board input as a blocker

In very rare circumstances the AlmaLinux OS Foundation Board will be required to provide input in things ALESCo is deciding on. In some cases the board may be required to approve a change, and in some cases they can abstain from an official position. However, as part of their oversight in the project, input will be required before a vote can be held in the following situations.

* Input from the board will be a blocking stakeholder before ALESCo can vote on RFCs that...
   - involve a change to trademarks 
   - involve a significant increase in needed resources
   - involve a new or expanded partnership of some kind

To ensure the board provides input, the current board chair will be added as a required reviewer to the pull request. A representative from the board will also attend the meeting where the vote is taking place.

As a note - this is not a veto right, but is only to ensure that the board has been able to advise ALESCo on any potential impacts of changes. 

## How to Submit an RFC

1. **Fork the Repository**: Click the "Fork" button at the top-right corner of this page to create your own copy of the repository.
2. **Create a New Branch**: In your forked repository, create a new branch for your RFC:

```bash
   git checkout -b rfc-title-of-your-proposal
```

3. **Copy the Template:** Copy the `0000-template.md` to `rfcs/XXXX-title-of-your-proposal.md`, replacing `XXXX` with the next available RFC number (use `0000` for the initial PR; the number will be assigned during the process).
4. **Write the RFC:** Fill out the template with your proposal. Be thorough and clear.
5. **Submit a Pull Request:** Push your branch to your fork and submit a pull request to this repository's main branch.
6. **Engage in Discussion:** Monitor the pull request comments and participate in the discussion, making revisions as necessary.

## RFC Lifecycle

The RFC process has the following states:

0. **Submitted**: The RFC is submitted and requires an initial review for formatting, etc. Once it has been confirmed to meet the needs of the RFC process, the "accepted for discussion" tag will be added.
1. **Open for discussion**: The RFC is under initial and active discussion and may undergo several revisions.
2. **FCP (Final Comment Period)**: ALESCo announces an FCP, signaling that the decision will be made soon. When the RFC enters this stage, the "vote at next meeting" tag will be added.
3. **Accepted**: The RFC is approved and merged.
4. **Rejected**: The RFC is not approved; reasons are documented.
5. **Withdrawn**: The RFC is withdrawn by the author.
6. **Superseded**: The RFC is replaced by a new proposal.

## RFC Template

The RFC template can be found [here](/rfcs/0000-template.md). Please use this template for all new RFCs.

## Contributing Guidelines

* **Clarity:** Write clear and concise proposals.
* **Respect:** Be respectful in discussions; adhere to the Code of Conduct.
* **Responsiveness:** Be available to answer questions and provide clarifications during the review process.
* **Documentation:** Update your RFC based on feedback received.

## Code of Conduct

All participants are expected to uphold the AlmaLinux Code of Conduct.

Which amounts to: be nice, be respectful, and be nerdy.

## License

By contributing to this repository, you agree that your contributions will be licensed under the Creative Commons Attribution Share Alike 4.0 International (CC-BY-SA-4.0`).

## Contact

* **Chat:** [AlmaLinux Mattermost](https://chat.almalinux.org/~alesco)
* **Forums:** [AlmaLinux Community Forums](https://forums.almalinux.org/)
* **Mailing List:** [devel@lists.almalinux.org](https://lists.almalinux.org/mailman3/lists/devel.lists.almalinux.org/)

# Additional Notes

* **Assigning RFC Numbers:** RFC numbers are assigned by the repository maintainers when the pull request is labeled as "accepted for discussion." Use 0000 as the number in your initial submission.
* **File Naming Convention:** Name your RFC file as XXXX-title-of-your-proposal.md, where XXXX is the assigned RFC number.
* **Directory Structure:** All RFC files should be placed in the rfcs/ directory.
* **Updating RFC Status:** The status of the RFC should be updated in the document header as it progresses through the lifecycle stages.

# Frequently Asked Questions

## How long is the review period?
The standard review period is two weeks, but it can be extended if necessary to ensure adequate discussion.

## Who decides whether an RFC is accepted?
The final decision is made by ALESCo, taking into account community feedback and the overall impact on the project. There is no obligation for the RFC to be accepted by the committee.

## Who is responsible for implementing an RFC once it's accepted?

It is expected that contributors will be responsible for all or part of the work implementing the RFC.

## Can I withdraw my RFC?
Yes, you can withdraw your RFC at any point before it is accepted by closing the pull request and stating your intention.
