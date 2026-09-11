
## What this covers

6a is about Terraform's default backend, the local one, and how the `backend` block works in general. 6b is about state locking, the mechanism that prevents concurrent writes from corrupting state, which applies across all backends that support it, not just local.

## The backend block, general mechanics

A backend controls where Terraform stores its state data. It's configured as a nested block inside the top-level `terraform` block.

```hcl
terraform {
  backend "local" {
    path = "relative/path/to/terraform.tfstate"
  }
}
```

Hard limitations on backend blocks, worth knowing cold:

- Only one `backend` block is allowed per configuration.
- A backend block cannot reference variables, locals, or data source attributes. It must be fully literal. This is because the backend has to be resolved before Terraform can evaluate anything else in the configuration, there's a chicken-and-egg problem if the backend itself depended on values that require state or provider setup to compute.
- You also can't reference values declared inside a backend block anywhere else in your configuration.

If your configuration connects to HCP Terraform or Terraform Enterprise workspaces via a `cloud` block, you cannot also have a `backend` block, the two are mutually exclusive since HCP Terraform manages state automatically for you.

## The local backend, in depth

The default backend when no `backend` block is configured at all. Stores state as a JSON file on disk and locks it using the operating system's own file-locking APIs.


```hcl
terraform {
  backend "local" {
    path = "relative/path/to/terraform.tfstate"
  }
}
```

Configuration options:

- `path` (optional): where the state file lives. Defaults to `terraform.tfstate` relative to the root module.
- `workspace_dir` (optional): where non-default workspace state files are stored.

You can also read local backend state from a `terraform_remote_state` data source, useful if a separate configuration wants to read outputs from a local-backend-managed configuration:


```hcl
data "terraform_remote_state" "foo" {
  backend = "local"
  config = {
    path = "${path.module}/../../terraform.tfstate"
  }
}
```

Legacy command-line flags exist for local state (`-state`, `-state-out`, `-backup`) from before remote state existed, predating multiple workspaces. HashiCorp explicitly recommends against using these in new work, they conflict with workspace-based filename selection and are only preserved for backward compatibility with old wrapper-script workflows.

## Backend credentials

For backends needing access credentials (most remote backends), the explicit recommendation is: don't hardcode credentials in the backend block. Use environment variables or the credentials mechanism conventional for that target system instead.

The reason is concrete, not just best practice: Terraform writes the backend configuration in plaintext into `.terraform/terraform.tfstate` (the local backend-config tracking file, distinct from your actual `terraform.tfstate` data) and into any saved plan file. If you use `-backend-config` with literal credential values or hardcode them, those values end up in plaintext in both places. Since applying a saved plan later uses the backend configuration captured _at plan time_, time-limited credentials embedded that way can also simply expire before the apply runs.

## Partial configuration

You don't have to specify every backend argument in the configuration file itself. Omitted arguments must then be supplied at `terraform init` time, via one of three methods:


```bash
# via a file
terraform init -backend-config="./state.config"

# via inline key/value pairs
terraform init -backend-config="address=demo.consul.io" -backend-config="path=example_app/terraform_state"

# interactively, Terraform prompts for missing required values
terraform init
```

At minimum, an empty backend block declaring just the type is required in the configuration itself:

```hcl
terraform {
  backend "consul" {}
}
```

If multiple sources supply overlapping settings, command-line options override the main configuration, and later command-line options override earlier ones. The final merged result lands in `.terraform/`, which should never be committed to version control since it can contain credentials in plaintext.

## Changing or removing a backend

Change a backend's type or its configuration at any time, edit the block and re-run `terraform init`. Terraform detects the change automatically and asks whether to migrate existing state to the new configuration, including copying all workspaces if you have more than one. Even reconfiguring the _same_ backend type triggers this migration prompt, you can decline it if nothing about the actual state storage changed.

Removing a backend block entirely and reinitializing prompts Terraform to migrate state back to the default `local` backend.

**Before migrating to any new backend, back up your state manually** (`terraform state pull > backup.tfstate` or a plain file copy) regardless of how confident you are in the migration.

## State locking

Terraform locks state for any operation that could write to it, whenever the backend in use supports locking. Not every backend does, check each backend's individual documentation.

Key mechanics:

- **Fully automatic.** No command triggers it, no confirmation message appears unless acquiring the lock takes longer than expected, in which case Terraform prints a status message.
- **Blocks other writers.** If someone else holds the lock, your operation waits or fails, preventing two concurrent applies from corrupting the same state.
- **Can be disabled per-command** with `-lock=false`, but this defeats the entire purpose and isn't recommended.

### Force unlock

If automatic unlocking fails (a crashed process, a network drop mid-operation), the lock can get stuck. `terraform force-unlock` handles this, but requires a specific lock ID that Terraform prints when the failure occurs.

bash

```bash
terraform force-unlock <LOCK_ID>
```

This ID requirement exists as a safety mechanism, not a formality: it acts like a nonce, ensuring the unlock targets the exact lock that failed rather than blindly clearing whatever lock happens to exist. **Never force-unlock a lock someone else is actively using**, this should only be used to clear your own stuck lock.

## Common exam traps in this section

| Trap                                               | Why it's tricky                                                                                                |
| -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| Backend block referencing variables                | People assume normal HCL expression rules apply everywhere; the backend block is a real exception              |
| Non-local backend and local disk                   | Assuming state is always written locally as a cache; remote backends hold it in memory except on write failure |
| State locking as something you invoke              | It's fully automatic on any state-writing operation, not a separate step                                       |
| force-unlock needing a lock ID                     | Assuming any authorized user can just force-unlock freely                                                      |
| Migrating even when reconfiguring the same backend | Terraform still prompts for migration confirmation, even with no real storage change                           |

## Quick reference

| Concept                       | Key fact                                                                                           |
| ----------------------------- | -------------------------------------------------------------------------------------------------- |
| `backend` block limit         | Exactly one per configuration, no variable/local/data references allowed                           |
| Default backend               | `local`, file on disk, OS-level locking                                                            |
| Non-local backend disk writes | Never, except as a fallback after an unrecoverable write error                                     |
| Partial configuration         | Missing args supplied via `-backend-config` file, key/value pairs, or interactive prompt at `init` |
| State locking                 | Automatic on state-writing ops; disable with `-lock=false` (not recommended)                       |
| Stuck lock recovery           | `terraform force-unlock <LOCK_ID>`, ID required as a safety check                                  |