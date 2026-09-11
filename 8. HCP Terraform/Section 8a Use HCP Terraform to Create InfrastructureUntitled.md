## What this covers

HCP Terraform's core execution model: what a workspace is and how it differs from a local working directory (and from a CLI workspace), how remote runs actually execute, the run queue and locking mechanics, the three ways to start a run, and the subscription tiers that gate which features are available.

## Workspaces

A workspace is a group of infrastructure resources managed together, HCP Terraform's equivalent of a local working directory. Where local Terraform organizes infrastructure into meaningful groups via separate directories on disk, HCP Terraform uses workspaces to do the same job, each one self-contained with its own configuration, variables, state, and credentials.

| Component           | Local Terraform                      | HCP Terraform                                  |
| ------------------- | ------------------------------------ | ---------------------------------------------- |
| Configuration       | On disk                              | Linked VCS repo, or uploaded via API/CLI       |
| Variable values     | `.tfvars` files, CLI args, shell env | Stored in the workspace                        |
| State               | On disk or remote backend            | Stored in the workspace                        |
| Credentials/secrets | Shell env or prompted                | Stored in the workspace as sensitive variables |

HCP Terraform additionally retains, per workspace, that local Terraform has no equivalent for: **state versions** (backups of every previous state file, useful for history and recovery) and **run history** (every run's summary, logs, triggering change, and comments).

### HCP Terraform workspaces vs Terraform CLI workspaces

Two genuinely different concepts sharing a name, a real exam trap.

| Aspect            | HCP Terraform workspace                                                          | CLI workspace                                                                 |
| ----------------- | -------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| Required?         | Yes, at least one is mandatory to manage anything                                | No, entirely optional                                                         |
| Purpose           | Full unit of infrastructure management, config + state + variables + credentials | Isolates multiple state files within one working directory, one configuration |
| Role-based access | Yes, permissions are granted per HCP Terraform workspace                         | No access control concept                                                     |

### Organizing workspaces

Recommended pattern: break large monolithic configurations into smaller ones, each with its own workspace, and delegate permissions per workspace rather than managing one giant configuration as a single unit. Example: split a production environment's code into networking, application, and monitoring configurations, then create `networking-prod`, `app1-prod`, `monitoring-prod` workspaces, assigning separate teams to each. This mirrors microservices thinking, parallel team work, and makes reusing the same configuration for other environments (`app1-dev`) straightforward.

**Projects** group workspaces (and Stacks) together, letting you scope access to a collection rather than workspace-by-workspace or organization-wide, useful for aligning permissions with business units or team responsibilities.

### Workspace health

Available on Standard and Premium: automatic health assessments checking whether real infrastructure still matches configuration (**drift detection**) and whether custom conditions (your `validation`/`precondition`/`postcondition`/`check` blocks from section 4g) continue passing after provisioning (**continuous validation**). Can be enforced org-wide or left opt-in per workspace.

## Remote operations

By default, HCP Terraform runs Terraform on its own disposable virtual machines rather than your workstation, called **remote operations**. This consistency is what unlocks features that need a controlled execution environment: Sentinel/OPA policy enforcement, cost estimation, notifications, and VCS integration.

**Remote runs can be triggered four ways**: VCS webhook, HCP Terraform UI controls, API calls, or Terraform CLI. When triggered via CLI, the run still executes remotely, but its output streams to your terminal in real time, giving a local-feeling experience even though execution happens elsewhere.

**Disabling remote operations** (per-workspace Execution Mode setting, switched to Local) turns the workspace into a pure remote state backend, all actual execution happens on your own machine or CI workers instead. This trades away every feature that depends on remote execution, policy enforcement, cost estimation, and notifications included.

**HCP Terraform agents** (a paid feature) let HCP Terraform reach isolated, private, or on-prem infrastructure without opening public ingress, an agent polls HCP Terraform for changes and executes them locally within that private network. Agents also support **hooks**, custom programs run at defined points in a run's lifecycle, for example downloading software the run needs, or kicking off an external workflow via HTTP request.

### The exception: terraform import never runs remotely

Regardless of Execution Mode or CLI integration, `terraform import` always executes locally. The workspace acts purely as a remote state backend for this one command. Consequence: workspace-stored environment variables aren't available during an import, since those only get injected into HCP Terraform's remote execution environment, which import never uses. Any credentials the provider needs for the import must be supplied in your local shell. HashiCorp recommends `import` blocks (1.5+) over the CLI command for this reason among others, since blocks execute as part of a normal (remote-capable) apply.

## Run queue and workspace locking

Each workspace processes exactly one run at a time, in a queue. A new run added while one is in progress goes into `pending`, Terraform deliberately won't even start planning it yet, since the in-progress run might change what the pending one should do.

**Workspace locks** are a related but separate mechanism: an in-progress run automatically locks the workspace for its duration, and a user or team can also deliberately lock a workspace for maintenance, independent of any run happening.

**Two run types bypass both the queue and locks**, and only these two: plan-only (speculative) runs, and the planning stage specifically of saved-plan runs. Both can proceed immediately because neither touches real infrastructure or committed state at that stage. The moment a saved plan moves to apply, it re-enters normal queue and lock behavior like any other run. Destroy runs do **not** get this exception, since they do modify real infrastructure.

**A run locks to a specific configuration version and variable set the moment it starts.** Changes pushed or made afterward only affect future runs, never the one already in flight.

## Starting runs, three workflows

- **UI/VCS-driven** (primary mode): clicking "New run" in the UI, or a VCS webhook firing on push/PR.
- **API-driven**: calling the Runs API directly, more flexible, requires custom tooling.
- **CLI-driven**: running standard `terraform plan`/`apply` with CLI integration configured, execution happens remotely, output streams locally.

### Plans and applies

HCP Terraform always plans before applying, enforcing the same plan/apply separation as local Terraform. Default behavior waits for user approval before applying (**manual apply**); workspaces can instead be configured for **auto-apply**, applying successful plans automatically. Some plans can never auto-apply regardless of setting: those queued by run triggers, or started by a user without apply permission for that workspace.

If a plan shows no changes, HCP Terraform skips the apply step entirely, ending with status "Planned and finished" (overridable via the allow-empty-apply run mode).

### Speculative plans

Plan-only runs used to preview changes during editing/review, they show proposed changes and policy impact but **can never be applied under any circumstance**. Three ways they get created: automatically on a VCS pull request (posted as a PR check), via `terraform plan` locally with CLI integration configured, or via the Runs API when a configuration version is marked speculative. A failed speculative plan can be retried (requires plan-queuing permission), generating a new run against the same configuration version.

### Saved plans

Requires **Terraform CLI 1.6.0+** specifically. With CLI integration configured: `terraform plan -out <FILE>` performs and saves a plan, `terraform apply <FILE>` applies it later, `terraform show <FILE>` inspects it first. Saved plan runs affect the queue differently from normal runs (their planning stage can jump ahead, as covered above) and can sometimes be automatically discarded if conditions change before they're applied.

## Subscription plans and billing

### The four tiers

| Feature                                                                   | Free      | Essentials | Standard  | Premium   |
| ------------------------------------------------------------------------- | --------- | ---------- | --------- | --------- |
| Managed resources                                                         | Up to 500 | Unlimited  | Unlimited | Unlimited |
| Remote execution, VCS integration, private registry, team mgmt, run tasks | Yes       | Yes        | Yes       | Yes       |
| Concurrent runs (remote / agent)                                          | 1 / 1     | 3 / 3      | 10 / 10   | 30 / 30   |
| Policy enforcement, Cost estimation, SSO (SAML)                           | No        | No         | Yes       | Yes       |
| Audit logging, Custom RBAC, Priority support                              | No        | No         | No        | Yes       |

**Memorize the ladder shape, not a flat list**: Free and Essentials share the same core feature set, differing mainly in resource limits and billing model. Standard adds the governance-and-compliance layer (policy enforcement, cost estimation, SSO) as one bundle. Premium adds enterprise-grade controls (audit logging, custom RBAC, priority support) as the next bundle on top. Concurrency limits scale independently across all four tiers regardless of feature gating.

### Billing models

- **Pay-as-you-go**: monthly charge based on actual managed-resource-hours, flexible, scales with usage.
- **HashiCorp Flex**: committed annual usage with volume discounts, for predictable infrastructure needs.
- **Contract plans**: custom enterprise agreements negotiated directly with HashiCorp Sales.

### Managed resources, what counts and what doesn't

A managed resource is anything in a state file where `mode = "managed"`, counted starting from its first `plan` or `apply`. Workspace and Stacks resources combine into one total for billing purposes.

**Counts:** resources from any provider, resources multiplied by `count`/`for_each` (each instance counts separately), resources from modules and no-code-ready modules.

**Does not count:** `null_resource`, `terraform_data`, data sources (`mode = "data"`), local values, variables.

## Common exam traps in this section

| Trap                            | Why it's tricky                                                                                                    |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| HCP vs CLI workspaces           | Same name, genuinely different concepts, one required and permission-scoped, one optional and state-isolation-only |
| `terraform import` always local | The one command that never uses remote execution, even with CLI integration on                                     |
| Run queue/lock exceptions       | Only speculative runs and saved-plan *planning* qualify, not destroy or refresh-only                               |
| Policy enforcement tier         | Unlocks at Standard, not Premium, easy to lump in with "advanced enterprise" features                              |
| `count`/`for_each` and billing  | Each generated instance counts individually toward managed resources                                               |
| Saved plans CLI version         | 1.6.0 specifically, not 1.5.0 (which is when `import` blocks landed, a different feature)                          |

## Quick reference

| Concept                                  | Key fact                                                                           |
| ---------------------------------------- | ---------------------------------------------------------------------------------- |
| HCP Terraform workspace                  | Required, contains config + state + vars + credentials, unit of RBAC               |
| CLI workspace                            | Optional, isolates state files within one config/directory                         |
| Remote operations                        | Default; runs on HCP Terraform's disposable VMs, unlocks policy/cost/notifications |
| `terraform import`                       | Always local execution, even with remote/CLI integration configured                |
| Queue/lock exceptions                    | Speculative (plan-only) runs; planning stage of saved-plan runs                    |
| Saved plans min version                  | Terraform CLI 1.6.0+                                                               |
| Policy enforcement, cost estimation, SSO | Standard tier and up                                                               |
| Audit logging, custom RBAC               | Premium only                                                                       |
