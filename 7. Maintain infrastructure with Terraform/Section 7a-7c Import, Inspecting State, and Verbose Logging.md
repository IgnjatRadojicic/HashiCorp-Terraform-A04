
## What this covers

Section 7 is the practical, CLI-heavy maintenance toolkit: bringing existing infrastructure under Terraform's management (7a), inspecting and manipulating state from the command line (7b), and debugging unexpected behavior with verbose logs (7c).

## 7a: Importing existing infrastructure

### Two approaches, and which one to actually use

**The `import` block** (Terraform 1.5+, covered in depth back in section 2d) is the current recommended approach for most workflows. It's declarative, reviewable in a `terraform plan` before anything is applied, and can be run through `terraform apply` like any other change, including in CI/CD pipelines.

```hcl
import {
  to = aws_instance.example
  id = "i-abcd1234"
}

resource "aws_instance" "example" {
  # can start empty, fill in after reviewing the plan
}
```

As of 1.5+, `terraform plan -generate-config-out=generated.tf` can auto-generate the resource block's configuration to match the real object, removing most of the manual reverse-engineering that used to be required.

**The `terraform import` CLI command** is the older, imperative approach. It modifies state immediately with no plan step to review first.

```bash
terraform import aws_instance.example i-abcd1234
```

Still shows up in older tutorials, existing scripts, and one-off situations where writing a full `import` block feels like overkill. Not deprecated, just no longer the default recommendation.

### terraform import command mechanics

- **One resource per invocation.** Cannot import an entire collection (a VPC and all its subnets) in a single call, each object needs its own `terraform import` run.
- **ADDRESS** is any valid resource address, root module or nested in a child module, including indexed (`count`) or keyed (`for_each`) instances.

```bash
terraform import aws_instance.foo i-abcd1234
terraform import module.foo.aws_instance.bar i-abcd1234
terraform import 'aws_instance.baz[0]' i-abcd1234
terraform import 'aws_instance.baz["example"]' i-abcd1234
```

- **ID format is entirely provider- and resource-specific.** An EC2 instance uses its instance ID, a Route53 zone uses its zone ID. Always check that resource type's provider documentation.
- **Provider configuration for the import** is read from `.tf` files in the working directory (or a directory given via `-config`). Same restriction discussed earlier applies: **the provider configuration used for import cannot depend on a data source**, only on variables or literal values, since Terraform needs to resolve the provider before it can do anything else, including reading most data sources.
- **`-provider` flag is deprecated.** Previously used to override which provider config to import with; default resolution from the target resource's own configuration is now the recommended path.
- **One-object-to-one-address is a hard assumption.** Terraform expects every real remote object to map to exactly one resource address. Importing the same object into two different addresses causes unwanted behavior, Terraform's whole model assumes it created (or was told about) each object exactly once.

### Complex imports, the destroy-on-next-plan trap

Some resources cascade: importing one object implicitly brings related sub-resources along in the real infrastructure, even though only the primary object gets bound to your state via the import itself. A classic example: an AWS network ACL import doesn't automatically create matching `aws_network_acl_rule` entries in your configuration, even though the real ACL has real rules attached.

**The danger:** `terraform import` never errors or warns about this. The problem surfaces one step later, on your next `plan`. State now reflects that the real object has those associated rules (Terraform reads current reality), but your configuration has no corresponding resource blocks for them. Terraform's plan logic reads this as "configuration doesn't want these, destroy them." If you don't consult the import output and manually add resource blocks for every secondary resource, the very next `apply` can silently delete real infrastructure you never intended to touch.

## 7b: Inspecting and modifying state with the CLI

All `terraform state` subcommands work identically against local or remote state, remote just costs a network round-trip per read/write.

|Subcommand|Purpose|Modifies state?|Writes backup?|
|---|---|---|---|
|`state list`|List resource addresses currently tracked|No|No|
|`state show`|Print one resource's attributes|No|No|
|`state mv`|Rename or relocate a resource's address|Yes|Yes|
|`state rm`|Remove a resource from state (without destroying it)|Yes|Yes|
|`state pull`|Print raw state to stdout|No|No|
|`state push`|Overwrite remote state from a local file|Yes|Yes|
|`state replace-provider`|Change which provider a resource is associated with|Yes|Yes|

**Backups cannot be disabled for state-modifying subcommands.** `-backup` only controls the destination path of the backup file, never whether one gets written. This is deliberate, given how sensitive and hard-to-recover a corrupted state file is. If you don't want backup files accumulating, you clean them up manually, there's no flag to suppress them.

**Read-only subcommands never write backups**, since there's nothing to protect against, nothing was modified.

**Designed for Unix pipelines.** Output is structured to compose with `grep`, `awk`, and similar tools, useful for scripted or advanced filtering beyond what the subcommands offer natively.

## 7c: Verbose logging for debugging

### TF_LOG, the master switch

Setting `TF_LOG` to any value enables detailed logging to `stderr`. Accepted levels, in decreasing verbosity: `TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR`. A separate value, `JSON`, outputs TRACE-level-or-higher logs in a machine-parseable JSON encoding (explicitly noted as an unstable interface, subject to change without notice, intended for tooling rather than manual reading).

```bash
export TF_LOG=DEBUG
terraform plan
```

### Scoped logging: isolating Core vs provider

`TF_LOG_CORE` and `TF_LOG_PROVIDER` take the same level values as `TF_LOG`, but restrict logging to just one side of the architecture. This maps directly onto the Core-vs-plugin split from section 2b: Core handles the general Terraform workflow (graph building, state management, CLI), while each provider plugin handles its own resource-specific logic, communicating with Core over RPC. Scoping logs to one side is the practical way to narrow down whether a bug lives in Terraform itself or inside a specific provider's implementation, without wading through the other side's noise.

```bash
export TF_LOG_PROVIDER=TRACE  # only the provider plugin's logs
```

### TF_LOG_PATH, persistence, not activation

```bash
export TF_LOG=DEBUG
export TF_LOG_PATH=/tmp/terraform.log
terraform apply
```

`TF_LOG_PATH` forces logs to be appended to a specific file whenever logging is enabled. **It has no effect on its own.** Setting only `TF_LOG_PATH` without also setting `TF_LOG` (or a scoped variant) results in zero logging, the file isn't even created. `TF_LOG_PATH` is a destination setting, not a switch, the switch is always `TF_LOG` or its scoped variants.

### Bug reports

If filing a Terraform bug, HashiCorp's own guidance is to include the detailed log output, typically shared via a service like a Gist given its length.

## Common exam traps in this section

| Trap                                              | Why it's tricky                                                                                                        |
| ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Complex imports and next-plan destroys            | The `import` command itself never errors, the danger only surfaces on the following `plan`                             |
| Backup files being disableable                    | `-backup` only redirects the path, it never suppresses the backup entirely                                             |
| Read-only state subcommands still needing backups | Assuming all `state` subcommands carry the same backup requirement; only modifying ones do                             |
| `TF_LOG_PATH` alone enabling logging              | It's a destination, not a switch; `TF_LOG` (or a scoped variant) must also be set                                      |
| Import provider config depending on a data source | The provider config used for import must resolve from variables/literals only, not data sources                        |
| `terraform import` importing whole collections    | It only ever imports one resource per invocation, cascading resources need separate imports and matching config blocks |

## Quick reference

| Task                               | Command / mechanism                                                        |
| ---------------------------------- | -------------------------------------------------------------------------- |
| Declarative import (recommended)   | `import` block + `terraform apply`, optionally with `-generate-config-out` |
| One-off imperative import          | `terraform import <ADDRESS> <ID>`                                          |
| List tracked resources             | `terraform state list`                                                     |
| Inspect one resource               | `terraform state show <ADDRESS>`                                           |
| Rename/relocate in state           | `terraform state mv` (writes backup)                                       |
| Remove from state, keep real infra | `terraform state rm` (writes backup)                                       |
| Enable verbose logging             | `TF_LOG=<LEVEL>`                                                           |
| Isolate Core-only logs             | `TF_LOG_CORE=<LEVEL>`                                                      |
| Isolate provider-only logs         | `TF_LOG_PROVIDER=<LEVEL>`                                                  |
| Persist logs to a file             | `TF_LOG_PATH=<path>`, requires `TF_LOG` also set                           |