
## What this covers

6c is about configuring remote state storage via the backend block (building on 6a's general mechanics, applied to non-local backends). 6d is about handling two related but distinct problems: resource drift (real infrastructure diverging from state) and refactoring (deliberately reorganizing configuration/state without destroying anything).

## Configuring a remote backend

Same `backend` block syntax as local, different backend type and arguments.

```hcl
terraform {
  backend "remote" {
    organization = "example_corp"
    workspaces {
      name = "my-app-prod"
    }
  }
}
```

All the general backend rules from 6a apply here too: one block only, no variable references, credentials via environment variables rather than hardcoded, `terraform init` required after any change.

## State storage and locking, backend-agnostic behavior

Regardless of which backend you use, all standard Terraform commands (`terraform console`, `terraform state` subcommands, etc.) behave identically whether state is local or remote, the backend is transparent to your day-to-day workflow.

### Manual state pull and push

```bash
terraform state pull > backup.tfstate   # read remote state to stdout, save it
terraform state push backup.tfstate     # overwrite remote state, dangerous
```

`state push` is explicitly flagged as extremely dangerous. Terraform runs two safety checks before allowing it:

- **Lineage match.** Lineage is a unique ID assigned when a state is first created. If the destination's lineage doesn't match, it likely means you're pushing an entirely different state's history, not a legitimate update.
- **Serial comparison.** Every state has a monotonically increasing serial number. If the destination state has a higher serial than what you're pushing, that means changes have happened since your local copy was taken, and pushing would silently discard them.

Both checks can be bypassed with `-force`, but even HashiCorp's own guidance is to pull a backup first regardless.

## Refactoring state, planning phase

Before splitting or reorganizing state files, first identify why you're doing it and how to group resources.

### Common refactor triggers

- Long `apply` times from a monolithic configuration
- Resources with different management lifecycles (frequently-changing vs static) that benefit from separate blast radii
- Ownership boundaries splitting along team lines
- A repeated pattern in your configuration that would make a good reusable module

### Grouping considerations

- **Volatility.** Separate frequently-changing resources (compute scaling) from stable ones (networking), so routine changes to one don't risk accidental changes to the other.
- **Stateful vs stateless.** Keep databases and other stateful resources isolated from stateless compute, limiting the blast radius of any re-provisioning operation.
- **Team ownership.** Split workspaces along team responsibility lines so only people familiar with a given area can change it.

### Identifying dependencies before migrating

If resources you're about to move are referenced by resources staying behind, you need a way for the remaining configuration to still reach them after the split. Recommended dynamic approaches, in preference order: a resource-specific data source if your provider offers one (e.g. `aws_vpc` data source), `tfe_outputs` if using HCP Terraform/Enterprise to read another workspace's outputs, or `terraform_remote_state` for other remote or local backends. Avoid hardcoding values manually, that requires manual updates every time the underlying data changes. `terraform graph` can help visualize dependencies before you commit to a split.

## Migrating resources between state files

Two approaches exist. **`removed` + `import` blocks is the currently recommended approach** (requires Terraform 1.7+), since it's configuration-driven and leaves a readable history of what happened. **`terraform state mv`** (Terraform 1.0+) is now considered legacy for this purpose, still functional but with more manual risk.

### removed + import workflow

In the source configuration, replace the `resource` block with a `removed` block:

```hcl
removed {
  from = aws_instance.example
  lifecycle {
    destroy = false
  }
}
```

`terraform plan` then `terraform apply` removes it from state without touching the real infrastructure (`destroy = false` is the critical setting here, the default without it is to actually destroy the resource).

In the destination configuration, add the `resource` block back plus an `import` block:

```hcl
resource "aws_instance" "example" {
  instance_type = "t3.micro"
  ami           = data.aws_ami.example.id
}

import {
  id = "i-07b510cff5f79af00"
  to = aws_instance.example
}
```

`terraform plan` then `terraform apply` binds the existing real resource to this new state, without recreating it. You can leave the `removed`/`import` blocks in place afterward as a historical record, or remove them once the migration is confirmed stable.

### state mv workflow (legacy)

Pull both state files locally, move the resource between them with `terraform state mv -state source.tfstate -state-out destination.tfstate <address> <address>`, then push both files back if using a remote backend. After moving, you still have to manually update both configurations (remove the resource from the source's `.tf` files, add it to the destination's) and verify with `plan` that neither side shows unexpected changes.

## The moved block, a different kind of relocation

`moved` is not for migrating between state files, it's for when a resource's _address_ changes but it stays in the same configuration, a rename, or restructuring into/out of a module.

```hcl
moved {
  from = aws_instance.a
  to   = aws_instance.b
}
```

Before planning, Terraform checks state for an object at `from`, renames it internally to `to`, and plans as if `to` already existed. No destroy, no recreate, the underlying infrastructure is untouched. This is the mechanism you'd reach for after renaming a resource in your `.tf` files, so Terraform doesn't interpret the rename as "destroy old, create new."

## Refresh-only mode

`terraform apply -refresh-only` (or `plan -refresh-only`) reconciles Terraform's _state_ to match drift that occurred on real infrastructure outside of Terraform's control, without ever modifying the real infrastructure itself. Available in HCP Terraform's UI as "Refresh state," and requires Terraform CLI 0.15.4+.

Use case: someone manually changed a setting in the cloud console. Rather than letting the next normal `apply` "correct" that change back to match your configuration (which may not be what you want, if the manual change should actually be kept), refresh-only mode updates state to reflect reality, letting you decide afterward whether to update your configuration to match, or to revert the drifted setting through Terraform on a subsequent normal apply.

Related, but distinct: `-refresh=false` on a normal `plan`/`apply` does the opposite, it skips the automatic state-refresh step entirely, useful when you want to check for configuration drift without spending time re-querying every resource's current state first.

## Common exam traps in this section

| Trap                                      | Why it's tricky                                                                                                             |
| ----------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `removed`+`import` vs `state mv`          | Assuming they're interchangeable; `removed`+`import` is the current recommendation, `state mv` is legacy                    |
| `removed` block's `destroy` default       | Forgetting that omitting `lifecycle.destroy` (or setting it true) destroys real infrastructure, not just removes from state |
| `moved` vs `removed`                      | Conflating renaming within one config (`moved`) with relocating between separate state files (`removed`+`import`)           |
| Refresh-only touching real infrastructure | It only ever updates state, never modifies the actual remote objects                                                        |
| `state push` safety checks                | Assuming any push just overwrites; lineage and serial checks exist specifically to prevent accidental data loss             |

## Quick reference

| Tool                                       | Purpose                                                          | Touches real infrastructure? |
| ------------------------------------------ | ---------------------------------------------------------------- | ---------------------------- |
| `moved` block                              | Rename/relocate a resource address within the same configuration | No                           |
| `removed` block (`destroy = false`)        | Stop managing a resource, hand off elsewhere                     | No                           |
| `removed` block (default/`destroy = true`) | Stop managing AND destroy the resource                           | Yes, destroys it             |
| `import` block                             | Bind an existing real resource to a new state entry              | No, just binds               |
| `terraform state mv`                       | Legacy: move a resource between state files directly             | No                           |
| `apply -refresh-only`                      | Sync state to match external drift                               | No                           |
| Normal `apply`                             | Reconcile real infrastructure to match configuration             | Yes                          |
