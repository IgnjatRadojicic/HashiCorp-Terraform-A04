#### Resource blocks -- the core of every Terraform config

Resources are the most important element in Terraform. Every resource block describes one or more infrastructure objects -- virtual networks, compute instances, DNS records, IAM policies, anything a provider exposes.

Syntax:

```hcl
resource "<TYPE>" "<LOCAL_NAME>" {
  <ARGUMENTS>
}
```


```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}
```

- `aws_instance` is the type -- the provider prefix (`aws`) tells Terraform which provider owns it
- `web` is the local name -- how you reference this resource elsewhere in the config as `aws_instance.web`
- Everything inside the block is arguments

---

#### Arguments vs attributes -- know the difference

**Arguments** are inputs you provide in the config. They configure the resource. Some are required, some are optional, and which ones exist depends entirely on the provider and resource type.

**Attributes** are values exposed by the resource. They include everything you set as arguments, plus computed values that only exist after the resource is created -- things like `id`, `public_ip`, `arn`. You reference attributes from other resources using dot notation:

hcl

```hcl
resource "aws_instance" "web" {
  ami                    = data.aws_ami.ubuntu.id
  instance_type          = "t3.micro"
  vpc_security_group_ids = [aws_security_group.sg_web.id]  # attribute reference
}
```

`aws_security_group.sg_web.id` is the `id` attribute of the security group resource. This reference also creates a dependency -- Terraform will create the security group before the instance.

---

#### How dependencies are created

Terraform builds its dependency graph from references. When resource A references an attribute of resource B, Terraform knows B must exist before A. This is an implicit dependency -- you don't have to declare it, it is derived from the reference itself.


```hcl
resource "aws_security_group" "sg_web" {
  name = "web-sg"
  # ...
}

resource "aws_instance" "web" {
  vpc_security_group_ids = [aws_security_group.sg_web.id]
  # aws_security_group.sg_web must be created first
  # Terraform figures this out automatically from the reference
}
```

For dependencies that exist but aren't expressed through a reference, use `depends_on` explicitly. Use it sparingly -- if you need it a lot, the config probably has implicit coupling that should be made explicit through references.

---

#### Meta-arguments -- special arguments available on every resource

These are not defined by the provider -- they are built into Terraform itself and work on any resource type:

|Meta-argument|What it does|
|---|---|
|`depends_on`|Explicit dependency when no reference exists|
|`count`|Creates multiple instances of a resource using an integer|
|`for_each`|Creates multiple instances using a map or set|
|`provider`|Selects a non-default or aliased provider|
|`lifecycle`|Controls create/destroy behaviour|

The `lifecycle` block is worth knowing in detail:


```hcl
resource "aws_instance" "web" {
  ami           = var.ami
  instance_type = "t3.micro"

  lifecycle {
    create_before_destroy = true   # new resource created before old one is destroyed
    prevent_destroy       = true   # blocks any operation that would destroy this resource
    ignore_changes        = [tags] # ignores drift on specific attributes
  }
}
```

`prevent_destroy = true` does NOT prevent destruction if you remove the resource block entirely from config. It only blocks destruction through normal plan/apply when the resource still exists in config.

---

#### Multiple providers in one config

You declare all providers you need in `required_providers` and configure each with its own `provider` block:


```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  region = "eu-west-1"
}

provider "cloudflare" {
  api_token = var.cloudflare_token
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}

resource "cloudflare_record" "www" {
  zone_id = var.zone_id
  name    = "www"
  value   = aws_instance.web.public_ip  # cross-provider reference
  type    = "A"
}
```

The Cloudflare DNS record references the AWS instance's public IP. This creates a cross-provider dependency -- Terraform handles it the same way as same-provider dependencies through the DAG.

---

#### The provider meta-argument on resources

When you have aliased providers, use the `provider` meta-argument to select which configuration a resource uses:


```hcl
provider "aws" {
  region = "eu-west-1"
}

provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

resource "aws_s3_bucket" "eu" {
  bucket = "my-eu-bucket"
  # uses default provider automatically
}

resource "aws_s3_bucket" "us" {
  bucket   = "my-us-bucket"
  provider = aws.us_east   # explicitly selects the alias
}
```

---

#### Data sources alongside resources

Data sources read existing infrastructure without managing it. They use the `data` block and expose attributes just like resources do:


```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-22.04-amd64-server-*"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id   # reference data source attribute
  instance_type = "t3.micro"
}
```

In state, the mode field distinguishes them: `managed` for resource blocks, `data` for data sources. Data sources are never destroyed by Terraform.

A data source is a read-only API call that fetches an existing resource's attributes at plan time -- Terraform never creates or destroys it, it simply reads it and lets you reference its attributes like any other object.

---

#### The -replace flag -- replacing a specific resource

When you need to force Terraform to destroy and recreate a specific resource without changing the config and without running a full destroy, use the `-replace` flag:

bash

```bash
terraform plan -replace="aws_instance.web"    # preview the replacement
terraform apply -replace="aws_instance.web"   # execute it
```

This is the current recommended approach. The old `terraform taint` command achieved the same result but is now deprecated. Use `-replace` instead.

Use cases: a resource is in a broken state, you updated a provisioning script and need a fresh instance, or a cloud provider had an incident affecting a specific resource.

---

#### Quick reference -- resource-related commands

| Command                                | What it does                                              |
| -------------------------------------- | --------------------------------------------------------- |
| `terraform plan -replace="<address>"`  | Preview forced replacement of a specific resource         |
| `terraform apply -replace="<address>"` | Force destroy and recreate a specific resource            |
| `terraform state list`                 | List all resource addresses in state                      |
| `terraform show`                       | Human-readable view of all resources and their attributes |
| `terraform state show <address>`       | Detailed view of one specific resource                    |