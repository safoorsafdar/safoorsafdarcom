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

* `required_version` is your Terraform version validation. That ensures your terraform code syntax matches with the installed Terraform version. You can your terraform version with `terraform -v`. 
* `required_providers`.`source` is the name of NewRelic providers. That brings the new relic provider into terraform to interact with the new relic.
* `required_providers`.`version` ensure the new relic provider version that you wish to use. 



## Configure New Relic Provider

```hcl
provider "newrelic" {
  account_id = "<your account>"
  api_key    = "NRAK-<your api key>" # usually prefixed with 'NRAK'
  region     = "EU" # Valid regions are US and EU
}
 
```

## Data for New Relic

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

## Add appropriate Tags to your New Relic Application

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

## Workload Definition

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

## Alerts Settings

```hcl
resource "newrelic_alert_policy" "golden_signal_policy" {
  name = "GoldenSignal-ManagedPolicy"
}

resource "newrelic_alert_channel" "team_email" {
  name = "Email-Notification"
  type = "email"

  config {
    recipients              = "<Notification Email>"
    include_json_attachment = "1"
  }
}
```

# Slack notification channel

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

resource "newrelic_alert_policy_channel" "golden_signals" {
  policy_id   = newrelic_alert_policy.golden_signal_policy.id
  channel_ids = [newrelic_alert_channel.team_email.id, newrelic_alert_channel.slack_notification.id]
}
```

## Browser App-based Dashboard

```hcl
resource "newrelic_one_dashboard" "dashboard_website_performance" {
  name = "DCOP-WebsitePerformance"

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

### Add appropriate tags

```hcl
resource "newrelic_entity_tags" "dashboard_website_performance_tags" {
  guid = newrelic_one_dashboard.dashboard_website_performance.guid
  tag {
    key    = "Environment"
    values = ["Production"]
  }
}
```

## Alerts For Traffic

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

## Alerts for Saturation

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

## Alerts for Latency

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

## Alerts for Error

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