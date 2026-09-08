#### What this section covers

Variables and outputs form the interface of a Terraform module. Variables are the module's inputs similar to function arguements. Outputs are the module's return values. Locals sit in between they are internal, reusable expressions scoped to the module and not part of its public interface.

##### variable block

```hcl
variable "instance_type" {
  type        = string
  description = "EC2 instance type for the web server"
  default     = "t2.micro"
}
```
Reference it elsewhere in the module with `var.instance_type`.
### Reserved names

A variable's label cannot be any of the following, since they collide with existing meta-arguments and block names: `source`, `version`, `providers`, `count`, `for_each`, `lifecycle`, `depends_on`, `locals`. Declaring `variable "count" { ... }` fails outright, it is not a case of silent shadowing.
### type

Constrains what value the variable accepts. If omitted, the variable accepts any type. Always prefer an exact type constraint over `any` unless you are genuinely passing the value through untouched (see the `any` section below).
### default

Makes the variable optional. Without a `default`, the caller must supply a value. The `default` value must be a literal, it cannot reference other resources or data sources in the configuration.

hcl

```hcl
variable "region" {
  type    = string
  default = "us-east-1"
}
```

### description

Documents the variable from the point of view of the module consumer, not the maintainer. If you need notes for yourself as the module author, use a comment instead.

### validation

```hcl
variable "image_id" {
  type = string
  validation {
    condition     = length(var.image_id) > 4 && substr(var.image_id, 0, 4) == "ami-"
    error_message = "The image_id value must start with \"ami-\"."
  }
}
```

If `condition`evalulates to false, Terraform halts and displays `error_message`.

### sensitive

Redacts the variable's value from CLI plan and apply output. Any expression that uses a sensitive variable becomes sensitive automatically, so it propagates.

```hcl
variable "database_password" {
  type      = string
  sensitive = true
}
```

**Important limit:** `sensitive` is a display filter only. The value is still written to state in plaintext, and `terraform output -json` or `-raw` bypasses the redaction entirely. Do not treat `sensitive` as encryption or access control.

### nullable

Controls whether the caller can pass `null` for this variable. Defaults to `true`.

```hcl
variable "example" {
  type     = string
  nullable = false
}
```

If `nullable = false` and the caller passes `null` (or omits the variable with no default), Terraform raises a plan-time error. This is a hard block on the variable's top-level value, not just its nested elements. There is no fallback to a previous state value, variables are evaluated fresh on every run.

### ephemeral (Terraform 1.10+)

Marks a variable as never persisted to state or plan files. Useful for short-lived tokens or session credentials.

```hcl
variable "session_token" {
  type      = string
  ephemeral = true
}
```

An ephemeral variable's value can only be used in other ephemeral contexts: another child module's ephemeral output, another ephemeral variable, a write-only resource argument, an ephemeral resource block, provider configuration, or provisioner/connection configuration. Assigning an ephermal variable directly to a normal (non-write-only) resource attribute is rejected. Any expression that references an ephermal variable becomes ephermal itself.

### const

Allows a variable's value to be used during early Terraform operations, specifically inside a module's `source` or `version` arguments, which are evaluated before Terraform builds a plan.

```hcl
variable "module_version" {
  type  = string
  const = true
}
```

A `const` variable must resolve to a known, static value. It cannot depend on any dynamic result from a plan.

### deprecated (Terraform 1.15+)

Documents that a variable is being phased out. Terraform surfaces the message when the variable is set in a root module or passed by a caller.

```hcl
variable "old_input" {
  type       = string
  deprecated = "This variable is deprecated, please use 'new_input' instead."
}
```

## Variable value precedence

When the same variable receives a value from more than one source, Terraform resolves the conflict in this order, with later sources overriding earlier ones: (LOWEST TO HIGHEST)

1. Environment variables (`TF_VAR_name`)
2. `terraform.tfvars`
3. `terraform.tfvars.json`
4. `*.auto.tfvars` / `*.auto.tfvars.json`, processed alphabetically
5. `-var` and `-var-file` flags on the command line, in the order given

A `-var` flag on the CLI always wins over anything in a `.tfvars` file. This is a common exam trap.


## output block

### Basic shape

```hcl
output "instance_ip_addr" {
  value       = aws_instance.server.private_ip
  description = "The private IP address of the main server instance."
}
```

An output is like a return value. It has four practical purposes: exposing a child module's resource attributes to the parent module, displaying values in CLI output for the root module, letting other configurations read root module outputs through `terraform_remote_state`, and passing data to an automation tool.

### value

Required. Any valid expression. Terraform evaluates it and stores the result in state.

### description

Same convention as variables, write it from the consumer's point of view.

### sensitive

Redacts the output's value in CLI output. Same caveats as variable `sensitive`: state still stores the plaintext value, and `-json`/`-raw` still display it.

```hcl
output "db_password" {
  value     = aws_db_instance.db.password
  sensitive = true
}
```

### ephemeral (Terraform 1.10+)

Only valid in child module output blocks, not the root module. Prevents the output's value from being written to state or plan files, so it can pass ephemeral data (like a generated token) from a child module up to a caller without persisting it.

### depends_on

An explicit dependency for the output. Normally unnecessary, Terraform infers dependencies from what the `value` expression references. Only add `depends_on` when the real dependency is a side effect not visible in the expression itself, for example an IAM policy attachment that isn't referenced by attribute. Comment every explicit `depends_on` to explain why it's there.


```hcl
output "instance_ip_addr" {
  value       = aws_instance.server.private_ip
  depends_on = [
    # Unreachable unless this rule exists first
    aws_security_group_rule.local_access,
  ]
}
```

### precondition

Validates the output's value before Terraform exposes it or stores it in state. Structurally identical to a variable's `validation` block, both require `condition` and `error_message`.


```hcl
output "instance_public_ip" {
  value = aws_instance.web.public_ip
  precondition {
    condition     = length([for r in aws_security_group.web.ingress : r if r.to_port == 80]) > 0
    error_message = "Security group must allow HTTP ingress."
  }
}
```

The difference between the two is direction: `validation` checks an input before Terraform uses it, `precondition` checks a computed value before Terraform exposes it.

## Common exam traps in this section

| Trap                           | Why it's tricky                                                                                                       |
| ------------------------------ | --------------------------------------------------------------------------------------------------------------------- |
| `-var` vs `.tfvars` precedence | CLI flags win, people assume files are authoritative since they feel more "official"                                  |
| `sensitive` and `-json`        | `sensitive` only hides output in default CLI display, not `-json`, `-raw`, or state                                   |
| `nullable = false`             | Blocks `null` at the top level of the variable, not just nested elements, and there's no fallback to a prior value    |
| Reserved variable names        | `count`, `for_each`, `lifecycle`, `depends_on`, `locals`, `source`, `version`, `providers` all fail outright          |
| `depends_on` on outputs        | Usually unnecessary since Terraform infers it from the expression, only needed for invisible side-effect dependencies |

## Quick reference

| Block      | Required arg            | Key optional args                                                                                           |
| ---------- | ----------------------- | ----------------------------------------------------------------------------------------------------------- |
| `variable` | none (type recommended) | `type`, `default`, `description`, `validation`, `sensitive`, `nullable`, `ephemeral`, `const`, `deprecated` |
| `output`   | `value`                 | `description`, `sensitive`, `ephemeral`, `depends_on`, `precondition`                                       |