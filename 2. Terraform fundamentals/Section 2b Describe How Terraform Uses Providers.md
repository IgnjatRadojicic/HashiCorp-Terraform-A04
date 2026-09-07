#### The architectual split

Terraform is logically divided into two seperate parts that run as a seperate process and communicate over an RPC interface.

Terraform Core is the terraform binary you installed. It is a statically compiled binary written in Go. It has no knowledge of any cloud provider. Its runtime responsabilities are:
- Reading and interpolating configuration files and modules
- Building the resource graph
- Managing resource state
- Executing the plan
- Communicating with plugins over RPC

**Terraform Plugins** are seperate executable binaries also written in Go. They run as a completel seperate process from Core. When you run terraform apply, Core and the provider are two different running processes talking to each other over RPC. Core says Create this resource with these attributes the plugin makes the actual API call returns the result, Core writes it to state.

The key mental model: **Core owns the thinking plugins own the doing.**

#### Why the split matters

Because plugins are separate versioned binaries, Core and providers can be developed and released completely independently. HashiCorp can release Terraform 1.9 without touching the AWS provider. AWS can release provider v5.82 without waiting for a new Terraform Core release. That is why providers have their own version numbers separate from Terraform itself.

It also means anyone can write a provider. The RPC interface is the contract -- if your binary speaks the protocol, Terraform Core will talk to it.

#### Two plugin types

**Provider plugins** are what you interact with day to day. Their responsabilities:
- Initialising any libraries used to make API Calls
- Authentificating with the infrastructure provider
- Defining managed resources and data sources
- Definining functions that enable computational logic in practioneer configs.

**Provisioner plugins** execute commands or scripts on a resource after creation or before destruction. For example running a shell script on a new EC2 instance to install software. HashiCorp actively discourages their use. The recommended alternatives are proper config management tools like Ansible, or cloud-native options like EC2 user data scripts. Some provisioners are built directly into Core rather than being separate plugins.

For the exam: know provisioners exist, know they run scripts post-creation or pre-destruction, know HashiCorp discourages them

****
#### The discovery and selection process on terraform init

When `terraform init` runs it searches for plugins in this order and applies this logic:

**If an acceptable version is already installed locally** -- Terraform uses the newest locally installed acceptable version, even if the Registry has something newer. It avoids unnecessary downloads.

**If nothing acceptable is installed locally and the provider is in the Registry** -- Terraform downloads the newest acceptable version and saves it under `.terraform/providers/`.

**If nothing acceptable is installed and the provider is not in the Registry** -- init fails completely. You must manually install it, typically via a filesystem mirror.

---

#### The -upgrade nuance

bash

```bash
terraform init -upgrade
```

This re-checks the Registry for newer acceptable versions. However there is a specific condition where it will NOT download a newer version: if an acceptable version of the provider exists anywhere other than the automatic downloads directory (`.terraform/providers/`), such as a manual install or a configured mirror, Terraform will not override it.

It only upgrades providers whose only acceptable versions are in the automatic downloads directory. This is a tested exam nuance.

---

#### Common confusion -- init vs runtime responsibilities

The exam sometimes tries to trick you by mixing init-time actions with runtime responsibilities.

|Happens at init|Core runtime responsibility|
|---|---|
|Downloading providers|Reading config files|
|Writing the lock file|Building the resource graph|
|Selecting plugin versions|Managing state|
||Plan execution|
||RPC communication with plugins|

Downloading providers and writing the lock file are done by the `terraform init` CLI command, which is a Core command, but they are setup steps -- not what Core does when actually managing infrastructure. When a question asks about Core responsibilities, think runtime architecture, not CLI commands.