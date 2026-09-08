
## What this section covers

This section is about generating configuration dynamically rather than writing every value and block by hand. Two mechanisms do most of the work: for expressions, which transform and filter collections into new values, and dynamic blocks, which generate repeated nested blocks inside a resource. Built-in functions supply the transformations you call inside those expressions.

## For expressions

A for expression loops over a collection and produces a new value. This is distinct from `count` and `for_each` on a resource block, which loop to produce multiple whole resources. A for expression only ever produces a value, a list or a map, never a block.

### Transforming a list

```hcl
[for s in var.names : upper(s)]
```

### Filtering a list

Add a trailing `if` clause to include only elements that match a condition.

```hcl
[for s in var.names : s if s != ""]
```

### Combining transformation and filtering

```hcl
[for s in var.names : upper(s) if s != ""]
```

This uppercases every non-empty string and drops the empty ones in the same pass.

### Producing a map

Use `{}` and a `key => value` pair instead of a single expression.

```hcl
{for k, v in var.tags : k => upper(v)}
```

## Dynamic blocks

Some resource arguments are not plain values, they are nested blocks, for example `ingress` inside an `aws_security_group` resource. A for expression cannot generate a block, since it only produces values. `dynamic` exists specifically to solve this.

```hcl
resource "aws_security_group" "example" {
  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port = ingress.value.from_port
      to_port   = ingress.value.to_port
      protocol  = ingress.value.protocol
    }
  }
}
```

The label after `dynamic` (`"ingress"` here) becomes the iterator name available inside `content`. `ingress.value` refers to the current item from `for_each`, and `ingress.key` refers to its map key or list index, following the same semantics as `for_each` on a resource.

**Decision rule:** `for_each` on the resource block itself repeats the entire resource. `dynamic` inside a resource repeats one nested block within a single resource. A for expression repeats to build a value, never a block.

## Built-in functions

### Call syntax

A function call is a name followed by comma-separated arguments in parentheses.

```hcl
max(5, 12, 9)
```

Terraform does not support user-defined functions in the configuration language itself. If you need custom logic beyond what a chain of built-in functions can express, that logic belongs in a provider.

### Categories

Terraform's built-in functions span several categories: numeric, string, collection, encoding, filesystem, date and time, hash and crypto, IP network, and type conversion. The exam does not require memorizing every function, but a working knowledge of the common ones below is expected.

### Functions worth knowing cold

```hcl
lookup(var.map, "key", "default")   # get a map value, with a fallback if the key is missing
merge(map1, map2)                   # combine maps, later arguments win on key conflicts
concat(list1, list2)                # combine lists in order
element(var.list, index)            # get an item from a list by index
length(var.collection)              # count elements in any collection or string
coalesce(a, b, c)                   # first non-null argument
coalescelist(list1, list2)          # first non-empty list argument
try(expr1, expr2, ...)              # first expression that doesn't error
can(expr)                           # true/false, did the expression error
jsonencode(value)                   # HCL value -> JSON string
jsondecode(json_string)             # JSON string -> HCL value
templatefile(path, vars)            # render a template file with variables
toset(list) / tolist(set)           # explicit collection type conversion
```

### merge() conflict behavior

When keys collide across arguments, the later argument wins.

```hcl
merge({a = 1, b = 2}, {b = 3, c = 4})
# result: {a = 1, b = 3, c = 4}
```

### try() vs can()

Both handle expressions that might fail, but they return different things and are used differently.

`try()` returns a value: it evaluates each argument left to right and returns the first one that doesn't produce an error. Common with optional attributes that might not be present.

```hcl
value = try(var.config.optional_field, "default")
```

`can()` returns a boolean: `true` if the given expression evaluates without error, `false` if it doesn't. It never itself throws, which is exactly why it fits inside a `validation` block's `condition`, where you need a true/false test rather than a fallback value.

```hcl
validation {
  condition     = can(regex("^ami-", var.image_id))
  error_message = "Must be a valid AMI ID."
}
```

### jsonencode() vs jsondecode(), direction matters

`jsonencode(value)` takes an HCL value and turns it into a JSON-formatted string. Used when a resource argument expects a raw JSON string, IAM policy documents are the classic case.

```hcl
resource "aws_s3_object" "example" {
  content = jsonencode(var.settings)
}
```

`jsondecode(json_string)` does the reverse: takes a JSON string and turns it into a usable HCL value you can index into.

```hcl
locals {
  parsed = jsondecode(file("${path.module}/config.json"))
}
```

## Provider-defined functions

Providers can expose their own functions in addition to Terraform's built-ins. Calling one requires a `provider::<local-name>::` prefix, where `local-name` matches the entry declared in `required_providers`. This namespacing distinguishes provider functions from built-ins with the same or similar names.

```hcl
provider::terraform::encode_tfvars({
  example = "Hello!"
})
```

Functions defined by external providers are documented by those providers in the Terraform Registry, not in Terraform's own function documentation.

## terraform console

An interactive REPL for experimenting with expressions and functions without touching real infrastructure. Useful for checking exactly what a function returns before committing it to configuration.

```
$ terraform console
> max(5, 12, 9)
12
```

## Common exam traps in this section

|Trap|Why it's tricky|
|---|---|
|for expression vs dynamic block|Assuming a for expression can produce a nested block like `ingress`, it can only produce a value|
|try() vs can() return type|Assuming they're interchangeable, `try()` returns a value, `can()` returns a boolean|
|jsonencode() vs jsondecode() direction|Easy to swap which one goes HCL-to-JSON versus JSON-to-HCL|
|merge() conflict winner|Assuming the first map wins instead of the last|
|provider:: prefix requirement|Forgetting that provider-defined functions need explicit namespacing, unlike built-ins|

## Quick reference

|Construct|Produces|Use when|
|---|---|---|
|For expression|A value (list or map)|Transforming or filtering a collection into a new value|
|`dynamic` block|Repeated nested blocks|A resource argument is itself a block, not a value, and needs to repeat|
|`count` / `for_each` on resource|Multiple whole resources|You need N instances of an entire resource|

| Function       | Returns                  | Purpose                                    |
| -------------- | ------------------------ | ------------------------------------------ |
| `try()`        | First non-erroring value | Fallback chain                             |
| `can()`        | Boolean                  | Safe true/false test, used in `validation` |
| `jsonencode()` | JSON string              | HCL value to JSON                          |
| `jsondecode()` | HCL value                | JSON to HCL value                          |
| `merge()`      | Combined map             | Later argument wins on key conflict        |