# RFC: Declarative GitHub Organization Management with Peribolos

* **RFC Number:** `0008`
* **Author(s):** [benny Vasquez](benny@almalinux.org)
* **Status:** Draft
* **Created:** 2026-04-16 
* **Updated:** 2026-04-16 

## Abstract

Adopt declarative, version-controlled management of the AlmaLinux GitHub organization using [peribolos](https://docs.prow.k8s.io/docs/components/cli-tools/peribolos/). All org membership, team structure, and repository access would be defined in YAML files and applied automatically through pull requests, replacing manual changes made through the GitHub UI.

## Motivation

* **Problem Statement:** Today, changes to the AlmaLinux GitHub organization — adding or removing members, creating teams, granting repository access — are made manually through the GitHub web UI. This creates several problems:
  - **No audit trail beyond GitHub's own logs.** There is no easy way to review who was added to which team, when, or why. GitHub's audit log is available but not easily searchable or reviewable by the broader community.
  - **No review process.** Membership and access changes happen immediately when an admin clicks a button. There is no opportunity for a second set of eyes before changes take effect.
  - **Drift from any potentially documented state.** The members of the teams doesn't actually exist in a public state, and the wiki is often woefully out of date. This gives us a source of truth for who has access and what access is defined for each user.
  - **Limited auditability** With the access control all hidden behind a github login, there's a limited number of people who can perform an access audit. Right now,there are 53 people with direct access across 79 repos. Likely all of those would be better served by being in a team so we can more easily manage access long term.

* **Goals:**
  - Make all org membership and team structure changes go through pull requests with review
  - Provide a clear, version-controlled record of the org's current and historical state
  - Enable dry-run previews so reviewers can see exactly what a change will do before it's merged
  - Reduce the risk of accidental or unintentional misconfiguration (wrong permission level, etc)
  - create teams as a first step toward automating alerts and setting owners for each of our repos
  - Ultimately, to align GitHub team membership with the documented SIG structure on the wiki, so it's clear every time who should have access and who shouldn't

## Detailed Design

* **Proposal Details:**

  A new repository (`AlmaLinux/org-management`) would contain YAML files that declaratively describe the GitHub organization:

  - **`org.yaml`** — Organization-level settings (name, description, default permissions) plus the full admin and member rosters
  - **`teams/*.yaml`** — One file per team (SIG, working group, or functional team), defining the team's members, maintainers, and repository access

  A GitHub Actions workflow uses peribolos to sync this YAML state to the actual GitHub org:
  - On **pull requests**: runs a dry run (`--confirm=false`) so reviewers can see what would change
  - On **merge to main**: applies the changes (`--confirm=true`) to the live org

  Authentication uses a dedicated GitHub App with scoped permissions (org member read/write, org admin read/write), generating short-lived tokens per run. No long-lived personal access tokens are needed.

* **Implementation Plan:**

  1. **Seed the initial config** — Export current org state and cross-reference with wiki-documented SIG membership to produce the initial YAML files
  2. **Create the repository** — Set up `AlmaLinux/org-management` with the YAML files, merge script, and GitHub Actions workflow
  3. **Create and install the GitHub App** — Register a GitHub App with the required permissions and install it on the AlmaLinux org
  4. **Validate with dry runs** — Run peribolos in dry-run mode to confirm the YAML matches current org state before enabling auto-apply
  5. **Enable auto-apply** — Once validated, enable the merge-to-main workflow to apply changes automatically
  6. **Document the process** — Update documentation to direct people to the new PR-based workflow for org changes, and add automation on the wiki that any SIG page update should also include a note that they it might need to confirm the members are correct
  7. **Access Audit** - Add or remove teams as needed,  using the new teams, based on a comprehensive review of all of our current repos, and set a month 

* **Compatibility:**

  This is a change to process, not to any AlmaLinux packages or deliverables. Existing org members, teams, and repository access remain unchanged — the initial config is seeded from the current state. The only difference is that future changes go through PRs instead of the GitHub UI.

  Org admins retain the ability to make emergency changes through the UI if needed. The next peribolos run will detect any drift and can either reconcile it or flag it for review.

## Drawbacks

- **New dependency.** Peribolos is a Go binary from the Kubernetes Prow project. If the upstream project stops maintaining it, we would need to either fork it or find an alternative. The good news is that peribolos is actively maintained and widely used across the CNCF ecosystem.
- **Learning curve.** Contributors who want to request org changes need to learn the YAML format and submit a PR rather than asking an admin directly. The YAML format is straightforward, and the README documents common tasks with copy-paste examples.
- **GitHub App setup.** Requires creating and maintaining a GitHub App with org admin permissions. This is a one-time setup, but the App approach is more secure than a personal access token because tokens are short-lived, and tied to a single person's account.
- **Emergency changes are slightly slower.** If an admin needs to revoke access urgently, they can still do so through the UI, but the org-management repo would be temporarily out of sync until a PR reconciles the state.
- **Opens potential attack vector.** With the full teams and admins public, we open ourselves to a potential social engineering or phishing attack. 

## Benefit to AlmaLinux

- **Transparency.** Anyone can see the full org structure by reading the YAML files. Anyone can propose changes through a PR. This aligns with AlmaLinux's commitment to open governance.
- **Safety.** Built-in guardrails prevent accidental damage: a minimum of 3 admins is enforced, specific required admins cannot be removed, and no more than 25% of members can be removed in a single run.
- **Accountability.** Every change has a PR with a description, reviewer approval, and a permanent git history. The dry-run output shows exactly what will happen before it's applied.
- **Proven to work already.** This is the same approach used by the Kubernetes project ([kubernetes/org](https://github.com/kubernetes/org)) and others in the CNCF ecosystem to manage organizations with thousands of members. We would be adapting it for AlmaLinux's smaller scale.

## Scope

* **Proposal Owners:** benny Vasquez — initial setup, seeding config from current org state, GitHub App creation (already 80-90% done)
* **Other Developers:** Org admins review and merge PRs; SIG leads submit PRs for their team membership changes
* **Policies and guidelines:** A lightweight review policy should be established (e.g., team membership changes require approval from a team maintainer or org admin)
* **Trademark approval:** N/A

## Unresolved Questions

- **Review policy:** Who should be required to approve different types of changes? I'd propose: Team maintainers (SIG leaders, etc) must voice approval on PRs for their team, but are ultimately merged by one of the org admins.
- **Reconciliation cadence:** If someone makes an emergency change through the UI, should we have a scheduled workflow that detects drift, or is it sufficient to rely on the next PR-triggered run? I think the solution is a daily scheduled job that detects team drift from this repo along with a monthly task that detects individuals being added to repos, and both create issues on the org-managemetn repo and alerts org admins and SIG leaders.

## Acknowledgments

- The [Kubernetes project](https://github.com/kubernetes/org) for creating this approach and maintaining peribolos as open source tooling.
