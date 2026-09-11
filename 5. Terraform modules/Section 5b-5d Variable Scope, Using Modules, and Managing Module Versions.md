
## What this covers

These three sub-sections overlap heavily in the source docs, so they're combined here. 5b is about how variables, locals, and outputs define a module's interface and scoping boundary. 5c is about the patterns for actually composing modules together in a configuration. 5d, module versioning, was already fully covered under 5a's `version` argument, this recap just cross-references it rather than repeating it.

## Variable scope within modules

A module's `variable` blocks define its entire input interface, nothing else can reach into a module from outside it. This creates a real boundary: a module's internal resources, its `locals`, and any variables not exposed as outputs are invisible to whatever calls that module.

```hcl
# child module (./modules/example)
variable "instance_type" {
  type        = string
  description = "EC2 instance type for the web server"
  default     = "t2.micro"
}

resource "aws_instance" "web" {
  instance_type = var.instance_type
}
```

```hcl
# root module
module "example" {
  source        = "./modules/example"
  instance_type = "t3.micro"
}
```

The root module sets `instance_type` as an argument on the `module` block, matching the child's `variable` label. Inside the child, that value is only ever accessed as `var.instance_type`, the child has no idea where the value came from or whether it was a literal, another variable, or a resource attribute in the caller.

### locals stay strictly internal


```hcl
locals {
  app_name = "${var.project_name}-${var.environment}"
}

resource "aws_instance" "example" {
  tags = {
    Name = local.app_name
  }
}
```

A `locals` block scoped to a module is not visible to the caller or to any other module at all, not even a parent. If you need a computed value available outside the module, it has to be explicitly re-exposed through an `output`. There is no way to reach into a child module's `locals` from the outside, this is a hard boundary, not a convention.

### Outputs are the only way data flows back up


```hcl
# child module
output "instance_ip" {
  description = "Private IP address of the EC2 instance"
  value       = aws_instance.web.private_ip
}
```


```hcl
# root module, referencing the child's output
output "website_url" {
  value       = "https://${module.example.instance_ip}"
  description = "The URL of the web server, starting with https://."
}
```

The `module.<label>.<output>` syntax is the only way a parent reads anything out of a child. If a value isn't declared as an output, it simply doesn't exist outside that module's own files, no matter how deep in the resource graph it sits.

### The full scoping picture

| From this module's perspective | Can it see the caller's values?                      | Can the caller see it?                      |
| ------------------------------ | ---------------------------------------------------- | ------------------------------------------- |
| `variable`                     | Yes, this is exactly how the caller passes values in | No, the caller only sees what it itself set |
| `locals`                       | N/A, always module-internal                          | No, never, must be re-exposed via output    |
| `output`                       | N/A                                                  | Yes, via `module.<label>.<output>`          |

## Using modules in configuration

### Module composition, the flat pattern

Nesting modules deeply (child modules calling their own child modules, several layers down) is possible but discouraged. The recommended pattern is to keep the module tree flat, one level of child modules under the root, and wire modules together using expressions in the root module, the same way you'd wire together plain resources.

```hcl
module "network" {
  source = "./modules/aws-network"

  base_cidr_block = "10.0.0.0/8"
}

module "consul_cluster" {
  source = "./modules/aws-consul-cluster"

  vpc_id     = module.network.vpc_id
  subnet_ids = module.network.subnet_ids
}
```

This is called module composition, assembling small, focused, reusable modules into a larger system by passing outputs of one into inputs of another, rather than one giant module trying to do everything internally.

### Dependency inversion

A module should generally receive its dependencies as inputs rather than creating or discovering them internally. The `consul_cluster` module above takes `vpc_id` and `subnet_ids` as arguments instead of building its own VPC. This keeps the module decoupled from exactly how those values get produced, today they might come from another module's output, later they could come from a `data` source instead, and the `consul_cluster` module's own code never has to change.

hcl

```hcl
data "aws_vpc" "main" {
  tags = { Environment = "production" }
}

module "consul_cluster" {
  source = "./modules/aws-consul-cluster"

  vpc_id = data.aws_vpc.main.id
  # ...
}
```

### Conditional creation of objects

When some environments already have a resource and others need it created, the recommended pattern is not to build conditional logic inside the module. Instead, define an input variable typed loosely enough (usually an `object` with just the attributes the module actually needs) to accept either a resource or a data source result, and let the caller decide which one to pass in.

```hcl
variable "ami" {
  type = object({
    id           = string
    architecture = string
  })
}
```

```hcl
# Caller manages the AMI directly:
resource "aws_ami_copy" "example" {
  name              = "local-copy-of-ami"
  source_ami_id     = "ami-abc123"
  source_ami_region = "eu-west-1"
}

module "example" {
  source = "./modules/example"
  ami    = aws_ami_copy.example
}

# Or, caller references an AMI that already exists:
data "aws_ami" "example" {
  owner = "9999933333"
  tags = { application = "example-app" }
}

module "example" {
  source = "./modules/example"
  ami    = data.aws_ami.example
}
```

This stays consistent with Terraform's declarative style. Rather than a module trying to detect and branch on whether something exists, the caller states explicitly what it expects, and a future reader understands the intent without inspecting remote state.

### Assumptions and guarantees

Every module implicitly has assumptions (conditions that must hold for its configuration to work, an AMI must be x86_64) and guarantees (things callers can rely on, an instance will get a private DNS record). Validating configuration, especially with output preconditions, helps make these explicit rather than leaving them as unwritten expectations.


```hcl
output "api_base_url" {
  value = "https://${aws_instance.example.private_dns}:8433/"

  precondition {
    condition     = data.aws_ebs_volume.example.encrypted
    error_message = "The server's root volume is not encrypted."
  }
}
```

### Multi-cloud abstractions

Terraform itself deliberately doesn't unify different vendors' services behind one interface, that would force a lowest-common-denominator design and hide each provider's actual capabilities. You can build your own lightweight abstraction where it makes sense, by defining a shared object type as a module's input variable, then writing one implementation module per vendor that accepts that same shape.

```hcl
variable "recordsets" {
  type = list(object({
    name    = string
    type    = string
    ttl     = number
    records = list(string)
  }))
}
```

Any module implementing this variable, whether it targets Route53, Cloudflare, or another DNS provider, can be swapped in without touching the configuration that produces the recordset data. The abstraction lives entirely in the shared type definition, not in any Terraform-provided cross-vendor feature.

### Data-only modules

A module containing no `resource` blocks at all, only `data` sources, exists purely to encapsulate how some shared information gets retrieved.

```hcl
module "network" {
  source      = "./modules/join-network-aws"
  environment = "production"
}

module "k8s_cluster" {
  source     = "./modules/aws-k8s-cluster"
  subnet_ids = module.network.aws_subnet_ids
}
```

The benefit: the actual retrieval mechanism (direct API query via `data` sources, reading from Consul, reading another config's state via `terraform_remote_state`) can change without updating every configuration that consumes it, as long as the data-only module's own output shape stays the same.

## Outputs from child modules, additional detail beyond 4c

Everything documented in the 4c recap about the `output` block applies identically inside a module. Two points specific to the module context worth restating here:

**Ephemeral outputs are child-module-only.** You cannot set `ephemeral = true` on a root module's output block, Terraform rejects it outright. Ephemeral outputs exist specifically to pass a transient value (a short-lived token, say) up from a child module to whatever called it, without persisting it to state or plan. Since the root module is the top of the call chain, there's no further "up" to pass an ephemeral value to, so the restriction exists because there'd be no valid ephemeral consumer for it.

**Preconditions on module outputs still run before storage/exposure.** Same timing as any other output precondition, evaluated before Terraform stores the value in state or exposes it, whether that output belongs to the root module or a child.

## Module versions, 5d, cross-reference

Already covered in full under the 5a recap's `version` argument section: registry-only, semantic versioning, always constrain it, requires `terraform init` after changes, and local-path modules don't support it since they inherit their caller's version implicitly.

## Common exam traps in this section

|Trap|Why it's tricky|
|---|---|
|`locals` visibility|Assuming a parent module can somehow read a child's locals, it never can, only outputs cross the boundary|
|Conditional resource creation|Assuming the module should contain the create-or-lookup branching logic itself, the caller decides instead|
|Multi-cloud abstraction mechanism|Assuming Terraform has a built-in cross-vendor feature, it's just a shared object type convention you define yourself|
|Ephemeral output on root module|Forgetting this is rejected outright, not merely discouraged|
|Data-only modules|Assuming a module must contain resources to be useful, a pure-data-source module is a valid, recommended pattern|

## Quick reference

|Concept|Visible to caller?|Mechanism|
|---|---|---|
|`variable`|Set by caller|`module` block argument matching the variable label|
|`locals`|Never|N/A, must be re-exposed via `output`|
|`output`|Yes|`module.<label>.<output>`|

| Composition pattern                     | Use when                                                                                                            |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| Dependency inversion                    | A module needs infrastructure it shouldn't own itself (network, existing resources)                                 |
| Conditional creation via input variable | Some environments already have a resource, others need it created                                                   |
| Multi-cloud abstraction                 | Multiple vendors implement a similar concept and a shared interface is worth the lowest-common-denominator tradeoff |
| Data-only module                        | Retrieval logic for shared info needs to be centralized and swappable                                               |