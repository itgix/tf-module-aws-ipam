# Terraform AWS IPAM module

This module creates and manages AWS VPC IPAM (IP Address Manager) with a hierarchical pool structure, provisioned CIDRs, and cross-account sharing via AWS RAM. It supports up to 4 levels of nested pools (top-level + 3 nested levels), automatic operating region detection, and locale inheritance from parent to child pools.

Based on [aws-ia/terraform-aws-ipam](https://github.com/aws-ia/terraform-aws-ipam).

Part of the ITGix AWS Landing Zone — https://itgix.com/itgix-landing-zone/

## Requirements

- [Terraform](https://www.terraform.io/) >= 1.3.0
- [AWS provider](https://registry.terraform.io/providers/hashicorp/aws/latest) >= 4.53.0

## Usage

### Basic module block

Point `source` at this repository. Replace placeholders with your values.

```hcl
module "ipam" {
  source = "git::https://github.com/itgix/tf-module-aws-ipam.git?ref=v1.0"

  top_cidr        = ["10.0.0.0/8"]
  top_name        = "itgix-ipam-top"
  top_description = "Top-level pool for landing zone"

  pool_configurations = {
    "eu-central-1" = {
      description = "Regional pool for eu-central-1"
      cidr        = ["10.0.0.0/10"]
      locale      = "eu-central-1"

      sub_pools = {
        dev = {
          cidr                 = ["10.3.0.0/16"]
          description          = "Dev application VPC pool"
          ram_share_principals = ["123456789012"]
          allocation_resource_tags = { Environment = "dev" }
        }
        prod = {
          cidr                 = ["10.5.0.0/16"]
          description          = "Prod application VPC pool"
          ram_share_principals = ["345678901234"]
          allocation_resource_tags = { Environment = "prod" }
        }
      }
    }
  }

  tags = {
    ManagedBy = "Terraform"
    Project   = "itgix"
  }
}
```

### Outputs

| Output | Description |
|--------|-------------|
| `ipam_info` | The IPAM object information (if created) |
| `operating_regions` | List of all IPAM operating regions |
| `pool_names` | List of all pool names across all levels |
| `pool_level_0` | Map of the top-level pool |
| `pools_level_1` | Map of all pools at level 1 |
| `pools_level_2` | Map of all pools at level 2 |
| `pools_level_3` | Map of all pools at level 3 |

Use pool outputs to wire IPAM into VPC creation:

```hcl
# Allocate a VPC CIDR from the dev pool
resource "aws_vpc" "dev" {
  ipv4_ipam_pool_id   = module.ipam[0].pools_level_2["eu-central-1/dev"].id
  ipv4_netmask_length = 16
}
```

### Pool hierarchy

Pools are structured in levels via the `pool_configurations` nested map:

| Level | Configured via | Example |
|-------|---------------|---------|
| Level 0 (top) | `top_cidr`, `top_name`, `top_description` | `10.0.0.0/8` |
| Level 1 | Keys of `pool_configurations` | `eu-central-1` |
| Level 2 | `sub_pools` inside level 1 | `dev`, `prod`, `shared-services` |
| Level 3 | `sub_pools` inside level 2 | Optional further nesting |

Pool names and descriptions auto-generate from the key hierarchy (e.g. `eu-central-1/dev`). Override with explicit `name` or `description` on any pool.

Locales inherit downward — set `locale` once at level 1 and all children inherit it.

### RAM sharing (cross-account)

Add `ram_share_principals` to any pool at any level to share it via AWS RAM. Principals can be account IDs, OU IDs, or organization IDs.

```hcl
sub_pools = {
  dev = {
    cidr                 = ["10.3.0.0/16"]
    ram_share_principals = ["767398095708"]  # dev account
  }
}
```

### pool_configurations object

Each key in `pool_configurations` represents a pool. Pools can be nested up to 3 levels via the `sub_pools` attribute. Either `cidr` or `netmask_length` must be set, but not both.

| Name | Description |
|------|-------------|
| `cidr` | List of CIDRs to provision into the pool. Conflicts with `netmask_length` |
| `netmask_length` | Netmask length to request. Conflicts with `cidr` |
| `locale` | Locale to set for the pool. Inherited by child pools |
| `description` | Description for the pool. Defaults to the key hierarchy name |
| `name` | Name for the pool. Defaults to the key hierarchy name |
| `ram_share_principals` | List of valid RAM share principals (org-id, ou-id, or account id) |
| `allocation_default_netmask_length` | Default netmask length for allocations |
| `allocation_max_netmask_length` | Maximum netmask length for allocations |
| `allocation_min_netmask_length` | Minimum netmask length for allocations |
| `allocation_resource_tags` | Tags required on resources for allocation from this pool |
| `auto_import` | Auto-import setting |
| `tags` | Tags for the pool |
| `aws_service` | AWS service, relevant for public IPs |
| `publicly_advertisable` | Whether the pool is publicly advertisable |
| `sub_pools` | Nested map of child pools (repeats this same structure) |

## Variable reference (summary)

| Name | Description |
|------|-------------|
| `top_cidr` | Top-level CIDR blocks |
| `top_name` | Name of the top-level pool |
| `top_description` | Description of the top-level pool |
| `top_netmask_length` | Top-level netmask length (ipv6 only) |
| `top_ram_share_principals` | RAM share principals for the top-level pool |
| `top_auto_import` | Auto-import setting for the top-level pool |
| `top_locale` | Locale for the top-level pool (ipv6 contiguous blocks only) |
| `top_publicly_advertisable` | Whether the top-level pool is publicly advertisable |
| `top_public_ip_source` | Public IP source: `amazon` or `byoip` |
| `top_aws_service` | AWS service for public IPs: `ec2` |
| `top_cidr_authorization_contexts` | BYOIP authorization context documents |
| `pool_configurations` | Nested map describing the pool hierarchy (see above) |
| `address_family` | `ipv4` or `ipv6` |
| `create_ipam` | Whether to create an IPAM instance |
| `ipam_scope_id` | Existing scope ID (required if `create_ipam` is false) |
| `ipam_scope_type` | `public` or `private` |
| `tags` | Tags for the IPAM resource |

See [`variables.tf`](variables.tf) for full definitions.
