
## What this covers

Run triggers (connecting workspaces so one's successful apply automatically queues a run in another) and variable sets (reusable variable bundles applied across workspaces and Stacks). Projects, the third topic under this subsection heading, is identical content to what's already documented in the 8b recap, cross-reference there rather than duplicating it here.

## Run triggers

A run trigger connects a **downstream** (dependent) workspace to one or more **upstream** (source) workspaces, so that a successful apply in a source workspace automatically queues a new run in the downstream one. Configuration lives entirely on the downstream side, the source workspace needs no changes at all, it just applies normally and HCP Terraform notices the subscription.

**Why this exists**: designed for workspaces that depend on values or infrastructure produced elsewhere, typically read via a data source. Without a run trigger, a change in the upstream workspace could silently leave the downstream workspace stale, since nothing would prompt anyone to re-run it.

```
networking-prod applies successfully
        │
        ▼ (run trigger on app1-prod, source = networking-prod)
app1-prod automatically queues a new plan
```

**Key facts:**

- Up to **20 source workspaces** per downstream workspace.
- **Permissions**: creating a run trigger requires admin access to the downstream workspace, plus permission to read runs on the source workspace being connected. Deleting one only requires admin on the downstream workspace.
- **Triggered runs queue a plan, not an apply.** Whether that plan proceeds to apply depends on the downstream workspace's normal apply settings. Even with the workspace's regular auto-apply on, a trigger-queued run still won't auto-apply unless the separate **"Auto-apply run triggers"** setting is explicitly enabled, this setting is independent from the primary workspace auto-apply toggle.
- **Run details carry context**: a trigger-queued run in the downstream workspace links back to the source workspace and the specific successful run that triggered it. The source workspace's own plan/apply output similarly notes which workspaces it triggered a run in.

### Reading another workspace's outputs

`terraform_remote_state` is the traditional data source for reading a source workspace's root-level outputs. The source workspace must be explicitly configured to allow access before this works.

**`tfe_outputs`** (from the HCP Terraform/Enterprise provider) is the **recommended** alternative now, since it doesn't require full access to the source workspace's state just to read its outputs, a meaningfully tighter security surface than `terraform_remote_state`, which needs broader state access.

Run triggers and one of these data sources typically pair together: the data source is _how_ you read the upstream dependency's current value, the run trigger is _what_ ensures you re-read it promptly after it changes.

## Variables and variable sets

### Workspace-specific variables

Set per-workspace via the UI, the Variables API, or the `tfe` provider's `tfe_variable` resource (convenient for bulk management). Two categories: **Terraform** (maps to a `variable` block) and **environment** (a shell env var for the run).

**Permissions**: "Read variables" to view a workspace's variables and org's variable sets; "Read and write variables" to create/edit workspace-specific ones.

**Not available for Local execution mode workspaces at all.** HCP Terraform doesn't evaluate workspace variables or variable sets when execution happens outside its own infrastructure, so the Variables page itself doesn't apply there.

### Run-specific variables (CLI-level)

Terraform 1.1+ lets you set variable values for one specific plan/apply via `-var`/`-var-file` on the command line, or via `TF_VAR_<name>` environment variables. These overwrite workspace-specific and variable-set values sharing the same key, sitting near the top of the normal precedence chain, same behavior as local Terraform's variable precedence from section 4c.

### auto.tfvars files

Files ending `.auto.tfvars` (or `terraform.tfvars`) can supply variable values for workspaces on Terraform 0.10.0+. Loaded automatically on every run. A workspace-specific variable with the same key overwrites the file's value. HCP Terraform does **not** persist these file-sourced variables to the workspace's own Variables section, they load fresh each run but never show up as workspace variables in the UI.

### Variable sets

A reusable bundle of variables applied across multiple workspaces and Stacks at once, defined once, updated once, propagates everywhere it's attached.

**Ownership**: organization-owned or project-owned, determining which permissions are needed to manage it (owners team / "Manage all projects" / "Manage all workspaces" for org-owned; project write/maintain/admin/"Manage variable sets" for project-owned).

**Scope options**:

- Org-owned, **global**: auto-applies to every current and future workspace in the org.
- Org-owned, **scoped**: applied to specifically selected projects/workspaces/Stacks.
- Project-owned, **entire project**: applies to all current and future workspaces/Stacks in that project.
- Project-owned, **scoped**: applied to specifically selected workspaces/Stacks within the project.

**Conflict rule**: HCP Terraform errors outright if you try to declare the same variable key across multiple _global_ variable sets, since there'd be no way to resolve which one wins.

**Also unevaluated for Local execution mode**, same restriction as workspace-specific variables.

### Overwriting and priority

A **workspace-specific variable overwrites a variable-set value with the same key** by default, this is a deliberate, visible override (flagged with a yellow OVERWRITTEN indicator in the UI, clicking it shows which value actually wins). Variable sets can also overwrite each other's shared keys within the same workspace, even though each set is defined at the org/project level, the overwrite resolution happens per-workspace.

**Priority variable sets** are the one real exception to normal precedence: a priority set's values override _everything_ sharing that key at more specific scopes, including workspace-specific variables and even CLI-supplied `-var`/`.tfvars` values, which normally sit at the very top of the precedence chain. This is a genuine override of the standard order, worth remembering as the single case where a CLI value can lose.

### Variable sets and Stacks, a fundamentally different reference mechanism

Workspaces get variable sets _applied to them implicitly_ from outside, with the whole precedence/overwrite system resolving conflicts. Stacks work the opposite way: **explicit, named reference from inside the deployment configuration**, via a `store` block in `<name>.tfdeploy.hcl`.

```hcl
store "varset" "secrets" {
  name     = "Database passwords"
  category = "terraform"
}

deployment "production" {
  inputs = {
    database_password = store.varset.secrets.db_password
    instance_count     = 5
  }
}
```

The Stack must have access to the variable set (via project access or the set being globally available), then references specific keys with `store.varset.<varset-name>.<variable-name>` syntax directly in deployment inputs.

**Consequence: variable precedence doesn't apply to Stacks at all.** Since the reference is explicit and unambiguous by name/ID, there's no multi-source conflict to resolve, a Stack always uses exactly the value from the set it named, regardless of any org-level variable or other set sharing that key. Values update live too, changes to the variable set are reflected in future deployments automatically. If the referenced set is later removed from the Stack's org/project, the Stack fails to deploy, no fallback.

### Security

All variable values are encrypted via Vault's transit backend before storage. **Variable descriptions are stored in plaintext**, never put sensitive info there.

**Marking a variable sensitive makes it write-only, permanently.** Nobody, including whoever originally set it, can ever view its value again through the UI or the Variables API. New values can be set (overwriting the old one), but existing attributes otherwise can't be edited in place, to change anything besides the value, delete and recreate the variable.

Recommended: pass credentials as environment variables rather than Terraform variables where possible, since Terraform variable values flow through the run in full text and can end up in logs, state, or Sentinel policy mocks if the config sends them to an output or resource parameter. Environment variables aren't stored in state, but can still leak into log files if `TF_LOG=TRACE`.

**Character limits**: description 512 chars, key 128 chars, value 256 KB. HCL syntax is supported for Terraform-category variables (not environment variables) to enter lists/maps, toggled via the HCL checkbox.

**Dynamic credentials** are an alternative to static variable-based credentials: temporary, per-run credentials that eliminate manual secret rotation entirely.

## Common exam traps in this section

|Trap|Why it's tricky|
|---|---|
|Run trigger configuration location|Lives on the downstream workspace, not the source, easy to assume it's the other way around|
|Trigger queues plan, not apply|Requires the separate "Auto-apply run triggers" setting to actually apply automatically|
|`terraform_remote_state` vs `tfe_outputs`|`tfe_outputs` is now recommended specifically for its narrower state-access requirement|
|Priority variable sets|The one case that beats CLI-supplied `-var` values, a real exception to standard precedence|
|Stacks and variable precedence|Doesn't apply at all, explicit named reference replaces the whole resolution system|
|Sensitive variables are truly write-only|Not just hidden from casual UI browsing, unrecoverable even via API, even for the creator|
|Local execution mode and variables|Workspace variables and variable sets are simply not evaluated at all in Local mode|

## Quick reference

| Concept                                 | Key fact                                                                 |
| --------------------------------------- | ------------------------------------------------------------------------ |
| Run trigger max sources                 | 20 per workspace                                                         |
| Run trigger auto-apply                  | Separate setting, off by default, independent of workspace auto-apply    |
| Recommended cross-workspace output read | `tfe_outputs` over `terraform_remote_state`                              |
| Priority variable set                   | Overrides even CLI-supplied values                                       |
| Stack variable set reference            | `store` block, `store.varset.<name>.<key>` syntax                        |
| Sensitive variable                      | Write-only forever; delete and recreate to change anything but its value |
| Local execution mode                    | No variable or variable-set evaluation at all                            |