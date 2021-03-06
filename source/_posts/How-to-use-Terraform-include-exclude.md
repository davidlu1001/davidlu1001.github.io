---
title: How to use Terraform include / exclude with Makefile
tags:
  - Terraform
  - SRE
  - DevOps
  - Makefile
categories:
  - - SRE
  - - DevOps
  - [Terraform]
abbrlink: 3cbb6765
date: 2019-08-13 21:59:05
---

# Background

When using make plan command, if there're too many resources need to be applied, together with other unrelated resources that we don't want to touch, then we need to use Terraform [resource targeting](https://www.terraform.io/docs/commands/plan.html#resource-targeting):

```
PLAN_OPTIONS="-target="A" -target="B" ... -target="N"" make plan
```

But Terraform [doesn't support --exclude feature](https://github.com/hashicorp/terraform/issues/2253) for the target at the moment (and we don't want to copy & paste over 50 times for the targets), that's why need to find a way to implement `exclude / include` features within Makefile.

<!-- more -->

# How-to

- make plan

Run `make plan` to show pending changes, also generate `current.plan` that we use later to filter targets.

- filter targets

Then pick the resources you want by using `[INCLUDE | EXCLUDE]` which support POSIX Extended Regular Expressions (ERE)

E.g.

```
# use include to specify only the targets you want to apply
$ make plan_include INCLUDE='mark3a'
 
# output
terraform plan -out current.plan    -target="aws_alb_target_group_attachment.routers-mark3a" -target="module.routers-mark3a.aws_instance.main" -target="module.routers-mark3a.aws_route53_record.main_instance"
 
# exclude the targets
$ make plan_exclude EXCLUDE='aws_security_group|mark3b'
 
# output
terraform plan -out current.plan    -target="aws_alb_target_group_attachment.routers-mark3a" -target="aws_alb_target_group_attachment.routers-mark3c" -target="module.asg_databus_production.aws_autoscaling_group.main" -target="module.asg_router_production.aws_autoscaling_group.main" -target="module.asg_router_production.aws_launch_configuration.main" -target="module.routers-mark3a.aws_instance.main" -target="module.routers-mark3a.aws_route53_record.main_instance" -target="module.routers-mark3c.aws_instance.main" -target="module.routers-mark3c.aws_route53_record.main_instance"
```

- make apply

Finally can use `make apply` to apply the changes based on new `current.plan`

# Explain

Show the output generated by `make plan`, then process the output (e.g. clear ANSI color / match resource names after empty line etc.) to generate the final format (e.g. -target="A" -target="B" -target="C" ...)

in Makefile:

```
# For Terraform 0.11
define PLAN_OPTIONS_EXCLUDE
	$(shell terraform show current.plan | perl -pe 's//\n/ if $$. == 1' | perl -pe 's/\x1b\[[0-9;]*[mG]//g' | perl -ne 'if ($$p) { print unless /^$$/; $$p = 0 } $$p++ if /^$$/'| awk '{print $$2}' | grep -E -v '$(1)' | sed -e 's/^/-target="/g' -e 's/$$/"/g' | awk BEGIN{RS=EOF}'{gsub(/\n/," ");print}')
endef

define PLAN_OPTIONS_INCLUDE
	$(shell terraform show current.plan | perl -pe 's//\n/ if $$. == 1' | perl -pe 's/\x1b\[[0-9;]*[mG]//g' | perl -ne 'if ($$p) { print unless /^$$/; $$p = 0 } $$p++ if /^$$/'| awk '{print $$2}' | grep -E '$(1)' | sed -e 's/^/-target="/g' -e 's/$$/"/g' | awk BEGIN{RS=EOF}'{gsub(/\n/," ");print}')
endef

# For Terraform 0.12 (using -no-color to avoid dealing with terminal color)
define PLAN_OPTIONS_EXCLUDE
	$(shell terraform show -no-color current.plan | perl -nle 'if (/\s# (.*?)\s/) {print $$1}' | grep -E -v '$(1)' | sed -e 's/^/-target="/g' -e 's/$$/"/g' | xargs)
endef

define PLAN_OPTIONS_INCLUDE
	$(shell terraform show -no-color current.plan | perl -nle 'if (/\s# (.*?)\s/) {print $$1}' | grep -E '$(1)' | sed -e 's/^/-target="/g' -e 's/$$/"/g' | xargs)
endef
```

and then the macro `PLAN_OPTIONS_EXCLUDE` / `PLAN_OPTIONS_INCLUDE` can be triggered by `plan_exclude` / `plan_include` in Makefile:

```
plan_exclude:
  terraform plan -out current.plan $(strip $(call PLAN_OPTIONS_EXCLUDE,$(EXCLUDE)))

plan_include:
  terraform plan -out current.plan $(strip $(call PLAN_OPTIONS_INCLUDE,$(INCLUDE)))
```

# Gist

Final gist could be find [here](https://gist.github.com/davidlu1001/e832038299fff99d4a4b2c6a75d71b78)

# Limitations

Unfortunately, when the target list becomes very long (e.g. 49) You start to see `Too many command line arguments. Configuration path expected`, so this workaround is not enough...

Anyway Inverse targeting is becoming more and more important / necessary in huge terraform plans with lots of modules managed by different teams, so hope there's an official solution for that in the new future.