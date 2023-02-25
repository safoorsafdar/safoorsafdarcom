---
title: How to provision Kubernetes/Container Engine with Terraform on Oracle
  Cloud (OCI)
date: 2023-02-25T19:41:23.269Z
tags:
  - oci
  - kubernetes
  - devops
  - oke
  - oracle-cloud
---
Kubernetes has become a critical component in modern cloud infrastructure, providing a scalable and flexible platform for deploying and managing containers. However, setting up a Kubernetes cluster on Oracle Cloud (OCI) can be a challenge, especially for those new to the platform.

This article presents a comprehensive, step-by-step guide to effortlessly provisioning a Kubernetes/OKE cluster on OCI using Terraform. Whether you are an experienced cloud professional or just starting with Terraform and OCI, this guide is designed to streamline the provisioning process and simplify the deployment of your Kubernetes cluster.

With its clear and concise instructions, this article will be an indispensable resource for anyone looking to deploy a robust and scalable Kubernetes environment on OCI.

IAC (Infrastructure as Code) is a concept for managing infrastructure through code instead of manual processes. Terraform is a popular open-source tool that implements IAC, allowing for the automation of infrastructure provisioning and management across multiple cloud providers and on-premises data centers. It uses a declarative language to describe the desired state of the infrastructure.

The Terraform Oracle Cloud provider is a plugin for Terraform that enables management of resources in the Oracle Cloud Infrastructure using Terraform. It is open-source, allowing for customization and community contributions.

## Dive into the OKE OCI Terraform module

The OKE enables users to easily create and manage a Kubernetes cluster on OCI using Terraform. 

**🔗 The terraform module for OKE on OCI is open-source and can be found on the Oracle Cloud [GitHub](https://github.com/oracle-terraform-modules/terraform-oci-oke) repository.**

This terraform module provisions a comprehensive Kubernetes environment on Oracle Cloud (OCI) utilizing Terraform, creating the following resources by default:

1. Virtual Cloud Network (VCN) with Internet, NAT, and Service Gateways, as well as associated route tables.
2. A regional public subnet for the bastion host and its security list, as well as a regional private subnet for the operator host and its Network Security Group (NSG).
3. A public control plane subnet and a private regional worker subnet.
4. A public regional load balancer, a bastion host, and an private operator host.
5. A public Kubernetes Cluster with private worker nodes and NSGs for each of the control plane, workers, and load balancers.

Please note that the Kubernetes Control Plane Nodes run within Oracle's tenancy and are not displayed in the deployment.

While private clusters are recommended, this guide currently defaults to public deployment to allow users sufficient time to adjust other configurations, such as their CI/CD tools, etc.

## The important terraform input options to review

Terraform inputs and outputs are used to manage the configuration and state of infrastructure as code. Here are some important inputs and outputs related to Terraform:

**OCI Provider Parameters**: This input sets the parameters for the Oracle Cloud Infrastructure provider, including the tenancy, user, and fingerprint for the API key.

**SSH Keys:** This input is used to specify the SSH public key that Terraform will use to access instances for provisioning and configuration management.

**Node Pools:** This input defines the properties and parameters of the node pool, including size, shape, and image.

**OCIR:** This input sets the parameters for the Oracle Cloud Infrastructure Registry (OCIR) provider, including the tenancy, user, and fingerprint for the API key.

👉There are several option related to terraform inputs & outputs. If you want to review more check  [Terraform Options](https://github.com/oracle-terraform-modules/terraform-oci-oke/blob/main/docs/terraformoptions.adoc) *.*

## Getting Started Guide: Fast and Easy Examples

This section presents a step-by-step, quick start example to help you get up and running with Oracle Cloud Infrastructure (OCI) Container Engine for Kubernetes (OKE) using Terraform. This guide includes Terraform configuration snippets that demonstrate how to create and manage an OKE cluster using the Terraform Registry module.

* In your project root, create a `provider.tf` file.  

```yaml
provider "oci" {
  fingerprint      = var.api_fingerprint
  private_key      = var.api_private_key
  region           = var.region
  tenancy_ocid     = local.tenancy_id
  user_ocid        = local.user_id
}

provider "oci" {
  fingerprint      = var.api_fingerprint
  private_key      = var.api_private_key
  region           = var.home_region
  tenancy_ocid     = local.tenancy_id
  user_ocid        = local.user_id
  alias = "home"
}
```

* In your project root, create a `variables.tf` file and add variables for your project. You can copy the existing [variables.tf](https://github.com/oracle-terraform-modules/terraform-oci-oke/blob/main/variables.tf) in the OKE module root.

```yaml
# OCI Provider parameters
variable "api_fingerprint" {
  default     = ""
  description = "Fingerprint of the API private key to use with OCI API."
  type        = string
}

variable "api_private_key_path" {
  default     = ""
  description = "The path to the OCI API private key."
  type        = string
}

variable "home_region" {
  # List of regions: https://docs.cloud.oracle.com/iaas/Content/General/Concepts/regions.htm#ServiceAvailabilityAcrossRegions
  description = "The tenancy's home region. Required to perform identity operations."
  type        = string
}

# Automatically populated by Resource Manager
variable "region" {
  # List of regions: https://docs.cloud.oracle.com/iaas/Content/General/Concepts/regions.htm#ServiceAvailabilityAcrossRegions
  description = "The OCI region where OKE resources will be created."
  type        = string
}

# Overrides Resource Manager
variable "tenancy_id" {
  description = "The tenancy id of the OCI Cloud Account in which to create the resources."
  type        = string
  default     = ""
}

# Overrides Resource Manager
variable "user_id" {
  description = "The id of the user that terraform will use to create the resources."
  type        = string
  default     = ""
}

# General OCI parameters

# Overrides Resource Manager
variable "compartment_id" {
  description = "The compartment id where to create all resources."
  type        = string
  default     = ""
}
```

* In your project root, create a `versions.tf` file and add the following:

```yaml
terraform {
  required_providers {
    oci = {
      source                = "oracle/oci"
      configuration_aliases = [oci.home]
      version               = ">= 4.67.3"
    }
  }
  required_version = ">= 1.0.0"
}
```

* In your project root, create a main.tf file and add the following:

```yaml
module "oke" {
  source                                =   "oracle-terraform-modules/oke/oci"
  version                               =   "4.2.4"

  compartment_id                        =   var.compartment_id
  tenancy_id                            =   var.tenancy_id

  ssh_private_key_path                  =   var.ssh_private_key_path
  ssh_public_key_path                   =   var.ssh_public_key_path

  label_prefix                          =   var.label_prefix
  home_region                           =   var.home_region
  region                                =   var.region

  # add additional parameters for availability_domains, oke etc as you need

  providers = {
    oci.home = oci.home
  }
}
```

👆The above example is just a starting point and may require customization to meet the unique needs and requirements of your project.

Please note that as of the writing of this article, the available version of the OKE Terraform module is 4.2.4. It is advisable to verify the official version before utilizing this module in your project.

At this point, if you apply your Terraform changes, your OKE cluster will be provisioned and can be viewed in the Oracle Cloud Console under the Container Engine. Remember to select the correct compartment. However, it will not include any worker nodes to run your workloads. To attach worker nodes to your Kubernetes cluster, you can use the terraform input mentioned below.

```yaml
module "oke" {
# ...

node_pools = {
	np1 = {
	    shape            = "VM.Standard.E4.Flex",
	    ocpus            = 2,
	    memory           = 32,
	    node_pool_size   = 1,
	    boot_volume_size = 150,
	}
}

# ....
```

The code uses the **`node_pools`** variable to specify the properties of the worker node pool, with the pool being named **`np1`**.

The properties being defined include:

* **`shape`**: The shape of the compute instance to be used for the worker nodes, in this case **`VM.Standard.E4.Flex`**.
* **`ocpus`**: The number of OCPUs to be allocated to each worker node, in this case 2.
* **`memory`**: The amount of memory to be allocated to each worker node, in this case 32 GB.
* **`node_pool_size`**: The number of worker nodes to be created in this node pool, in this case 1.
* **`boot_volume_size`**: The size of the boot volume to be created for each worker node, in this case 150 GB.

Before proceeding with the deployment if you have not already, it is recommended to initialize Terraform and review the resource plan. You can do so by using the following command: 

```shell
terraform init
terraform plan
terraform apply
```

In addition to the default configuration offered by the OKE Terraform module, there are alternative options for provisioning and setting up a Kubernetes cluster with the Oracle Cloud Container Engine.

### Deploying a public Kubernetes controller plane

A public cluster is one that has no access restrictions. Users can access this cluster from anywhere. 

You can configure a Kubernetes cluster to be public and restrict its access to the CIDR blocks A.B.C.D/A and X.Y.Z.X/Z by setting the following parameters:

`control_plane_type = "public"
 control_plane_allowed_cidrs = ["A.B.C.D/A","X.Y.Z.X/Z"]`

A Kubernetes API deployed in public mode has a publicly accessible endpoint, while a private mode deployment limits access to only the operator host.

### Deploying public or private worker nodes

When creating the Worker nodes, you have the option to choose between public or private subnets. The following parameter can be used to make this determination.

`worker_type = "private" # or public`

Deploying worker nodes in public mode results in the creation of public subnets and a direct connection to the Internet Gateway. Each worker node will have both private and public IP addresses. The private IP addresses are assigned from the worker subnet's range, while the public IP addresses come from Oracle's pool of public IP addresses.

To ensure proper security configuration and allow NodePort access, enabling of NodePort and SSH access is required. You can use below mentioned terraform variable

`allow_node_port_access = true
 allow_worker_ssh_access = true`

When you deploy the worker subnet in private mode, it will be deployed as a private subnet and route to the NAT Gateway.

Controlling the internet access for worker nodes and pods can be accomplished through the following options.

`allow_worker_internet_access = true`

`allow_pod_internet_access = true`

## Accessing the provisioned OKE cluster

After OKE is provisioned, accessing the cluster is another important step because it provides the ability to manage and maintain the resources in the cluster. With access to the cluster, administrators can perform several tasks.

![](https://lh5.googleusercontent.com/Ug0wYWqVunn5RiGgNFDSrA1w6QLb1fa-AP_V2bSz51b5zvOy387ruwMteZkmtdIWUeJ1eltYQ2SmECMn3p4UqlD5PdeB6Qq2Ea3Ou-YWY6ITiV1b3RdM4uCsp_kWkXlvVG2kmNo5jFNdDHC2ekfx4MU)

Restrict access to the bastion host by specifying a list of CIDR blocks in the `bastion_access`
parameter. By default, access is open from any location.

You can use the bastion host for:

* SSH to worker nodes
* SSH to operator host for managing Kubernetes cluster.

From ssh to the bastion, copy the terraform output command at the end of its run:

`ssh_to_bastion = ssh -i /path/to/private_key opc@bastion_ip`

From ssh to the worker nodes, you can follow this step:

`ssh -i /path/to/private_key -J <username>@bastion_ip opc@worker_node_private_ip`

## Takeaway

In conclusion, provisioning a Kubernetes cluster or Oracle Kubernetes Engine (OKE) using Terraform or Oracle Cloud Infrastructure (OCI) can greatly simplify and automate the process of setting up and managing a cloud-based system. 

This article might help you boost some invaluable knowledge that will streamline your infrastructure deployment and management process.

By following the steps outlined in this blog, you should now have a solid foundation for provisioning your own Kubernetes and Container Engine infrastructure on OCI with Terraform. You're one step closer to taking full advantage of the benefits that this powerful tool offers, such as increased efficiency, reduced downtime, and better collaboration.

So what's next? It's time to put all that knowledge into practice. Use what you've learned to provision your own Kubernetes and Container Engine infrastructure on OCI with Terraform. And remember, if you ever need a refresher or run into any roadblocks, this article is here for you