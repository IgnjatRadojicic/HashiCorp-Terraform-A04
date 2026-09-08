
## What this section covers

Terraform offers four distinct ways to validate configuration, each running at a different point in the workflow and each with different consequences on failure. Choosing the right one depends on two questions: what phase should catch the problem, and should a failure block the operation or just warn.

## Input variable validation

Runs immediately, before Terraform generates a plan. Verifies that a variable's value meets requirements beyond its type constraint, format, range, naming convention, whatever the module author needs to enforce.

```hcl
variable "image_id" {
  type        = string
  description = "The id of the machine image (AMI) to use for the server."

  validation {
    condition     = length(var.image_id) > 4 && substr(var.image_id, 0, 4) == "ami-"
    error_message = "The image_id value must be a valid AMI id, starting with \"ami-\"."
  }
}
```

If `condition` evaluates to `false`, Terraform errors immediately with `error_message` and stops before any plan is built. This catches bad input earlier and with a clearer message than letting the provider API reject it later. Minimum version: 0.13.0.

## Preconditions

Attached inside a resource's, data source's, or output's `lifecycle` block. Verify an assumption before Terraform tries to create the resource, read the data source, or expose the output.

```hcl
resource "aws_instance" "example" {
  instance_type = "t3.micro"
  ami           = data.aws_ami.example.id

  lifecycle {
    precondition {
      condition     = data.aws_ami.example.architecture == "x86_64"
      error_message = "The selected AMI must be for the x86_64 architecture."
    }
  }
}
```

Preconditions evaluate while Terraform builds its plan, and they take precedence over any argument error the provider itself would raise for the same misconfiguration. That means if a precondition would catch a problem, Terraform shows the precondition's (usually clearer) `error_message` instead of letting the provider fail first.

Outputs can also carry a precondition:
```hcl
output "instance_public_ip" {
  value = aws_instance.web.public_ip

  precondition {
    condition     = length([for rule in aws_security_group.web.ingress : rule if rule.to_port == 80 || rule.to_port == 443]) > 0
    error_message = "Security group must allow HTTP (port 80) or HTTPS (port 443) traffic."
  }
}
```

Minimum version: 1.2.0.

## Postconditions

Also attached inside a `lifecycle` block, but they check the result after Terraform plans and applies a resource, or after reading a data source. Use postconditions for guarantees you need the resulting infrastructure to hold, not assumptions about the input.

```hcl
data "aws_ami" "example" {
  id = var.aws_ami_id

  lifecycle {
    postcondition {
      condition     = self.tags["Component"] == "nomad-server"
      error_message = "tags[\"Component\"] must be \"nomad-server\"."
    }
  }
}
```

`self` refers to the resource or data source the `lifecycle` block is attached to, giving access to its own resulting attributes without spelling out the full resource address. If a postcondition fails, Terraform halts and prevents downstream actions that depend on that resource or data source, but it does not undo anything Terraform already did.

Minimum version: 1.2.0.

### Choosing between a precondition and a postcondition

The same rule can often be expressed either way, so the deciding factor is what you're actually verifying and when it's knowable. Use a precondition for an assumption you want confirmed before Terraform acts, an AMI's architecture, an input's shape. Use a postcondition for a guarantee about the result Terraform actually produced, a resource landed in a subnet with a private DNS record, an AMI has a specific tag.

Practical considerations when deciding: if a resource has many dependencies, one postcondition on the resource itself is often more pragmatic than a precondition on every individual dependency. And if a postcondition lives in a different module than a related precondition, keeping both can be worthwhile, since each module then verifies its own assumptions independently as the two evolve separately.

## Check blocks

A standalone block, not attached to any specific resource's `lifecycle`. Runs as the final step of a plan or apply, after everything else has been planned or provisioned.

```hcl
check "health_check" {
  data "http" "terraform_io" {
    url = "https://www.terraform.io"
  }

  assert {
    condition     = data.http.terraform_io.status_code == 200
    error_message = "${data.http.terraform_io.url} returned an unhealthy status code"
  }
}
```

The defining feature: a failed check produces a warning, not an error, and the operation continues regardless. This makes check blocks the right tool for validating something about your infrastructure as a whole without risking a blocked deployment over it, and for continuous validation in HCP Terraform, where health checks re-run check blocks, preconditions, and postconditions on a schedule to catch drift or external changes.

Minimum version: 1.5.0.

## Order of validation

Terraform validates as early as it practically can, given what information is available at each stage:

1. Input variable validations run immediately, before any plan exists.
2. Preconditions run after Terraform builds a plan, but before it creates the resource, reads the data source, or exposes the output.
3. Postconditions run after planning and applying changes.
4. Check blocks run last, at the end of plan and apply, and on every scheduled health assessment in HCP Terraform.

The exact timing of preconditions, postconditions, and checks relative to apply can shift depending on whether the relevant value is known during planning or only becomes known after apply. If Terraform already has the value (an image ID available before apply), it validates during planning. If the value only exists after apply (like an AWS-assigned root volume ID), the check is deferred until then.

## Error messages

Every validation type requires `error_message`, and it must be present regardless of which type you're using. The expression can be any string-producing expression, literals, heredocs, template expressions, or the `format()` function for interpolating structured values into the message. Multi-line messages are supported, and Terraform does not wrap lines that start with leading whitespace.

## Common exam traps in this section

|Trap|Why it's tricky|
|---|---|
|Which validation type blocks vs warns|check blocks are the only one that doesn't halt the operation on failure|
|Minimum version per type|Three different version gates: 0.13 for variable validation, 1.2 for pre/postconditions, 1.5 for check blocks|
|What `self` refers to|Not documented as a standalone concept, only shown in examples, easy to miss its meaning|
|Precondition vs provider error precedence|Preconditions win, so the clearer custom message shows instead of the raw provider error|
|Precondition timing vs postcondition timing|Precondition checks an assumption before creation, postcondition checks a guarantee after|

## Quick reference

| Validation type       | Attached to                                      | Runs                                                  | On failure                          |
| --------------------- | ------------------------------------------------ | ----------------------------------------------------- | ----------------------------------- |
| Variable `validation` | `variable` block                                 | Before plan                                           | Halts                               |
| `precondition`        | `lifecycle` block on resource/data source/output | Before creation, during plan                          | Halts                               |
| `postcondition`       | `lifecycle` block on resource/data source        | After plan and apply                                  | Halts, no rollback of prior actions |
| `check`               | Standalone `check` block                         | End of plan/apply, and on HCP Terraform health checks | Warns only, continues               |