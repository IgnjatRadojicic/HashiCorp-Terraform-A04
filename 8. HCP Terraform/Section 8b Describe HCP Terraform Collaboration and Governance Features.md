

## What this covers

The features that support teams working together safely at scale: the Explorer for org-wide visibility, policy enforcement, health assessments (drift detection and continuous validation), the private registry, change requests, projects, and teams.

## Explorer for workspace visibility

Org-wide search and reporting tool, accessed via Explorer in the top-level side navigation. Surfaces data across four resource types: **Workspaces, Modules, Providers, Terraform versions**.

**Permissions required**: organization owner, or "View all workspaces" or greater.

**Workflow**: click a resource type or a pre-built use case card, results render in a sortable table, clicking a hyperlinked field drills into a more specific view (a workspace's module count links to that workspace's associated modules).

**Notable pre-built use cases**: top module/provider/Terraform versions by usage, workspaces without VCS, workspaces with failed continuous validation checks, drifted workspaces, latest-updated and oldest-applied workspaces, runs by status.

**Custom filter conditions**: build queries manually with target field + operator + value rows. Multiple conditions combine with logical AND. No OR logic available.

**Saved views**: store the _query definition_, not a snapshot of results. Revisiting a saved view always re-runs the query fresh, HCP Terraform does not cache or persist query results or history. "Save as" creates a new view from a modified existing one, without overwriting the original.

## Policy enforcement

Three frameworks now, not two:

| Framework               | Supports Stacks?    | Notes                                                                                |
| ----------------------- | ------------------- | ------------------------------------------------------------------------------------ |
| Terraform policy (beta) | Yes, and workspaces | Native HCL-based; requires Terraform v1.16alpha+, opt-in via org pre-release setting |
| Sentinel                | Workspaces only     | HashiCorp's own policy-as-code framework                                             |
| OPA                     | Workspaces only     | Open Policy Agent, uses the Rego language                                            |

**Key structural facts**: a policy set can only contain policies from a single framework, but multiple policy sets (potentially using different frameworks) can apply to the same workspace or Stack simultaneously. Policies can be scoped globally or to specific projects, workspaces, Stacks, or deployments. Evaluated against the Terraform plan during each run.

**Recommended workflow**: store policy configuration in VCS for policy-as-code auditability, rather than authoring directly in the UI (which is possible but not recommended). Pre-written policies exist for common standards like PCI DSS.

**Enforcement outcome**: depending on enforcement level, a failed policy can stop the run outright. Users with the right permission can override a failed policy.

## Health assessments

Two evaluation types under one umbrella feature, available on Standard and Premium:

- **Drift detection**: does real infrastructure still match your Terraform configuration.
- **Continuous validation**: do custom conditions (`validation`, `precondition`, `postcondition`, `check` blocks) still pass after provisioning.

### Configuration drift vs state drift

A distinction the docs draw a hard line around. **Configuration drift** is when an external change invalidates your configuration, the classic "someone edited it in the console" case, this is what drift detection catches. **State drift** is when an external change happens but doesn't invalidate the configuration, this is a different problem, remediated with refresh-only mode (section 6), not by drift detection. Drift detection does not substitute for `-refresh-only`.

### Requirements and permissions

| Requirement                             | Value                                                                                                       |
| --------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| Drift detection alone                   | Terraform 0.15.4+                                                                                           |
| Drift detection + continuous validation | Terraform 1.3.0+                                                                                            |
| Execution mode                          | Remote or Agent (not Local)                                                                                 |
| Latest run status                       | Must be successful; errored/canceled/discarded pauses assessments until a successful run occurs             |
| Prior apply                             | At least one successful apply must exist; no assessments run against workspaces with no real infrastructure |

Permissions: read access to view health status, organization owner to change org-level health settings, workspace admin to change workspace-level settings or trigger on-demand assessments.

### Scheduling

Enabling health assessments at the **organization level overrides** workspace-level settings, workspace-level enablement only applies when the org isn't enforcing it globally. First assessment timing depends on active runs at enable time (next period if idle, after completion if a speculative plan is active). Subsequent assessments are always scheduled a fixed interval out from whichever is more recent, the last apply or the last assessment.

**A new run always cancels an in-progress assessment**, rescheduling it for the next period rather than letting it interfere. This can effectively starve assessments in workspaces with very frequent runs. **On-demand assessments** (UI only) reset the automatic schedule, the next automatic assessment waits for the next normal period rather than stacking immediately after.

Health assessments don't count against your concurrency limit, though HCP Terraform batches them internally at scale to avoid overwhelming large organizations.

### Continuous validation's false-positive trap

Health assessments generate a **speculative plan** to check current state. Any `check` block referencing a data source gets that data source re-queried fresh as part of that speculative plan. If the real-world value the data source returns has changed since your last actual apply (a newer AMI became available, say), the check can fail even though your actually-deployed infrastructure hasn't changed at all, since HCP Terraform never modifies infrastructure during an assessment, only evaluates against the freshly-queried value. **Mitigation**: use a nested data source that queries your specific deployed resource's actual current configuration, rather than one that returns "the latest available X" generically.

```hcl
check "ami_version_check" {
  data "aws_instance" "hashiapp_current" {
    instance_tags = { Name = "hashiapp" }
  }
  assert {
    condition     = aws_instance.hashiapp.ami == data.hcp_packer_artifact.hashiapp_image.external_identifier
    error_message = "Must use the latest available AMI."
  }
}
```

### Viewing results

Drift results live under a workspace's Health > Drift page, showing drifted resource counts and proposed remediation. Continuous validation results live under Health > Continuous validation, showing pass/fail per named object. If a resource has multiple pre/postconditions, individual results are hidden unless they fail, an all-pass object just reports "passed" as a whole.

### Resolving drift

Two options once drift is detected: **overwrite** (queue a normal apply, reverting real infrastructure to match configuration) or **update configuration** (modify `.tf` files to adopt the drifted value, preventing it from being reverted on the next apply). Same decision framework as the refresh-only tutorial from earlier.

## Private registry

Functions like the public Terraform Registry, but scoped to your organization, with versioning and search support.

**Public providers/modules**: HCP Terraform can auto-synchronize the public registry's content into your private registry, letting you centrally designate which public artifacts are org-recommended.

**Private providers/modules**: hosted only in your org's registry, visible only to org members (Terraform Enterprise additionally allows sharing across configured organizations).

**Governance hook**: Sentinel policies can constrain private registry usage org-wide, for example requiring all non-root modules to come from the private registry specifically, or requiring recent module versions. This is the direct link between policy enforcement and the private registry, policy is the enforcement mechanism, the registry is what's being governed.

## Change requests

Standard/Premium only. A lightweight, **manual** backlog mechanism, not an automated gate. Administrators create change requests directly from Explorer query results (found a workspace using a deprecated module version, say), attaching a descriptive message. The workspace tracks the request; whoever resolves it archives it. Team notifications can alert a workspace's owning team when a new change request lands on it.

**Distinguish clearly from policy enforcement**: policies automatically evaluate and can block runs based on plan content. Change requests never block anything automatically, they're a to-do list item a human created and a human resolves.

## Projects

Organize workspaces and Stacks into groups with their own permission scope, more granular than org-level permissions, less granular than per-resource grants. Managing project permissions requires Essentials/Standard/Premium (projects themselves are usable on all tiers, permission management is the gated part).

**Every workspace and Stack belongs to exactly one project, always.** New resources default to the org's **Default Project** (renamable, never deletable) unless a different project is specified at creation.

**Permission ladder for workspace creation**:

- "Manage Workspaces" → can create/manage workspaces, but only into the Default Project; cannot see other projects' metadata
- "Manage Projects & Workspaces" or admin role on a specific project → required to create workspaces into projects other than Default
- "Manage all Projects" → can view/edit/delete/assign access for all of an org's projects

### Execution mode, four options

- **Organization Default**: inherits the org's setting (itself Remote or Local)
- **Remote**: plan/apply run on HCP Terraform/Enterprise infrastructure
- **Local**: plan/apply run on machines you control, HCP Terraform only stores/syncs state
- **Agent**: plan/apply managed by HCP Terraform but executed via an agent

Settable at org, project, or workspace level, with workspace overriding project overriding org default. **Stacks do not support Local execution mode at all**, a hard exception, only workspaces can opt into it.

## Teams

Groups of users within an organization; belonging to at least one team makes someone a member of that org. Team management available on Essentials, Standard, Premium (not Free's basic tier structure, though Free still has the owners team).

Teams are granted workspace/project/Stack/org permissions, letting members start runs, manage variables, read/write state, and more. A team's permissions only apply within its own organization, though an individual user can belong to multiple teams across multiple orgs.

**Team API tokens**: not tied to a specific user, managed from Organization settings, useful for automation that shouldn't depend on any one person's credentials.

### The owners team

Every org has exactly one, automatically created with the org's creator as first member. **Free organizations cap it at 5 members; paid organizations have no cap.** Cannot be deleted, and cannot be left empty, if it has exactly one member, you must add another before that member can be removed.

## Common exam traps in this section

|Trap|Why it's tricky|
|---|---|
|Configuration drift vs state drift|Drift detection only catches the former; the latter needs `-refresh-only`, not drift detection|
|Which policy frameworks support Stacks|Only Terraform policy does; Sentinel and OPA are workspace-only|
|Two-tier health assessment version requirement|0.15.4+ for drift alone, 1.3.0+ additionally for continuous validation|
|Continuous validation false positives|Speculative plans re-query data sources fresh, can flag changes that haven't actually happened to real infrastructure yet|
|Change requests vs policy enforcement|Change requests never block runs automatically; policies do|
|Default Project inescapability|Every workspace/Stack has exactly one project, always, no exceptions|
|"Manage Workspaces" permission scope|Lets someone create workspaces, but only into Default Project, not "cannot create at all"|
|Stacks and Local execution mode|Explicitly unsupported, workspaces-only capability|
|Owners team floor|Can never be emptied, even with only one member remaining|
|Explorer saved views|Store the query, not a results snapshot, always re-run live|

## Quick reference

|Feature|Minimum tier|
|---|---|
|Explorer|Requires org owner or "View all workspaces"+ permission|
|Health assessments (drift + continuous validation)|Standard, Premium|
|Change requests|Standard, Premium|
|Policy enforcement (Sentinel/OPA/Terraform policy)|Standard, Premium|
|Projects (usable)|All tiers|
|Project permission management|Essentials, Standard, Premium|
|Team management|Essentials, Standard, Premium|

|Concept|Key fact|
|---|---|
|Drift detection min version|Terraform 0.15.4+|
|Continuous validation min version|Terraform 1.3.0+|
|Stacks-compatible policy frameworks|Terraform policy only|
|Execution modes|Organization Default, Remote, Local, Agent (Stacks: no Local)|
|Owners team size cap|5 on Free, unlimited on paid|