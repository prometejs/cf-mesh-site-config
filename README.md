# cf-site-config

Ansible configuration for Cloudflare WARP-connector sites.

```mermaid
graph LR
    %% Style definitions
    classDef default fill:transparent,stroke:#333,stroke-width:1px;
    classDef linkNode fill:transparent,stroke:#0288d1,stroke-width:1px,font-weight:bold;
    classDef activeLinkNodeClass fill:transparent,stroke:#2e7d32,stroke-width:4px,font-weight:bold;

    %% Diagram nodes
    T["cf-terraform-infra<br/><hr/>creates tunnels + tf-states"]:::linkNode
    C["cf-cloud-init<br/><hr/>first-boot provisioning"]:::
    A["cf-site-config<br/><hr/>day-2 config"]:::activeLinkNodeClass

    %% Flow connections with text notes
    T --> C
    C --- A

    %% Clickable hyperlinks (Fixed with 'href')
    click T href "https://github.com/prometejs/cf-terraform-infra" "Open Terraform Repo"
    click C href "https://github.com/prometejs/cf-cloud-init" "Open Cloud-Init Repo"
```

## Inventory contract

Inventory is **sourced live from the Terraform state** produced by
[`cf-terraform-infra`](https://github.com/prometejs/cf-terraform-infra) (separate and completely independent repository) via the
[`cloud.terraform.terraform_state`](https://github.com/ansible-collections/cloud.terraform)
inventory plugin. The plugin reads the S3-backed state directly — no checkout
of cf-terraform-infra is needed. Terraform state is the single source of truth.

Each connector site provisioned by `cf-terraform-infra` appears as one host.

**Groups**

- `connectors` — every discoverable site host (limitted to connector nodes for now)
- `dev` / `prod` — parent group per environment (workspace), with `connectors` as a child

**Host vars**

| Var                | Type            | Notes                                                  |
| ------------------ | --------------- | ------------------------------------------------------ |
| `ansible_host`     | string          | Connector IP — SSH target                              |
| `tunnel_token`     | string (secret) | Cloudflare WARP connector registration token          |
| `site_cidr`        | string          | This site's subnet, e.g. `10.20.0.0/24`                |
| `private_hostname` | string          | e.g. `dev-site-a.dev.prometejs.network`                |
| `ha_enabled`       | string bool     | `"true"` or `"false"`                                  |
| `ha_peer_ip`       | string          | Empty when not in an HA pair                           |
| `environment`      | string          | `dev` or `prod`                                        |
| `remote_cidrs`     | JSON string     | List of other sites' CIDRs — parse with `\| from_json` |

## Repository coupling

If `cf-terraform-infra` ever changes its S3 backend (bucket / key prefix /
region), update the inventory files here to match.

## Prerequisites

- `terraform` CLI on `PATH` (the inventory plugin shells out to it for `init` / `show`)
- AWS credentials with read access to the S3 state bucket (env vars or `~/.aws/credentials`)
- `ansible-core >= 2.16`, Python `>= 3.11`

## Quickstart (local)

```bash
ansible-galaxy collection install -r requirements.yml

# Pick the workspace whose state you want to read
export TF_WORKSPACE=dev

# Inventory check — groups + hosts
ansible-inventory -i inventories/dev/terraform_state.yml --graph

# Sanity playbook — pings every connector and dumps the contract fields
ansible-playbook -i inventories/dev/terraform_state.yml playbooks/ping.yml

# Target a single host
ansible-playbook -i inventories/dev/terraform_state.yml playbooks/ping.yml -l dev-site-a
```

**Don't** run `ansible-inventory --vars` in shared terminals or CI logs — it dumps `tunnel_token` in plain text.

## CI / Deployment

- [`lint.yml`](.github/workflows/lint.yml): static checks
- [`apply.yml`](.github/workflows/apply.yml): runs Ansible against real hosts

**Triggers**

- `workflow_dispatch` — manual run. Inputs: `workspace` (`dev`/`prod`) and an
  optional `limit` pattern (e.g. `dev-site-a` or `dev-site-a,dev-site-b`).
- `repository_dispatch` with type `cf-tf-applied` — fired by 
  `cf-terraform-infra` after a successful `terraform apply`. Payload:
  `{ workspace, new_sites }`. The workflow applies only against `new_sites`.

**Promotion gates**

`dev` and `prod` map to GitHub Environments. Configure required reviewers on
the `prod` environment to gate prod applies behind manual approval.

**Required secrets and variables**

| Name                            | Kind        | Scope               | Purpose                          |
| ------------------------------- | ----------- | ------------------- | -------------------------------- |
| `AWS_ACCESS_KEY_ID`             | secret      | Environment (dev/prod) | Read TF state from S3         |
| `AWS_SECRET_ACCESS_KEY`         | secret      | Environment (dev/prod) | Read TF state from S3         |
| `CF_WARP_AUTH_CLIENT_ID`        | secret      | Environment (dev/prod) | WARP service-token client ID  |
| `CF_WARP_AUTH_CLIENT_SECRET`    | secret      | Environment (dev/prod) | WARP service-token secret     |
| `CF_WARP_ORG`                   | variable    | Environment / repo  | Zero Trust team domain           |
| `AWS_DEFAULT_REGION`            | variable    | Repo                | Defaults to `eu-west-2`          |

The static-AWS-creds pattern matches `cf-terraform-infra`. Migrating both
repos to AWS OIDC is a future iteration.