
## What this section covers

A module is a collection of resources Terraform manages together. Every Terraform configuration has at least a root module, and the root module can call child modules. This section is specifically about the `source` argument, where module code can live and how Terraform retrieves it.

## The module block itself

```hcl
module "consul" {
  source  = "hashicorp/consul/aws"
  version = "0.1.0"

  servers = 3
}
```

Full set of built-in arguments a module block supports: `source` (required), `version`, `count`, `for_each`, `providers`, `depends_on`, plus whatever module-specific input variables the module itself defines. `count` and `for_each` are mutually exclusive, same as on resources.

## Source types

### Local paths

Prefix with `./` or `../`.

```hcl
module "consul" {
  source = "./consul"
}
```

Absolute paths (starting with `/` or a drive letter) also work, but are discouraged since they couple your config to one machine's filesystem layout. Local modules do not support the `version` argument, they're loaded from the same source tree as their caller and always share its version by definition.

### Terraform Registry

The primary way modules get shared. Syntax is `<NAMESPACE>/<NAME>/<PROVIDER>`.

hcl

```hcl
module "consul" {
  source  = "hashicorp/consul/aws"
  version = "0.1.0"
}
```

Private registries (HCP Terraform, Terraform Enterprise) use the same shape with a hostname prefix.

```hcl
module "vpc" {
  source  = "app.terraform.io/example-corp/vpc/aws"
  version = "0.9.3"
}
```

Registry modules follow semantic versioning, and you should always constrain the version to avoid pulling in unreviewed breaking changes on your next `init`.

### Git-based sources (GitHub, generic Git, BitBucket, Mercurial)

```hcl
module "consul" {
  source = "github.com/hashicorp/example"
}

module "vpc" {
  source = "git::https://example.com/network.git"
}
```

Terraform shells out to `git clone` (or `hg clone` for Mercurial), using your local system's Git config and credentials. SSH is the most common choice for private repos in automated pipelines, since it needs no interactive credential prompt. Two useful query parameters: `ref` (branch, tag, or commit SHA to check out) and `depth` (shallow clone depth, defaults to 1; using `depth` forces `ref` to be a named branch or tag, not a raw commit SHA, since shallow clones can't reliably resolve arbitrary commits).

### Archive and object storage sources (HTTP URL, S3, GCS)

For an HTTPS URL pointing at a recognized archive extension (`.zip`, `.tar.gz`, etc.), Terraform treats the archive contents directly as the module source, skipping any redirect logic. `s3::` and `gcs::` prefixes pull an archive object from a bucket, using each cloud's standard credential chain (environment variables, shared credentials files, instance profile, etc.).

### Subdirectories within a package

Use `//` to mark where the repository or archive root ends and the module's subdirectory begins.

```hcl
module "consul" {
  source = "hashicorp/consul/aws//modules/consul-cluster"
}

module "vpc" {
  source = "git::https://example.com/network.git//modules/vpc?ref=v1.2.0"
}
```

Terraform extracts the whole package to local disk but only reads the module from that subdirectory. Query parameters like `ref` go after the subdirectory segment.

## Finding modules on the registry

Every registry page has a search field matching against module name, provider, and description. You can filter to Partner modules specifically, which HashiCorp has reviewed for stability and compatibility, as opposed to unreviewed community modules.

Registry integration requires Terraform 0.10.6+, with full version constraint support arriving in 0.11.0. Before that, Terraform always pulled the latest version regardless of any constraint you wrote.

## terraform init and module changes

Running `terraform init` is what actually downloads and caches modules referenced by your configuration. You must re-run `init` any time you change a module's `source` or `version` argument, Terraform doesn't pick up source or version changes automatically on `plan` or `apply`.

## Common exam traps in this section

|Trap|Why it's tricky|
|---|---|
|Registry syntax vs local path|Forgetting the `./` prefix makes Terraform interpret a path as a registry address instead of local|
|`version` only works on registry sources|Assuming it applies universally; local paths inherit their caller's version by definition|
|`//` subdirectory marker|Easy to miss that this is required syntax, not just a stylistic slash|
|`depth` forcing `ref` to a named branch|Shallow clones can't reliably resolve a raw commit SHA, so `depth` + `ref` requires a branch or tag|
|Re-running `init` after source/version changes|Terraform won't silently pick these up on `plan`/`apply`|

## Quick reference

| Source type                      | Prefix / format                                        |
| -------------------------------- | ------------------------------------------------------ |
| Local path                       | `./path` or `../path`                                  |
| Public registry                  | `<NAMESPACE>/<NAME>/<PROVIDER>`                        |
| Private registry (HCP Terraform) | `app.terraform.io/<NAMESPACE>/<NAME>/<PROVIDER>`       |
| GitHub (HTTPS)                   | `github.com/<ORG>/<REPO>`                              |
| GitHub (SSH)                     | `git::github.com/<ORG>/<REPO>` or `git@github.com:...` |
| Generic Git                      | `git::<protocol>://<host>/<path>`                      |
| BitBucket                        | `bitbucket.org/<path>`                                 |
| Mercurial                        | `hg::<protocol>://<host>/<path>`                       |
| HTTP archive/vanity URL          | `https://...`                                          |
| S3                               | `s3::https://<bucket-url>`                             |
| GCS                              | `gcs::https://www.googleapis.com/storage/v1/<bu`       |