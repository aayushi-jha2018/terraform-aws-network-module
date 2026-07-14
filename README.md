# terraform-aws-network-module

A reusable, tested Terraform module that provisions a standard AWS networking layer: a VPC with public and private subnets spread across availability zones, an internet gateway, an optional NAT gateway for private-subnet egress, route tables, and a configurable default security group.

This is not a toy snippet. It is built and validated the way a real infrastructure module should be: with input validation via typed variables, a runnable example, and a CI pipeline that actually runs `terraform fmt`, `terraform validate`, and `terraform plan` on every push — with no AWS account or credentials required.

## Architecture

```
                         ┌─────────────────────────────┐
                         │            VPC               │
                         │                              │
  Internet ── IGW ───────┼── public subnet(s) ──┐       │
                         │        │              │       │
                         │   route table         │       │
                         │        │         NAT gateway  │
                         │        │              │       │
                         │  private subnet(s) ───┘       │
                         │        │                      │
                         │   route table                │
                         │                              │
                         │   default security group      │
                         └─────────────────────────────┘
```

- Public subnets route outbound traffic through an internet gateway and (optionally) auto-assign public IPs.
- Private subnets route outbound traffic through a single NAT gateway placed in the first public subnet (toggle with `enable_nat_gateway`).
- Subnets are distributed across the availability zones you provide, wrapping around if there are more subnets than AZs.
- A default security group is created with a configurable list of ingress rules and an open egress rule, ready to attach to instances or other resources built on top of this module.

## Usage

```hcl
module "network" {
  source = "github.com/aayushi-jha2018/terraform-aws-network-module"

  name     = "my-app"
  vpc_cidr = "10.0.0.0/16"

  availability_zones   = ["us-east-1a", "us-east-1b"]
  public_subnet_cidrs  = ["10.0.0.0/24", "10.0.1.0/24"]
  private_subnet_cidrs = ["10.0.10.0/24", "10.0.11.0/24"]
  enable_nat_gateway   = true

  ingress_rules = [
    {
      description = "Allow HTTPS from anywhere"
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  ]

  tags = {
    Environment = "production"
  }
}
```

A complete, runnable copy of this usage lives in [`examples/basic`](examples/basic) — the exact same configuration validated by CI.

## Inputs

| Name | Description | Type | Default |
|---|---|---|---|
| `name` | Name prefix applied to all resources | `string` | – (required) |
| `vpc_cidr` | CIDR block for the VPC | `string` | `"10.0.0.0/16"` |
| `availability_zones` | AZs to spread subnets across | `list(string)` | `["us-east-1a", "us-east-1b"]` |
| `public_subnet_cidrs` | CIDR blocks for public subnets | `list(string)` | `["10.0.0.0/24", "10.0.1.0/24"]` |
| `private_subnet_cidrs` | CIDR blocks for private subnets | `list(string)` | `["10.0.10.0/24", "10.0.11.0/24"]` |
| `enable_nat_gateway` | Whether to create a NAT gateway for private-subnet egress | `bool` | `true` |
| `ingress_rules` | Ingress rules for the default security group | `list(object)` | `[]` |
| `tags` | Tags applied to all resources | `map(string)` | `{}` |

## Outputs

| Name | Description |
|---|---|
| `vpc_id` | ID of the created VPC |
| `vpc_cidr_block` | CIDR block of the created VPC |
| `public_subnet_ids` | IDs of the public subnets |
| `private_subnet_ids` | IDs of the private subnets |
| `nat_gateway_ids` | IDs of the NAT gateway(s), if enabled |
| `default_security_group_id` | ID of the default security group |

## Continuous integration

Every push runs [`.github/workflows/ci.yml`](.github/workflows/ci.yml), which:

1. Runs `terraform fmt -check -recursive` to enforce consistent formatting across the module and example.
2. Runs `terraform init -backend=false` and `terraform validate` on the root module to catch syntax and configuration errors.
3. Runs the same init/validate steps against [`examples/basic`](examples/basic), the real usage example.
4. Runs `terraform plan` against the example using fake AWS credentials (`access_key = "test"`, `secret_key = "test"`) combined with the provider's `skip_credentials_validation`, `skip_requesting_account_id`, and `skip_metadata_api_check` flags. This lets Terraform build a full, real execution plan — showing every resource that would be created — without ever contacting an actual AWS account.

No cloud credentials or secrets are stored or used anywhere in this repository. The plan step is a genuine `terraform plan` run, not a mocked or hand-written example — you can see the real resource-by-resource output in the Actions tab.

## Project structure

```
.
├── main.tf                 # VPC, subnets, gateways, route tables, security group
├── variables.tf            # Typed input variables with sane defaults
├── outputs.tf              # VPC/subnet/gateway/security-group outputs
├── versions.tf             # Terraform + provider version constraints
├── examples/
│   └── basic/
│       └── main.tf         # Runnable example, validated by CI
└── .github/workflows/ci.yml
```

## What this demonstrates

- Writing a Terraform module as a reusable, parameterized unit rather than a one-off script.
- Designing inputs/outputs so the module composes cleanly into a larger stack.
- Validating infrastructure-as-code in CI without requiring cloud credentials, using the standard fake-credentials-plus-skip-flags pattern.
- Providing a runnable example that doubles as living documentation and as the thing CI actually checks.
