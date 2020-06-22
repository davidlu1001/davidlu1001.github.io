---
title: Terraform Upgrade to 0.12
tags:
  - Terraform
  - AWS
  - SRE
  - DevOps
categories:
  - - SRE
  - - DevOps
  - - Terraform
abbrlink: 8ca14daf
date: 2020-06-22 13:29:03
---

# Steps

## 1. Check existing terraform version
```
terraform -v
```

## 2. Upgrade to Terraform 0.11 first (if applicable)
If the current version is not `0.11` then upgrade it to `0.11.14` first. If you are on 0.11.x, please make sure you are on 0.11.14.

## 3. Pre-upgrade Checklist
Terraform `v0.11.14` introduced a temporary helper command `terraform 0.12checklist`, which analyses the configuration to detect any required steps that will be easier to perform before upgrading.

## 4. Initialisation in 0.11
```
terraform init
```

## 5. `Plan` and make sure no errors are shown
```
terraform plan
```

## 6. `Apply`
to ensure that your real infrastructure and Terraform state are consistent with the current configuration.
```
terraform apply
```

## 7. Do the `checklist` check
to see if there are any pre-upgrade steps in the checklist

```
terraform 0.12checklist
```

## 8. Resolve suggestions from checklist
Depending upon the suggestions above, take the steps & change the tf scripts.

If you are using templates, then you need to update the template provider as well.

Run above command until there are no suggestions.

## 9. Switch to terraform 0.12
Switch to terraform 0.12 (choose one of the following methods, ranked in order of recommendation)

- Using [tfswitch](https://github.com/warrensbox/terraform-switcher) (recommend) / [tfenv](https://github.com/tfutils/tfenv) to switch to 0.12

- Using docker-terraform

- Unpin the old 0.11 version (if applicable) and upgrade to Terraform 0.12: `brew upgrade terraform`

## 10. Initialisation in 0.12

For those repos that ref terraform-modules, need to:

- Make sure use git module to refer terraform-modules:

    e.g.
```
source = "git@github.com:{user}/terraform-modules.git//{module_name}?ref={branch_name}"
```
- replace branch name to refer testing branch in `terraform-modules` (if applicable)
- update `.gitmodules` first to use testing branch:

    e.g.
```
cat .gitmodules
 
[submodule "common/modules"]
 
    path = common/modules
 
    url = git@github.com:{user}/terraform-modules.git
 
    branch = {branch_name}
```

Then run command to update submodules:

```
git submodule update --recursive --remote
```

> P.S.
>
> Re-running init with modules already installed will install the sources for any modules that were added to configuration since the last init, but will not change any already-installed modules.
>
> Use `-upgrade` to override this behavior, updating all modules to the latest available source code..
>
> So use this command when there're changes from terraform-modules:

```
terraform init -upgrade
```

## 11. Check for errors
```
terraform validate
```

## 12. Auto update the code
with below command. Terraform v0.12 includes a new command `terraform 0.12upgrade` that will read the configuration files for a module written for Terraform 0.11 and update them in-place to use the cleaner Terraform 0.12 syntax and also adjust for use of features that have changed behaviour in the 0.12 Terraform language.

```
terraform 0.12upgrade
```

### 12.1. Symlink
If there is a symlink under the directory, the directory of the actual file pointed by the symlink needs to be upgraded in the last, to avoid the situation where 0.11/0.12 codes coexist. This will terminate the execution of the `0.12upgrade` command.

And also need to delete symlink first, before running `0.12upgrade`, then restore the symlink files.

The logic for each directory would be something like this:

e.g.
```
# init
terraform init -upgrade

# clean
rm -rf .terraform

# record symlink
find . -type l -name "*.tf" -ls | awk -F' ./' '{print $NF}' | awk -F' -> ' '{print "ln -s "$2,$1}' > /tmp/all_ln.txt

# remove
find . -type l -name "*.tf" | sed -e "s#\./##g" | xargs echo "rm" | bash

# upgrade
terraform 0.12upgrade -yes

# restore symlink
cat /tmp/all_ln.txt | bash

# plan
terraform plan
```

### 12.2. Error on batch upgrade
According to the official doc, run the following command for batch upgrade:
```
gfind . -name '*.tf' -printf "%h\n" | sort | uniq | xargs -n1 terraform init

gfind . -name '*.tf' -printf "%h\n" | sort | uniq | xargs -n1 terraform 0.12upgrade -yes
```

but got bunch of errors:
```
Error: error resolving providers:

- provider.template: no suitable version installed
  version requirements: "(any version)"
  versions installed: none
```
Turned out the init command will ***overwrite*** the `.terraform/plugins` dir each time:

e.g.

First init module `./A`:
```
ls -rthl .terraform/plugins/darwin_amd64/
lock.json*                              terraform-provider-aws_v2.63.0_x4*      terraform-provider-template_v2.1.2_x4*
```

Then init module `./B`:
```
ls -rthl .terraform/plugins/darwin_amd64/
lock.json*                          terraform-provider-aws_v2.63.0_x4*
```

So need to create script to process each directory one by one (first `init` then `upgrade`) instead of using `xargs` for batch upgrade.

## 13. Fix syntax issues

Might need to manual fix syntax issues (that can not be auto-upgraded):

- Invalid expression value: number required

```
Error: Incorrect value type
 
  on .terraform/modules/service-crayola/service/iam_db_auth.tf line 3, in data "template_file" "iam-db-auth-policy":
   3:   count    = var.enable_iam_db_auth
 
Invalid expression value: number required.
```

Use a conditional expression to select the count based on the variable:

```
count = var.enable_iam_db_auth ? 1 : 0
```

- Incorrect attribute value type

```
Error: Incorrect attribute value type
 
  on .terraform/modules/container-crayola/container/main.tf line 38, in resource "aws_iam_policy_attachment" "ci-attach":
  38:   users      = [var.username, var.extra_ci_users]
    |----------------
    | var.extra_ci_users is empty tuple
    | var.username is "ci-travis-crayola"
 
Inappropriate value for attribute "users": element 1: string required.
```
 
Referring to https://www.terraform.io/upgrade-guides/0-12.html#referring-to-list-variables

The solution is just `remove the redundant list brackets` or add a `flatten` function.

- Unsupported attribute for lifecycle/ignore_changes

```
Error: Unsupported attribute
 
  on .terraform/modules/balances-api_alb_target_group/alb-target-group/target-group.tf line 46, in resource "aws_alb_target_group" "main":
  46:       healthy_threshold,
 
This object has no argument, nested block, or exported attribute named
"healthy_threshold".
 
 
Error: Unsupported attribute
 
  on .terraform/modules/balances-api_task_definition/task-def-volumeless/main.tf line 8, in resource "aws_ecs_task_definition" "main":
   8:     ignore_changes        = [image]
 
This object has no argument, nested block, or exported attribute named
"image".
```

Ref: https://www.terraform.io/docs/configuration/resources.html#lifecycle-lifecycle-customizations

`ignore_changes` can only support list of attribute names, but `image` is the sub-attribute name for `container_definitions`

- Deperacated condition for lb_listener_rule

```
Warning: "condition.0.values": [DEPRECATED] use 'host_header' or 'path_pattern' attribute instead
 
  on crafty-microservice-main.tf line 30, in resource "aws_lb_listener_rule" "crafty_rule_0":
  30: resource "aws_lb_listener_rule" "crafty_rule_0" {
 
(and one more similar warning elsewhere)
 
Warning: "condition.0.field": [DEPRECATED] use 'host_header' or 'path_pattern' attribute instead
 
  on crafty-microservice-main.tf line 30, in resource "aws_lb_listener_rule" "crafty_rule_0":
  30: resource "aws_lb_listener_rule" "crafty_rule_0" {
 
(and one more similar warning elsewhere)
```

Just need to switch the format as follows:
 
**From**:

```
condition {
  field  = "path-pattern"
  values = ["*"]
}
```
 
**To**: 
 
```
condition {
  path_pattern {
    values = ["*"]
  }
}
```

**From**:
 
```
  condition {
    field  = "host-header"
    values = ["abc.hq.com"]
  }
```
 
**To**: 
 
```
condition {
  host_header {
    values = ["abc.hq.com"]
  }
}
```

This was a change introduced in terraform-providers/terraform-provider-aws#8268, which was released in AWS Provider 2.42.0.

Ref: https://github.com/brikis98/terraform-up-and-running-code/issues/43#issuecomment-569441777

Doc: https://www.terraform.io/docs/providers/aws/r/lb_listener_rule.html


- After fix
After fix syntax issues in terraform-modules, need to update modules and re-run init / upgrade when necessary:

```
# git update submodules
git submodule update --recursive --remote
 
# upgrade modules to local .terraform/modules
terraform init -upgrade
```

## 14. Plan
```
terraform plan
```

Be aware of using correct aws role to execute plan / apply for different directories.

And might also need to fix syntax errors manually for those files can not be upgraded automatically (e.g. templates)

## 15. Apply
```
terraform apply
```

## 16.Post Actions

### IDE support (VSCode)

When using vscode extension for Terraform (`v1.x`), it might show syntax error as follow:

```
Unknown token: 2:26 IDENT var.cidr
 
Peek Problem (⌥F8)
No quick fixes available
```

Need to add the following configs in VSCode `settings.json` to enable support for terraform 0.12

```
"terraform.languageServer": {
        "enabled": true,
        "args": []
    },
"terraform.indexing": {
    "enabled": false,
    "liveIndexing": false
  }
```

Unfortunately after installing the latest VSCdoe extension (e.g. `2.0.1` - which officially support Terraform 0.12) it seems NOT working properly.

Found bunch of errors:

- [Workspace not initialized](https://github.com/hashicorp/vscode-terraform/issues/329)
- [Nothing works after update to 2.x](https://github.com/hashicorp/vscode-terraform/issues/386)
- ["Server return 404" when trying to install an old version](https://github.com/microsoft/vscode/issues/99699)

The old version 1.40 used to work, so the ***workaround***:

- Remove 2.0.1 extension from VS Code
Download 1.4.0 of the vsix from [here](https://github.com/hashicorp/vscode-terraform/releases/download/v1.4.0/mauve.terraform-v1.4.0.vsix)
- Manually install it: `code --install-extension mauve.terraform-v1.4.0.vsix`
- Restart VS Code
- When prompted install language server, choose: `v.0.0.11-beta2`
- If no prompt can install manually: `⇧⌘P`, then type `Terraform: Enable/Disable Language Server` and install `v.0.0.11-beta2`

## Reference

- [Upgrade Guilds](https://www.terraform.io/upgrade-guides/0-12.html)
- [Useful 0.12 code examples](https://github.com/hashicorp/terraform-guides/tree/master/infrastructure-as-code/terraform-0.12-examples)
- [Terraform v0.12 - Syntax check broken: Unknown token IDENT](https://github.com/hashicorp/vscode-terraform/issues/195#issuecomment-578194160)
- Fix syntax issues
  - [syntax issue while upgrading from 11 to 12](https://discuss.hashicorp.com/t/syntax-issues-while-upgrading-from-11-14-to-version-12/4356)
  - [map var no longer merge when overriden](https://www.terraform.io/upgrade-guides/0-12.html#map-variables-no-longer-merge-when-overridden)
  - [refering to list var](https://www.terraform.io/upgrade-guides/0-12.html#referring-to-list-variables)