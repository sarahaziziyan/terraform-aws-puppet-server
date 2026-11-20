# terraform-aws-puppet-server

Terraform module that runs a Puppet Server on AWS EC2. The server configures web nodes and deploys a website from a Git repository. Nodes enrol automatically when their certificate request carries a shared secret kept in SSM Parameter Store.

## What it creates

- An EC2 instance (`t3.medium`, Ubuntu 22.04 from Canonical, 30 GB encrypted root volume, IMDSv2 required)
- A security group named `puppet-master`:
  - TCP 8140 (Puppet) from the web nodes' security group only
  - TCP 22 (SSH) from EC2 Instance Connect only, using the region's AWS-managed prefix list
  - All outbound traffic
- A boot script that:
  - installs Puppet Server 8 and sets its certificate name to `puppet_master_fqdn`
  - checks that the instance role can read the autosign secret, then installs an autosign script that reads the secret from SSM for each request
  - installs `puppetlabs-vcsrepo` and writes the `profile::website` class and `site.pp`

The module does not create DNS records; point a record at the `private_ip` output.

## Before you use it

The instance needs an existing instance profile whose role may call `ssm:GetParameter` on the SecureString parameter that holds the autosign secret. The module reads both by name and creates no IAM resources.

Web nodes must:

- use the EC2 instance ID as their Puppet certname (`site.pp` matches `/^i-[0-9a-f]+$/`)
- send the secret in their certificate request as `challengePassword` (OID `1.2.840.113549.1.9.7`)
- publish the facts `site_repo_url` and `site_repo_branch`

## Usage

```hcl
module "puppet_server" {
  source = "git::https://github.com/sarahaziziyan/terraform-aws-puppet-server.git?ref=v1.0.0" # x-release-please-version

  vpc_id                = "vpc-0123456789abcdef0"
  subnet_id             = "subnet-0aaaaaaaaaaaaaaa1"
  key_name              = "my-key"
  web_security_group_id = aws_security_group.web.id
  puppet_master_fqdn    = "puppet.internal.example.com"

  autosign_secret_parameter_name = "/puppet/autosign-secret"
  instance_profile_name          = "puppet-master-autosign-reader"
}
```

## Requirements

| Name | Version |
|---|---|
| Terraform | `>= 1.7` |
| AWS provider | `~> 6.0` |

## Inputs

All inputs are required.

| Name | Type | Description |
|---|---|---|
| `vpc_id` | `string` | VPC for the server's security group |
| `subnet_id` | `string` | Subnet to launch the server in |
| `key_name` | `string` | Name of an existing EC2 key pair |
| `web_security_group_id` | `string` | Security group of the web nodes allowed to reach port 8140 |
| `puppet_master_fqdn` | `string` | Fully qualified name of the server. It becomes the certname and is fixed when the Puppet CA is created |
| `autosign_secret_parameter_name` | `string` | Name of the SSM SecureString parameter that holds the autosign secret |
| `instance_profile_name` | `string` | Instance profile that may read that parameter |

## Outputs

| Name | Description |
|---|---|
| `private_ip` | Private IP of the server |
| `instance_profile_name` | Instance profile attached to the server |

## Things to know

- Terraform ignores changes to the instance's `user_data` and `ami` after creation. A new Ubuntu image or an edited boot script never replaces the server or its certificate authority. To rebuild it on purpose, replace `aws_instance.puppet_master`; every node then needs a new certificate.
- The security group name `puppet-master` is fixed, so only one copy of this module can run per VPC.

## Tests

The unit tests mock AWS, so they need no credentials and create nothing.

```bash
terraform init
terraform test
```

GitHub Actions runs `terraform fmt -check`, `terraform validate` and `terraform test` on every pull request (`.github/workflows/test.yml`).

## Releases

Versions follow [Semantic Versioning](https://semver.org/) and are created by [release-please](https://github.com/googleapis/release-please) (`.github/workflows/release.yml`). Nobody creates tags by hand.

1. Merge pull requests to `main` with [Conventional Commit](https://www.conventionalcommits.org/) titles:

   | Title | Release |
   |---|---|
   | `fix: ...` | Patch, e.g. `v1.0.1` |
   | `feat: ...` | Minor, e.g. `v1.1.0` |
   | `feat!: ...`, `fix!: ...` or a `BREAKING CHANGE:` footer | Major, e.g. `v2.0.0` |
   | `docs:`, `test:`, `ci:`, `chore:`, `refactor:` | No release on their own |

2. release-please keeps a pull request named `chore(main): release ...` open, with the next version and the `CHANGELOG.md` entry.
3. Merge that pull request when you want to release. It creates the tag and a GitHub Release, and updates the version in the usage example above.

A change is **breaking** if it renames or removes an input or output, adds a required input, or makes Terraform replace `aws_instance.puppet_master` (a new server has a new certificate authority, so every node must be replaced).

Never move or delete a published tag; release a new version instead.
