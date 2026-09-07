
---

### What providers actually are

Terraform itself has no knowledge of AWS, Azure, or any cloud. Providers are plugins that wrap each service's API and translate your HCL into actual API calls. They are distributed separately from Terraform with their own versioning and release cadence. They live on the Terraform Registry at `registry.terraform.io`.

Each provider adds a set of resource types and data sources that Terraform can manage. Without providers, Terraform cannot manage any infrastructure at all.

---

### The five provider tiers

|Tier|Who maintains it|Trust level|
|---|---|---|
|Official|HashiCorp|Highest|
|Partner Premier|Qualified technology partners, vetted by HashiCorp|High|
|Partner|Third-party companies against their own APIs|High|
|Community|Individual or group maintainers, no HashiCorp vetting|Lower|
|Archived|Official or Partner providers no longer maintained|Treat as red flag|

Tier affects how much you scrutinise a provider before production use. It does not change how you declare or use it in HCL.

---

### Declaring providers -- two blocks, two jobs

```hcl
# required_providers -- what you need and what version range is acceptable
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# provider block -- how to configure it (credentials, region etc.)
provider "aws" {
  region = "eu-west-1"
}
```

The `version` argument inside the `provider` block itself is **deprecated**. Never put version constraints there. They always go in `required_providers`.

If you omit the `provider` block entirely for a provider that needs no configuration (like `random`), Terraform assumes an empty default. If the provider requires arguments and you omit them, Terraform throws an error.

---

### Source addresses

Format is `hostname/namespace/type`. The hostname defaults to `registry.terraform.io` and is usually omitted. You only write the full address when using a private or mirror registry.

```
registry.terraform.io/hashicorp/aws
         |                |       |
      hostname         namespace  type
    (optional)
```

If you omit `source` entirely, Terraform assumes `registry.terraform.io/hashicorp/<local_name>`. This works but is not recommended in modern configs.

For in-house private providers, you can use an arbitrary internal hostname even if it does not resolve in DNS:

```hcl
source = "terraform.example.com/examplecorp/ourcloud"
```

Terraform uses this as a unique identifier pointing to a local filesystem mirror directory, not an actual HTTP address.

---

### The built-in provider

One provider is baked directly into Terraform itself and does not need to be downloaded. It enables the `terraform_remote_state` data source. Its source address is `terraform.io/builtin/terraform`. You do not need to declare it in `required_providers` to use it, though you will sometimes see it appear in error messages.

---

### Version constraint syntax -- know these cold

|Constraint|Meaning|
|---|---|
|`= 5.0.0`|Exactly this version|
|`!= 5.0.0`|Anything except this|
|`>= 5.0.0`|This version or higher|
|`~> 5.0`|`>= 5.0, < 6.0` -- minor and patch float, major locked|
|`~> 5.0.0`|`>= 5.0.0, < 5.1.0` -- patch only floats, minor locked|
|`>= 5.0, < 6.0`|Explicit range|

The rule for `~>`: more segments equals tighter constraint. The last number floats, everything to the left is locked.

**Best practice differs between root and reusable modules:**

- Root modules should cap the maximum version with `~>` to prevent accidental breaking upgrades
- Reusable modules should only specify a minimum with `>=` and let the root module manage the ceiling. Using `~>` in a reusable module forces all callers to upgrade simultaneously when the ceiling conflicts, which causes unnecessary pain

---

### Local name conflicts

If two providers share the same type name, use compound local names with a dash:

```hcl
terraform {
  required_providers {
    hashicorp-http = {
      source  = "hashicorp/http"
      version = "~> 2.0"
    }
    mycorp-http = {
      source  = "mycorp/http"
      version = "~> 1.0"
    }
  }
}

provider "mycorp-http" { }

data "http" "example" {
  provider = hashicorp-http
}
```

When you use a non-preferred local name, you must specify the `provider` meta-argument on every affected resource since Terraform cannot guess it from the resource type prefix.

---

### terraform init -- what it actually does

```bash
terraform init
```

- Downloads providers declared in `required_providers` into `.terraform/providers/`
- Creates `.terraform.lock.hcl` on first run
- On subsequent runs, installs the exact version recorded in the lock file, not the latest
- Does not re-download if the provider is already present and matches the lock file

```bash
terraform init -upgrade
```

- Ignores existing lock file selections
- Re-resolves within your version constraints
- Picks the newest allowed version
- Rewrites the lock file with the new version and new hashes
- Commit the updated lock file after reviewing the diff in git

---

### The lock file -- .terraform.lock.hcl

```hcl
provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.82.0"     # exact version selected
  constraints = "~> 5.0"     # informational only, not used for decisions
  hashes = [
    "h1:xRCd...",            # hash of extracted file contents, cross-platform
    "zh:0843...",            # hash of the zip archive as downloaded
  ]
}
```

**It only locks providers, not modules.** Module versions are not recorded here. To pin a module version you must use an exact version constraint in the module block itself:

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"   # only way to pin a module
}
```

**Always commit it to git.** It guarantees every teammate and every CI run gets the exact same provider binary you tested with.

**Never edit it by hand.** Let `terraform init` and `terraform init -upgrade` manage it.

**Lock file entry is removed automatically** when you remove the last resource using a provider from both your config and state. If you re-add the provider later it is treated as brand new with no memory of the previous version.

**`constraints` is informational only.** Terraform records it to help humans understand how the version was selected, but does not use it when making install decisions.

---

### The trust and verification system

The model is called "trust on first use". When you add a provider for the first time, you verify it however you choose, and from that point Terraform trusts any future init that produces matching hashes.

When HashiCorp publishes a provider they sign the hash file with their private GPG key. When you run `terraform init`:

1. Terraform downloads the provider binary
2. Downloads HashiCorp's published hash file
3. Verifies the signature using HashiCorp's public key, which is baked into the Terraform binary itself -- no API call needed for the key
4. Hashes the downloaded binary locally
5. Compares against the verified hash file

If anything does not match, Terraform refuses to install and throws an error. On subsequent inits, the lock file hashes are used for verification without hitting the Registry again.

**Critical distinction:** checksums verify integrity -- the file was not tampered with in transit. They do not verify intent. A community provider that passes checksum verification could still be abandoned or malicious. The hash confirms the binary matches what the maintainer published, nothing more.

**`h1:` vs `zh:` prefixes:**

- `zh:` (zip hash) -- SHA256 of the zip archive as downloaded. Legacy scheme, used for registry installs
- `h1:` (hash scheme 1) -- SHA256 of the extracted file contents. Preferred scheme, works across zip files, unpacked directories, and recompressed archives

Terraform stores both so the lock file works across your whole team regardless of OS. As you use a config on new platforms, Terraform opportunistically adds `h1:` hashes for those platforms.

**Mirror installs break cross-platform hashes.** If you install from a filesystem mirror instead of the Registry, Terraform can only verify the hash for your current platform. Fix it by pre-populating hashes for all platforms:

```bash
terraform providers lock \
  -platform=linux_amd64 \
  -platform=windows_amd64 \
  -platform=darwin_amd64
```

---

### Multiple provider configurations -- aliases

Without an alias you get one configuration per provider type. With aliases you can have multiple:

```hcl
provider "aws" {
  region = "eu-west-1"    # default, used automatically by all aws_ resources
}

provider "aws" {
  alias  = "us_east"
  region = "us-east-1"   # aliased, must be explicitly selected
}

resource "aws_s3_bucket" "us_bucket" {
  provider = aws.us_east
}
```

**Provider without a default -- implied empty config.** If every `provider` block uses an alias and none is left unaliased, Terraform creates an implied empty default. Resources that do not specify a `provider` argument use that empty default. If the provider requires arguments like a region, those resources will error:

```hcl
provider "aws" { alias = "east" ... }
provider "aws" { alias = "west" ... }

resource "aws_s3_bucket" "example" {
  # no provider arg = implied empty default = will error if aws requires config
}
```

---

### Provider configuration with variables

You can use `var.` and `local.` values inside a provider block. You cannot reference computed resource attributes because Terraform needs provider config resolved before running any resources:

```hcl
provider "aws" {
  region = var.aws_region       # fine
  # region = aws_instance.web.availability_zone  # not allowed
}
```

---

### Provider inheritance in modules

Default providers flow down from root to child automatically. Aliased providers never flow down automatically -- they must be wired explicitly.

**Child declares the need:**

```hcl
# ./modules/vpc/main.tf
terraform {
  required_providers {
    aws = {
      source               = "hashicorp/aws"
      version              = ">= 5.0"
      configuration_aliases = [aws.us_east]
    }
  }
}
```

**Root fulfills it:**

```hcl
module "vpc_east" {
  source = "./modules/vpc"

  providers = {
    aws.us_east = aws.us_east
  }
}
```

Child modules do not inherit provider source or version requirements -- you must declare those explicitly inside the child module even when the configuration itself flows down from root.

If the root calls the module without a `providers` argument, the child gets the default provider only. If the child declared `configuration_aliases` and the root does not pass it, Terraform throws an error.

One line summary: **child declares the need, root fulfills it.** Same pattern as dependency injection.

---

### Plugin cache

To avoid re-downloading the same provider binary across multiple projects, enable a shared cache in your CLI config file:

Windows: `%APPDATA%\terraform.rc` Linux/Mac: `~/.terraformrc`

```hcl
plugin_cache_dir = "$HOME/.terraform.d/plugin-cache"
```

Once set, a second project using the same provider version links to the cached binary instead of downloading it again.

---

### What to gitignore

```
.terraform/              # provider binaries, regenerable from lock file
*.tfvars                 # if they contain secrets
terraform.tfstate        # state file, contains plaintext secrets
terraform.tfstate.backup
```

Commit: `.terraform.lock.hcl` and all `.tf` files.

---

### Quick reference -- commands for this section

| Command                            | What it does                                                |
| ---------------------------------- | ----------------------------------------------------------- |
| `terraform init`                   | Downloads providers, creates or respects lock file          |
| `terraform init -upgrade`          | Re-resolves providers, updates lock file                    |
| `terraform providers`              | Lists providers required by current config                  |
| `terraform providers lock`         | Pre-populates hashes for specific platforms                 |
| `terraform providers mirror <dir>` | Downloads providers to a local directory for air-gapped use |