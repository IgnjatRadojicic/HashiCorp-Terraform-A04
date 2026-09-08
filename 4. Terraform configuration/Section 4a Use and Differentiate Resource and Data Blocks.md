

---

A resource is any infrastructure object you want Terraform to create and manage. Virtual networks, compute instances, DNS records, and IAM policies are all examples. The kinds of resources available depend on which providers you have installed.

hcl

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}
```

When you apply a configuration, Terraform performs the following operations on resources:

- Creates resources in the configuration that do not yet exist as real infrastructure objects
- Destroys resources that exist in state but no longer exist in the configuration
- Updates resources in place if their arguments have changed
- Destroys and recreates resources whose arguments have changed but cannot be updated in place due to remote API limitations
- Updates the state file so configuration, real infrastructure, and state all match

---

#### Data sources

A data source is a read-only API call that fetches an existing resource's attributes at plan time. Terraform never creates or destroys it. It simply reads from the provider and lets you reference the returned attributes anywhere in your configuration.


```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-22.04-amd64-server-*"]
  }
}
```

Each data block is associated with a single data source. Terraform can only perform read operations on data sources. The combination of type and label within a data block must be unique across your configuration.

---

#### When Terraform reads data sources

Terraform attempts to read data sources during the planning phase. It defers reading to the apply phase in three situations. First, when arguments in the data block refer to computed values Terraform cannot predict during plan. Second, when the data block depends on a resource configured to change in the current plan. Third, when custom conditions in the data block depend on a changing resource.

When a data source is deferred, all its interpolated attributes show as `(known after apply)` in the plan output. Any resources that reference those deferred values cannot be provisioned until apply either.

When arguments refer to non-computed values, Terraform reads the data source during the refresh phase before planning. This ensures fetched data is available during planning and the diff shows real values.

---

#### Specialized local-only data sources

Some data sources generate data purely within the Terraform process during the operation itself. They make no outbound API calls and have no external dependencies. Terraform recalculates them fresh on every plan.

Examples:

- `data "template_file"` renders a template string locally
- `data "local_file"` reads a file from the local filesystem
- `data "aws_iam_policy_document"` constructs a JSON IAM policy document locally

The data they produce exists only while Terraform is running. They require installing the relevant provider but never make an outbound call to that provider's cloud service.

---

#### Referencing data source output


```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
}
```

The reference syntax is always `data.<TYPE>.<LABEL>.<ATTRIBUTE>`. The `data.` prefix distinguishes a data source reference from a managed resource reference which carries no prefix.

---

#### Data sources with meta-arguments

Data blocks support the same meta-arguments as resource blocks.

`depends_on` defers the data source query until after the specified dependency completes. Use it when a dependency exists but is not expressed through a direct reference.

`count` and `for_each` create multiple instances of a data source. Reference them with `data.<TYPE>.<LABEL>[<KEY>]`.

`provider` selects an aliased provider configuration for the data source query.

`lifecycle` with `postcondition` validates assumptions about the data returned before Terraform uses it.

