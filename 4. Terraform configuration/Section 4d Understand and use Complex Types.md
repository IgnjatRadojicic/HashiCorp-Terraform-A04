
### What this section covers

A complex type groups multiple values into one valie. Terraform splits complex types into two categories. Collection types group values of the same type. Structural types group values of different types under a fixed schema.

### Collection types

All three collection types require an element type as an arguement to their constructor.
Every element inside a given collection must be that same type

#### list(...)

An ordered sequence of values, index by consecutive whole numbers starting at zero.

```hcl
variable "availability_zones" {
  type      = list(string)
  default   = ["us-west-1a", "us-west-1b"] 
}
```
The bare keyword `list` is shorthand for `list(any)`, kept for backward compatibility with older configurations. New code should always specify the element type explicitly.

#### map(...)
A collection of values identified by string keys rather than position.
```hcl
variable "instance_tags" {
  type = map(string)
  default = {
    Name        = "web-server"
    Environment = "production"
  }
}
```

Maps can be written with `{}`, and keys and values separated by `:` or `=`. Both of the following define the same map:

```hcl
{ "foo": "bar", "bar": "baz" }
{ foo = "bar", bar = "baz" }
```

Quote a key only when it starts with a number, contains spaces, or contains special characters. `terraform fmt` vertically aligns `=` signs but ignores `:` delimiters, so `=` is the more idiomatic style in new configuration.
### set(...)

An unordered collection of unique values with no positional index.

```hcl
variable "allowed_ports" {
  type    = set(number)
  default = [22, 80, 443]
}
```

**The for_each connection.** This is the single most exam-relevant fact tied to collection types: `for_each` on a resource or module block only accepts a map or a set of strings. It rejects a plain list outright. If your source data is a list, convert it first:

```hcl
resource "aws_subnet" "this" {
  for_each   = toset(var.subnet_cidrs)
  cidr_block = each.value
}
```
### Structural types

Structural types require a schema since the elements inside are allow to differ in type

### object(...)

A collection of named attributes, each with its own declared type.

```hcl
variable "server_config" {
  type = object({
    name = string
    age  = number
  })
}
```

A value matches an object type if it contains at least all the required keys with correctly typed values. Extra attributes beyond the schema are allowed, they are just silently discarded during conversion. This means a `map -> object -> map` round trip can lose data, and passing an entire resource reference (with a dozen attributes) into a narrower object type constraint works fine, Terraform just keeps what the schema asks for and drops the rest.

### tuple(...)

A fixed-length, ordered sequence where each position has its own declared type.

```hcl
variable "server_row" {
  type = tuple([string, number, bool])
}
# Matches a value like: ["a", 15, true]
```

Unlike list-to-tuple-friendly conversions elsewhere, a list can only convert to a tuple type if it has exactly the number of elements the tuple requires, no more, no fewer.

## Optional object attributes

By default, Terraform errors if a required object attribute is missing. Marking an attribute `optional` changes that behavior.

```hcl
variable "with_optional_attribute" {
  type = object({
    a = string                # required
    b = optional(string)      # optional, defaults to null
    c = optional(number, 127) # optional, defaults to 127
  })
}
```

`optional()` takes one required argument, the attribute's type, and one optional argument, its default value. Without an explicit default, Terraform uses `null` of the appropriate type.

**Guarantee:** an optional attribute with a non-null default is never `null` inside the receiving module. Terraform substitutes the default both when the caller omits the attribute entirely and when the caller explicitly sets it to `null`. This removes the need for extra null-checks downstream.

**Nested defaults apply top-down.** When an object type has optional attributes at multiple nesting levels, Terraform applies the outer object's default first, then processes any nested optional defaults within that resulting value.

```hcl
variable "buckets" {
  type = list(object({
    name    = string
    enabled = optional(bool, true)
    website = optional(object({
      index_document = optional(string, "index.html")
      error_document = optional(string, "error.html")
    }), {})
  }))
}
```

If a caller omits `website` entirely, Terraform fills in the outer default `{}` first, then applies `index_document` and `error_document` defaults within that empty object, producing a fully populated `website` object.

**Conditionally leaving an attribute unset.** Use a conditional expression with `null` as one arm to dynamically decide whether an optional attribute gets its default or an explicit override:


```hcl
website = {
  index_document = var.legacy_filenames ? "INDEX.HTM" : null
}
```

When the condition is `false`, the result is `null`, which triggers the attribute's own default rather than leaving it unset in some undefined way.

## Type conversion between similar kinds

Terraform automatically converts between "similar" complex types wherever possible, which is why most documentation glosses over the list-vs-tuple and map-vs-object distinction.

**Objects and maps are similar.** A map, or a larger object, converts to an object type if it has at least the object's required keys. Extra keys are discarded.

**Tuples and lists are similar.** A list converts to a tuple only with an exact element count match.

**Sets are almost similar to both.** Converting a list or tuple to a set discards duplicates and loses ordering. Converting a set back to a list or tuple produces an arbitrary order, except for a set of strings, which comes out in lexicographical (alphabetical) order specifically. That guarantee does not extend to other element types.

Terraform also converts elements recursively within a complex type. If a module expects `list(string)` and receives the tuple `["a", 15, true]`, Terraform converts it element-by-element to `["a", "15", "true"]`. Conversion fails only when an element is fundamentally incompatible, for example a tuple cannot convert to a plain `string`, so an object like `{name = ["Kristy", "Claudia"], age = 12}` cannot satisfy a `map(string)` constraint.

## The any type constraint

`any` is a placeholder, not a real type. When Terraform encounters `any` in a type constraint, it inspects the actual value provided and infers the single concrete type that would make the constraint valid, resolved fresh at each call site.

```hcl
variable "settings" {
  type = any
}

resource "aws_s3_object" "example" {
  content = jsonencode(var.settings)
}
```

This is appropriate only when the module passes the value through untouched, without inspecting its structure or expecting a specific type internally. If the module accesses attributes, indexes into it, or treats it as a string or number anywhere, `any` is the wrong choice, write the exact type instead.

**How inference works with collections.** For `list(any)`, Terraform looks at the given value and finds a single type that fits every element:

- `["a", "b", "c"]` is a tuple of three strings. Tuple-to-list conversion applies, all elements are strings, so `any` resolves to `string`, giving `list(string)`.
- `["a", 1, "b"]` still resolves to `list(string)`, because `1` converts to `"1"` under primitive conversion rules. Result: `["a", "1", "b"]`.
- `["a", [], "b"]` fails outright. A string and an empty tuple share no common convertible type, so Terraform rejects the value.

The same inference principle applies to `map(any)` and `set(any)`.

## Common exam traps in this section

|Trap|Why it's tricky|
|---|---|
|`for_each` requires map or set of strings|People know `for_each` exists but forget it rejects a plain list without `toset()`|
|Extra object attributes are discarded, not rejected|Comes from stricter statically typed languages where this would be a compile error|
|List-to-tuple needs an exact count|Easy to assume it behaves like the more forgiving map-to-object conversion|
|Set-to-list order is only defined for strings|Assuming ordering guarantees extend to numbers or other element types|
|Optional attribute defaults apply top-down|Assuming nested defaults resolve before the outer default, or all at once|
|`any` resolves per call site, not permanently|Assuming a variable typed `any` stays untyped across every use forever|

## Quick reference

|Type|Category|Ordered?|Mixed types allowed?|Shorthand|
|---|---|---|---|---|
|`list(...)`|Collection|Yes|No|`list` = `list(any)`|
|`map(...)`|Collection|No (keyed)|No|`map` = `map(any)`|
|`set(...)`|Collection|No|No|none|
|`object(...)`|Structural|N/A (named)|Yes|none|
|`tuple(...)`|Structural|Yes|Yes|none|

| Conversion        | Rule                                                           |
| ----------------- | -------------------------------------------------------------- |
| map -> object     | Needs at least the required keys, extras discarded             |
| list -> tuple     | Needs exact element count                                      |
| list/tuple -> set | Duplicates discarded, order lost                               |
| set -> list/tuple | Arbitrary order, except strings which come out lexicographical |