---
title: Get started with NewRelic Provider for Terraform
date: 2022-04-02T12:56:01.540Z
tags:
  - devops
  - terraform
  - newrelic
  - iac
---


```hcl
# Configure terraform
terraform {
  required_version = "~> 1.0"
  required_providers {
    newrelic = {
      source  = "newrelic/newrelic"
    }
  }
}
```

**What is happening here?**

`required_version` is *your* Terraform version - ensure it matches. You can run `terraform -v` to check.

`source` is the New Relic Terraform provider. This brings the NR TF providers into your scripts to be interacted with.

`version` within the NewRelic block is the version of the New Relic provider that you wish to use.

```hcl
# Configure the New Relic provider
provider "newrelic" {
  api_key = "<New Relic key>"
  account_id    = <account ID>
  region = "US"
}
```

**What is happening here?**

Giving our authentication credentials lets our script interact through the provider. Switch region to "EU" if you use the New Relic EU platform - i.e. login through one.eu.newrelic.com

```hcl
# Read an APM application resource
data "newrelic_entity" "node-workshop-app" {
  name = "Node Workshop"
  domain = "APM"
  type = "APPLICATION"
}
```

**What is happening here?**

Here, we are *reading* a resource from our New Relic account. Where we have `"node-workshop-app"` - this is the name locally within Terraform. We can give it any name we like, it is how we will reference this object.
Where we have name `"Node Workshop"` - this is the name of the entity within New Relic. 
By first fetching this resource, we can then reference it in our Terraform to do things such as apply an alert condition to it.

```hcl
# Create an alert policy
resource "newrelic_alert_policy" "node-alerts" {
  name = "Demo Node Alerts via Terraform"
}
```

**What is happening here?**

This will create the resource of an alert policy. The `"node-alerts"` is the local reference within Terraform.
The `name` the attribute is the name for the alert policy as seen in New Relic.

```hcl
# Add a condition
resource "newrelic_nrql_alert_condition" "node-workshop-slow-txn" {
  policy_id                    = newrelic_alert_policy.node-alerts.id
  type                         = "static"
  name                         = "Slow Transactions"
  description                  = "Alert when transactions are taking too long"
  runbook_url                  = "https://www.example.com"
  enabled                      = true
  value_function               = "single_value"
  violation_time_limit_seconds = 3600

  nrql {
    query             = "SELECT average(duration) FROM Transaction where appName = '${data.newrelic_entity.node-workshop-app.name}'"
    evaluation_offset = 3
  }

  critical {
    operator              = "above"
    threshold             = 5.5
    threshold_duration    = 300
    threshold_occurrences = "ALL"
  }
}
```

**What is happening here?**

First up we are specifying the name for the condition locally. `"node-workshop-slow-txn"` is how we can reference this later.
We are also saying the `policy_id` that the condition belongs to, because in New Relic a condition must belong to a policy. We will reference this to the policy we created earlier `node_alerts`.
In the `query` you can see that to make our NRQL useful it is helpful to reference the name of the APM application we called earlier.
The rest is meta information we need to define the condition.

At this point if we perform `terraform apply` (assuming a `terraform init` happened some point earlier) you can see that Terraform is ready to perfor two actions, that is creating the alert policy we defined, and create the condition we defined.

```hcl
# Add a notification channel
resource "newrelic_alert_channel" "email-gary" {
  name = "email Gary Spencer"
  type = "email"

  config {
    recipients              = "gspencer@newrelic.com"
    include_json_attachment = "1"
  }
}
```

**What is happening here?**

We are creating a notification channel. We are giving it locally within Terraform the reference `"email-gary"`. 
Within the definition we give it a New Relic facing name `"email Gary Spencer"`.
We can create this standalone because in New Relic a notification channel can exist stand alone and does not have to be associated with any notification channel - it is also the case that a notification channel can be attached to many policies.
The rest is the meta information necessary to create a notification channel of type email.

And now if we perform a `terraform plan` you can see that three actions are ready to be performed by terraform. The two earlier and now the creation of a notification channel.

```hcl
# Link the channel to the policy
resource "newrelic_alert_policy_channel" "email-to-gary" {
  policy_id  = newrelic_alert_policy.node-alerts.id
  channel_ids = [
    newrelic_alert_channel.email-gary.id
  ]
}
```

**What is happening here?**
Now to link the notification channel to the policy created earlier. You can see that we reference the `"email-gary"` channel in the resource and again in `newrelic_alert_channel.id`. 
We then also reference the earlier created policy_id. This creates the mapping

If we now perform `terraform apply` you can see that four actions are ready to be performed. The three earlier, followed by assigning this notification channel to this policy.

*Why do you not link a condition to a policy in the same way?*
When creating a condition, the policy must live in the meta config of the condition. This is because (unlike alert channels) conditions cannot exist standalone to be later assigned.

If after a `terraform apply` you are happy with the stated actions, you can perform `terraform apply` which will actually make the changes in New Relic. New Relic will also perform some checks at this stage too.

You should then see *Apply complete!* and the changes made in New Relic.

```hcl
# Add a Simple Ping Synthetics monitor
resource "newrelic_synthetics_monitor" "node-workshop-home-synthetics" {
  name = "Node Workshop Homepage"
  type = "SIMPLE"
  frequency = 60
  status = "ENABLED"
  locations = ["AWS_US_EAST_1", "AWS_EU_WEST_2"]                # Locations at: https://docs.newrelic.com/docs/synthetics/synthetic-monitoring/administration/synthetic-public-minion-ips#location

  uri                       = "http://nodeworkshop.eu-west-2.elasticbeanstalk.com/"               # Required for type "SIMPLE" and "BROWSER"
  validation_string         = "welcome to the New Relic Node Workshop"                            # Optional for type "SIMPLE" and "BROWSER"
  verify_ssl                = false                                                               # Optional for type "SIMPLE" and "BROWSER"
}
```

**What is happening here?**

Here we are making a new basic Synthetic ping check. Again we have the local reference and the name as appears in New Relic.
There is also meta information necessary to create the script such as locations, frequency, and the URL to be checked.

## Deleting what you made

We can test destruction by performing `terraform plan -destroy`

If you are happy with the actions listed then you can perform `terraform destroy` and you should get the same success notification.