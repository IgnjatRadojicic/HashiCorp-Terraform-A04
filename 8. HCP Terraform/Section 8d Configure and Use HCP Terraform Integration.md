
## What this covers

Connecting the Terraform CLI to HCP Terraform via the `cloud` block, authenticating with `terraform login`, and the three ways to migrate existing state into HCP Terraform or Terraform Enterprise, manual CLI, manual API, or the (now deprecated, but still exam-relevant) `tf-migrate` tool.

## Connecting the CLI to HCP Terraform

Four-step process: provide credentials, define connection settings, initialize, optionally migrate state.

### Credentials

`terraform login` is the recommended path, an interactive command that launches a browser to obtain an API token. **Only works interactively**, since it needs a browser session, unattended automation (CI/CD) must configure credentials manually in the CLI configuration file instead.

```bash
terraform login          # defaults to app.terraform.io
terraform login app.terraform.io/eu   # HCP Europe
```

By default the token is stored **in plaintext** in `credentials.tfrc.json`. Terraform explicitly discloses the save location before writing, giving you a chance to cancel. A **credentials helper program** can be configured to delegate storage to an external secrets system instead of the default plaintext file.

### The cloud block

```hcl
terraform {
  cloud {
    organization = "my-org"
    hostname     = "app.terraform.io"  # optional, this is the default

    workspaces {
      project = "networking-development"
      tags = {
        layer  = "networking"
        source = "cli"
      }
    }
  }
}
```

**Arguments**: `organization` (required), `hostname` (optional, defaults to `app.terraform.io`), and within `workspaces`, either `name` (one specific existing workspace) or `tags` (matches/creates workspaces by tag, map or legacy key-only list style), **mutually exclusive with each other**. `workspaces.project` additionally scopes tag/name matching to workspaces within that project.

If no existing workspace matches the configured tags, `terraform init` prompts to create one with those tags applied.

### Initialize

`terraform init` after adding or changing the `cloud` block. By default, `plan`/`apply` uploads a copy of the working directory's configuration to HCP Terraform each time, controllable via `.terraformignore` (see below).

## Migrating from backend "remote" to cloud

The `cloud` block is the modern replacement for the older `backend "remote"` block (Terraform 1.1+; use `remote` for 1.0 and older). Migration is close to a direct swap, with one real capability gap:

```hcl
terraform {
-  backend "remote" {
+  cloud {
     organization = "my-org"
     workspaces {
-      prefix = "my-app-"
+      tags = {
+        app = "mine"
+      }
     }
   }
 }
```

**`prefix` has no equivalent in the `cloud` block.** The old `remote` backend let one configuration represent multiple workspaces sharing a name prefix, letting you `terraform workspace select prod` using just the short suffix. `tags` replaces the multi-workspace-matching use case, but **after migrating, you must reference workspaces by their full name**, `terraform workspace select my-app-prod` instead of the old short `prod`. A genuine behavior change, not just a syntax rename.

## Migrating state, three methods

### 1. Manually via CLI

Add the `cloud` block, run `terraform init`. If the directory has existing state under a prior backend, Terraform prompts to migrate it into the new HCP Terraform workspace(s) automatically.

**A key naming friction point**: a local CLI workspace represents one of several _environments sharing a single configuration_ (`prod`, `staging`, `dev`). An HCP Terraform workspace must be a fully independent, org-wide-uniquely-named unit. Migrating `prod`/`staging`/`dev` from one config can't just carry those short names over, so Terraform prompts for renaming, typically to a `<COMPONENT>-<ENVIRONMENT>-<REGION>` pattern (`networking-prod-us-east`, `networking-staging-us-east`).

### 2. Manually via the API

For scripted or bulk migration outside the CLI flow:

1. Base64-encode the state file, generate an MD5 hash of it.
2. Create the destination workspace via `POST /organizations/:organization_name/workspaces`, if it doesn't already exist.
3. Lock the workspace: `POST /workspaces/:workspace_id/actions/lock`.
4. Upload state: `POST /workspaces/:workspace_id/state-versions`, with the MD5 hash in `data.attributes.md5` and the base64-encoded state in `data.attributes.state`.
5. Unlock: `POST /workspaces/:workspace_id/actions/unlock`.

Requires an API token with appropriate permissions. This lock-upload-unlock pattern mirrors the state locking mechanics from section 6, migration is itself treated as a state-modifying operation requiring the same protection.

### 3. Automatically via tf-migrate (deprecated, but historically the "bulk" tool)

A **separate binary**, not bundled with HCP Terraform or the Terraform CLI, downloaded independently. Note: recent HashiCorp documentation marks this tool as deprecated and no longer supported, worth knowing current status even though it may still appear in exam material written while it was current.

**Two-stage workflow, state migration:**

- **`tf-migrate prepare`**: recursively scans the current directory for state files, generates migration configuration into a `_hcp-migrate-configs` directory (kept out of version control via auto-added `.gitignore` entry), and interactively prompts for the target organization, whether to create a git branch, and whether to open a pull request. Requires local backend config changes committed to source control first. Reuses the same API token `terraform login` obtained by default.
    - **Workspace naming**: one workspace per state file, `<DIRECTORY NAME>-<LOCAL WORKSPACE NAME>`.
    - **Project naming**: one project per directory path (`./frontend/networking/terraform.tfstate` → project `frontend_networking`), path must be 3-40 characters, letters/numbers/spaces/hyphens/underscores only.
    - Carries over variables from existing Terraform configuration automatically.
    - Creates a **VCS-driven** workspace if the local repo has a GitHub/GitLab/Bitbucket remote and a matching org OAuth connection exists; otherwise falls back to **CLI-driven**.
- **`tf-migrate execute`**: implements the staged changes for real, performing the actual migration.

**Separate sub-workflow, workspace-to-Stack migration** (currently beta):

1. **`tf-migrate modules create`**: wraps root-module resources into a child module (required structurally for Stack components); doesn't touch state.
2. **`tf-migrate stacks prepare`**: generates Stack configuration from the modularized code.
3. **`tf-migrate stacks execute`**: runs `init`/`plan`/`apply` against the generated Stack configuration, creating the project and Stack, migrating existing state into it.

**Requirements for the Stack migration path**: tf-migrate 2.0+, Terraform 1.13+, existing configuration already deployed to one or more HCP Terraform workspaces, Stacks enabled on the org.

### Safety rule across all three methods

**Only migrate state into a workspace that has never performed a run.** This mirrors the lineage/serial safety checks on `terraform state push` from section 6, migrating into a workspace with its own existing run history risks a state-history collision.

## .terraformignore

Controls what gets uploaded to HCP Terraform during a CLI-driven `plan`/`apply`. Syntax mirrors `.gitignore`: `#` comments, blank lines ignored, trailing `/` for directory-only patterns, leading `!` to negate a pattern (place negations early, or avoid them, for performance on large directories).

**Default exclusions apply unconditionally, with no `.terraformignore` file required at all:**

- `.git/`
- `.terraform/`, **except `.terraform/modules`**, which is still uploaded since cached module source needs to travel with the configuration for the remote run to actually build it correctly.

A `.terraformignore` file only adds _further_ exclusions beyond these two automatic defaults, it isn't what switches them on in the first place.

## Common exam traps in this section

| Trap                                            | Why it's tricky                                                                                               |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `cloud` block dropping `prefix`                 | Real capability loss vs `backend "remote"`, full workspace names required afterward                           |
| CLI workspace vs HCP workspace naming collision | Root cause of the rename prompt during migration, ties back to the 8a distinction                             |
| `terraform login` token storage                 | Plaintext by default, explicitly disclosed, not hidden or encrypted unless a credentials helper is configured |
| `terraform login` and automation                | Interactive-only; CI/CD needs manual credential configuration instead                                         |
| `.terraform/modules` upload exception           | The one subdirectory of `.terraform/` still uploaded despite the general default exclusion                    |
| `workspaces.name` vs `workspaces.tags`          | Mutually exclusive in the `cloud` block                                                                       |
| Migration destination workspace state           | Must have zero prior runs, to avoid a state history collision                                                 |
| `tf-migrate`'s two-command pattern              | `prepare` stages and generates config/PR, `execute` performs the actual migration, two distinct steps         |

## Quick reference

| Task                                  | Mechanism                                                                                      |
| ------------------------------------- | ---------------------------------------------------------------------------------------------- |
| Connect CLI to HCP Terraform          | `cloud` block in `terraform` block + `terraform init`                                          |
| Authenticate interactively            | `terraform login` (plaintext token by default)                                                 |
| Authenticate for automation           | Manual credentials in CLI config file                                                          |
| Migrate one workspace's state via CLI | `cloud` block + `terraform init`, follow prompts                                               |
| Migrate state via API                 | base64 + MD5 → lock → POST state-versions → unlock                                             |
| Bulk/automatic migration              | `tf-migrate prepare` then `tf-migrate execute` (deprecated tool)                               |
| Workspace → Stack migration           | `tf-migrate modules create` → `stacks prepare` → `stacks execute`                              |
| Exclude files from CLI upload         | `.terraformignore`, `.git/` and `.terraform/` (minus `modules`) excluded by default regardless |