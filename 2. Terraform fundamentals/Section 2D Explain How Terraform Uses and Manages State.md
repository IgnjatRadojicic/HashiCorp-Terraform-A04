
#### What state is

Terraform state is a JSON file (`terraform.tfstate`) that maps your configuration to real-world resources. It is the bridge between what your `.tf` files describe and what actually exists in the cloud.

Without state, Terraform would have no way to know that `resource "aws_instance" "web"` in your config corresponds to instance `i-0a1b2c3d4e5f` in AWS. State is what holds that binding.

#### Four purposes of state

**1. Mapping to the real world**

Terraform needs a database to map config to real resources. Early prototypes tried using cloud tags instead of a state file, but this failed because not all resources support tags and not all providers support tags. Terraform uses its own state structure instead.

Each remote object must be bound to exactly one resource instance. If one remote object is bound to two resource instances the mapping becomes ambiguous and Terraform may behave unexpectedly. This is why you must be careful when importing resources, each object imported to one resource instance only.

**2. Metadata**

State stores metadata alongside resource mappings -- most importantly, dependency information. This is critical for destruction order.

When you delete a resource from your config, Terraform can no longer derive dependency order from the config alone because the config no longer has that resource. But state still has the last known dependency tree, so Terraform knows the correct order to destroy things. For example: instances must be destroyed before the security groups they reference, subnets before the VPC they belong to.

State also stores which provider configuration was most recently used with each resource, which matters when multiple aliased providers are present.

**3. Performance**

State caches all attribute values for every resource. For small infrastructures Terraform can query all resources live from the cloud on every plan. For large infrastructures this is too slow -- cloud providers have API rate limits and each resource query takes hundreds of milliseconds.

For large configs, teams use `-refresh=false` to skip the live query entirely and treat cached state as the record of truth. This is why state must be kept accurate.

**4. Syncing**

Local state is fine for solo work. In a team, everyone must work from the same state file or operations will apply to different remote objects. Remote state solves this, and state locking prevents two users from running apply simultaneously which would corrupt state.

#### State file structure

The state file is plain JSON. You should never edit it by hand -- use CLI commands instead. Here is what it contains:

json

```json
{
  "version": 4,
  "terraform_version": "1.9.0",
  "resources": [
    {
      "mode": "managed",
      "type": "aws_instance",
      "name": "web",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "attributes": {
            "id":            "i-0a1b2c3d4e5f",
            "ami":           "ami-0c55b159cbfafe1f0",
            "public_ip":     "54.12.34.56"
          },
          "dependencies": [
            "aws_security_group.sg_web"
          ]
        }
      ]
    }
  ]
}
```

Key fields:

- `mode` -- either `managed` (a resource block) or `data` (a data source). Data sources appear in state but are never destroyed.
- `type` -- the resource type from the provider
- `name` -- the local name from your config
- `attributes` -- every attribute value the provider returned, including computed ones like `public_ip` and `id`
- `dependencies` -- the dependency list Terraform uses for ordering, especially destruction ordering

---

#### State CLI commands -- how to interact with state safely

Never touch the `.tfstate` file directly. Use these commands:

|Command|What it does|
|---|---|
|`terraform show`|Human-readable view of all resources and their attributes|
|`terraform state list`|List of all resource addresses in state|
|`terraform state show <address>`|Detailed view of one specific resource|
|`terraform state mv <src> <dst>`|Move or rename a resource in state|
|`terraform state rm <address>`|Remove a resource from state without destroying it (legacy)|
|`terraform import <address> <id>`|Bring an existing real resource into state|
|`terraform refresh`|Update state to match reality (largely superseded)|

---

#### terraform state mv -- what it does and what it does not do

`terraform state mv` moves a resource between state files or renames it within the same state. It updates state only -- it does NOT touch your `.tf` config files.

bash

```bash
# rename within same state
terraform state mv aws_instance.example aws_instance.web

# move to a different state file
terraform state mv -state-out=../other.tfstate aws_instance.example aws_instance.example
```

After running this, your config still has the old name. The next `terraform plan` will show a diff unless you also update the config manually to match. State and config are completely separate -- `state mv` only touches one of them.

Use case: renaming a resource or moving it into a module without destroying and recreating the real infrastructure.

#### Removing a resource from state without destroying it

**Legacy approach -- terraform state rm:**

bash

```bash
terraform state rm aws_instance.example
```

Removes the resource from state immediately. The real infrastructure is untouched. No plan preview -- it just does it.

**Modern approach -- removed block (Terraform 1.7+):**


```hcl
removed {
  from = aws_instance.example

  lifecycle {
    destroy = false   # prevents the real resource being destroyed
  }
}
```

Comment out or delete the original resource block, add the `removed` block, then run `terraform apply`. Terraform removes it from state but leaves the real infrastructure alone. The advantage is it goes through the normal plan/apply workflow so you can review exactly what will happen before committing.

After removing, if you want to bring the resource back under Terraform management, use `terraform import`.

#### terraform import -- bringing existing resources into state

If a resource was created outside of Terraform -- manually in the console, by another tool, or by a different Terraform config -- you can bring it under management:

bash

```bash
terraform import aws_instance.web i-0a1b2c3d4e5f
```

This writes the resource into state against the address you specify. You still need the resource block in your config to match. Terraform 1.5+ also supports configuration-driven import using an `import` block, which lets you preview the import through the normal plan workflow and can auto-generate config.

---

#### terraform refresh vs -refresh-only

**terraform refresh** is a standalone command that updates the state file to match real-world resources. It runs and makes state changes immediately with no preview. It does NOT touch your config files. It is now considered the legacy approach.

**-refresh-only flag** is the modern replacement:

bash

```bash
terraform plan -refresh-only    # shows what state changes would be made, no action
terraform apply -refresh-only   # makes those state changes after you review and approve
```

Same result as `terraform refresh` but through the normal plan/apply workflow with a review step before anything changes. Always prefer this over the standalone `terraform refresh`.

Also worth knowing: `terraform plan` and `terraform apply` both run an implicit in-memory refresh automatically before creating the execution plan. In normal workflows you rarely need to explicitly refresh at all -- drift is picked up automatically.

---

#### Remote state and locking

Local state works fine for solo projects. For teams, remote state is essential.

hcl

```hcl
terraform {
  backend "s3" {
    bucket         = "my-tfstate-bucket"
    key            = "prod/terraform.tfstate"
    region         = "eu-west-1"
    dynamodb_table = "terraform-locks"
  }
}
```

The DynamoDB table provides state locking -- only one `terraform apply` can run at a time. Without locking, two users applying simultaneously would both read the same state, make changes independently, and write back conflicting versions -- corrupting state.

HCP Terraform handles remote state and locking automatically without any backend configuration.

---

#### What to never do

- Never edit the state file by hand -- any manual change risks corrupting it and causing resources to be destroyed or recreated unexpectedly
- Never commit state to git -- it contains plaintext secrets that providers write into it
- Never let two applies run simultaneously without locking -- corrupted state is difficult to recover from

---

#### Quick reference -- state-related flags

| Flag                                   | What it does                                        |
| -------------------------------------- | --------------------------------------------------- |
| `terraform plan -refresh-only`         | Preview state updates from real-world drift         |
| `terraform apply -refresh-only`        | Apply state updates without changing infrastructure |
| `terraform apply -replace="<address>"` | Force destroy and recreate a specific resource      |
| `terraform plan -refresh=false`        | Skip live cloud queries, use cached state as truth  |
| `terraform apply -target="<address>"`  | Apply changes to one specific resource only         |