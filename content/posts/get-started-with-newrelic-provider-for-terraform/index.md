---
title: Get started with NewRelic Provider for Terraform
date: 2022-04-04T11:54:38.735Z
tags:
  - devops
  - terraform
  - newrelic
  - iac
---
## Provider Defination
```hcl
terraform {
  required_version = "~> 1.0"
  required_providers {
    newrelic = {
      source = "newrelic/newrelic"
    }
  }
}

provider "newrelic" {
  account_id = "<your account>"
  api_key    = "NRAK-<your api key>" # usually prefixed with 'NRAK'
  region     = "EU"                               # Valid regions are US and EU
}

```

## Data for New Relic
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


## Add appropriate Tags to your New Relic Application
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

## Workload Defination

resource "newrelic_workload" "workload_production" {
  name       = "Production-WorkLoad"
  account_id = data.newrelic_account.acc.account_id

  entity_search_query {
    query = "tags.accountId='${data.newrelic_account.acc.account_id}' AND tags.Environment='Production'"
  }

  scope_account_ids = [data.newrelic_account.acc.account_id]
}


## Alerts Settings
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

# Slack notification channel
resource "newrelic_alert_channel" "slack_notification" {
  name = "Slack-Notification"
  type = "slack"

  config {
    # Use the URL provided in your New Relic Slack integration
    url     = "Slack Hooks "
    channel = "proj-alerts"
  }
}

resource "newrelic_alert_policy_channel" "golden_signals" {
  policy_id   = newrelic_alert_policy.golden_signal_policy.id
  channel_ids = [newrelic_alert_channel.team_email.id, newrelic_alert_channel.slack_notification.id]
}


## Browser App-based Dashboard

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

### Add appropriate tags
resource "newrelic_entity_tags" "dashboard_website_performance_tags" {
  guid = newrelic_one_dashboard.dashboard_website_performance.guid
  tag {
    key    = "Environment"
    values = ["Production"]
  }
}


## Alerts For Traffic
```
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
```
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

```
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

```
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

