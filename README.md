# Sidecar - Terraform Azure VM

A quick start to deploy a sidecar to Azure VM using Terraform!

This quick start guide uses our [Terraform module for Azure VM](https://registry.terraform.io/modules/cyralinc/sidecar-vm/azure/latest).
The source code for this module is available in the public GitHub repository
[terraform-azure-sidecar-vm](https://github.com/cyralinc/terraform-azure-sidecar-vm).

---

## Deployment

### Architecture

![Deployment architecture](https://raw.githubusercontent.com/cyralinc/terraform-azure-sidecar-vm/main/images/azure_architecture.png)

### Requirements

* Make sure you have access to your Azure environment with credentials that have sufficient permissions to deploy the sidecar. The minimum permissions must allow for the creation of the elements shown previously.

* Install [Terraform](https://www.terraform.io).

See the Terraform module's requirements in file [versions.tf](https://github.com/cyralinc/terraform-azure-sidecar-vm/blob/main/versions.tf).

### Examples

#### Quick Start

* Save the code below in a `.tf` file (ex `sidecar.tf`) in a new folder.
    * Fill the parameters `sidecar_id`, `control_plane`, `client_id` and 
    `client_secret` with the information from the `Cyral Templates` option
    in the `Deployment` tab of your sidecar details.
    * Fill the parameter `subnets` with existing subnets that allows 
    network connectivity to the Cyral control plane (outbound HTTPS and gRPC traffic using port `443`)
    and to the database you plan to protect with this sidecar.

* Open a command line terminal in the new folder.
* Configure the Azure credentials directly in the `.tf` file or provide them through environment variables.
* Run `terraform init` followed by `terraform apply`.

```hcl
provider "azurerm" {
  # This feature is to immediately destroy secrets when `terraform destroy`
  # is executed. We advise you to remove it for production sidecars.
  features {
    key_vault {
      purge_soft_delete_on_destroy = true
    }
  }
}

module "cyral_sidecar" {
  source = "cyralinc/sidecar-vm/azure"
  version = "~> 1.0" # terraform module version
  
  sidecar_id    = ""
  control_plane = ""
  client_id     = ""
  client_secret = ""

  # Leave empty if you prefer to perform upgrades directly
  # from the control plane.
  sidecar_version = ""

  # Considering MongoDB ports are from the range 27017 to 27019
  sidecar_ports = [443, 3306, 5432, 27017, 27018, 27019]

  # Subnets to use to deploy VMs
  subnets = [""]
  
  # Location that will be used to deploy the resource group
  # containing the sidecar resources
  resource_group_location = ""

  # Path to the public key that will be used to SSH into the VMs
  admin_ssh_key = file("/Users/me/.ssh/id_rsa.pub")

  #############################################################
  #                       DANGER ZONE
  # The following parameters will expose your sidecar to the
  # internet. This is a quick set up to test with databases
  # containing dummy data. Never use this configuration if you
  # are binding your sidecar to a database that contains any
  # production/real data unless you understand all the
  # implications and risks involved.

  public_load_balancer = true

  # Source address prefixes for SSH into the VM instances
  ssh_source_address_prefixes = ["0.0.0.0/0"]
  # Source address prefixes to access ports defined in `sidecar_ports`
  db_source_address_prefixes = ["0.0.0.0/0"]
  # Source address prefixes to monitor the VM instances (port 9000)
  monitoring_source_address_prefixes = ["0.0.0.0/0"]
  #############################################################
}
```

The quick start example above will create the simplest configuration possible on your Aure subscription
and deploy a single sidecar instance behind a load balancer.

Deploying a test sidecar is the easiest way to have a sidecar up and running to
understand the basic concepts of our product.

In case the databases you are protecting with the Cyral sidecar also live on Azure, make sure to
configure the database security group in such way that the inbound rules allows traffic from
the sidecar on the database ports. If the databases do not live on Azure,
analyze what is the proper networking configuration to allow connectivity from the VM
instances to the protected databases.

#### Production Starting Point

* Save the code below in a `.tf` file (ex `sidecar.tf`) in a new folder.
    * Fill the parameters `sidecar_id`, `control_plane`, `client_id` and 
    `client_secret` with the information from the `Cyral Templates` option
    in the `Deployment` tab of your sidecar details.
    * Fill the parameters `vpc_id` and `subnets` with an existing VPC and subnets that allows 
    network connectivity to the Cyral control plane (outbound HTTPS and gRPC traffic using port `443`)
    and to the database you plan to protect with this sidecar.
    * Fill the remaining parameters as intructed in the comments.
* Open a command line terminal in this new folder.
* Configure the Azure credentials directly in the `.tf` file or provide them through environment variables.
* Run `terraform init` followed by `terraform apply`.

```hcl
provider "azurerm" {
}

module "cyral_sidecar" {
  source = "cyralinc/sidecar-vm/azure"
  version = "~> 1.0" # terraform module version
  
  sidecar_id    = ""
  control_plane = ""
  client_id     = ""
  client_secret = ""

  # Assign the version that will be used by the sidecar instances.
  # Remove the parameter or leave it empty should you prefer to
  # perform upgrades directly from the control plane using the
  # 1-click upgrade.
  sidecar_version = ""

  # Considering MongoDB ports are from the range 27017 to 27019
  sidecar_ports = [443, 3306, 5432, 27017, 27018, 27019]

  # For production use cases, provide multiple subnets in
  # different availability zones.
  subnets = [
    "<subnet-az1-id>",
    "<subnet-az2-id>",
    "<subnet-azN-id>"
  ]

  # Use 2 instances `m5.large`
  auto_scale_min = 2
  auto_scale_default = 2
  auto_scale_max = 4

  instance_type = "m5.large"
  asg_min = 1
  asg_desired = 2
  asg_max = 4
  
  # Location that will be used to deploy the resource group
  # containing the sidecar resources
  resource_group_location = ""

  # Path to the public key that will be used to SSH into the VMs
  admin_ssh_key = file("/Users/me/.ssh/id_rsa.pub")

  # Restrict the inbound to SSH into the VM instances
  ssh_source_address_prefixes = ["0.0.0.0/0"]
  # Restrict the inbound to database connection to the VM instances
  # using the ports declared in `sidecar_ports`
  db_source_address_prefixes = ["0.0.0.0/0"]
  # Restrict the inbound to monitor the VM instances (port 9000)
  monitoring_source_address_prefixes = ["0.0.0.0/0"]
}
```

The example above will create a production-grade configuration and assumes you understand
the basic concepts of a Cyral sidecar.

For a production configuration, we recommend that you provide multiple subnets in different
availability zones and properly assess the dimensions and number of VM instances required
for your production workload.

In order to properly secure your sidecar, define appropriate inbound CIDRs using variables
`ssh_source_address_prefixe`, `db_source_address_prefixer` and `monitoring_source_address_prefixe`. See the
input variables documentation in the [module's input section](https://registry.terraform.io/modules/cyralinc/sidecar-vm/azure/latest?tab=inputs)
for more information.

In case the databases you are protecting with the Cyral sidecar also live on Azure, make sure to
configure the database security group in such way that the inbound rules allows traffic from
the sidecar on the database ports. If the databases do not live on Azure,
analyze what is the proper networking configuration to allow connectivity from the VM
instances to the protected databases.

### Parameters

See the full list of parameters in the [module's docs](https://registry.terraform.io/modules/cyralinc/sidecar-vm/azure/latest?tab=inputs).

---

## Upgrade

This quick start supports [1-click upgrade](https://cyral.com/docs/sidecars/manage/upgrade#1-click-upgrade).

Instructions for sidecar upgrade are available
in the [module's docs](https://github.com/cyralinc/terraform-azure-sidecar-vm#upgrade).

---

## Advanced

Instructions for advanced configurations are available
in the [module's docs](https://github.com/cyralinc/terraform-azure-sidecar-vm#advanced).