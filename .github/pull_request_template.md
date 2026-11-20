## What and why

<!-- What does this change, and why? -->

## Release

The pull request title becomes the commit message on main, and release-please picks the version from it:
`fix:` = patch, `feat:` = minor, `feat!:` (or a `BREAKING CHANGE:` footer) = major.

- [ ] Does `terraform plan` in a project using this module **replace** `aws_instance.puppet_master`? A replaced server gets a new certificate authority and every web node must be replaced. If yes, this is a breaking change: use `feat!:` / `fix!:` and explain the upgrade in the description.
- [ ] Does it rename or remove an input or output, or add a required input? If yes, this is a breaking change.
- [ ] Tests updated (`terraform test`)
