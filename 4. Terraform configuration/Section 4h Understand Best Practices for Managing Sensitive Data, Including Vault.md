
## What this section covers

Terraform's state and plan files can contain sensitive values, credentials, tokens, initial passwords, by nature of tracking detailed resource attributes. This section covers the mechanisms available to hide those values from output, prevent them from being stored at all, and how the Vault provider fits into that picture.

## Two separate goals: hiding vs not storing

The first decision point is whether you want to hide a value from CLI and HCP Terraform UI display, or prevent Terraform from storing that value anywhere at all. These are different problems with different tools.

## sensitive: hides display, still stores the value

Add `sensitive = true` to a `variable` or `output` block to redact that value from CLI plan/apply output and the HCP Terraform UI.

```hcl
variable "database_password" {
  type      = string
  sensitive = true
}
```

Any expression that references a sensitive variable or output automatically becomes sensitive too, so it propagates through the config without extra tagging.

```hcl
resource "aws_db_instance" "main" {
  password = var.database_password
}
# password argument is redacted in plan/apply output as (sensitive value)
```

**The limit that matters most:** `sensitive` only affects display. Terraform still writes the real value into both state and plan files in plaintext. Anyone with access to those files can read it. And `terraform output -json` or `-raw` bypasses the CLI redaction entirely, printing the plaintext value. Treat `sensitive` as protection against accidental exposure in logs and terminal output, not as encryption or access control.

## ephemeral: prevents storage entirely

Available from Terraform 1.10+. An ephemeral value exists only at runtime and is never written to state or plan files. There are three ways to define one.

### ephemeral argument on variables and child module outputs

```hcl
variable "api_token" {
  type      = string
  sensitive = true
  ephemeral = true
}
```

You can only reference an ephemeral variable in other ephemeral contexts: the `locals` block, another ephemeral variable, a child module output with the `ephemeral` argument, a write-only resource argument, an `ephemeral` block, provider configuration, or provisioner/connection blocks. The root module cannot have an ephemeral output, only child modules can.

```hcl
provider "example" {
  api_token = var.api_token
}
```

### The ephemeral block

Declares a temporary ephemeral resource that exists only for the current run.

```hcl
ephemeral "random_password" "db_password" {
  length           = 16
  override_special = "!#$%&*()-_=+[]{}<>:?"
}
```

Reference it only in other ephemeral contexts, most commonly a write-only argument.

**Important consequence:** if you don't capture the value somewhere persistent, another resource, a proper secret store, it is simply gone after the run completes. Terraform doesn't keep a hidden backup, ephemeral means the value only ever existed during that one operation.

### Write-only arguments

Provider-defined arguments (typically ending in `_wo`) that accept a value during an operation without ever storing it in state or plan. This is provider specific AWS / Azure provide this.

```hcl
resource "aws_db_instance" "example" {
  password_wo         = ephemeral.random_password.db_password.result
  password_wo_version = 1
}
```

**Why the paired `_wo_version` exists:** Terraform normally detects a changed value by diffing the new value against what's recorded in state. A write-only value is never in state, so there's nothing to diff against. The version integer is the workaround, bump it whenever you want Terraform to treat the write-only value as changed and reapply it. Terraform compares the version number, not the underlying secret.

### Combining sensitive and ephemeral

For a variable or child module output you want both hidden from display and never stored, use both arguments together.

```hcl
variable "database_password" {
  type      = string
  sensitive = true
  ephemeral = true
}
```

## Vault provider

The Vault provider lets Terraform read from, write to, and configure HashiCorp Vault. It serves two distinct use cases with different risk profiles, worth keeping separate in your head.

### Use case 1: Configuring and populating Vault

Terraform acts as the administrator, writing secrets into Vault.

```hcl
resource "vault_generic_secret" "example" {
  path = "secret/foo"
  data_json = jsonencode({
    "foo"   = "bar",
    "pizza" = "cheese"
  })
}
```

Here, the risk is that Terraform has no mechanism to redact secrets provided via configuration, so anything written to Vault this way also lands in state and plan files in cleartext. Treat those files with the same care as any other secret store.

### Use case 2: Reading Vault credentials to authenticate other providers

Terraform reads a secret or dynamically-leased credential from Vault, then uses it to configure another provider (like AWS), so operators only need a Vault token rather than the underlying cloud credentials directly.

```hcl
provider "vault" {
  auth_login {
    path = "auth/approle/login"
    parameters = {
      role_id   = var.login_approle_role_id
      secret_id = var.login_approle_secret_id
    }
  }
}
```

The mitigation here: Terraform requests a short-lived child token from Vault (20-minute default TTL, configurable via `max_lease_ttl_seconds`), which limits how long any credentials leased under that token remain valid, since Vault can revoke them after expiry.

**The gap this doesn't cover:** the short TTL only helps with dynamically leased secrets that Vault can actually revoke. Static secrets read from Vault's generic or KV secret backend are not leases, Vault has nothing to revoke, so they persist in your state file in plaintext for as long as that state file exists, TTL notwithstanding.

### Vault authentication methods, at a glance

The provider supports numerous auth engines (userpass, AWS, TLS cert, GCP, Kerberos, Radius, OCI, OIDC, JWT, Azure, token file, and a generic path-based fallback), each with its own `auth_login_<method>` configuration block. Memorizing every argument for every method is not exam-relevant, the concept that matters is that credentials for the `vault` provider block itself are best supplied via environment variables rather than hardcoded in configuration, and that the provider issues itself a limited child token regardless of which method authenticated it (unless `skip_child_token` is explicitly, and inadvisably, set to true).

### Simplified

Vault is a key-value store, I can define a static one in the sense of putting my password into their locker. Or I can connect the specific provider resource to Vault which then leases the password to me and refreshes and resets the password in Azure/AWS.

## State security best practices

Beyond the mechanisms above, general hygiene for any state file that might contain sensitive data:

- Store state remotely rather than as a local plaintext file
- Encrypt state at rest (backend-dependent, HCP Terraform and the S3 backend with `encrypt` both support this)
- Use access controls to limit who can read the state
- Use audit logs to track state access over time

## Common exam traps in this section

|Trap|Why it's tricky|
|---|---|
|sensitive vs ephemeral|sensitive hides display only, ephemeral prevents storage, they solve different problems and can be combined|
|`-json`/`-raw` bypass sensitive|People assume sensitive is a hard block on ever seeing the value anywhere|
|Ephemeral values without a capture step|Assuming Terraform saves it somewhere automatically, it doesn't, unreferenced ephemeral values are simply lost|
|`_wo_version`'s purpose|Easy to assume it's a TTL or usage-count limit rather than a change-detection proxy|
|Vault TTL protection scope|Assuming the short-lived token protects all secrets equally, it only helps with revocable dynamic leases, not static KV secrets|

## Quick reference

|Mechanism|Hides from CLI/UI display|Omits from state/plan|
|---|---|---|
|`sensitive`|Yes|No|
|`ephemeral`|No (not its purpose)|Yes|
|`sensitive` + `ephemeral`|Yes|Yes|
|Write-only argument (`_wo`)|N/A (provider-defined arg)|Yes, paired with `_wo_version` for change detection|

| Vault use case                     | What's at risk                                                                                                |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| Populating Vault (writing secrets) | Written secrets land in state/plan in cleartext, no redaction mechanism exists                                |
| Reading Vault credentials          | Read secrets land in state/plan; short-lived child token mitigates dynamic leases only, not static KV secrets |