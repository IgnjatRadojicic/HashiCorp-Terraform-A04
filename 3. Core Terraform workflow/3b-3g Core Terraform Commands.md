
---

### terraform init (3b)

`terraform init` initializes a working directory containing Terraform configuration files. It is the first command you run after writing a new config or cloning one from version control. It is safe to run multiple times and will never delete your existing configuration or state.

It performs four jobs in order:

**Backend initialization** configures the remote backend declared in your config. If you are switching backends, you must pass either `-reconfigure` or `-migrate-state`.

`-reconfigure` discards any existing backend configuration and reinitializes from scratch with no state migration. Use this when you want a clean slate.

`-migrate-state` attempts to copy existing state to the new backend. Use this when you want to carry your state over to the new backend.

`-backend=false` skips backend initialization entirely. Useful in CI when you want to validate or format config without touching remote state.

**Child module installation** downloads any remote modules declared in your config into `.terraform/modules/`. Re-running init installs any modules added since the last init but does not update already installed modules.

`-get=false` skips module installation. Only use this when the working directory was already previously initialized.

**Plugin installation** downloads providers declared in `required_providers` into `.terraform/providers/`. After successful installation Terraform writes the provider selections to the lock file.

`-upgrade` tells Terraform to ignore the existing lock file selections and install the newest version of each provider that satisfies the version constraints. Always review the updated lock file diff before committing.

`-lockfile=readonly` suppresses lock file changes but still verifies checksums against what is already recorded. Useful when a third party tool manages the lock file and you want to prevent Terraform from modifying it.

`-plugin-dir=PATH` forces Terraform to read plugins only from a specified local directory. Used for air-gapped environments or when testing a locally built provider.

**Lock file** is written or updated after successful provider installation. Always commit `.terraform.lock.hcl` to version control.

The `-from-module=SOURCE` option copies a module into the target directory before initializing. Useful for bootstrapping a new config from an example or a version controlled source.

```bash
terraform init                        # standard initialization
terraform init -upgrade               # upgrade providers within constraints
terraform init -backend=false         # skip backend, useful for validate in CI
terraform init -reconfigure           # reinitialize backend, discard old config
terraform init -migrate-state         # reinitialize backend, copy existing state
terraform init -from-module=MODULE    # copy module into directory then init
```

---

### terraform validate (3c)

`terraform validate` checks that configuration files are syntactically valid and internally consistent. It refers only to the configuration and never accesses any remote service, provider API, or existing state.

What it catches:

- Misspelled block or argument names
- Missing required arguments
- Wrong argument types
- References to variables or resources that do not exist in the config

What it does not catch:

- Whether attribute values are actually valid for the provider -- that happens at plan time when the provider API is called
- Whether credentials are valid
- Whether referenced remote resources exist

Validate requires an initialized working directory with providers and modules installed. To initialize for validation without touching a backend:

```bash
terraform init -backend=false
terraform validate
```

It is safe to run automatically as a post-save check in an editor or as a CI step before plan runs. Plan and apply both run an implied validate automatically, so explicit validate is most useful as an early gate before reaching plan.

```bash
terraform validate           # human readable output
terraform validate -json     # machine readable output for CI integrations
```

---

### terraform plan (3d)

`terraform plan` evaluates your configuration, compares the desired state against real infrastructure using state data and provider API calls, and produces a description of the changes needed. It does not make any changes to real infrastructure.

**Three planning modes:**

`terraform plan` runs the default mode. It calculates what needs to be created, updated, or destroyed to reach the desired state.

`terraform plan -destroy` runs destroy mode. It shows what would be destroyed if you ran `terraform destroy`. Nothing is executed.

`terraform plan -refresh-only` runs refresh only mode. It shows what would change in the state file to match real world drift. No infrastructure changes are planned.

**Saving a plan:**

```bash
terraform plan -out=plan.out    # save the plan as a binary artifact
```

A saved plan file can be passed to `terraform apply` to execute exactly those changes with no re-plan and no confirmation prompt. This two-step workflow is standard in CI pipelines because it guarantees the reviewed plan is exactly what gets applied.

```bash
terraform show plan.out         # inspect a saved plan before applying
```

**Useful planning options:**

```bash
terraform plan -replace="aws_instance.web"     # force replacement of a specific resource
terraform plan -target="aws_instance.web"      # plan only a specific resource
terraform plan -var="environment=prod"         # pass a variable value
terraform plan -var-file="prod.tfvars"         # pass a variable file
terraform plan -refresh=false                  # skip live cloud queries, use cached state
```

**The resource graph** underpins every plan. Terraform builds a directed acyclic graph of all resources before doing anything, resolves dependencies from references, and executes independent resources in parallel. The default parallelism is 10 concurrent operations. Override with `-parallelism=n`.

---

### terraform apply (3e)

`terraform apply` executes the operations proposed in a plan.

**Automatic plan mode** runs when you call apply without a saved plan file. Terraform creates a new plan, prompts for approval, and then executes. All planning modes and options available to `terraform plan` are available here.

**Saved plan mode** runs when you pass a saved plan file. Terraform executes exactly those changes without prompting. Passing the file is treated as your approval. You cannot specify additional planning modes or options when applying a saved plan because those decisions were already made when the plan was created.

```bash
terraform apply                          # automatic plan mode, prompts for approval
terraform apply plan.out                 # saved plan mode, no prompt
terraform apply -auto-approve            # skip confirmation prompt, dangerous without review
terraform apply -replace="aws_instance.web"   # force replacement of specific resource
terraform apply -target="aws_instance.web"    # apply only a specific resource
terraform apply -var="environment=prod"       # pass a variable at apply time
terraform apply -destroy                      # destroy all managed resources
terraform apply -parallelism=5               # limit concurrent operations
```

**What happens during apply internally:**

1. Acquires a state lock
2. Runs a plan if no saved plan file was passed
3. Waits for approval unless `-auto-approve` is set or a saved plan file was passed
4. Executes each operation in dependency order, parallel where possible
5. Updates state after each resource operation completes
6. Releases the state lock on completion

**What happens when apply errors:**

Terraform does not roll back. It logs the error, updates state with any changes that completed successfully, unlocks state, and exits. Your infrastructure may be partially applied. Fix the error and run `terraform apply` again. Terraform will leave already-created resources alone and only retry what failed.

**The `-auto-approve` warning:**

Using `-auto-approve` in automation is common but requires that no one can change infrastructure outside of your Terraform workflow. If drift can occur, skipping the review step means you might apply against a stale plan.

---

### terraform destroy (3f)

`terraform destroy` deprovisions all objects managed by a Terraform configuration. It is a convenience alias for `terraform apply -destroy` and accepts most of the same options. It does not accept a saved plan file.

```bash
terraform destroy                              # destroy everything, prompts for approval
terraform plan -destroy                        # preview destruction without executing
terraform apply -destroy                       # identical to terraform destroy
terraform destroy -target="aws_instance.web"  # destroy a specific resource and its dependencies
```

Typically used for ephemeral development environments, test infrastructure, or learning environments where you want to clean up everything when finished. You would not typically run this against long-lived production infrastructure.

---

### terraform fmt (3g)

`terraform fmt` rewrites Terraform configuration files to match the canonical HCL format and style. It applies a subset of the Terraform language style conventions along with minor adjustments for readability.

It is intentionally opinionated with no customization options. The goal is consistency across Terraform codebases. If you disagree with its formatting decisions you may choose not to use it or use a third party formatter, but fmt itself cannot be configured.

The canonical format may change in minor ways between Terraform versions. After upgrading Terraform, run `terraform fmt` proactively on your modules.

```bash
terraform fmt              # format all .tf files in the current directory
terraform fmt -recursive   # also format files in subdirectories
terraform fmt -check       # check formatting without modifying files, exits non-zero if files need formatting
terraform fmt -diff        # show what would change without modifying files
terraform fmt -list=false  # do not list files whose formatting differs
```

The `-check` flag is the important one for CI. It exits with a non-zero status code if any files need formatting without modifying them. A CI pipeline that runs `terraform fmt -check` will fail if a developer pushes unformatted code, forcing them to run `terraform fmt` locally before pushing.

A typical CI formatting gate:

```bash
terraform fmt -check -recursive -diff
```

This checks all files recursively, exits non-zero if anything needs formatting, and shows the diff so the developer knows exactly what to fix.

---

### Command quick reference

| Command                            | What it does                                                 |
| ---------------------------------- | ------------------------------------------------------------ |
| `terraform init`                   | Initialize working directory, download providers and modules |
| `terraform init -upgrade`          | Re-resolve providers, update lock file                       |
| `terraform init -backend=false`    | Initialize without backend, for validate in CI               |
| `terraform init -reconfigure`      | Reinitialize backend, discard existing config                |
| `terraform init -migrate-state`    | Reinitialize backend, copy existing state                    |
| `terraform validate`               | Check syntax and internal consistency offline                |
| `terraform validate -json`         | Machine readable validation output                           |
| `terraform plan`                   | Preview changes without executing                            |
| `terraform plan -out=plan.out`     | Save plan as binary artifact                                 |
| `terraform plan -destroy`          | Preview full destruction                                     |
| `terraform plan -refresh-only`     | Preview state sync from real world                           |
| `terraform plan -replace="<addr>"` | Preview forced replacement of a resource                     |
| `terraform plan -refresh=false`    | Skip live queries, use cached state                          |
| `terraform apply`                  | Execute plan, prompts for approval                           |
| `terraform apply plan.out`         | Execute saved plan, no prompt                                |
| `terraform apply -auto-approve`    | Execute without confirmation prompt                          |
| `terraform apply -destroy`         | Destroy all managed resources                                |
| `terraform destroy`                | Alias for terraform apply -destroy                           |
| `terraform plan -destroy`          | Preview destruction without executing                        |
| `terraform fmt`                    | Format files in current directory                            |
| `terraform fmt -recursive`         | Format files including subdirectories                        |
| `terraform fmt -check`             | Check formatting without modifying, CI gate                  |
| `terraform fmt -diff`              | Show formatting diff without modifying                       |