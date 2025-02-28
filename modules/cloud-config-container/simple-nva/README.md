# Google Simple NVA Module

This module allows for the creation of a NVA (Network Virtual Appliance) to be used for experiments and as a stub for future appliances deployment.

This NVA can be used to interconnect up to 8 VPCs.

## Examples

### Simple example

```hcl
locals {
  network_interfaces = [
    {
      addresses  = null
      name       = "dev"
      nat        = false
      network    = "dev_vpc_self_link"
      routes     = ["10.128.0.0/9"]
      subnetwork = "dev_vpc_nva_subnet_self_link"
    },
    {
      addresses  = null
      name       = "prod"
      nat        = false
      network    = "prod_vpc_self_link"
      routes     = ["10.0.0.0/9"]
      subnetwork = "prod_vpc_nva_subnet_self_link"
    }
  ]
}

module "cos-nva" {
  source               = "./fabric/modules/cloud-config-container/simple-nva"
  enable_health_checks = true
  network_interfaces   = local.network_interfaces
  # files = {
  #   "/var/lib/cloud/scripts/per-boot/firewall-rules.sh" = {
  #     content     = file("./your_path/to/firewall-rules.sh")
  #     owner       = "root"
  #     permissions = 0700
  #   }
  # }
}

module "vm" {
  source             = "./fabric/modules/compute-vm"
  project_id         = "my-project"
  zone               = "europe-west8-b"
  name               = "cos-nva"
  network_interfaces = local.network_interfaces
  metadata = {
    user-data              = module.cos-nva.cloud_config
    google-logging-enabled = true
  }
  boot_disk = {
    initialize_params = {
      image = "projects/cos-cloud/global/images/family/cos-stable"
      type  = "pd-ssd"
      size  = 10
    }
  }
  tags = ["nva", "ssh"]
}
# tftest modules=1 resources=1
```

### Example with advanced routing capabilities

Find below a sample terraform example for bootstrapping a simple NVA powered by [COS](https://cloud.google.com/container-optimized-os/docs) and running [FRRouting](https://frrouting.org/) container.
Please find below a sample frr.conf file based on the documentation available [here](https://docs.frrouting.org/en/latest/basic.html) for hosting a BGP service with ASN 65001 on FRR container establishing a BGP session with a remote neighbor with IP address 10.128.0.2 and ASN 65002.

```
# tftest-file id=frr_conf path=./frr.conf
# Example frr.conmf file

log syslog informational
no ipv6 forwarding
router bgp 65001
 neighbor 10.128.0.2 remote-as 65002
line vty
```

Following code assumes a file in the same folder named frr.conf exists. 

```hcl
locals {
  network_interfaces = [
    {
      addresses           = null
      name                = "dev"
      nat                 = false
      network             = "dev_vpc_self_link"
      routes              = ["10.128.0.0/9"]
      subnetwork          = "dev_vpc_nva_subnet_self_link"
      enable_masquerading = true
      non_masq_cidrs      = ["10.0.0.0/8"]
    },
    {
      addresses  = null
      name       = "prod"
      nat        = false
      network    = "prod_vpc_self_link"
      routes     = ["10.0.0.0/9"]
      subnetwork = "prod_vpc_nva_subnet_self_link"
    }
  ]
}

module "cos-nva" {
  source               = "./fabric/modules/cloud-config-container/simple-nva"
  enable_health_checks = true
  network_interfaces   = local.network_interfaces
  frr_config           = { config_file = "./frr.conf", daemons_enabled = ["bgpd"] }
  optional_run_cmds    = ["ls -l"]
}

module "vm" {
  source             = "./fabric/modules/compute-vm"
  project_id         = "my-project"
  zone               = "europe-west8-b"
  name               = "cos-nva"
  network_interfaces = local.network_interfaces
  metadata = {
    user-data              = module.cos-nva.cloud_config
    google-logging-enabled = true
  }
  boot_disk = {
    image = "projects/cos-cloud/global/images/family/cos-stable"
    type  = "pd-ssd"
    size  = 10
  }
  tags = ["nva", "ssh"]
}
# tftest modules=1 resources=1 files=frr_conf
```
<!-- BEGIN TFDOC -->

## Variables

| name | description | type | required | default |
|---|---|:---:|:---:|:---:|
| [network_interfaces](variables.tf#L75) | Network interfaces configuration. | <code title="list&#40;object&#40;&#123;&#10;  routes              &#61; optional&#40;list&#40;string&#41;&#41;&#10;  enable_masquerading &#61; optional&#40;bool, false&#41;&#10;  non_masq_cidrs      &#61; optional&#40;list&#40;string&#41;&#41;&#10;&#125;&#41;&#41;">list&#40;object&#40;&#123;&#8230;&#125;&#41;&#41;</code> | ✓ |  |
| [cloud_config](variables.tf#L17) | Cloud config template path. If null default will be used. | <code>string</code> |  | <code>null</code> |
| [enable_health_checks](variables.tf#L23) | Configures routing to enable responses to health check probes. | <code>bool</code> |  | <code>false</code> |
| [files](variables.tf#L29) | Map of extra files to create on the instance, path as key. Owner and permissions will use defaults if null. | <code title="map&#40;object&#40;&#123;&#10;  content     &#61; string&#10;  owner       &#61; string&#10;  permissions &#61; string&#10;&#125;&#41;&#41;">map&#40;object&#40;&#123;&#8230;&#125;&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [frr_config](variables.tf#L39) | FRR configuration for container running on the NVA. | <code title="object&#40;&#123;&#10;  daemons_enabled &#61; optional&#40;list&#40;string&#41;&#41;&#10;  config_file     &#61; string&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> |  | <code>null</code> |
| [optional_run_cmds](variables.tf#L84) | Optional Cloud Init run commands to execute. | <code>list&#40;string&#41;</code> |  | <code>&#91;&#93;</code> |

## Outputs

| name | description | sensitive |
|---|---|:---:|
| [cloud_config](outputs.tf#L17) | Rendered cloud-config file to be passed as user-data instance metadata. |  |

<!-- END TFDOC -->
