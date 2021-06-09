# terraform-azurerm-panos-bootstrap

## This module is deprecated
Please use the bootstrap module located at https://github.com/PaloAltoNetworks/terraform-azurerm-vmseries-modules instead.

## Overview

The terraform-azurerm-panos-bootstrap module is used to create an Azure file share that to be used for bootstrapping Palo Alto Networks VM-Series virtual firewall instances.  A bootstrap package must include an `init-cfg.txt` file that provides the basic configuration details to configure the VM-Series instance and register it with its Panorama management console.  This file will be generated by this module using the variables provided.  

The bootstrap package may optionally include a PAN-OS software image, application and threat signature updates, VM-Series plug-ins, and/or license files.

## Directory and file structure
The root directory of the Terraform plan calling this module should include a `files` directory containing a subdirectory structure similar to the one below.

```
files
├── config
├── content
├── license
├── plugins
└── software
```

## Example

```terraform
#
# main.tf
#

provider "azurerm" {
  subscription_id = var.azure_subscription_id
  tenant_id       = var.azure_tenant_id
  client_id       = var.azure_client_id
  client_secret   = var.azure_client_secret
}


module "panos-bootstrap" {
  source  = "PaloAltoNetworks/panos-bootstrap/azurerm"
  version = "1.0.4"

  azure_resource_group = var.azure_resource_group
  azure_location       = var.azure_location

  hostname         = "my-firewall"
  panorama-server  = "panorama1.example.org"
  panorama-server2 = "panorama2.example.org"
  tplname          = "My Firewall Template"
  dgname           = "My Firewalls"
  vm-auth-key      = "supersecretauthkey"
}
```

## Requirements

The [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) must be installed on the host executing the Terraform plan.

## Instructions

1. Define a `main.tf` file that calls the module and provides any required and optional variables.
2. Define a `variables.tf` file that declares the variables that will be utilized.
3. (OPTIONAL) Define an `output.tf` file to capture and display the module return values.
4. Create the directories `files/config`, `files/software`, `files/content`, `files/license`, and `files/plugins`.
5. (OPTIONAL) Add software images, content updates, plugins, and license files to their respective subdirectories.
6. (OPTIONAL) Define a `terraform.tfvars` file containing the required variables and associated values.
7. Initialize the providers and modules with the `terraform init` command.
8. Validate the plan using the `terraform plan` command.
9. Apply the plan using the `terraform apply` command. 

## Utilization

The module output will provide values for the `storage_account`, `access_key`, and `share_name`.  These values can then be used in a `azurerm_virtual_machine` resource to instantiate a VM-Series instance.  They are used in the `os_profile{custom_data}` parameter.

```terraform
resource "azurerm_virtual_machine" "vmseries" {
  count                        = var.vm_count
  name                         = "${var.name}${count.index + 1}"
  location                     = var.location
  resource_group_name          = var.resource_group_name
  vm_size                      = var.size
  primary_network_interface_id = element(azurerm_network_interface.nic0.*.id, count.index)

  network_interface_ids = [
    element(azurerm_network_interface.nic0.*.id, count.index),
    element(azurerm_network_interface.nic1.*.id, count.index),
    element(azurerm_network_interface.nic2.*.id, count.index),
  ]

  availability_set_id = azurerm_availability_set.default.id

  os_profile_linux_config {
    disable_password_authentication = false
  }

  plan {
    name      = var.license
    publisher = "paloaltonetworks"
    product   = "vmseries1"
  }

  storage_image_reference {
    publisher = "paloaltonetworks"
    offer     = "vmseries1"
    sku       = var.license
    version   = var.panos
  }

  storage_os_disk {
    name              = "${var.name}${count.index + 1}-osdisk"
    caching           = "ReadWrite"
    create_option     = "FromImage"
    managed_disk_type = "Standard_LRS"
  }

  os_profile {
    computer_name  = "${var.name}${count.index + 1}"
    admin_username = var.username
    admin_password = var.password
    custom_data = base64encode(
      join(
        ",",
        [
          "storage-account=${var.storage_account}",
          "access-key=${var.access_key}",
          "file-share=${var.share_name}",
          "share-directory=${var.share_directory}"
        ],
      )
    )
  }
}
```


## References
* [VM-Series Firewall Bootstrap Workflow](https://docs.paloaltonetworks.com/vm-series/10-0/vm-series-deployment/bootstrap-the-vm-series-firewall/vm-series-firewall-bootstrap-workflow.html#id59fe5979-c29d-42aa-8e72-14a2c12855f6)
* [Bootstrap the VM-Series Firewall on Azure](https://docs.paloaltonetworks.com/vm-series/10-0/vm-series-deployment/bootstrap-the-vm-series-firewall/bootstrap-the-vm-series-firewall-in-azure.html#idd51f75b8-e579-44d6-a809-2fafcfe4b3b6)
* [Prepare the Bootstrap Package](https://docs.paloaltonetworks.com/vm-series/10-0/vm-series-deployment/bootstrap-the-vm-series-firewall/prepare-the-bootstrap-package.html#id5575318c-1de8-497a-960a-1d7417feefa6)
