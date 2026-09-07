
#### What IaC actually is

Infrastructure as Code means your infrastructure is defined in file that live in version control, not in someones head or a sequence of console clicks. The file is the source of truth. Anyone can read it, review it, reproduce it.

The properties this gives you:
- Reporductability - spin up an identical environment anytime.
- Auditability - every change is a git commit with a diff and an author.
- Automation - no manual steps means no human error.
- Consistency - dev staging and prod can be identical except for a few variables.

What it does NOT give you: understanding of the underlying provider. You still need to know what a S3 bucket is what a VPC does what IAM Policies control. IAC automated the how not the what.

#### Declarative vs imperative

**Imperative** (bash, Ansible): you write the steps. "Call this API, then this one, then this one." The script doesn't know or care about current state run it twice and things break.

**Declarative** (Terraform): you describe the end state. "I want 2 EC2 instances of type t3.micro." Terraform figures out what API calls to make to get there. Run it twice, second run is a no-op because you're already there.

This is idempotency. Same input, same output, no matter how many times you run it. It's the foundational guarantee Terraform makes.

#### How Terraform manages multi-cloud
Terraform itself has no knowledge of AWS, Azure of GCP. It's a generic engine. Providers are plugins that wrap each services API and translate your HCL into the right calls. 

There are providers for:
- Cloud Platforms: Aws Azure GCP DigitalOcean
- DNS: Route53, Cloudflare
- Source control: GitHub, GitLab
- Kubernetes, DataDog, PaperDuty, Vault..
#### Terraform vs Everything else
**vs CloudFormation**w - CloudFormation is AWS-only and AWS-managed (no state file to worry about), with deeper integration for cutting-edge AWS features. Terraform is cloud-agnostic with a larger community. For the exam: Terraform is slightly behind on brand new AWS features because the provider has to catch up.

**vs Ansible/Chef/Puppet** - these are configuration management tools. They configure software on existing machines (install packages, manage files, run services). Terraform provisions the machines themselves. They're complementary, Terraform creates the VM, Ansible configures it. Terraform is not a replacement for config management.

**vs Pulumi** - same provider ecosystem, but you write real code (Python, TypeScript, Go) instead of HCL. Advantage: full language features - real loops, conditionals, type safety, reusable classes. Disadvantage: higher barrier for non-developers, HCL is more readable for most infra configs. Terraform dominates in market share; Pulumi wins on complex dynamic infrastructure.

#### The dependancy graph
Terraform doesn't run resources top to bottom. Before touching anything it builds a directed acyclic graph of all resources, resolves dependencies and runs independent resources in parallel.

Dependencies are created by references:

```terraform
resource "aws_subnet" "public" {
  vpc_id = aws_vpc.main.id   # this reference = dependency edge
}
```

Terraform sees this and knows: Create VPC first then subnet. Two subnets that both depend on the VPC but not on each other get created in paallel.

For cases where you have a dependency that isn't expressed through a reference use depends_on, but sparingly. Needing it alot singlas implicit coupling that should be made explicit.

#### Drift
Drift is when the real world diverges from your state file. Someone clicks something in the console an external process modifies a resource somethng gets deleted outside Terraform. Terraorm is push-based and doesn't watch for drift continuously. It detects drift when you run a plan. Commands:

```bash
terraform plan -refresh-only    # shows drift, plans nothing
terraform apply -refresh-only   # syncs state file to reality, no infra changes
```

Terraform detects drift but does NOT automatically fix it. It shows it as a planned change on the next apply. Some teams run Terraform on a CI schedule specifically to catch and remediate drift regularly.

#### What actually happens on apply

Terraform picks the least destructive action available:

|Plan symbol|Meaning|
|---|---|
|`+`|Create|
|`-`|Destroy|
|`~`|Update in place|
|`-/+`|Destroy and recreate|

Update in place is the default for most attribute changes. Destroy and recreate happens when the provider doesn't support live updates for that attribute the plan output flags this with `# forces replacement`. You always see this before anything is executed. Never a surprise.

Immutable infrastructure (always replace, never modify) is a deliberate architectural choice, not Terraform's default behavior.

### Standardize configurations

Terraform supports reusable configuration components called [modules](https://developer.hashicorp.com/terraform/language/modules) that define configurable collections of infrastructure, saving time and encouraging best practices. You can use publicly available modules from the Terraform Registry, or write your own.

---

#### Things the study guide doesn't emphasise

- Terraform is **not** idempotent because it's magic:  it's idempotent because of the plan phase. Plan diffs desired state against the state file and executes only the delta.
- The state file is written in plain JSON, contains everything the provider returned (including secrets), and should never be committed to git.
- HCL files can be split across as many `.tf` files as you want, Terraform merges them all at runtime. `main.tf`, `variables.tf`, `outputs.tf` is a convention, not a requirement.
- The `.terraform/` directory created by `terraform init` contains downloaded provider binaries. It's regenerable and should be gitignored.
- Terraform's Registry (`registry.terraform.io`) is where providers and modules live. Providers are published by HashiCorp, cloud vendors (official), and the community (verified/unverified).