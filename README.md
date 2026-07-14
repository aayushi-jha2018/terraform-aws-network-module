# terraform-aws-network-module

Reusable Terraform module for a standard AWS networking layer -- VPC, public and private subnets across availability zones, an internet gateway, an optional NAT gateway, route tables, and a configurable default security group. Something to build on top of, not a snippet to copy into a blog post.

## Usage

```hcl
module "network" {
  source = "github.com/aayushi-jha2018/terraform-aws-network-module"

  name               = "my-app"
  vpc_cidr           = "10.0.0.0/16"
  availability_zones = ["us-east-1a", "us-east-1b"]

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

A runnable copy of this lives in `examples/basic` -- it's the exact config CI validates on every push.

## How it fits together

Public subnets route out through an internet gateway. Private subnets route out through a single NAT gateway sitting in the first public subnet (toggle this with `enable_nat_gateway`). Subnets get spread across whatever AZs you pass in, wrapping around if you give it more subnets than AZs. A default security group comes out the other end with whatever ingress rules you configured and an open egress rule, ready to attach to anything built on top.

## Inputs / Outputs

| Input | Description | Default |
|---|---|---|
| name | Prefix applied to all resource names | required |
| vpc_cidr | CIDR block for the VPC | 10.0.0.0/16 |
| availability_zones | AZs to spread subnets across | ["us-east-1a", "us-east-1b"] |
| public_subnet_cidrs | CIDR blocks for public subnets | ["10.0.0.0/24", "10.0.1.0/24"] |
| private_subnet_cidrs | CIDR blocks for private subnets | ["10.0.10.0/24", "10.0.11.0/24"] |
| enable_nat_gateway | Create a NAT gateway for private egress | true |
| ingress_rules | Ingress rules for the default security group | [] |
| tags | Tags applied to all resources | {} |

Outputs: `vpc_id`, `vpc_cidr_block`, `public_subnet_ids`, `private_subnet_ids`, `nat_gateway_ids`, `default_security_group_id`.

## Design notes

The example only deploys a single NAT gateway, in one AZ. That's fine for a demo and for plenty of real low-traffic setups, but it's a single point of failure -- if that AZ has a bad day, every private subnet loses egress. A production version of this module would take a `one_nat_gateway_per_az` flag and provision one NAT gateway (and EIP) per AZ instead of sharing one for the whole VPC. I kept it to one here mainly to keep the example's plan output small and readable in CI logs, not because it's the right default for a real account.

I also intentionally avoided the `aws_availability_zones` data source, even though it's the more common way to write this kind of module. That data source makes a real API call against an AWS account at plan time, which would break the whole point of this repo -- a `terraform plan` that runs in CI with zero cloud credentials. Passing `availability_zones` in as a plain variable is a little more manual for callers, but it's what keeps the CI plan honest and fully offline.

## CI

`.github/workflows/ci.yml` runs on every push: `terraform fmt -check`, then `terraform init -backend=false` plus `terraform validate` against both the root module and `examples/basic`, then a real `terraform plan` against the example. The plan step uses fake AWS credentials (`access_key = "test"`) plus the provider's `skip_credentials_validation` / `skip_requesting_account_id` / `skip_metadata_api_check` flags -- a standard pattern that lets Terraform build a genuine, full execution plan without ever touching a real AWS account. Check the Actions tab if you want to see the actual resource-by-resource plan output -- it isn't mocked.

## Structure

```
.
├── main.tf              # VPC, subnets, gateways, route tables, security group
├── variables.tf         # typed inputs with sane defaults
├── outputs.tf
├── versions.tf
├── examples/basic/       # runnable example, the thing CI actually plans
└── .github/workflows/ci.yml
```

MIT licensed.
