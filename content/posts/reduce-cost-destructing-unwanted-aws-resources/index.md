---
title: Reduce your cost by destructing unwanted AWS resources
date: 2021-12-19
tags:
  - process-automation
  - aws
  - cost-optimization
---

Utilize AWS-Nuke to reduce AWS cost by programmatically destroying any resources in an AWS account that are not considered Default and AWS-Managed.


Amazon Web Services (AWS) allows you to reduce costs and continuously optimize your use of computing power and storage while building modern, scalable applications. But cost optimization might be more complicated than you originally anticipated and you keep getting unexpected bills. The following are some of the main areas where you could be wasting money in the AWS:

- **Ghost Resources** - The biggest benefit of the cloud is that you can spin new resources very quickly. But, over time these resources are unintentionally forgotten and there is no automatic way to delete them. These resources can be but are not limited to unattached EBS volumes, aged EBS snapshots, aged database backups, over-provisioned, or some feature testing.

- **Engineer Sandbox Account** - A junior engineer forgot to destroy infrastructure after testing Terraform script. Or maybe the solution you're preparing is very costly and you want to delete it as soon as you test a specific part of Terraform. This is more suitable, if multi account structure is set up using AWS Organization.

We have to create ways to destroy unwanted resources so that we can save money. Let's call unwanted resources "garbage" and the tool that will destroy these resources is a garbage collector. The cloud garbage collector is [AWS-Nuke][aws-nuke], a project that provides a mechanism to destroy all garbage (unwanted resources) or any resources you want from your account(s).

> **Caution!** _Be aware that "aws-nuke" is a very destructive tool, hence you have to be very careful while using it. Otherwise, you might destroy production data._

"AWS-Nuke" is a very extensive tool that lets you use a declarative approach to destroy AWS resources. At its core, AWS-Nuke makes API calls to AWS resources on your behalf. The command `aws-nuke resource-types` can help you review the full list of supported resources in an account.

Also, AWS-Nuke uses a YAML configuration. This configuration contains the rules of the whitelist and the resources that you want to keep.

```yaml
regions:
- eu-west-1
- global
account-blocklist:
- "999999999999" # production
accounts:
  "000000000000": # aws-nuke-example
    filters:
      IAMUser:
      - "my-user"
      IAMUserPolicyAttachment:
      - "my-user -> AdministratorAccess"
      IAMUserAccessKey:
      - "my-user -> ABCDEFGHIJKLMNOPQRST"
```

That configuration is like that:

- Only scan the `eu-west-1` instance and include resources that are global.
- `account-blocklist` excludes a defined account to scan or destroy any resources in it. In our case, the production account is excluded.
- In `accounts`, there are criteria to ignore resources in Nuke. Nuke will destroy everything from the mentioned account except resources that match filters. In the given configuration, user has access keys with the name "ABCDEFGHIJKLMNOPQRST" and a policy attached (AdministratorAccess).

With this config we can run `aws-nuke`:

```shell
aws-nuke -c config/nuke-config.yml --profile aws-nuke-example
```

We can execute `aws-nuke` in docker by using the following command:

```shell
docker run \
  --rm -it \
  -v /full-path/to/nuke-config.yml:/home/aws-nuke/config.yml \
  -v /home/user/.aws:/home/aws-nuke/.aws \
  quay.io/rebuy/aws-nuke:v2.11.0 \
  --profile default \
  --config /home/aws-nuke/config.yml
```

This command was last updated for the v2.11.0 version of `aws-nuke`. Take a look at the latest releases to update the docker image tag.

By default, `aws-nuke` will not destroy any resource when you run this command. It will show you a list of all garbage (unwanted resources). You have to add `--no-dry-run` actually to destroy the garbage (unwanted resources). It will ask you twice to confirm before running a nuke on your Amazon Web Services account.

The above example shows you an ad hoc way to run "aws-nuke", but it is more effective to run in some schedule and automation to deploy it. There is a project [1Strategy/automated-aws-multi-account-cleanup][aws-nuke-automation]. This project uses an Organization Unit (OU) to identify the list of accounts and then iterate over accounts to destroy garbage (unwanted resources) based on defined criteria from the configuration.

>“Cost optimization is a continual process of refinement and improvement over the span of a workload’s lifecycle.”
>
> &mdash;<cite>[AWS Cost Optimization][aws-cost-optimization]</cite>
<!-- You can use a variety of tools and approaches to analyze and optimize the cost of your AWS resources. You can choose the cost optimization techniques that are appropriate for your business and workload. -->

Cost optimization can be complicated to achieve, there are various tools and approaches available to analyze and optimize the cost of your AWS resources. You can choose the cost optimization techniques that are more suitable for your workload and business.

AWS-Nuke can help you to reduce your cost by destructing unwanted resources. Use the configuration to define your whitelist with filters and presets. For more information and configuration details, take a look at [AWS-Nuke][aws-nuke].

[aws-nuke]: https://github.com/rebuy-de/aws-nuke
[soc]: https://en.wikipedia.org/wiki/Separation_of_concerns
[aws-cost-optimization]: https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/welcome.html
[aws-nuke-automation]: https://github.com/1Strategy/automated-aws-multi-account-cleanup