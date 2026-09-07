
---

### The three steps

The core Terraform workflow has three steps regardless of whether you are working alone, in a team, or with HCP Terraform.

**Write** authors infrastructure as code in your editor. Store it in version control even as an individual.

**Plan** previews what Terraform will do before touching anything. The plan output shows exactly what will be created, changed, or destroyed.

**Apply** provisions the infrastructure. Terraform executes the plan and updates state.

This is a loop. Every change starts the cycle again from Write.

---

### Working as an individual

```bash
# start a new config
git init my-infra && cd my-infra

# write config
vim main.tf

# initialize -- download providers, set up backend
terraform init

# tight feedback loop while authoring
vim main.tf
terraform plan    # run plans repeatedly to flush out errors
vim main.tf

# commit when the plan looks right
git add main.tf
git commit -m "managing infrastructure as code"

# final review and apply
terraform apply

# push to remote
git push origin main
```

The individual workflow parallels writing application code. Edit, test, commit, push. `terraform apply` shows the plan one more time before executing so you get a final review even without a separate plan step.

---

### Working as a team

Teams introduce two problems the individual workflow does not have: colliding changes and sensitive credentials.

**Branching** keeps team members from stepping on each other. Everyone works on feature branches and resolves conflicts through the normal git merge workflow.

```bash
git checkout -b add-load-balancer
```

**Sensitive inputs** grow as the team grows. API keys, certificates, and other secrets required to run a plan become a security risk and operational burden when every team member arranges them locally. Teams typically move to a shared CI environment where Terraform operations run centrally.

**Plan review on pull requests** is where the team evaluates proposed changes. Some teams manually paste plan output into PRs. Others configure CI to post it automatically. Reviewers evaluate both the code change and the resulting plan before approving.

**The final apply plan** runs against the shared main branch and the latest state file after a PR is merged. This plan can differ from the PR speculative plan for two reasons. First, merge order means another PR may have merged first and changed something. Second, infrastructure drift means a manual change may have been made to real resources since the PR was reviewed. The team always reviews the final concrete plan before applying.

---

### The core workflow enhanced by HCP Terraform

HCP Terraform solves the collaboration friction that emerges at team and organisation scale.

**Write** becomes simpler because HCP Terraform provides a centralised secure store for input variables and state. Team members only need an HCP Terraform API key to run speculative plans against the latest state using all remotely stored variables. No local credential management is needed.

```hcl
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      tags = {
        layer = "networking"
      }
    }
  }
}
```

**Plan** becomes automatic. When a PR is opened, HCP Terraform runs a speculative plan and posts status updates directly to the PR. Reviewers see at a glance whether there are infrastructure changes. Clicking through shows the full plan detail in the HCP UI.

**Apply** becomes a team event. After merge, HCP presents the concrete plan in the UI for review and approval. Team members can discuss in comments before confirming. Once confirmed, HCP runs the apply remotely and streams progress live to anyone watching.

State is stored in HCP and never touches anyone's local machine. Locking happens automatically. No teammate ever needs to pull state, push state, or coordinate timing with anyone else.

---

### Key distinctions for the exam

A speculative plan is read only. It runs against a branch and is used for PR review. It is never applied directly.

A concrete plan runs against the merged main branch and the latest state file. It is the one that actually gets applied. Always review the concrete plan before confirming, not just the PR speculative plan.

**Individual vs team vs HCP workflow:**

|Concern|Individual|Team|HCP Terraform|
|---|---|---|---|
|State|Local file|Shared backend needed|Managed automatically|
|Credentials|Local|Burden on each member or CI|Stored centrally in HCP|
|Plan review|Personal judgement|PR comments, manual or CI|Automatic speculative plans on PRs|
|Apply execution|Local terminal|CI pipeline or local|Remote in HCP, streamed live|
|Locking|Not needed|DynamoDB or similar|Automatic|

---

### The workflow as a loop

```
write config
        |
terraform plan repeated until plan looks right
        |
commit and push to feature branch
        |
PR opened, speculative plan runs automatically
        |
team reviews code and plan output
        |
PR merged to main
        |
concrete plan runs against latest state
        |
team reviews and approves in HCP UI
        |
terraform apply executes remotely
        |
state updated in HCP
        |
next change starts the loop again
```