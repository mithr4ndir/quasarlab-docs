# Terraform

`terraform-quasarlab` is the IaC for things that talk to provider APIs: Proxmox VMs, Cloudflare DNS, GitHub repos, NPM proxy hosts. Anything inside the K8s cluster is ArgoCD's job; anything that talks to a provider is Terraform's.

## Day-to-day

```bash
terraform init                              # one-time per workspace, plus after backend changes
terraform validate                          # syntax + provider schema check
terraform fmt -recursive                    # canonical formatting
terraform plan -out plan.tfplan             # always plan into a file, review, then apply that file
terraform apply plan.tfplan                 # exactly what was planned, no surprises
```

`plan -out` then `apply <file>` is the only safe pattern when something else might be changing the same resources between planning and applying.

## State management

```bash
terraform state list
terraform state show <addr>
terraform state mv <from> <to>              # refactor without rebuild
terraform state rm <addr>                   # forget without destroying (use carefully)
```

State lives in a Postgres backend on `192.168.1.123` so multiple machines can plan against the same lab. Locking is on. If a plan crashed and the lock is stuck:

```bash
terraform force-unlock <lock-id>            # only when you are sure no one else is applying
```

## Drift and import

```bash
# What does the cloud actually look like vs what state says?
terraform plan -refresh-only

# Bring an existing resource under management:
terraform import <addr> <provider-id>
```

Drift discovery runs in CI weekly so I do not get surprised by manual changes.

## Workspaces

```bash
terraform workspace list
terraform workspace new   <name>
terraform workspace select <name>
```

I keep one workspace per environment. The lab has only one environment, so workspaces are mostly unused, but the muscle memory transfers when working on multi-env codebases.

## Providers I use

| Provider | What for |
|----------|----------|
| `bpg/proxmox` | VM provisioning, cloud-init, snapshots |
| `cloudflare/cloudflare` | DNS, tunnel routing |
| `integrations/github` | Repo settings, branch protection, PAT-bound deploy keys |
| `nginxinc/nginx-proxy-manager` (community) | NPM hosts and certificates, where applicable |

## Module structure

```
terraform-quasarlab/
  main.tf              # provider blocks, root module
  vms/                 # one .tf file per VM, each calls the proxmox-vm module
  dns/                 # cloudflare records by zone
  modules/
    proxmox-vm/        # standardized VM (cloud-init, tags, network, disks)
    cf-record/         # CNAME/A/AAAA wrapper with sane defaults
```

Reusable modules live in `modules/`. I do not pull in Terraform Registry modules unless I have read them; opaque modules are a supply-chain risk.

## Things I reach for when something is wrong

| Question | Command |
|----------|---------|
| Did my last apply do what I thought? | `terraform show <plan-file>` or `terraform state show <addr>` |
| Is my state out of sync with reality? | `terraform plan -refresh-only` |
| Why is plan so slow? | `TF_LOG=DEBUG terraform plan 2> tf.log` and check provider call counts |
| Did someone mutate a resource by hand? | Run drift discovery, see the diff |
