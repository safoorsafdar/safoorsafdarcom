---
title: Get started with NewRelic Provider for Terraform
date: 2022-04-04T13:12:33.021Z
tags:
  - devops
  - terraform
  - newrelic
  - iac
---
[Terraform](https://www.terraform.io/) is a popular infrastructure-as-code software tool built by HashiCorp. You can use it to provision all kinds of infrastructure and services, including New Relic dashboards and alerts.

On the other hand, New Relic is a web application performance service designed to work in real-time with your live web app. 

In this get started guide, you can learn how to set up New Relic. More specifically, you provision an alert policy, alert conditions, alert channels, and Dashboard. 

## Before you begin

To use this guide, you should have some basic knowledge of both New Relic and Terraform. If you haven't deployed a New Relic open-source agent yet, [install New Relic](https://docs.newrelic.com/docs/agents/manage-apm-agents/installation/install-agent) for your application. Also, [install the Terraform CLI](https://learn.hashicorp.com/collections/terraform/cli).

## Bootstrap Terraform

`mkdir your-project && cd your-project`

`touch main.tf`

Now, let's instruct Terraform to install and use the NewRelic provider by setting the `terraform` and `required_providers` block in `main.tf`

```hcl
terraform {
  # Require Terraform version 1.x.x (recommended)
  required_version = "~> 1.0"
  
  # Require the latest 2.x version of the New Relic provider
  required_providers {
    newrelic = {
      source = "newrelic/newrelic"
      version = "~> 2.0"
    }
  }
}
```

**What is happening here?**

* `required_version` is your Terraform version validation. That ensures your terraform code syntax matches with the installed Terraform version. You can check your terraform version with `terraform -v`. 
* `required_providers.source` is the name of NewRelic providers. That brings the new relic provider into terraform to interact with the new relic.
* `required_providers.version` ensures the new relic provider version that you wish to use. 

## Configure New Relic Provider

With Terraform all set, let's configure the New Relic `provider` with the following items:

```hcl
provider "newrelic" {
  account_id = "<your account>"
  api_key    = "NRAK-<your api key>" # usually prefixed with 'NRAK'
  region     = "EU" # Valid regions are US and EU
}
 
```

* `account_id` - Your New Relic Account ID. Visit to learn more https://docs.newrelic.com/docs/accounts/install-new-relic/account-setup/account-id
* `api_key` - Your Personal New Relic API Key. Visit to learn more https://docs.newrelic.com/docs/apis/get-started/intro-apis/types-new-relic-api-keys#user-api-key
* `region` - Valid regions are the US and EU. Your region is `US` if the New Relic page is located at `one.newrelic.com` and `EU` if your account is located at `one.eu.newrelic.com`

You can also configure the New Relic provider using the environment variable. This is a useful way to set default values instead of hard coding into code and publishing it to the repository. 

The table below shows the available environment variables equivalent to attributes and all of these are required attributes. 

| Schema Attribute | Equivalent Env Variable |
| ---------------- | ----------------------- |
| account_id       | NEW_RELIC_ACCOUNT_ID    |
| api_key          | NEW_RELIC_API_KEY       |
| region           | NEW_RELIC_REGION        |

With your new relic provider configured, initialize the Terraform:
`terraform init`

## Reference your New Relic Application in Terraform

With a New Relic provider configured and initialized, you can define various resources for your application. 

As you will be targeting a specific application. You can use `newrelic_entity` data to fetch information from New Relic to reference in terraform code. 

```hcl
data "newrelic_entity" "app_apm" {
  name   = "<Your APM application Name>" # Must be an exact match to your application name in New Relic
  domain = "APM"        # or BROWSER, INFRA, MOBILE, SYNTH, depending on your entity's domain
  type   = "APPLICATION"
}

data "newrelic_entity" "app_browser" {
  name   = "<Your Browser application name>" # Must be an exact match to your application name in New Relic
  domain = "BROWSER"                   # or BROWSER, INFRA, MOBILE, SYNTH, depending on your entity's domain
  type   = "APPLICATION"
}

data "newrelic_account" "acc" {
  scope      = "global"
  account_id = "<your account>"
}
```

* `newrelic_entity.app_apm` is to fetch information for APM New Relic Application. 
* `newrelic_entity.app_browser` is to fetch information for Browser New Relic as in my case we have APM and Browser application on New Relic. 
* `newrelic_account` is to get information about the New Relic account so you can reference it later in your code. 

> **Info!** At this point, you should be able to test your terraform code with a dry run: `terraform plan`, as a response to the `plan` command, you should see Terraform execution plan.

It's considered a best practice to tag all your resources on Cloud. Similarly, we can tag our resources on New Relic with `newrelic_entity_tags`. Let's tag our APM and Browser New Relic application. 

```hcl
resource "newrelic_entity_tags" "app_apm_tags" {
  guid = data.newrelic_entity.app_apm.guid
  tag {
    key    = "Environment"
    values = ["Production"]
  }
}

resource "newrelic_entity_tags" "app_browser_tags" {
  guid = data.newrelic_entity.app_browser.guid
  tag {
    key    = "Environment"
    values = ["Production"]
  }
}
```

## Create a New Relic alert Policy

```hcl
resource "newrelic_alert_policy" "golden_signal_policy" {
  name = "Golden Signal - Managed Policy "
}
```
- `name` is the name of the Alert Policy. 

> **Info!** At this point, you can apply your terraform code with `terraform apply`. Every time you `apply` changes, Terraform asks you to confirm the actions you've told it to run. Type "yes".

## Provision alert conditions for defined Alert Policy

Next, add alert conditions for your application based on the four golden signals: latency, traffic, errors, and saturation. Apply these alert conditions to the alert policy you created in the previous step.

### Traffic

```hcl
# Low throughput
resource "newrelic_alert_condition" "throughput_web" {
  policy_id       = newrelic_alert_policy.golden_signal_policy.id
  name            = "Low Throughput (Web)"
  type            = "apm_app_metric"
  entities        = [data.newrelic_entity.app_apm.application_id]
  metric          = "throughput_web"
  condition_scope = "application"

  # Define a critical alert threshold that will
  # trigger after 5 minutes below 5 requests per minute.
  term {
    priority      = "critical"
    duration      = 5
    operator      = "below"
    threshold     = "5"
    time_function = "all"
  }
}
```

### Saturation

```hcl
# High CPU usage
resource "newrelic_infra_alert_condition" "high_cpu_utils" {
  policy_id   = newrelic_alert_policy.golden_signal_policy.id
  name        = "${local.alarm_label_prefix}:High_CPU_Utilisation"
  type        = "infra_metric"
  event       = "SystemSample"
  select      = "cpuPercent"
  comparison  = "above"
  runbook_url = "https://www.example.com"
  where       = "(`applicationId` = '${data.newrelic_entity.app_akgalleria.application_id}')"

  # Define a critical alert threshold that will trigger after 5 minutes above 90% CPU utilization.
  critical {
    duration      = 5
    value         = 90
    time_function = "all"
  }
}
```

### Latency

```hcl
# Response time
resource "newrelic_alert_condition" "response_time_web" {
  policy_id       = newrelic_alert_policy.golden_signal_policy.id
  name            = "High Response Time (Web) - ${data.newrelic_entity.app_akgalleria.name}"
  type            = "apm_app_metric"
  entities        = [data.newrelic_entity.app_akgalleria.application_id]
  metric          = "response_time_web"
  runbook_url     = "https://www.example.com"
  condition_scope = "application"

  term {
    duration      = 5
    operator      = "above"
    priority      = "critical"
    threshold     = "5"
    time_function = "all"
  }
}
```

### Errors

```hcl
# Error percentage
resource "newrelic_alert_condition" "error_percentage" {
  policy_id       = newrelic_alert_policy.golden_signal_policy.id
  name            = "High Error Percentage"
  type            = "apm_app_metric"
  entities        = [data.newrelic_entity.app_akgalleria.application_id]
  metric          = "error_percentage"
  runbook_url     = "https://www.example.com"
  condition_scope = "application"

  # Define a critical alert threshold that will trigger after 5 minutes above a 5% error rate.
  term {
    duration      = 5
    operator      = "above"
    threshold     = "5"
    time_function = "all"
  }
}
```

## Get notified when an alert triggers

There are multiple ways available from New Relic to get notified when an alert triggers. For starters, we can use Email and Slack notifications. 

```hcl
resource "newrelic_alert_channel" "team_email" {
  name = "Email-Notification"
  type = "email"

  config {
    recipients              = "<Notification Email>"
    include_json_attachment = "1"
  }
}
```

If you want to specify multiple recipients, use a comma-delimited list of emails.

```hcl
resource "newrelic_alert_channel" "slack_notification" {
  name = "Slack-Notification"
  type = "slack"

  config {
    # Use the URL provided in your New Relic Slack integration
    url     = "<Slack Hooks>"
    channel = "proj-alerts"
  }
}
```

You need to add New Relic Slack App to your Slack account. and select the Slack channel to send the notification. 

Last, but not least, you need to associate them with the respective New Relic alert policy. Create `newrelic_alert_policy_channel`.

```hcl
resource "newrelic_alert_policy_channel" "golden_signals" {
  policy_id   = newrelic_alert_policy.golden_signal_policy.id
  channel_ids = [newrelic_alert_channel.team_email.id, newrelic_alert_channel.slack_notification.id]
}
```

Currently, I am not able to find a possible way to segregate the alerts based on priority. For Example, send warning notifications to the Slack channel and critical notifications to the email channel. 

One way might be possible to achieve mentioned problem, We can separate alert policies for warning and critical and associate channels according to alert policy. 

## Browser Dashboard
We can also provision the New Relic dashboard with `newrelic_one_dashboard`.

```hcl
resource "newrelic_one_dashboard" "dashboard_website_performance" {
  name = "Website Performance"

  page {
    name = "Sessions"

widget_line {
  title  = "Unique User Sessions"
  row    = 1
  column = 1

  nrql_query {
    query = "FROM PageView SELECT uniqueCount(session) WHERE appName='${data.newrelic_entity.app_browser.name}' TIMESERIES"
  }
}

widget_markdown {
  title  = "Dashboard Note"
  row    = 1
  column = 9

  text = "### Helpful Links\n\n* [New Relic One](https://one.newrelic.com)\n* [Developer Portal](https://developer.newrelic.com)"
}
  }
}
```

We are creating "Website Performance" with the example "Unique User Sessions" and the markdown widget. 

and add the appropriate tag(s) to the dashboard:

```hcl
resource "newrelic_entity_tags" "dashboard_website_performance_tags" {
  guid = newrelic_one_dashboard.dashboard_website_performance.guid
  tag {
    key    = "Environment"
    values = ["Production"]
  }
}
```

## Organize New Relic resources in Work Load

```hcl
resource "newrelic_workload" "workload_production" {
  name       = "Production-WorkLoad"
  account_id = data.newrelic_account.acc.account_id

  entity_search_query {
    query = "tags.accountId='${data.newrelic_account.acc.account_id}' AND tags.Environment='Production'"
  }

  scope_account_ids = [data.newrelic_account.acc.account_id]
}
```

This will provision New Relic Work Load with the 

* `name` of "Production-WorkLoad" 
* `account_id` on the defined account 
* and `entity_search_query` would be based on environment tags we defined in the implementation of our resources with `newrelic_entity_tags`. 

You may also want to consider automating this process in your CI/CD pipeline.  Use Terraform's [recommended practices guide](https://www.terraform.io/docs/cloud/guides/recommended-practices/index.html) to learn more about their recommended workflow and how to evolve your provisioning practices.

Congratulations! You're officially practicing observability-as-code. Review the [New Relic Terraform provider documentation](https://registry.terraform.io/providers/newrelic/newrelic/latest/docs) to learn how you can take your configuration to the next level.