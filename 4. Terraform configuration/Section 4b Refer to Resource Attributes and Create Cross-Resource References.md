
#### Named value types and their prefixes

Terraform makes several categories of named values available as expressions:

```hcl
var.environment              # input variable
local.full_name              # local value
module.vpc.subnet_id         # child module output
data.aws_ami.ubuntu.id       # data source attribute
aws_instance.web.public_ip   # managed resource attribute
terraform.workspace          # current workspace name
path.module                  # filesystem path of current module
path.root                    # filesystem path of root module
path.cwd                     # filesystem path of original working directory
count.index                  # current index in a count loop
each.key / each.value        # current key and value in a for_each loop
self                         # used inside provisioner and connection blocks
```

These are not real objects. You cannot iterate over the parent or use square bracket notation to replace dot-separated paths. You use them exactly as written.

---

#### Resource attribute references

```hcl
resource "aws_instance" "example" {
  ami           = "ami-abc123"
  instance_type = "t2.micro"

  ebs_block_device {
    device_name = "sda2"
    volume_size = 16
  }
}
```


```hcl
aws_instance.example.ami                              # the ami argument value
aws_instance.example.id                               # computed attribute set by AWS
aws_instance.example.ebs_block_device[*].device_name  # all device names as a list
```

The splat expression `[*]` collects a specific attribute from all instances of a nested block into a list.

---

#### References on count and for_each resources

When a resource uses `count` it becomes a list of objects:

```hcl
aws_instance.web[*].id    # list of all instance IDs
aws_instance.web[0].id    # ID of first instance
aws_instance.web[2].id    # ID of third instance
```

When a resource uses `for_each` it becomes a map of objects:

```hcl
aws_instance.web["prod"].id              # ID of prod instance
[for v in aws_instance.web : v.id]      # list of all IDs via for expression
values(aws_instance.web)[*].id          # splat on for_each using values()
```

Splat expressions work directly on `count` resources because they produce a list. They do not work directly on `for_each` resources because those produce a map. Use `values()` to convert the map to a list first before applying a splat.

---

#### Sensitive resource attributes

Provider developers can mark certain resource attributes as sensitive. Terraform shows `(sensitive value)` in plan and apply output instead of the real value. Any value derived from a sensitive attribute is also treated as sensitive automatically.

If you reference a sensitive resource attribute in an output, Terraform requires you to mark the output as sensitive explicitly:

```hcl
output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true
}
```

Sensitive values are still stored in plaintext in the state file. Anyone with access to state can read them.

---

#### Unknown values

Some resource attribute values cannot be known during plan because the remote system generates them on creation. Auto-generated IDs, assigned IP addresses, and ARNs are common examples. These appear in plan output as `(known after apply)`.

Terraform handles unknown values automatically in expressions. An operation involving an unknown value produces another unknown value as its result. Three constraints apply:

`count` cannot be unknown because Terraform must evaluate it during plan to determine how many instances to create.

A data source that depends on an unknown value is deferred to the apply phase.

An output that references an unknown value produces an unknown output in the parent module.