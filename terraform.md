# Terraform — Complete Notes (Basics → Advanced + Interview Prep)

---

## Table of Contents
1. [Introduction](#1-introduction)
2. [Terraform vs Other Tools](#2-terraform-vs-other-tools)
3. [Terraform Architecture & Workflow](#3-terraform-architecture--workflow)
4. [Installation & Setup](#4-installation--setup)
5. [Core Concepts](#5-core-concepts)
6. [HCL Syntax Basics](#6-hcl-syntax-basics)
7. [Providers](#7-providers)
8. [Resources](#8-resources)
9. [Data Sources](#9-data-sources)
10. [Variables](#10-variables)
11. [Outputs](#11-outputs)
12. [State Management](#12-state-management)
13. [Remote Backends](#13-remote-backends)
14. [State Locking](#14-state-locking)
15. [Terraform Commands](#15-terraform-commands)
16. [Provisioners](#16-provisioners)
17. [Meta-Arguments (count, for_each, depends_on, lifecycle)](#17-meta-arguments)
18. [Modules](#18-modules)
19. [Workspaces](#19-workspaces)
20. [Expressions & Functions](#20-expressions--functions)
21. [Import & State Manipulation](#21-import--state-manipulation)
22. [Terraform Cloud / Enterprise](#22-terraform-cloud--enterprise)
23. [Security Best Practices](#23-security-best-practices)
24. [Terraform in CI/CD](#24-terraform-in-cicd)
25. [Troubleshooting](#25-troubleshooting)
26. [Best Practices Summary](#26-best-practices-summary)
27. [Cheat Sheet](#27-cheat-sheet)
28. [Interview Questions & Answers](#28-interview-questions--answers)

---

## 1. Introduction

**Terraform** is an open-source **Infrastructure as Code (IaC)** tool by HashiCorp that lets you define, provision, and manage cloud and on-prem infrastructure using a declarative configuration language called **HCL (HashiCorp Configuration Language)**.

Instead of manually clicking through a cloud console or writing imperative scripts, you describe the **desired end state** of your infrastructure, and Terraform figures out the steps needed to reach it — creating, updating, or destroying resources as required.

**Why Terraform?**
- **Declarative** — describe *what* you want, not *how* to get there
- **Provider-agnostic** — same workflow across AWS, Azure, GCP, Kubernetes, and 3000+ providers
- **Plan before apply** — preview exactly what will change before it happens
- **State tracking** — knows what it created and can detect drift
- **Version-controllable** — infrastructure defined as code, reviewable via Git/PRs
- **Reusable** — modules let you package and share infrastructure patterns

---

## 2. Terraform vs Other Tools

| Tool | Type | Approach | Scope |
|---|---|---|---|
| **Terraform** | Provisioning (IaC) | Declarative | Multi-cloud infrastructure |
| **CloudFormation** | Provisioning (IaC) | Declarative | AWS only |
| **Ansible** | Configuration management | Mostly procedural/imperative | Server/OS configuration, app deployment |
| **Pulumi** | Provisioning (IaC) | Declarative, but uses real programming languages | Multi-cloud infrastructure |
| **Chef/Puppet** | Configuration management | Declarative (agent-based) | Server configuration |

**Key distinction (common interview question):** Terraform provisions infrastructure (servers, networks, databases). Ansible configures what's running *on* that infrastructure (installing packages, managing config files). They're often used together — Terraform creates the VM, Ansible configures it.

---

## 3. Terraform Architecture & Workflow

```
   .tf config files
         │
         ▼
   ┌───────────┐   reads/writes   ┌─────────────┐
   │ Terraform  │ ───────────────▶ │ State file   │
   │   Core     │ ◀─────────────── │ (terraform.  │
   └─────┬─────┘                  │  tfstate)     │
         │                         └─────────────┘
         │ RPC
         ▼
   ┌───────────────┐
   │ Provider Plugin │  (e.g. AWS, Azure, GCP)
   └────────┬────────┘
            │ API calls
            ▼
   ┌───────────────┐
   │  Cloud / Infra  │
   └───────────────┘
```

**Core workflow (the famous 4 steps):**
1. **Write** — author `.tf` configuration files describing desired infrastructure
2. **Init** (`terraform init`) — downloads required providers/modules, sets up backend
3. **Plan** (`terraform plan`) — computes the diff between current state and desired config, shows what will change
4. **Apply** (`terraform apply`) — executes the plan, calling provider APIs to create/update/destroy resources, then updates the state file

**Terraform Core** parses configuration, builds a **resource dependency graph**, and determines the correct order of operations (creating dependent resources after their dependencies, destroying in reverse order). **Provider plugins** translate Terraform's generic resource operations into actual API calls for a specific platform.

---

## 4. Installation & Setup

```bash
# macOS
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Linux
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# Verify
terraform -version
```

Basic project structure:
```
project/
  main.tf          # primary resource definitions
  variables.tf      # input variable declarations
  outputs.tf         # output value declarations
  providers.tf        # provider configuration
  terraform.tfvars     # variable values (often gitignored if sensitive)
  versions.tf            # required_version & required_providers
```

---

## 5. Core Concepts

| Term | Definition |
|---|---|
| **Provider** | Plugin that lets Terraform interact with an API (AWS, Azure, GCP, etc.) |
| **Resource** | A single infrastructure object Terraform manages (e.g., an EC2 instance) |
| **Data Source** | A read-only query to fetch info about existing infrastructure not managed by this config |
| **State** | A file (`terraform.tfstate`) tracking what Terraform has created and its current attributes |
| **Module** | A reusable, self-contained package of `.tf` configuration |
| **Plan** | A preview of changes Terraform will make |
| **Backend** | Where the state file is stored (local, S3, Terraform Cloud, etc.) |
| **Workspace** | A named instance of state, allowing multiple environments from the same config |
| **Provisioner** | A way to run scripts on a resource after creation (last-resort tool) |

---

## 6. HCL Syntax Basics

```hcl
# Block syntax
<block_type> "<label1>" "<label2>" {
  key = value
}

# Example: a resource block
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "web-server"
  }
}
```

**Comments:** `#` or `//` for single-line, `/* */` for multi-line.

**Data types:** `string`, `number`, `bool`, `list`/`tuple`, `map`/`object`, `null`.

```hcl
locals {
  name      = "my-app"          # string
  count_val = 3                  # number
  enabled   = true                # bool
  azs       = ["us-east-1a", "us-east-1b"]   # list
  tags      = { Env = "prod", Team = "infra" } # map
}
```

---

## 7. Providers

Providers are plugins that translate HCL into API calls for a specific platform (AWS, Azure, GCP, Kubernetes, GitHub, Datadog, etc. — 3000+ providers in the registry).

```hcl
terraform {
  required_version = ">= 1.7.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region  = "us-east-1"
  profile = "default"
}

# Multiple configurations of the same provider (aliasing)
provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

resource "aws_instance" "west_server" {
  provider      = aws.west
  ami           = "ami-xxxx"
  instance_type = "t2.micro"
}
```

```bash
terraform init    # downloads provider plugins into .terraform/
```

---

## 8. Resources

A **resource** block describes one piece of infrastructure.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "web-server"
  }
}

# Referencing another resource's attribute
resource "aws_eip" "web_ip" {
  instance = aws_instance.web.id
}
```

**Resource addressing:** `<resource_type>.<resource_name>` (e.g., `aws_instance.web`) — used for references, `terraform state` commands, and `import`/`taint` targeting.

**CRUD lifecycle:** Terraform compares desired config to current state to decide whether to **create**, **update in-place**, **destroy and recreate**, or **leave unchanged** a resource. Some attribute changes force a full replace (e.g., changing an EC2 instance's AMI) — Terraform plan output marks these clearly (`-/+` vs `~`).

---

## 9. Data Sources

Used to **read** information about existing infrastructure (managed by Terraform or not) without managing its lifecycle.

```hcl
data "aws_ami" "latest_amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.latest_amazon_linux.id
  instance_type = "t2.micro"
}
```

**Resource vs Data Source (common interview question):** A `resource` block tells Terraform to **create and manage** something; a `data` block just **looks up/reads** existing information (e.g., an existing VPC ID, the latest AMI) and Terraform has no control over its lifecycle.

---

## 10. Variables

```hcl
# variables.tf
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "subnet_ids" {
  type = list(string)
}

variable "tags" {
  type    = map(string)
  default = {}
}

variable "db_password" {
  type      = string
  sensitive = true     # hides value in plan/apply output and logs
}
```

```hcl
# Using a variable
resource "aws_instance" "web" {
  instance_type = var.instance_type
}
```

**Ways to set variable values (in order of precedence, highest wins):**
1. `-var` CLI flag: `terraform apply -var="instance_type=t3.micro"`
2. `-var-file` CLI flag
3. `*.auto.tfvars` / `*.auto.tfvars.json` files (auto-loaded)
4. `terraform.tfvars` file (auto-loaded)
5. Environment variables: `TF_VAR_instance_type=t3.micro`
6. Default value in the `variable` block

```hcl
# terraform.tfvars
instance_type = "t3.micro"
subnet_ids    = ["subnet-abc", "subnet-def"]
```

**Variable validation:**
```hcl
variable "environment" {
  type = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}
```

---

## 11. Outputs

Expose values from your configuration — useful for chaining modules, displaying info after apply, or feeding into other tools.

```hcl
# outputs.tf
output "instance_ip" {
  description = "Public IP of the web server"
  value       = aws_instance.web.public_ip
}

output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true     # redacted from CLI output, still stored in state
}
```

```bash
terraform output                  # show all outputs
terraform output instance_ip      # show a specific output
terraform output -json            # machine-readable
```

---

## 12. State Management

The **state file** (`terraform.tfstate`, JSON format) is Terraform's record of what infrastructure it manages and the last-known attributes of every resource. It's how Terraform knows the difference between "desired config" and "what actually exists" without querying every single API on every run (though it does refresh on plan/apply).

**Why state matters:**
- Maps real-world resources to your configuration's resource blocks
- Stores resource metadata/dependency graph for performance
- Used to detect **drift** (when real infrastructure differs from state, e.g., someone manually changed something in the console)
- Required for `terraform plan` to know what already exists

**Critical facts:**
- State **can contain sensitive data** (passwords, keys) in plaintext — must be protected (encrypted backend, restricted access)
- **Never edit `terraform.tfstate` by hand** — use `terraform state` subcommands instead
- Local state (default) is a single file — risky for teams (no locking, easy to lose, can't collaborate) — **always use a remote backend in production/team settings**

```bash
terraform state list                          # list all resources in state
terraform state show aws_instance.web         # show attributes of one resource
terraform state mv aws_instance.web aws_instance.web2   # rename/move in state
terraform state rm aws_instance.web           # remove from state (doesn't destroy real resource!)
terraform state pull                          # download remote state, print to stdout
terraform state push                          # upload local state to remote backend
```

---

## 13. Remote Backends

A **backend** determines where state is stored and how operations (plan/apply) are executed.

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "envs/prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"   # for state locking
    encrypt        = true
  }
}
```

**Common backends:** S3 (+ DynamoDB for locking), Azure Storage, Google Cloud Storage, Terraform Cloud, Consul.

**Why use a remote backend:**
- **Team collaboration** — everyone reads/writes the same shared state
- **State locking** — prevents two people from running `apply` simultaneously and corrupting state
- **Durability/versioning** — S3 versioning, encryption at rest
- Some backends (Terraform Cloud) also provide **remote execution** (plan/apply runs in the cloud, not on your laptop)

```bash
terraform init -migrate-state   # migrate from local to remote backend (or vice versa)
```

---

## 14. State Locking

When using a backend that supports locking (S3+DynamoDB, Terraform Cloud, Consul), Terraform acquires a **lock** before any operation that writes state, preventing concurrent `apply` runs from corrupting it.

```bash
terraform plan      # acquires lock automatically, releases after
terraform force-unlock <LOCK_ID>   # manually remove a stuck lock (use carefully!)
```

---

## 15. Terraform Commands

```bash
terraform init                  # initialize working directory, download providers/modules
terraform validate              # check syntax/internal consistency (no API calls)
terraform fmt                   # auto-format .tf files to canonical style
terraform plan                  # preview changes
terraform plan -out=plan.tfplan # save plan to a file for later apply
terraform apply                 # apply changes (prompts for confirmation)
terraform apply plan.tfplan     # apply a previously saved plan
terraform apply -auto-approve   # skip confirmation prompt (use carefully, e.g. in CI)
terraform destroy               # destroy all managed infrastructure
terraform destroy -target=aws_instance.web   # destroy a specific resource
terraform show                  # human-readable view of state or a plan file
terraform graph                 # output dependency graph (Graphviz format)
terraform taint aws_instance.web    # mark resource for recreation on next apply (deprecated → use -replace)
terraform apply -replace="aws_instance.web" # force recreate a specific resource
terraform console               # interactive REPL for testing expressions
terraform workspace list/new/select
terraform providers             # show required providers
terraform version
```

---

## 16. Provisioners

Provisioners run scripts/commands on a resource during creation or destruction — generally considered a **last resort** (prefer baking config into images via Packer, or using dedicated config management tools like Ansible).

```hcl
resource "aws_instance" "web" {
  ami           = "ami-xxxx"
  instance_type = "t2.micro"

  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx"
    ]
    connection {
      type        = "ssh"
      user        = "ubuntu"
      private_key = file("~/.ssh/id_rsa")
      host        = self.public_ip
    }
  }

  provisioner "local-exec" {
    command = "echo ${self.public_ip} >> inventory.txt"
  }
}
```

**Types:** `local-exec` (runs on the machine running Terraform), `remote-exec` (runs on the created resource via SSH/WinRM), `file` (copies files to the resource).

**Why "last resort"?** Provisioners are imperative (breaks the declarative model), Terraform has no real visibility into their success/failure beyond exit code, and they tightly couple infrastructure provisioning to configuration steps. Prefer pre-baked images (Packer/AMIs), cloud-init/`user_data`, or a proper config management tool.

---

## 17. Meta-Arguments

### count
Creates multiple instances of a resource based on a number.
```hcl
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-xxxx"
  instance_type = "t2.micro"
  tags = {
    Name = "web-${count.index}"
  }
}
# Reference: aws_instance.web[0], aws_instance.web[1], ...
```

### for_each
Creates multiple instances based on a map or set — generally **preferred over `count`** because resources are tracked by key (not index), so adding/removing an item doesn't shift/recreate unrelated resources.
```hcl
resource "aws_instance" "web" {
  for_each      = toset(["app1", "app2", "app3"])
  ami           = "ami-xxxx"
  instance_type = "t2.micro"
  tags = {
    Name = each.key
  }
}
# Reference: aws_instance.web["app1"]
```

### depends_on
Explicitly declares a dependency Terraform can't infer automatically (most dependencies are inferred from references; this is for non-obvious ones, e.g., IAM propagation delays).
```hcl
resource "aws_instance" "web" {
  depends_on = [aws_iam_role_policy.example]
  # ...
}
```

### lifecycle
```hcl
resource "aws_instance" "web" {
  # ...
  lifecycle {
    create_before_destroy = true     # create replacement before destroying old (avoid downtime)
    prevent_destroy        = true     # block accidental destroy (errors out on terraform destroy)
    ignore_changes          = [tags]   # ignore drift on specific attributes (e.g., changed outside TF)
  }
}
```

---

## 18. Modules

A **module** is a reusable container for multiple `.tf` files defining related resources — the primary mechanism for code reuse and abstraction in Terraform. Every Terraform configuration has at least one module (the **root module**).

```hcl
# modules/ec2-instance/main.tf
variable "instance_type" {}
variable "ami" {}

resource "aws_instance" "this" {
  ami           = var.ami
  instance_type = var.instance_type
}

output "instance_id" {
  value = aws_instance.this.id
}
```

```hcl
# root main.tf — using the module
module "web_server" {
  source        = "./modules/ec2-instance"
  ami           = "ami-xxxx"
  instance_type = "t2.micro"
}

output "web_id" {
  value = module.web_server.instance_id
}
```

**Module sources:** local path (`./modules/x`), Terraform Registry (`hashicorp/vpc/aws`), Git repo, S3 bucket, HTTP URL.

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.8.1"
  # ... inputs
}
```

```bash
terraform get      # download/update modules
terraform init     # also downloads modules referenced in config
```

**Benefits:** DRY infrastructure code, encapsulation, versioning, consistent patterns across teams/environments.

---

## 19. Workspaces

Workspaces let you maintain **multiple distinct states** from a single configuration — commonly used for managing dev/staging/prod variants without duplicating code (though using separate root configurations/directories per environment is often preferred for production for stronger isolation).

```bash
terraform workspace list
terraform workspace new dev
terraform workspace select dev
terraform workspace show
```

```hcl
resource "aws_instance" "web" {
  instance_type = terraform.workspace == "prod" ? "t3.large" : "t2.micro"
  tags = {
    Environment = terraform.workspace
  }
}
```

**Caveat (interview-relevant):** Workspaces share the same backend config and same `.tf` files — they're good for quick environment variants but don't give you separate backend permissions/credentials per environment. Many teams instead use **separate directories or separate Terraform Cloud workspaces** per environment for stronger isolation and access control.

---

## 20. Expressions & Functions

```hcl
# Conditional expression
instance_type = var.environment == "prod" ? "t3.large" : "t2.micro"

# String interpolation
name = "${var.project}-${var.environment}"

# For expression (transform a list/map)
locals {
  upper_names = [for name in var.names : upper(name)]
  name_map    = { for idx, name in var.names : idx => name }
}

# Splat expression
all_ids = aws_instance.web[*].id
```

**Common built-in functions:**
```hcl
length(list)
join(",", list)
split(",", string)
lookup(map, key, default)
merge(map1, map2)
coalesce(val1, val2, val3)      # first non-null value
file("path/to/file")
templatefile("template.tpl", { var1 = "value" })
jsonencode(value)
cidrsubnet(prefix, newbits, netnum)
timestamp()
```

---

## 21. Import & State Manipulation

Bring existing, manually-created infrastructure under Terraform management.

```bash
# Classic CLI import (Terraform < 1.5 style, still works)
terraform import aws_instance.web i-0abcd1234efgh5678
```

```hcl
# Modern declarative import block (Terraform 1.5+)
import {
  to = aws_instance.web
  id = "i-0abcd1234efgh5678"
}
```
After importing, you still need to **write the matching resource configuration** yourself (`terraform plan` will show a diff if your config doesn't match reality) — import only populates state, it doesn't generate `.tf` code (though `terraform plan -generate-config-out=generated.tf` can help draft it in newer versions).

```bash
terraform state rm <resource>     # stop managing a resource (leaves real infra untouched)
terraform state mv <src> <dest>   # rename a resource without destroy/recreate
```

---

## 22. Terraform Cloud / Enterprise

A SaaS (Cloud) / self-hosted (Enterprise) platform from HashiCorp providing:
- **Remote state storage** with locking, built-in
- **Remote execution** — plan/apply runs in HashiCorp's infrastructure, not your laptop
- **VCS integration** — auto-triggers runs on Git push/PR
- **Policy as Code (Sentinel / OPA)** — enforce governance rules before apply (e.g., "no public S3 buckets", "must have cost tag")
- **Private Module Registry** — share modules internally
- **Team access controls, audit logging, cost estimation**

```hcl
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "prod-infra"
    }
  }
}
```

---

## 23. Security Best Practices

1. **Never commit `.tfstate` or `.tfvars` with secrets to version control** — add to `.gitignore`.
2. **Use a remote backend with encryption at rest** (S3 with SSE, Terraform Cloud).
3. **Mark sensitive variables/outputs** with `sensitive = true` (note: still stored in plaintext in state — this only hides CLI output).
4. **Use a secrets manager** (Vault, AWS Secrets Manager, SSM Parameter Store) and reference secrets via data sources rather than hardcoding.
5. **Restrict state file access** via IAM/backend permissions — state often contains sensitive plaintext data.
6. **Use least-privilege IAM roles** for the credentials Terraform runs with.
7. **Enable state locking** to prevent concurrent corrupting writes.
8. **Pin provider and module versions** (`required_providers`, `version` constraints) for reproducibility.
9. **Use `terraform plan` review (and require it in PRs)** before any apply — never blind-apply in production.
10. **Scan configurations** with tools like `tfsec`, `checkov`, or `terrascan` for security misconfigurations.
11. **Use policy-as-code** (Sentinel/OPA) in team settings to enforce guardrails automatically.

---

## 24. Terraform in CI/CD

Typical GitOps-style pipeline:
```yaml
# Example: GitHub Actions snippet
- name: Terraform Init
  run: terraform init

- name: Terraform Validate
  run: terraform validate

- name: Terraform Plan
  run: terraform plan -out=tfplan

- name: Terraform Apply (on merge to main)
  if: github.ref == 'refs/heads/main'
  run: terraform apply -auto-approve tfplan
```

**Common practices:**
- Run `plan` on every PR, post the diff as a PR comment for review
- Only run `apply` after merge/approval (often manually gated for prod)
- Store plan output as an artifact and `apply` the *exact same* saved plan (avoid re-planning right before apply, which could plan against changed state)
- Use remote backend + locking so CI runs don't clash with local runs
- Separate pipelines/credentials per environment (dev/staging/prod) with least-privilege service accounts

---

## 25. Troubleshooting

```bash
TF_LOG=DEBUG terraform apply        # verbose debug logging
TF_LOG=TRACE terraform plan         # most verbose
terraform validate                  # catch syntax errors early
terraform plan                      # always check before apply
terraform state list                # confirm what TF thinks it manages
terraform refresh                   # (deprecated standalone; now part of plan) sync state with real infra
```

**Common issues:**
| Issue | Cause / Fix |
|---|---|
| "Resource already exists" error on apply | Resource was created outside Terraform — `terraform import` it instead |
| State lock stuck | Previous run crashed mid-operation — `terraform force-unlock <ID>` (verify no other run is active first) |
| Plan shows unexpected destroy/recreate | Changed an immutable attribute, or attribute drifted outside Terraform |
| "Error: Provider configuration not present" | Module needs explicit provider passed via `providers = {}` block |
| Drift (real infra ≠ state) | Someone changed resources manually — run `terraform plan` to detect, then reconcile via apply or `ignore_changes` |

---

## 26. Best Practices Summary

- Always run `terraform plan` and review before `apply`
- Use remote backends with locking for any team/production work
- Never hand-edit the state file
- Pin Terraform, provider, and module versions
- Break large configurations into modules; keep modules small and focused
- Use `for_each` over `count` when items can be added/removed out of order
- Use separate state per environment (directories or workspaces)
- Keep secrets out of `.tf`/`.tfvars` files — use secret managers
- Use consistent naming/tagging conventions across resources
- Run `terraform fmt` and `terraform validate` in CI
- Use `prevent_destroy` lifecycle rule on critical resources (databases, etc.)
- Document modules with clear variable descriptions and examples

---

## 27. Cheat Sheet

```bash
terraform init
terraform validate
terraform fmt
terraform plan
terraform apply [-auto-approve]
terraform destroy
terraform show
terraform output
terraform state list / show / mv / rm
terraform import <resource_addr> <id>
terraform workspace list / new / select
terraform console
terraform graph
terraform apply -replace="<resource_addr>"
terraform force-unlock <LOCK_ID>
```

---

## 28. Interview Questions & Answers

**Q1: What is Terraform and what problem does it solve?**
A: Terraform is a declarative Infrastructure-as-Code tool that lets you define cloud/on-prem infrastructure in configuration files, then automatically creates, updates, or destroys resources to match that desired state — replacing manual console clicks or imperative scripts with a repeatable, versioned, provider-agnostic workflow.

**Q2: Explain the core Terraform workflow.**
A: Write configuration → `terraform init` (download providers/modules, configure backend) → `terraform plan` (preview the diff between desired config and current state) → `terraform apply` (execute the plan and update state).

**Q3: What is Terraform state and why is it needed?**
A: A file (`terraform.tfstate`) that maps your configuration's resources to real-world infrastructure IDs and tracks their last-known attributes. Terraform uses it to compute diffs efficiently on `plan`, detect drift, and know what to update/destroy — without state, Terraform wouldn't know what it has already created.

**Q4: Why use a remote backend instead of local state?**
A: Remote backends (S3, Terraform Cloud, etc.) enable team collaboration via a shared state, provide **state locking** to prevent concurrent corrupting writes, offer durability/versioning/encryption, and (for some backends) remote execution — local state is a single file at risk of loss, conflicts, and has no locking.

**Q5: Difference between `count` and `for_each`?**
A: `count` creates N instances indexed numerically (`resource[0]`, `resource[1]`) — removing an item from the middle of a list shifts indices and can cause unwanted recreation of unrelated resources. `for_each` creates instances keyed by a map/set value, so each instance is tracked independently by its key — adding/removing one item doesn't affect others. `for_each` is generally preferred when items aren't a simple fixed count.

**Q6: What's the difference between a `resource` and a `data` block?**
A: A `resource` block tells Terraform to create and manage the full lifecycle of an infrastructure object. A `data` block performs a read-only lookup of existing information (which may or may not be managed by Terraform) — Terraform never creates, updates, or destroys anything referenced via `data`.

**Q7: What is a Terraform module and why use them?**
A: A module is a reusable, self-contained set of `.tf` files representing a logical group of resources (e.g., "a standard VPC setup"). Modules enable code reuse, encapsulation, and consistency — instead of copy-pasting the same resource blocks across environments/projects, you parameterize them once and instantiate via `module` blocks with different inputs.

**Q8: How does Terraform determine the order in which to create/destroy resources?**
A: It builds a **dependency graph** from implicit references between resources (e.g., one resource's attribute referencing another's id) and explicit `depends_on` declarations, then performs operations in the correct topological order — creating dependencies first, destroying in reverse order. Independent resources can be created in parallel.

**Q9: What happens if someone manually changes a resource outside Terraform (drift)?**
A: On the next `terraform plan` (which refreshes state against real infrastructure), Terraform detects the difference between actual infrastructure and what's recorded in state/config, and shows a plan to reconcile it — typically reverting the manual change back to match the configuration, unless `ignore_changes` is set for that attribute.

**Q10: What's the purpose of `terraform plan -out=tfplan` followed by `terraform apply tfplan`?**
A: Saving the plan to a file and applying that exact saved plan guarantees you apply precisely what was reviewed — avoiding a race condition where infrastructure or state changes between a separate `plan` and `apply` invocation, which is especially important in CI/CD pipelines.

**Q11: What are provisioners, and why are they considered a last resort?**
A: Provisioners (`local-exec`, `remote-exec`, `file`) run scripts on a resource during creation/destruction. They're imperative and break Terraform's declarative model, Terraform has limited visibility into their actual success beyond exit codes, and they couple provisioning to configuration steps. Preferred alternatives: pre-baked machine images (Packer), cloud-init/`user_data`, or dedicated configuration management tools (Ansible, Chef).

**Q12: How do you manage secrets in Terraform?**
A: Avoid hardcoding secrets in `.tf`/`.tfvars` files; mark sensitive variables/outputs with `sensitive = true` (hides CLI output, though the value is still stored in state plaintext); pull secrets at runtime from a secret manager (Vault, AWS Secrets Manager, SSM) via data sources; ensure the state backend itself is encrypted and access-restricted since state can contain secrets in plaintext.

**Q13: What is the difference between `terraform taint` (or `-replace`) and `terraform destroy`?**
A: `-replace`/`taint` marks a *specific resource* to be destroyed and recreated on the *next apply*, while leaving everything else untouched and the configuration unchanged. `terraform destroy` tears down *all* resources managed by the current configuration/state (or a targeted subset with `-target`).

**Q14: What are Terraform workspaces and what's their limitation?**
A: Workspaces let one configuration maintain multiple separate state files (e.g., dev/staging/prod) without duplicating `.tf` code. The limitation: all workspaces share the same backend configuration and code — there's no built-in way to give different workspaces different credentials/permissions, so many teams prefer separate directories or separate Terraform Cloud workspaces for stronger environment isolation.

**Q15: How would you import an existing, manually-created resource into Terraform?**
A: Write a `resource` block matching the real resource's type/name (even with placeholder values initially), then run `terraform import <resource_address> <real_resource_id>` (or use a declarative `import` block in newer Terraform versions) to populate state. Then run `terraform plan` and adjust your configuration until the plan shows no changes, confirming your `.tf` code accurately matches reality.

**Q16: What is the `lifecycle` block used for? Give examples.**
A: It customizes how Terraform manages a resource's lifecycle: `create_before_destroy` (provision the replacement before destroying the old one, avoiding downtime), `prevent_destroy` (errors out if someone tries to destroy this resource, protecting critical infra like databases), and `ignore_changes` (tells Terraform to ignore drift on specific attributes, useful when something is legitimately managed outside Terraform, like autoscaling-adjusted tags).

**Q17: How does Terraform handle dependencies between resources across different providers (e.g., AWS and Cloudflare)?**
A: The same way as same-provider dependencies — by reference. If a Cloudflare DNS record's value references an AWS load balancer's DNS name attribute (`aws_lb.main.dns_name`), Terraform's dependency graph automatically ensures the AWS resource is created first, regardless of which provider plugin owns each resource.

**Q18: What's the difference between `terraform refresh` behavior in plan vs the old standalone command?**
A: Historically `terraform refresh` was a standalone command that updated state to match real infrastructure without modifying configuration or resources. Since Terraform 0.15+/1.x, this refresh happens automatically as part of `terraform plan` (and `apply`) by default, so the standalone command is largely unnecessary now (though `-refresh=false` can skip it for speed when you're confident state is accurate).

**Q19: How do you structure Terraform code for multiple environments (dev/staging/prod)?**
A: Common patterns: (1) separate directories per environment, each with its own backend config and `.tfvars`, reusing shared modules; (2) Terraform workspaces sharing one configuration with environment-specific variables; (3) separate Terraform Cloud workspaces per environment with distinct variable sets and credentials. Separate directories/workspaces in Terraform Cloud are generally preferred for production due to stronger isolation of state, credentials, and blast radius.

**Q20: What is Sentinel (or OPA) in the context of Terraform?**
A: Policy-as-code frameworks integrated with Terraform Cloud/Enterprise that let organizations enforce governance rules automatically before an `apply` can proceed — e.g., "no S3 buckets without encryption," "instance type must be from an approved list," "must have a cost-center tag" — preventing non-compliant infrastructure from ever being provisioned.

---

### Final interview tip
Be ready to explain the **plan/apply/state** cycle clearly, the difference between **`count` vs `for_each`**, why **state locking and remote backends matter for teams**, and to walk through **how Terraform decides resource creation order via the dependency graph**. Also expect at least one practical question like "how would you import an existing resource" or "how do you handle secrets" — these come up very often in real interviews.
